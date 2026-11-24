---
layout: post
title: "DevOps - CDInfante Estatísticas: Turning AARAM's PDFs into a Season"
date: 2026-11-24 15:31:13 +00
categories: devops web
tags: react typescript vite pwa cloudflare python pdf data madeira
---

The first question after a race is easy. What was my time? The second is harder. Was it better than last year? The answer is usually buried in a PDF.

AARAM, the Madeira athletics association, publishes the official classifications as PDFs. Each PDF is a good record of one race. It is a poor record of a season. You cannot filter it, sort it, or compare it with the race before. You can only read it.

So I built [CDInfante Estatísticas](https://estatisticas.cdinfante.org) with our club, Clube Desportivo Infante Dom Henrique. It takes the public AARAM PDFs from the Madeira a Correr road circuit and the Trail Madeira circuit and turns them into a searchable archive. It covers every club in the regional circuit, not only ours.

Today it holds 272 races and 45,226 results from four seasons, 2023/24 to 2026/27. That is 4,824 athletes and 159 clubs. The data is rebuilt every night.

## What the Site Does

The overview shows the season at a glance: races, results, athletes, clubs, the fastest man and woman, and how participation moved across the season. The results page filters by season, race, club, gender, category, and race type. You can type a name. The filters live in the URL, so a filtered view is a link you can send.

Each athlete has a profile: every race, the pace progression, club history, transfers, best performances, and a trophy cabinet. Each club has a profile with its roster, its best athletes, its team podiums, and its race calendar. Each race has the full classification, the previous editions, the official PDF, and a finish line.

The finish line is my favourite part. It plots every timed finisher on one horizontal axis. A table tells you who won. The line tells you how the race felt: where the field broke, how deep the pack was, how lonely the front was. Filters keep the other runners visible as grey dots, so you can compare without losing the shape of the race.

## A PDF Is a Document, Not an API

The AARAM pages list the classifications race by race. The pipeline downloads the page, follows the PDF links, and caches the files. Then it parses them with pdfplumber.

That sounds simple until you look at the PDFs. They are not one format. The modern principal classification has one athlete per row and three trailing time columns. The youth classification has one time column and ranks inside age groups. The historical pages use a two-column layout with points and no time columns. Some years use a compact category. Some use the full club name.

The parser has a small taxonomy for these layouts. It reads the header, finds the rank markers, and matches the rows. The rule is simple: read the header, find the columns, and test against a real copy of the page. Never parse a PDF by position alone.

In November, AARAM replaced the São Martinho youth PDF with a version that includes `Ritmo/km`, the pace per kilometre. Every youth row stopped matching. The race vanished from the build, and the season count shrank. Validation failed on every run for a few days.

The fix was small: accept an optional trailing pace column. The first guard was blunt: a current-season PDF that yields no results failed the build. That was honest, but it stopped the other races. A missing race is a bug. A wrong race is worse.

## A Broken PDF Must Not Erase History

The second fix came later the same day. If a source PDF fails to download or parse, the build now reuses the last committed race. It does not drop the race, and it does not block the other races. The workflow prints a warning, and the app keeps serving the last good data.

This is the rule I keep coming back to: fail closed on new data, but never delete published history because a source had a bad day. The current file is the fallback. The source is not the only copy.

The pipeline also validates the PDF date against the archive date. If the two differ by more than two days, the PDF is dropped with a warning. Old archive pages sometimes link the wrong file. A PDF can be official and still be filed under the wrong race.

## Identity Is Harder Than Parsing

A modern PDF lists the same athlete twice: once in the gender ranking and once in the category ranking. The pipeline merges the two rows. It matches on name, dorsal, and gender, with birth year as a fallback. It keeps the better category rank.

Inside one classification, the dorsal is a useful tie-breaker. Across seasons, it is not an identity. Dorsals are reused, and a dorsal can change between races. The identity key is the normalised name, birth year, and gender. The display name is the most common spelling in the data.

This is not perfect. Two athletes with the same name and birth year can merge. One athlete with a changed name can split into two profiles. Fuzzy matching would fix some of those cases, and create worse ones. A split profile is annoying. A merged profile is a lie. I chose annoying.

Clubs have the same problem. A club is stored on the result, not on the athlete. A transfer is a separate record. It never rewrites the past. AARAM published 314 transfer rows for the last two seasons. Only 7 of them match an athlete profile exactly. The rest are shown as transfer records without a profile link. That is honest. A wrong link is worse than no link.

## The Build Has to Say No

The pipeline runs every night at 02:00 UTC. It installs Python 3.13, runs the tests, rebuilds road and trail, and validates every generated file. If something changed, it commits the JSON as `github-actions[bot]`.

The validation is not decorative. It checks the index totals against the season files. It checks that seasons run from October to September and that race dates fall inside them. It checks for duplicate race ids, duplicate athletes in one race, dangling athlete, club, and race references, relay bibs, invalid times and ranks, and pace plausibility. It also checks that a closed season does not shrink. A closed season is history. History does not get smaller because a page moved.

The current season is different. It can grow. It can also shrink if a source corrects a mistake. That path is manual: `allow_shrink` is an input on the workflow, and it is off by default. A human has to say the source is right.

The source can also be down. If the current road page is unavailable, the build reports a warning and keeps the last data. If the trail page is unavailable, it reuses the last generated current season. The app does not go blank because AARAM had a bad morning.

## A Static App with a Heavy Backpack

There is no backend and no database. The Python pipeline writes JSON. The React app reads it. Cloudflare serves the static files from a Worker.

The whole snapshot is about 12 MB of JSON, less over the wire with gzip. That is a deliberate trade. It makes every filter instant and lets the app work offline as a PWA. The service worker caches the data. On a phone, the first load is the cost. After that, the app is local.

The trade has a ceiling. If the archive grows to many more seasons, the first load becomes the problem. The plan is to split the largest collections by season and load them lazily. For now, the simple version is the right version. A static file cabinet is easier to keep alive than a database.

## The Parts That Are Not Official

The site has a few things that are not official data, and it says so.

The trophy cabinet counts every 1st, 2nd, and 3rd place across overall, gender, and category. Club cabinets add a participation trophy for every race where the club had the largest field, with ties counted, and a team podium when the club finished in the top three by finishers. Overall podiums are filled gold, silver, or bronze. Category and gender podiums are outlined. The difference matters: a category win is real, but it is not the same as an overall win.

The accumulated load chart is a model. It combines distance, pace intensity, distance impact, race duration, trail weight, weekly frequency, and cross-modality. The formula is in a disclosure under the chart. It is useful for comparing an athlete with their own history. It is not an official training load, and the site does not pretend it is.

The newest change adds total race time and total distance to the athlete season metrics, and running totals on the load chart. Small numbers, but they answer the question people ask after a good season: how much did I actually do?

## What Is Still Open

The archive is not complete. Three source PDFs are scanned images or empty, and they yield no results. Three relay events are skipped because the rows are teams, not people. 39 races have no reliable distance, mostly youth races, so they have no pace or progression. Some clubs exist only as an acronym in an old classification.

Name variants can still split an athlete. Transfers only link when the name matches exactly. The distance for some youth races is unknown, and the site leaves it unknown. A blank pace is better than a fabricated one.

These are not bugs to hide. They are the shape of the source. A public archive built from PDFs will always have edges. The job is to make the edges visible, not to smooth them over with guesses.

## Lessons

- A PDF is a document, not an API. Read the header, find the columns, and test against the real page.
- Fail closed on new data. Keep the last good file when a source breaks.
- Never join athletes by dorsal. Use the name, birth year, and gender, and accept that some profiles will split.
- Keep missing values missing. A blank pace is better than a made-up one.
- A static app can feel instant. The 12 MB is a real cost, but it buys offline use and simple operations.
- The data is the product. The interface is the shop window. Most of the work is in the pipeline, the tests, and the validation.

## Summary

CDInfante Estatísticas started as a way to answer one question after a race. It now holds four seasons of Madeira athletics: 272 races, 45,226 results, 4,824 athletes, and 159 clubs. It covers road and trail, every club in the circuit, and both languages.

It is built with our club, but it is for the whole athletics community on the island. The data comes from AARAM. The parsing mistakes are mine, and the site shows where the data stops.

You can use it at [estatisticas.cdinfante.org](https://estatisticas.cdinfante.org/). If a result looks wrong, the official PDF is one click away.
