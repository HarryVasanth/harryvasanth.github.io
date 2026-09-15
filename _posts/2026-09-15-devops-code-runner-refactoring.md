---
layout: post
title: "DevOps - Rebuilding the Code Runner: Concurrency, Security, and Zip Bombs"
date: 2026-09-15 01:55:38 +01
categories: devops go
tags: go devops docker concurrency security refactoring
---


## The Invisible Service

Every educational platform that runs learner code needs a grading engine. The engine is the part of the platform that takes a submission, executes it, and decides whether it passes or fails. In our platform this job belongs to a small Go service called the runner.

The runner sits quietly behind the API. Learners never talk to it directly and most of them do not know it exists. Yet it is the most load-bearing piece of infrastructure we run. Every exercise, every learner, and every submission flows through it. When the runner has a bad day, the whole platform has a bad day, and so does every learning centre and every learner who happens to be using it at that moment.

This post is the story of how we rebuilt that service. It is not a story about rewriting something for fun. It is a story about a production service that worked, that grew organically over several years, and that slowly accumulated the kind of problems that only show up under real load. We found those problems one at a time, fixed most of them with small surgical changes, and learned a hard lesson when one large rewrite backfired.

Before we get to the fixes, it helps to understand exactly what the runner does and how a submission becomes a result.

## What the Runner Does

The runner is a stateless HTTP service written in Go. It has no database, no persistent storage, and no state to speak of. That simplicity is deliberate. A stateless service can be scaled horizontally, load-balanced, and replaced at will, which is exactly what we wanted for a service that occasionally needs to handle a wave of submissions at the start of an exam.

The service listens on a single endpoint. The URL path carries the name of the Docker image to use, and the request body carries the learner's code as a ZIP archive. Query parameters provide the context: which file to test, which exercise to run, which domain the request belongs to, and which user submitted it.

When a request arrives, the runner does the following:

1. Read the request body and parse it as a ZIP archive.
2. Convert the ZIP into a TAR archive, because that is the format Docker expects.
3. Make sure the test image is present and up to date.
4. Create a Docker volume to hold the learner's files.
5. Copy the TAR contents into that volume using a throwaway helper container.
6. Create the real test container, mounting the volume read-only.
7. Start the test container and wait for it to finish.
8. Collect the logs, demux stdout and stderr, and trim them to a sane size.
9. Return a JSON object with the test output and a pass/fail flag.

The test environment is deliberately hostile to the code it runs. The learner's code executes inside a container with the network disabled, a read-only root filesystem, a writable `/jail` directory on tmpfs, and hard limits on memory, CPU, and process count. This is untrusted code, so the isolation is not optional. It is the whole point of the service.

## The Monolith That Grew

The first version of this pipeline was a single function of roughly 400 lines. It did every one of the nine steps above inline, in one long sequence, with almost no comments and a handful of magic numbers. It was the kind of code that makes perfect sense while you are writing it and becomes progressively harder to read with every passing week.

The function worked. Requests came in, tests ran, results went back. But a 400-line function is a magnet for subtle problems, and this one had accumulated its fair share over the years. The problems were not visible during a happy-path demo. They only revealed themselves under load, under attack, or under the specific kind of chaos that many learners submitting exercises at the same time can generate.

What follows is the list of problems we found, roughly in the order we found them, together with the fix for each and the reasoning behind it.

## Problem 1: The Image Pull Stampede

Every test run needs the correct Docker image for that exercise. The runner refreshes each image about once a minute to make sure it is running a recent version of the tests rather than a stale one.

The original code did this refresh with a global mutex held for the entire duration of the pull. That single line of design caused two separate problems.

The first problem was serialisation. A Docker image pull can take a long time, sometimes several minutes on a slow connection or a cold cache. While one request held the mutex, every other request in the service was blocked, waiting for a lock that had nothing to do with them. One slow pull meant the entire service stopped. That is a textbook example of a shared lock protecting far more than it should.

The second problem was the thundering herd. When several learners submitted at the same moment, all of them asked for the same image, and the code was not smart enough to share the work. Each request checked the timestamp, decided the image was too old, and started its own pull. Ten simultaneous requests meant ten simultaneous pulls of the same image, all racing each other and hammering the registry.

The fix came from two standard Go tools working together.

The first is `singleflight`, from the `golang.org/x/sync` package. The name describes the behaviour exactly: when several goroutines ask for the same thing at once, only one of them does the work and the rest wait for its result. It turns a stampede into a single shared request. One image, one pull, no matter how many learners want it.

