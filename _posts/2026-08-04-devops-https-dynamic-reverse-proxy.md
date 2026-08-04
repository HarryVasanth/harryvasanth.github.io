---
layout: post
title: "DevOps - A Self-Serving HTTPS Edge: Dynamic Reverse Proxying with Caddy and Docker"
date: 2026-08-04 19:37:50 +01
categories: devops networking
tags: caddy docker tls https reverse-proxy devops networking
---

## The TLS Problem at the Edge

Every learning centre runs the platform on its own server. Each server hosts a handful of services: the backend API, the frontend, a Git service, a code runner, and a few supporting tools. Every one of those services needs its own subdomain, and every subdomain needs a valid TLS certificate. `api.harryvasanth.com`, `git.harryvasanth.com`, `web.harryvasanth.com`, and so on.

If you manage a handful of servers, you can handle this by hand. You write an nginx or Caddy config, you run certbot, you reload, you hope nothing changed. It works, and it is miserable. Multiply that by a fleet of learning centre servers, each running a slightly different set of containers, and the manual approach stops working entirely. Every new service means editing a config file on every server that needs it. Every certificate renewal is a potential failure point. It does not scale, and it consumes time that should be spent on the platform itself.

The problem is not TLS, and it is not reverse proxying. Both are solved problems. The problem is *discovery*. The proxy does not know which services exist, which domains they serve, or which ports they listen on. In a normal setup you tell the proxy everything by hand. We wanted the opposite: we wanted the containers to tell the proxy about themselves, and for the proxy to configure itself automatically.

## The Idea: A Proxy That Discovers Itself

The solution is a small Go service, simply called `https`, that sits at the edge of every learning centre server. It is built on two solid foundations: Caddy for the actual HTTP and TLS handling, and the Docker socket for discovery.

The contract between a service and the edge is one Docker label. Any container that wants to be reachable from the internet declares it like this:

```dockerfile
LABEL org.harryvasanth.https="git.harryvasanth.com:8080,api.harryvasanth.com:8081"
```

The label is a comma-separated list of `domain:port` pairs. When the container starts, the `https` service sees the event, reads the label, and adds those domains to its proxy map. When the container dies, the domains are removed. That is the whole contract. No config files, no restarts, no manual steps. The container owns its routing, and the edge follows.

This is the same principle that makes load balancers and service meshes feel magical. The routing table is derived from the running system rather than maintained alongside it. The difference here is that the derivation is trivial: one label, one watcher, one config reload.

## How the Service Works

The service is a single Go binary that does three jobs. It runs Caddy as a child process, it watches Docker for changes, and it keeps Caddy's configuration in sync with what it sees.

### The Docker Watcher

The heart of the service is the Docker events stream. It subscribes to container lifecycle events, filtered to exactly two things: containers carrying the `org.harryvasanth.https` label, and the `start` and `die` events. A `start` event adds the container's domains to the map. A `die` event removes them.

There is an important detail at boot. When the service starts, it must not wait for events to learn about the world. It first lists the containers that already carry the label, parses each one, and populates the map. Only then does it subscribe to the event stream. This cold-start sync is what makes the service safe to restart at any time. The config rebuilds itself from scratch, no state required.

### The Two Templates

Caddy accepts its configuration as JSON through its admin API. We generate that JSON from a Go template. There are two templates, one for production and one for development, because the two environments do TLS very differently.

In production, Caddy handles certificates itself through Let's Encrypt. The template is simple: a listener on port 443, a shared block that applies compression and a hardened HSTS header, one route per domain that reverse-proxies to the right container, and a catch-all that answers unknown hosts with a 404. The per-domain routes are the interesting part. Each one matches a single hostname and terminates there, so Caddy never needs to look beyond the first match.

In development, the domain resolves to `127.0.0.1`, which Let's Encrypt will never sign. So development uses certificates generated locally with `mkcert`, and the template loads them explicitly from the certificate store inside the container.

### The Reload

When the proxy map changes, the service renders the template and pushes the result to Caddy's admin API with a single POST to `/load`. Caddy swaps the running configuration with no downtime. The whole cycle, from container start to a live route, takes well under a second.

## Production Versus Development

The service has to behave differently in the two environments, and it decides which one it is in without being told. On startup it resolves the first domain it is asked to serve. If that domain points at a loopback address, it is a development box and the development template wins. Otherwise it is production.

That auto-detection is neat, but it is not the whole story. The deployment script does the real environment work. When the target domain resolves to localhost, the script checks for `mkcert`, checks whether the local root certificate authority is installed, installs it if it is missing, and generates a certificate for the domain. On a machine without `mkcert`, the script now stops and tells you exactly how to install it on your operating system, instead of failing somewhere deep in a Docker build. It is a small quality-of-life change that saves a surprising amount of confusion.

