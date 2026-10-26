---
layout: post
title: "DevOps - Run CDInfante: The Second Lap"
date: 2026-10-26 11:27:10 +00
categories: devops web
tags: react typescript vite pwa leaflet cloudflare scraping automation madeira security accessibility
---

The March post on [Run CDInfante](https://harryvasanth.com/posts/devops-run-cdinfante-running-app/), the free app built with Clube Desportivo Infante Dom Henrique for runners, hikers, swimmers, and anyone else who spends time outside in Madeira and Porto Santo, covered what the app does and how the pieces fit together. This is the follow-up. The app has not been rewritten. It has been used, questioned, and hardened. Here is what changed, why it changed, and what stayed the same.

The first version was built around a simple idea: put the websites we checked before every session into the same page. The second lap was about making that page trustworthy when a source changed, the network dropped, or the user was standing on a ridge with weak signal.

## The Short Version

The app still does the same job. The changes since March are mostly underneath:

- A new port meteo card for Funchal, Caniçal, and Porto Santo.
- A dashboard you can reorder, hide, and swap around.
- A consolidated map tile source.
- A larger scraper set, with shared guardrails.
- A security and quality review.
- Cloudflare Workers as the only deploy target.
- Accessibility fixes, a lighter browser bundle, and a better failure path.

The rest of this post explains what each change did to the app, and what it taught about keeping a small free service alive.

## A New Sense: The Port Stations

The biggest user-facing addition since March is the port meteo card. It covers Funchal, Caniçal, and Porto Santo. For each port, APRAM publishes a weather station, a tide gauge, and a wave buoy on the same page.

The weather station gives temperature, humidity, pressure, and wind. The tide gauge gives the low and high tides, with height and time. The wave buoy gives significant wave height, mean period, wave direction, and water temperature. For a swimmer or a triathlete, that is the difference between a useful forecast and a guess. For a runner, the tide times are a reminder that the seafront route changes shape during the day.

The card is also a map layer. The ports appear as markers, and the popup shows the same data. The map toggle row now includes water, toilets, alerts, trails, marine, and meteo. It is still readable on a phone, but only just.

The hard part was the data. APRAM does not publish an API for these stations. The page is a normal web page with separate sections, and the data is written in Portuguese. The tide times are the awkward part. The phase labels, low tide and high tide, are not table cells. They are text inside a small drawing. A normal table parser finds nothing, so the parser has to read the drawing, find the phase, and then read the neighbouring cells for the height and time.

That is the kind of detail that makes a scraper either work for years or break on a Tuesday. The parser now has a test that uses a copy of the real page, including the drawing. The scraper also refuses to publish if a port returns nothing. It would rather fail and leave the previous data in place than show a card full of empty values.

The port stations are the clearest example of how the app evolved. It did not add a new screen. It added a new source, a new parser, a new test, and a new guard. The screen is the easy part.

## The Dashboard Became Personal

In March, the dashboard could collapse and expand each panel. That was useful, but it was still a fixed layout. The current version has an edit mode. You can move a card up or down, hide it, and swap the columns. You can do the same with the weather locations. The settings screen still handles the location list and the number of ships, but the arrangement now lives on the dashboard itself.

The defaults are deliberate. On a phone, the cards start collapsed, because a long page on a small screen is worse than a short one. On a desktop, they start open. The card order is ships, warnings, trails, marine, and meteo. The weather column takes the wider share on a wide screen. None of this is clever, but it is the difference between an app that feels like a wall of panels and an app that feels like yours.

The layout is remembered in the browser. The reset button used to wipe all local storage for the site. Now it clears only the keys the app owns, including the theme and the language, and leaves everything else alone. That is a small thing until someone has another app on the same domain, or a test key in the console.

The settings also moved into a shared place, and the update prompt became its own small banner. The main app is smaller than it was, but still larger than it should be. The settings are still passed through too many layers. That is an open item, and it is honest work for another day.

The app also shows its version now. When someone reports a bug, the first question is no longer "which build are you on?" The version is on the screen.

## The Map Changed Its Skin

The March version used separate tile sets for light and dark mode. The current version uses a consolidated source and a dark-mode filter. That is a compromise. A filtered tile is not the same as a tile designed for dark mode. But there is less to cache and less to keep in sync.

The map still has the same job: show the island, the routes, the water, the toilets, the alerts, the marine points, the port stations, and the ships. The amenity markers now share a clustering setup, which keeps dense areas readable. The map is still a large part of the app. It still needs to be broken into smaller layers. That work is open, and it is the kind of change that needs a visual check rather than a quick refactor.

The map is also where the app's data model is most visible. Every layer comes from a different source, with a different refresh rate and a different failure mode. The map does not care. It renders what it has, and the rest of the app keeps working.

## The Data Layer Got Boring, in a Good Way

The app's value is not the map or the cards. It is the data layer. That layer grew, and the new scraper came with a parser and tests.

Here is what each scraper does:

- **Ships.** The port authority publishes the Funchal movement schedule as a set of cards, with a card for each vessel. The scraper reads the card heading, then the arrival, departure, origin, destination, agent, and port of call. It turns the Portuguese dates into a standard format.
- **Trails.** The forest institute publishes the official PR routes in a table, with distance, description, and status. The scraper reads the table, normalises the Portuguese status into open, closed, or partially open, and attaches trailhead coordinates from a fixed map. The official table does not publish coordinates, so the map is maintained by hand.
- **Amenities.** The scraper asks OpenStreetMap's Overpass service for drinking water and toilets inside a bounding box around Madeira and Porto Santo. It returns hundreds of points at the time of writing. The request identifies the app and a contact address, because Overpass is a shared public service.
- **Marine.** The marine forecast comes from Open-Meteo, for a set of coastal locations. The coordinates are pushed well offshore so they land in the water and the forecast is about the sea, not the coast. The scraper works in chunks and pauses between requests.
- **Meteo.** The APRAM port stations described above. Separate sections, a parser, and tests.

The shared rules are the part that matters:

- Retries with backoff when a source fails or rate-limits.
- A timeout on every request.
- A cap on the output file.
- A cap on the length of scraped text.
- A refusal to write when the scrape finds nothing.
- A low-count guard when the number of records is suspiciously small.
- A fill-ratio guard when too many numeric fields are empty.

The philosophy is simple: fail closed. A scraper that writes an empty file is worse than a scraper that stops. The app can survive an hour of stale data. It cannot survive a data file that looks valid but is not.

The data workflow runs the scrapers every hour. Each scraper runs even if an earlier scraper fails, and a final step fails the job if any of them did. That way a broken source does not hide the state of the others. The commit job only runs if the scrape job succeeds, so a partial run never reaches the live branch.

The workflow also split the permissions. The job that installs dependencies and runs the scrapers can read the repository but cannot write to it. The job that commits the data has write access, and it runs after the data has been passed across as an artifact. That matters because installing dependencies runs their install scripts. A hostile package could read a token during install. In the old workflow, that token could push to the repository. Now the job that installs dependencies cannot push at all.

The commit step also ignores timestamp-only changes. Every scrape rewrites the "scraped at" time, so the workflow filters that line out and commits only when something real changed. Most of the repository's history is data updates. That looks absurd until you remember what the app is: a small front end on top of a data pipeline that runs every hour. The commit history is mostly the pipeline doing its job.

## The Browser Got Lighter

The browser bundle still carried a request library that did not need to be there. It was marked as a development tool, but the build pulled it into a bundle that loaded on every page. The app only needed a simple request with a timeout. The services now use the browser's own request support, and the library stays in the scraper scripts where it is needed for retries.

The fonts changed too. The font link used to block the first paint. It now loads without blocking, with a fallback for browsers that do not run scripts. The fonts could be hosted locally or loaded without blocking. The non-blocking load was the compromise. Hosting them is still the cleaner answer.

The service worker no longer precaches every image the site ships. The large social image and the screenshots are excluded. That saves a noticeable amount of install weight, which matters on a phone. The caching approach is unchanged in shape: map tiles are cached first, the data files are served from cache and refreshed in the background, and the live weather and warnings prefer the network. The difference is that only map tiles are cached blindly. The data files are cached only when the response is a success, so a cached error page cannot pretend to be data.

The update check moved from an hourly timer to a check when the tab becomes visible. It runs at most once an hour. The old timer made a network request every hour even when the app was in the background. On a phone, that is battery and data for no reason.

The lazy loading is still there, but it does not do what the March post said it did. The main sections render on the first screen, so the split does not defer the first paint. It lets the chunks load in parallel. The March post said the lazy loading kept the initial bundle small. That was not quite right.

The error boundary is new. Each main section is wrapped in an error boundary. If the map fails, the dashboard still works. If the dashboard fails, the map still works. The fallback shows a short message and a retry button. It is a small change, and it turns a white screen into a bad section.

## The Security Review

The security and quality pass covered the whole project, not just the dependencies. Most of the findings are closed. A few remain open or partial.

The highest-severity findings came first.

The first was the dependency tree. There were several high advisories. Most were in build tools, which do not run for visitors. The finding that mattered was a request library that was in the browser bundle and declared in the wrong section of the project. Updating the toolchain and moving the runtime libraries into the right place fixed the declaration. Removing the library from the browser fixed the bundle.

The second was the Content Security Policy. The site now ships a set of security headers with the page. The policy allows scripts from the app itself, because the app does not use inline scripts. It allows inline styles because the map and interface libraries need them. It allows the specific APIs and tile sources the app actually calls. The policy is a second line of defence. It does not replace clean code, but it means a future mistake has less room to run.

The third was the workflow token. The old data workflow gave the job that installed dependencies permission to write to the repository. A compromised dependency could have read the token and pushed to the live branch. The new workflow splits the scrape and commit jobs, gives the write permission only to the commit job, and passes the data between them as an artifact. This was the most important change in the whole pass, and it is invisible to users.

The medium items included the output limits, the location prompt, the cache rules, and the third-party data flow. The location item is a good example of a product change hiding in the security work. The map used to ask for location permission on load. Every visitor got a prompt before they asked for anything. The prompt now waits for the locate button. The browser is more likely to grant it, and the user is less surprised.

The cache item is subtler. The service worker used to cache any response for every route, including cross-origin responses the browser cannot inspect. That is correct for map tiles. It is wrong for data. A cached error page can look like data for a long time. The data caches now store only successful responses. Map tiles are still cached blindly.

The low items were mostly maintenance. The reset button no longer wipes all local storage. The viewport no longer blocks zoom. The GitHub Actions are pinned to commit SHAs.

The zoom item is worth calling out. The March post said the app disabled user zoom on mobile so it felt like a native app. That was a bad trade. It made the interface feel more app-like for people who did not need to zoom, and it made the app unusable for people who did. The viewport now allows zoom. The app still feels like an app. Accessibility won, and it should have won earlier.

The quality side covered performance, best practice, and architecture. The interesting fixes were the shared data loading, the shared caching, the error boundary, the check workflow, a shared source of truth for the version, the translation layer, and the separation of the ship logic from the network call. The ship logic now lives on its own, with tests. It takes a list of ships and a time, and returns the docked count, the next available slot, and the formatted list. That is easier to test than a version that also fetches the data.

Some findings were left open.

The remote data source is an example. The app still fetches the data files from the repository's main branch and falls back to the copy bundled with the app. The cleaner option is to use only the local files or pin the branch to a specific commit. The remote fetch stays, with the reason written down. The data is inert JSON. The app renders it as text. It does not execute it. Fresh data without a deploy is worth more than a pinned branch for this app. The risk is accepted, not forgotten.

The animation library is a similar case. Several parts of the interface use it for fades and small movements, and a CSS animation could replace most of it. That is a visual change, and it needs a design check, so it stays open.

The last items are size. The main app and the map are still larger than they should be. Moving the settings into a shared place, and grouping the map markers, moved in the right direction, but the main sections and the map layers are still doing too much in the same place. They are on the list.

A review is a work order, not a verdict. Some findings are worth fixing, some are worth accepting in writing, and some need a design decision before anyone touches the code. The value is in the list, not in a perfect score.

## The Deploy Got Simpler

The app runs on Cloudflare Workers. The configuration sets the worker name, the fallback for unknown paths, and basic monitoring. The build output is static assets. The security headers ship with the site, so there is no separate proxy to configure.

The old GitHub Pages workflow is still in the repository, but it runs only on a manual start. It stays as a reference, not a live path. Deleting it would be cleaner, but it remains for now, with the reason written down. A fresh project would probably delete it and keep the deployment notes with the project instead.

The deploy is straightforward. The data workflow commits to the live branch every hour. If the Cloudflare Git integration is on, that commit starts a build. If it is off, the deploy is a manual command after the data commit. Either way, the app and the data share the same home.

## What Did Not Change

The principles did not change.

There is still no backend, no database, and no login. The app is a static site with a data layer. The weather and warnings are fetched live. The ships, trails, amenities, marine, and port stations are scraped ahead of time into JSON. The split is still right: live data where freshness matters, static data where rate limits, CORS, and offline matter more.

The app is still free. There are no ads, no premium tier, no analytics, and no cookies. The only personal data it stores is your own settings, in your own browser. There is no account, and there is nothing to sign up for.

It is still bilingual, still has a dark mode, and still installs as a PWA. It still works offline, which on Madeira is not a gimmick. It is a tunnel, a valley, or a ridge with weak signal.

The core question is the same as it was in March: what is it like outside, and is the route open? The app answers that question with weather, warnings, trail status, sea conditions, ships, and the map. The port stations are a new answer to the same question.

## What Is Still Open

The open items are limited, but they are not small in effort.

The remote data source is accepted. The animation library is open. The main app and the map are still larger than they should be. None of these is a bug. They are the next round of work.

There is also the ordinary maintenance that never appears on a list like this: dependency updates, a source page that changes its markup, a new trail, a new port, a new language string. The app is not finished, and it should not pretend to be.

## Lessons from the Second Lap

The first lap was about getting the app to work. The second lap was about getting it to keep working. These are the lessons that stuck.

- A feature is the easy part. The data source is the hard part. The port meteo card took a parser, a test, and a guard. The card itself was the small part.
- Scrapers must fail closed. A broken scrape that writes an empty file is worse than a scrape that stops. The previous file is always better than a confident lie.
- Static data was the right call. It is fast, cheap, offline-capable, and free of CORS. The app has no server to patch and no database to back up.
- Security and accessibility are not separate tracks. The zoom block and the location prompt were product decisions with a cost. They showed up because the whole app was in scope, not just the dependencies.
- Keep the app boring. The interesting work is in the guards, the cache, the headers, and the logs. The user should see a page that loads, not a system that impresses.
- Write down the risks you accept. An accepted risk with a reason is engineering. An accepted risk without a reason is neglect.

## Summary

The second lap was not about adding more. It was about making what was already there trustworthy. The port stations and the dashboard edit mode are the changes most users will notice. The rest is under the floor: scrapers that fail closed, a workflow that cannot leak a token, a browser bundle without a request library, a consolidated tile source, a security policy, an error boundary, and a reset button that does not clear someone else's data.

The app is still small, still free, and still works when the signal drops. The club kept using it through all of it, which is the only test that matters. The next lap is already on the list.

You can use it at [run.cdinfante.org](https://run.cdinfante.org/). If something looks wrong, the reset button is in the settings, and the version number is in the footer.