The second fix was about the lock itself. We changed the logic to check the image age under a short lock, release the lock immediately, and only then start the pull. The lock now protects a map lookup, which takes nanoseconds, instead of a network operation, which takes seconds. This is the check-then-act pattern, and it matters here because the pull can safely happen outside the lock.

We also added a fallback for the moments when a pull fails. If the registry is unreachable, or the pull times out, but the service already has an older copy of the image on disk, the runner uses that copy instead of failing the request. A registry outage no longer takes down every exercise in the platform; the worst case is that learners run against a slightly older version of the tests for a minute or two.

## Problem 2: Unbounded Container Creation

The service had no limit on how many containers it would create at once. That sounds benign until you think about what a real submission wave looks like. An exam starts, a hundred learners click submit within the same minute, and suddenly the runner tries to run a hundred Docker containers at the same time.

Each of those containers reserves memory, CPU, and PIDs on the host. Docker is efficient, but it is not free. Under a big enough wave, the host could exhaust its resources, slow to a crawl, or start killing things. The symptom we saw was not a crash; it was a gradual, painful degradation of latency across the whole service.

The fix is a classic Go pattern: a buffered channel used as a semaphore. The channel has a fixed capacity, and before the runner starts real work on a request, it tries to put a token into the channel. If the channel is full, the request waits. When the request finishes, it releases the token and lets the next one in.

It is a simple pattern, and it gives us a hard ceiling on concurrency. The ceiling is configurable through the `MAX_PARALLEL` environment variable, which matters because the right number depends on the host. We started optimistically at 100, watched the host begin to struggle, and walked it back to 2. Two running containers at a time is a conservative number, but the runner is fast, and the queue drains quickly. It is much better to have a small, stable queue than to have the host fall over.

## Problem 3: The Zip Bomb

A ZIP archive is a compressed format, which means the size you see on the wire is not the size you get on disk. A tiny 1MB archive can claim to contain several gigabytes of uncompressed data. An attacker, or a learner with a broken tool, can send a small file that expands into something enormous.

The original code read the entire request body and unpacked it without asking a single question about what it was about to receive. This is a textbook denial-of-service vector. A single malicious request could exhaust memory, fill the disk, or both, and there was nothing stopping it.

The fix was a pair of guards placed before any extraction happens:

- A maximum number of files in the archive.
- A maximum total uncompressed size.

The first guard stops an archive that contains a ridiculous number of tiny files, which is its own kind of resource exhaustion. The second guard stops an archive that claims an absurd amount of decompressed data. Both checks read only the ZIP central directory, which is cheap, so they run before we spend any real effort.

The interesting part was the tuning. We started with a strict limit of 32 files, which felt generous for a typical exercise submission. It was not. Real learner repositories routinely contain more than 32 files, and once the limit went live, legitimate submissions started failing. So we raised it to 1024 files and kept the 256MB uncompressed-size cap. The combination has been quiet ever since, which is a good reminder that validation limits are not abstract numbers. They need to be calibrated against real usage.

## Problem 4: A Panic Killed the Service

Go is a language that turns errors into values, but it still has panics. A panic is an unrecoverable error: an out-of-bounds access, a nil dereference, or a call that explicitly panics. In a normal Go program, an unhandled panic crashes the process.

That is exactly what could happen in the runner. The code had a few places where it called a helper that panicked on unexpected errors, and it had a spot where a Docker API response could trigger one. The HTTP handler had no recovery mechanism, so any panic crashed the whole process. One bad submission, one edge case, one unexpected response, and the service was gone until someone restarted it.

The fix is small and powerful: a `recover()` at the top of the HTTP handler. In Go, `defer` runs even when a function panics, so the handler can intercept the panic, log it, and return a structured error response instead of dying. The process survives, the connection gets a clean JSON 500 response, and the service keeps serving the next request.

This is one of those changes that looks trivial and is quietly critical. The runner went from a service where a single panic could take down every exam to a service where the worst a panic can do is fail one request.

## Problem 5: Security Was Not Default

The runner exists to execute untrusted code. That is the job. So it is fair to ask what happens if the code being executed is malicious, and the answer, in the original version, was not reassuring. The test containers ran with the default Docker security posture, which is designed for convenience, not for running hostile code.

We hardened the test container configuration across several dimensions:

