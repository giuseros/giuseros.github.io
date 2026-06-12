---
layout: post
title: "Finding my travel history in 5 minutes with geotrace"
date: 2026-06-12
tags: [python, ai, photos, geocoding]
---

I needed to apply for my UK passport, and that means listing every trip I've taken out of the country over the last few years. I know people who have spent *days* digging through old calendars, emails, and boarding passes to reconstruct this — and a few who gave up and paid someone to do it.

But the answer was sitting on my disk the whole time: my photos. Every picture my phone takes is quietly stamped with where and when it was taken. So I vibe-coded a little app that walks through a photo archive, pulls the GPS and timestamp out of each image (and out of the Google Photos Takeout sidecars), figures out which country I was in on any given day, and spits out a clean list of every trip away from "home".

It took about five minutes. Five minutes to turn 18,000 photos into a tidy table of *"Italy, 5–9 April 2024"* rows. Amazing.

A couple of things I liked about how it turned out:

- **It runs entirely offline.** Your photos and your coordinates are about as personal as data gets, so the reverse-geocoding (turning latitude/longitude into a country) uses a local dataset. Nothing leaves the machine.
- **It's honest about its limits.** It's an aid, not legal proof — trip dates are the first and last geotagged photo of a stay, so I still cross-check the important ones against boarding passes.

I cleaned it up and put it on GitHub: [**geotrace**](https://github.com/giuseros/geotrace). If you've got a pile of photos and a form to fill in, it might save you a week.