## The Problems We Hit

The original version of this service was written in 2021. It worked, and it stayed mostly untouched for years, which is a compliment in itself. But when we finally went through it properly, we found a collection of issues that only surface under real operation. This is what we fixed.

### The Configuration Was a Wall of Nested Blocks

The original Caddy templates were built around `subroute` handlers. Every domain got its own nested block containing a headers handler, a compression handler, and a reverse proxy. The development template was over 80 lines of deeply nested JSON, with each domain repeating the same three handlers and the same per-domain certificate selection rules.

This was not just ugly. It was a maintenance hazard, because a change to the shared behaviour meant touching every block. We flattened the whole thing. The shared handlers now appear once, at the top of the server block, and each domain gets a single, minimal reverse-proxy route. The per-domain certificate tagging logic turned out to be unnecessary once the certificates were loaded by name, so it disappeared. The development template dropped from over 80 lines to about 30, and the production template lost its repetition as well.

### Reload Storms

The original code reloaded Caddy's configuration on every single Docker event. That is fine when one container starts. It is a disaster when a whole platform deploys and thirty containers start within the same minute. Every event triggered a full config render and a push to Caddy, and the pushes piled up on each other.

The fix is a debounce. We wait half a second after the most recent event before touching Caddy. If another event arrives in that window, the timer resets and the wait starts again. The result is that a burst of thirty events produces one config reload, not thirty. The service stays calm while the platform churns.

### A Race Waiting to Happen

The proxy map is shared between two worlds. The event loop writes to it, and the reload function reads from it, potentially from a timer goroutine. In the original code that map had no protection at all. We added a read-write mutex around it. The event loop takes the write lock, the reload takes the read lock, and the two stop tripping over each other. It is a boring change, and it is exactly the kind of change you should make before the race bites you in production rather than after.

### The Boot Order

The original startup sequence had a subtle bug lurking in it. If the initial sync found no containers, the service would bail out early and never initialise its template, which meant it would never proxy anything even after containers appeared. The logic is now explicit: sync the existing world first, initialise the template, and only then start waiting for events. The service also stops treating an empty map as a reason to give up.

### A Certificate Path Bug

In development, the certificate loading referenced the cert files by a relative path, which is fragile inside a container with a different working directory. The fix was to reference them by their absolute path in the container's certificate store. It is the kind of bug that works on your machine and mysteriously breaks in the container, and the fix is exactly one line.

### Hardening the Headers

While we were in the templates, we also made the edge a little more opinionated. The production HSTS header now includes `includeSubDomains; preload`, which tells browsers to treat every subdomain as HTTPS-only and to submit the domain to the preload list. Unknown hosts now get a clean 404 instead of an ambiguous response. Small changes, but they harden the edge for free.

### The Build and the Script

The Dockerfile got the usual long-overdue care: a proper multi-stage build, a `.dockerignore`, dependency layer caching, and a stripped binary. The deployment script was standardised with strict shell options, consistent variable names, proper cleanup of stale containers and volumes, and the `mkcert` checks described earlier. The Go toolchain and the Docker client dependency were bumped along the way, and the codebase moved onto current Go releases.

## The Honest Afterword: Cloudflare

There is a piece of this story that deserves to be told straight. The custom HTTPS edge worked, and it still works. But as the platform spread to more learning centres, a managed edge became the better answer for most of them. Cloudflare's proxy handles certificate issuance and renewal automatically, even in regions with geo-blocking, and it brings a WAF, DDoS protection, and a CDN along for free.

So the recommended configuration shifted. For most learning centres, the custom `https` service is now disabled and Cloudflare terminates TLS instead. The service remains genuinely useful in one scenario: the hybrid setup. A learning centre hosts the platform on its own server with intermittent internet connectivity. Internal requests are routed locally through the `https` service, so the platform works even when the internet drops. External requests go through Cloudflare. The local edge and the managed edge split the traffic, and the platform keeps working in both worlds.

This is a good reminder that a custom solution is not always the forever answer. The `https` service was the right tool when it was written, and it is still the right tool for the hybrid case. But part of the job is recognising when a managed service does the job better, and letting it.

## Summary

The `https` service solves a boring problem in an elegant way. Containers declare their domains in a single label, a small Go watcher listens to Docker, and Caddy does the rest. No config files, no manual certificate choreography, no per-server edits. When we went through the code properly, we simplified the templates, added debouncing and thread safety, fixed the boot order and a certificate path bug, and hardened the edge for production. We also learned when to step aside and let a managed edge take over.

For a platform that is deployed onto hundreds of servers by people who should not need to think about reverse proxies, that self-service property is the whole point. The edge configures itself, and the people running a learning centre never have to touch it.