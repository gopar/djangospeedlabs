---
author: Gopar
pubDatetime: 2026-02-11
title: Optimizing Django Queries Starts With Context, Not Indexes
slug: optimizing-django-queries-starts-with-context-not-indexes
featured: false
draft: true
tags:
  - Django
description:
  A slow list endpoint wasn’t about missing indexes — it was about loading more data than the UI needed.
---

The main entry point for a website was a listing page. The issue? Database was taking too long to return results.
The fix? Asking the right questions to reduce load on db.

## Table of Contents


## Slow endpoint.

This listing api endpoint was taking around 2-3 seconds to load all the data it neded. This was made worse that the UI
allowed the user to do 'search as you type' functionality, which further increased load on the database. At first you
might be inclined to staart profiling the queyr that is being used, but in this case (or in almost all cases) this would
be the wrong direction to go. At first we need to understand what we are loading and why. Once we starting asking and
undestanding we realized one thing. That was that we are loading way too much information that is irrelevant tot he user
when they are searching for a specific entry. In this UI, we have 9 columns that we are showing. Now only 3 of those
were actually relevant to the user. So how and when should we display the rest of the columns to the user? We can set a
timer that after X period of time (eg 2 seconds) we fetch the remaining columns from the database and show them along
side, meanwhile we can display a skeleton or spinning wheel that is commonly seen when loading data that a user would be
familiar with.


## Immediate instinct: optimize query.

## Before doing that — ask:

- What does the UI actually render?

- What does the user actually need?

## Discovery:

- We only need N columns.

- FK fields aren’t needed immediately.

## Fix:

.only() / .defer()

Lazy loading.

Secondary fetch after 2s.

## Takeaway:

Optimization starts with understanding data requirements.

That’s a strong performance-authority narrative.