- Drop all Linux capabilities, so the container cannot perform privileged operations.
- Set `no-new-privileges`, so even if a setuid binary exists inside the image, it cannot escalate.
- Disable the network entirely. The learner's code has no business talking to the internet.
- Mount the root filesystem read-only, so the code cannot modify the image it runs in.
- Put the writable `/jail` directory on a tmpfs with explicit size and inode limits.
- Cap the number of PIDs, the memory, and the CPU available to the container.

Together, these options give us a fairly tight box. The code can compute, but it cannot reach the network, cannot write to the filesystem it runs on, cannot spawn unbounded processes, and cannot escalate privileges. That is the correct posture for a service whose entire purpose is to run code it does not trust.

There is an honest caveat to this section. The hardening was not free. After we enabled the capability drops, some test images stopped working, because they relied on capabilities that we had removed. We had a choice: keep the hardening and fix every affected image, or relax the hardening and keep everything working. We chose pragmatism. The capability drop and the privilege flag were commented out with a note explaining why, and fixing the affected images became a tracked work item.

The lesson here is that security hardening is not a switch you flip once. It is an ongoing conversation between the platform and the images that run on it. The configuration we want is the strict one. The configuration we shipped, temporarily, is the compatible one, and we are working our way back.

## Problem 6: The Logs Told Us Nothing

Debugging a distributed system is only as good as its logs, and the runner's logs were close to useless for anything beyond "something happened". A typical line contained a timestamp, a duration, and a vague event name like `container start`. That is enough to know the service did things, but not enough to know which thing, for whom, or why.

When a learner reported a failed submission, we could not tell from the logs which domain it came from, which exercise it was, or which user submitted it. We were flying blind, correlating requests by hand using timestamps and guesswork.

We changed the log line to carry the full context. Every request now logs its unique ID, the domain, the exercise, the username, and the image name, followed by a numbered stage and two durations: the total time since the request started and the time spent on the last stage.

Before, a log line looked like this:

> `10.0.0.7#12 | 3.142s | container start`

After the change, the same event looks like this:

> `10.0.0.7#12 | prod | sortlist | alice | ghcr.io/harryvasanth/test-go | 3.142s (+0.211s) | [Step 8] test container started`

The difference is enormous. A single line now tells a complete story: who asked, for what, against which image, and how long every stage took. When something is slow, the step durations tell us exactly where the time went. When something fails, the metadata tells us which learner and exercise to look at.

We also numbered the pipeline stages from `[Step 1]` to `[Step 10]`, which makes a request read like a checklist. It is a small formatting choice, but it turns a wall of log text into something scannable.

## The Timeout Tuning

Learners are humans, and humans write infinite loops. It is not an attack; it is a mistake, and it happens constantly in an educational platform. The runner needs to survive it.

The original code had a hard timeout on the container wait, which is the right instinct. When the timeout expires, the runner sends `SIGKILL` to the container and returns a clear message to the learner. The message is worth calling out, because it is a good example of product thinking in infrastructure code:

> `💥 Timeout: Did you write an infinite loop? (<30s)`

That message does two jobs. It tells the learner their submission did not finish, and it gives them the most likely reason, in a friendly, slightly humorous way. The timeout itself had a restless life. We started at 15 seconds, raised it to 30, then to 45, and eventually settled back on 30. Each change was a response to a real exercise that needed a little more room to finish. The 30 seconds we run with today is a deliberate compromise between giving legitimate tests enough time and keeping the service responsive.

## The Structure Pass

Some of the best changes in this project were not new features. They were passes that made the code easier to reason about.

We replaced the request ID counter, which was a mutex-guarded map keyed by client IP, with a simple atomic counter. The map version had a subtle problem: it grew without bound, so we had added a hack to clear it when it exceeded 1000 entries, which threw away correlation history. The atomic counter is simpler, faster, and has no memory leak. It is also an honest improvement in privacy, since we no longer keep a map of which IP did what.

We removed a package-level context variable in favour of passing `context.Background()` explicitly. The shared variable looked convenient but made it impossible to know which context a call was using. Explicit is better.

And we added a comment to every stage explaining why it exists, not just what it does. The difference between the two is the difference between documentation and an essay. A comment that says "create a volume to store the student's code" tells a future engineer why this step is here and what would break if it disappeared.

This change barely moved the needle in terms of behaviour, yet it was one of the most valuable things we did. When we were finished, the file read like something a human had taken care over, rather than something that had simply grown. That alone was worth the effort.

