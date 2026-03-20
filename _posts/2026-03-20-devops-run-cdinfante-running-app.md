---
layout: post
title: "DevOps - Run CDInfante: A Community App for Madeira's Runners, Swimmers, and Trail Lovers"
date: 2026-03-20 09:51:01 +01
categories: devops web
tags: react typescript vite pwa leaflet cloudflare scraping automation madeira
---


## The Story: From a Running Club to the Whole Island

The app started with a simple, practical need. I run with a local club, the Clube Desportivo Infante Dom Henrique, on the island of Madeira. Anyone who trains outside in Madeira learns the same lesson quickly: the island's weather does not care about your plans. One side of a mountain can be in thick cloud while the other is in blazing sun. A beach that was calm at breakfast can be dangerous by the afternoon. A favourite trail can be closed overnight because of a landslip or a fire risk.

We kept checking five different websites before every session. The weather forecast. The official weather warnings. The port schedule, to see whether a cruise ship was going to block the seafront where we run. The trail status pages for the levadas and the mountain paths. The sea forecast for the swims. It was a ritual, and it was a chore.

So I built a single app that gathers all of it in one place. It was meant for our members, and we used it and tested it internally for a few months before letting anyone else near it. The club found it genuinely handy, so we figured the same scattered-information problem must affect every runner, hiker, swimmer, and outdoor person on Madeira and Porto Santo, not just us. We published it to the whole community. Today it lives at [run.cdinfante.org](https://run.cdinfante.org/), and anyone on either island can use it.

This post explains what the app does, how it is built, and how the pieces fit together.

You can try the webapp yourself: **[run.cdinfante.org](https://run.cdinfante.org/)**. It works in any browser, and on a phone you can add it to your home screen so it behaves like a native app.

## The Problem the App Solves

Madeira is a small island with a lot of microclimates and a lot of outdoor activity. Athletes need answers to specific, practical questions before they go out:

- What is the weather like in Funchal versus the mountains today?
- Are there official warnings in force, and where?
- Is a cruise ship docked at Terminal Sul, and when will the berth be free?
- Are the levada trails open, closed, or partially blocked?
- How big are the waves, and are there dangerous currents at the beach?
- Where is the nearest drinking fountain or public toilet on a route?

Each of these questions has an official answer somewhere, but it lives on a different government or agency website, in a different format, sometimes in a different language. The app's real value is not any single data source. It is the fact that one page answers all of them at once, in a format that works on a phone in a pocket.

## The Features

### Weather That Knows the Island

The weather section is the heart of the app. It covers 67 locations across Madeira and Porto Santo, from the town centres to the mountain peaks and the north coast villages. Each location shows current conditions: temperature, wind speed and gusts, wind direction, humidity, UV index, precipitation, visibility, and cloud cover. Below that come the daily high and low, sunrise and sunset, and the probability of precipitation.

The app also shows air quality using the European AQI, including PM2.5, PM10, and dust, alongside pollen levels for nine allergen types. For people with asthma or allergies, that is not a nice-to-have. It is the difference between a good run and a miserable one.

Weather is one of the few things the app fetches live, because it genuinely changes every few minutes. Even so, there is a backup service. If the primary weather API is unreachable, the app silently falls back to a secondary source and labels the card with a small "backup" badge, so the user knows the data is still fresh, just from a different provider.

### Official Weather Warnings

Weather forecasts are useful. Official warnings are essential. The app pulls live warnings from IPMA, Portugal's weather agency, for four regions: the North Coast, the Mountain Regions, the South Coast, and Porto Santo. It covers eight warning types: wind, rain, extreme heat, extreme cold, sea agitation, fog, snow, and thunderstorms.

Each warning is colour-coded by severity, using the same green, yellow, orange, and red scale the agency uses. Tapping a warning links out to the official IPMA detail page, because the app shows you the alert but never tries to replace the authority that issued it.

### The Cruise Ship Tracker

Funchal's seafront at Pontinha is a popular running route, and it has a recurring problem: cruise ships. When a ship is docked at Terminal Sul, large parts of the seafront are closed off to the public and the berth is occupied. Runners want to know whether they can use the promenade, and when.

The ship tracker answers the exact question we used to have to phone the port about: are there ships in today, and is the south terminal closed to the public? It shows the arrivals and departures for Funchal, marks the ships currently docked at Terminal Sul, and counts them. The interesting part is the "next available" calculation. When no ship is docked, the berth is free right now. When ships are docked, the app works out when the last of the overlapping stays ends, walking through the departures to find the moment the berth genuinely frees up. It is a small scheduling problem, and it is exactly the kind of detail that makes the app feel thoughtful. Each ship links to the APRAM schedule and to MarineTraffic for a live view.

### Hiking Trails

Madeira's levadas and peaks are the island's main draw for walkers. The official trails are the PR routes, and their status changes often: a section closes for maintenance, a storm makes a path unsafe, a fire risk shuts a forest trail.

The app lists the official PR trails for Madeira and Porto Santo, with their distance, description, and current status. The status is dug out of the official IFCN PDFs that are buried inside the forest institute's website, so you never have to hunt through documents to find out that your favourite levada is closed today. Status is colour-coded, open, closed, or partially open, in a grid of badges so a glance tells you which routes are usable today. Each trail links to IFCN, the forest institute that maintains the official information, for full details.

### Sea Conditions

Swimming and sea activities need the marine forecast. The app provides sea temperature, wave height, wave period, and wave direction for coastal locations, and splits the waves into their two components: wind waves and swells. It also shows ocean current velocity and direction. The whole section exists for the triathletes and swimmers who like to cap a training session with a dip, because knowing the wave and current conditions is what separates a refreshing swim from a struggle.

There is a neat detail in how the marine locations were chosen. The coordinates are deliberately pushed a few hundred metres offshore, so they plot strictly in the water rather than on the land. That keeps the markers on the map honest and makes the marine API return real sea data rather than a coastal approximation. Locations are grouped by island, with Madeira, Porto Santo, and the Desertas islands covered.

### The Interactive Map

The map ties everything together spatially. It is a Leaflet map centred on Madeira, using OpenStreetMap tiles, and it layers the community data on top. Each layer can be toggled independently: drinking fountains, public toilets, weather alerts, hiking trails, and marine conditions. The water fountains and toilets are the quiet heroes here, a genuine lifesaver on long runs when the nearest public toilet is the difference between a planned route and an emergency detour. Dense areas cluster their markers so the map stays readable. The user's current location appears as a blue dot.

The map adapts to the theme. In light mode it uses a clean, light tile set. In dark mode it switches to dark tiles, which matters a lot at night when you want to check a route without blinding yourself. Map tiles are cached aggressively, because they are immutable, and the map is designed to keep working even when the connection drops.

### Dashboard and Settings

The dashboard arranges everything into a responsive grid: ships, warnings, trails, marine conditions on one side, weather cards on the other. Each panel can be collapsed or expanded, and the state persists between visits, so the app remembers how you like to use it. Weather cards can be expanded to reveal the full detail for a location.

The settings screen lets each user make the app their own. You can add or remove weather locations from the searchable list of 67, and you can set how many ships appear in the tracker, from one to twelve. One button resets everything back to the defaults, which is also the first thing we tell someone to try when they hit a glitch.

### A Progressive Web App

The app is installable. On a phone you just open it in the browser and choose "Add to Home Screen", and it behaves like a native app: standalone window, its own icon, full screen. The same works on desktop. It works offline, which is not a gimmick on an island with mountain valleys and tunnels where the signal drops constantly. The service worker caches the map tiles, the API responses, and the static data, and when a new version of the app is available, the user gets a polite update prompt instead of a silent break.

### Two Languages, Both Islands

The audience is bilingual, so the app ships in English (en-GB) and Portuguese (pt-PT), and it remembers the choice. It respects the system colour scheme, uses semantic HTML and ARIA throughout, and disables user zoom on mobile so the interface feels like a native app.

### Free, Private, and Ad-Free

The app is 100% free, and that is a deliberate product decision, not a pricing accident. There are no ads, no premium tier, and no analytics. There is no tracking and no cookie banner, because there are no cookies doing anything that would need one. The only thing the app stores is your own settings, in your own browser. A free community app for outdoor people should feel exactly that clean.

## How It Is Put Together

### A Static App with a Living Data Layer

The single most important architectural decision is that the app is a static site. It is a React single-page application built with Vite and TypeScript, and it compiles to plain HTML, JavaScript, and CSS. There is no application server, no database, and no login. Everything that makes it useful comes from data.

The data comes in two tiers. Live, fast-changing data, weather and warnings, is fetched from public APIs in the browser. Everything else, ships, trails, sea conditions, fountains, and toilets, is scraped ahead of time into static JSON files that ship with the app. This split is deliberate, and it drives most of the other decisions.

Static JSON is fast, because it is served from the same static host as the app, or from a content delivery network. It has no rate limits, because the scraping happens once an hour on a server, not in a thousand browsers. It works offline, because the service worker can cache it like any other asset. And it avoids CORS entirely, because many of the source websites do not allow cross-origin requests from browsers.

### The Scraper Farm

Behind the static JSON files is a small fleet of scraper scripts. Each one knows how to read one official source and turn it into clean, typed JSON:

- The ship scraper reads the APRAM port movement page for Funchal, parses the HTML for each vessel, and normalises the Portuguese date and time format into ISO timestamps.
- The trail scraper digs the status, distance, and description for each PR route out of the official IFCN PDFs, because that is the only place the information is published.
- The amenity scraper pulls drinking fountains and public toilets from OpenStreetMap's Overpass API, so the community map has real data instead of hand-drawn points.
- The marine scraper calls the Open-Meteo marine API for a curated list of coastal locations, in chunks, with a delay between requests to stay polite.

The scrapers are defensive on purpose. If a scraper finds nothing at all, it exits with an error rather than writing an empty file. That prevents a layout change on the source website from silently destroying a week of good data. The marine scraper goes further: before it writes, it checks how many of the expected values are actually filled, and if the fill ratio is too low, it aborts and keeps the previous good file. The philosophy is simple, never let a broken scrape overwrite good data.

They share a small HTTP helper that wraps axios with retries and exponential backoff, so a transient network blip does not kill a run.

### The Data Refresh Loop

A GitHub Actions workflow runs all four scrapers every hour, on a schedule. Each scraper runs independently and reports its own success or failure. If any scraper fails, the workflow fails, so a regression in one data source is noticed rather than ignored.

When the scrapers produce changes, the workflow commits them and pushes the update. There is a subtle piece of bookkeeping here. Every scrape rewrites a "scraped at" timestamp, which would change the file even when the actual data is identical. The workflow filters that field out of the diff, so it only commits when there is a real change, and the commit history stays clean instead of recording the same data every hour.

### The Browser-Side Data Flow

In the browser, the fetching layer is built on TanStack Query, which gives every request a cache, a staleness window, and retries. The query client is tuned for this specific use case. It uses offline-first network mode, so when the phone is in a tunnel with no signal, a request does not fail loudly. It serves whatever is in the cache, and the UI stays calm.

The static data has a layered fallback. The app tries the freshest copy first, the raw file in the GitHub repository, then falls back to the copy bundled with the app. Both are cached by the service worker, so the fallback is effectively instant after the first visit.

The live weather calls are protected in two ways. A concurrency lock caps the number of simultaneous weather requests, because the free weather API rate-limits aggressively and a dashboard with six weather cards would trip it instantly. And every live request has a timeout, so a hung third-party API can never hang the app.

### The Caching Strategy

The service worker is not a generic catch-all. Each data source gets a deliberately chosen caching strategy:

- Map tiles are cached with CacheFirst, because they are immutable and heavy, and revalidating them is pure waste.
- The scraped JSON data uses StaleWhileRevalidate, so the user always sees the last known good data and the cache refreshes in the background.
- The live APIs, weather and warnings, use NetworkFirst with a short network timeout. The app prefers fresh data but will happily fall back to a recent cached copy when the network is slow or down.

The app also checks for service worker updates every hour and asks the user before applying an update. Nobody likes an app that refreshes itself mid-run.

### Performance and Feel

The app is a single page, but it is built like a larger system. The heavy sections, the map and the dashboard, are lazy-loaded so the initial bundle stays small. Vendor code is split into separate chunks, so Leaflet, Framer Motion, React Query, and React each have their own cacheable file. Every animated visual detail is built to avoid React re-renders: the ambient background glow writes directly to the DOM inside a requestAnimationFrame callback, with a passive event listener, instead of triggering a re-render on every mouse move.

The HTML does DNS prefetching for every third-party API the app talks to, shaving latency off the first request to each service. Fonts are preconnected. The whole thing feels fast, which matters when someone is checking the conditions before heading out the door.

## How It Was Thought Out Ahead

Most of the app's value is not in any single feature. It is in the decisions made before the features were built. These are the principles that shaped it.

**Fail gracefully, never lie.** Every data source has a fallback or a clearly labelled "backup" state. When something fails, the app either shows the last known good data or tells you the data is from a backup service. It never shows you nothing, and it never pretends a fallback is the primary source.

**Respect the free APIs.** The scrapers run once an hour, not every minute. Browser requests are concurrency-limited, cached, and given timeouts. A community app on free public APIs survives by being a polite citizen, and being polite is also what keeps the app reliable.

**Protect good data.** The scrapers would rather fail loudly than overwrite good files with broken scrapes. The update workflow refuses to commit timestamp-only changes. After months of hourly runs, the data files stay clean and trustworthy.

**Work offline by default.** On an island of tunnels and mountain valleys, offline is not an edge case. It is the norm on half the routes. Caching strategies were chosen per data source, and the app treats cached data as a feature, not a fallback.

**Keep the authority in the loop.** The app summarises official data but always links back to the source, IPMA, IFCN, APRAM. It is a front door to the official information, not a replacement for it.

## Summary

Run CDInfante started as a small tool for a running club and became an app used by the whole outdoor community on Madeira and Porto Santo. It gathers weather, official warnings, cruise ship schedules, trail status, and sea conditions into one installable, offline-capable page.

The architecture reflects that journey. A static front end, no backend to maintain, live APIs for fast-moving data, a scraper farm that feeds a static data layer, and a caching strategy chosen per data source. The whole system is built to fail gracefully, respect free APIs, protect good data, and keep working when the signal disappears.

The best part of the project is that the people it was built for use it every day, and the community it started with is exactly the community that helped it grow.