## The Big Rewrite That Did Not Survive

After the incremental fixes, we got ambitious. The monolith was working, but it was still a monolith, and we wanted to give it the treatment it deserved. We planned a full rewrite of the pipeline with three main goals.

The first goal was structure. Split the single function into named helpers, one per stage, with clear inputs and outputs. The second goal was streaming. The original code converted the ZIP to a TAR by buffering the entire result in memory, which is wasteful for large submissions. We wanted to stream the conversion through an `io.Pipe`, so the data flows from the ZIP reader to the Docker API without ever landing in a big buffer. The third goal was context propagation. We wanted the request context to flow through the whole pipeline, so that a cancelled or timed-out request actually stops the work in progress, and we wanted graceful shutdown so the service could drain active requests when it restarted.

On paper, the rewrite was a thing of beauty. Every stage had a name and a purpose, the streaming conversion meant we no longer held whole archives in memory, and the constants finally gave every limit a home. The code was cleaner than anything we had shipped in that file before.

Production did not agree. A week later, we pulled the whole thing out and went back to the previous state. We chose not to write the reasoning down at the time. The lesson was still forming, and it deserved more room than a quick note could give it. It gets that room here instead.

It is worth being honest about what this episode taught us, because it is the most interesting lesson in this whole post. The rewrite was not bad code. It was risky in exactly the way that large changes to timing and streaming behaviour are always risky. A service under load behaves differently from a service in a staging environment, and a rewrite that changes when bytes are read, how they flow, and what happens on cancellation changes behaviour in ways that are hard to predict from reading the code.

From there, we rebuilt in small steps. The good ideas from the rewrite did not die with it. Pieces like the named constants and the output cap can be reintroduced one at a time, each with its own review and its own observation period.

## The Supporting Work

Not everything in this effort was about the hot path. Several changes improved the service without touching the request flow at all.

The Dockerfile got a long-overdue cleanup. We tightened the multi-stage build, copied only the dependency files before downloading modules so the layer cache actually works, and added `-s -w` flags to strip debug information from the binary. The result is a smaller image and faster rebuilds. We also trimmed the base image by installing the timezone data, copying the UTC zone, and removing it again, so the final image carries no dead weight.

The Go version and its dependencies were bumped on a schedule. Over the course of the effort we moved from Go 1.22 to 1.26, handled a deprecation in the Docker client API, and let dependabot keep the transitive dependencies current. None of these changes is exciting, and all of them are necessary. A service that runs untrusted code cannot afford to sit on outdated dependencies.

We also ran `shellcheck` against the deployment scripts and fixed everything it flagged. It is a small thing, but the scripts are the surface that operators touch, and clean, consistent scripts are part of operational hygiene.

## Lessons Learned

Looking back at the whole effort, these are the lessons that stuck.

1. **Fix one problem at a time.** Small, targeted changes are easy to review, easy to reason about, and easy to undo when they are wrong. Almost every good change in this project was small.
2. **Tune with data, not guesses.** The parallelism default went from 100 to 2 because we watched the host. The timeout settled at 30 seconds because exercises told us so. Both numbers came from observation, not intuition.
3. **Deduplicate shared work.** `singleflight` turned an image pull stampede into a single shared request. If several callers want the same expensive result, let one of them compute it.
4. **Validate input before doing work.** The zip bomb guards run before any extraction. Validation that happens after the damage is validation that does not matter.
5. **Never trust a panic.** Recover at the boundary, log what happened, and return a clean error. A process that survives one bad request is a service that can keep serving the next thousand.
6. **Log the metadata.** A request ID without the user, the exercise, and the image is not enough to debug anything.
7. **A big rewrite is a bet.** Incremental changes shipped and survived. The grand rewrite did not survive a week. Be suspicious of your own ambition.

## Summary

The runner is the silent gatekeeper of every submission in our curriculum. It started life as a monolithic function with hidden production problems, and it grew into a service that needed attention. The refactoring fixed concurrency, security, input validation, and observability in small, deliberate steps. The rewrite attempt showed that not every change survives contact with production, and that the humble incremental fix is often the right answer.

The service now handles concurrent learners safely, rejects malicious archives before they can do damage, keeps itself alive through panics, and tells us exactly what happened for every request. It is still a work in progress, because production services always are. But it is in a much better place than the day we started, and we learned more from the process than from any of the individual fixes.