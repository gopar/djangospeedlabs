---
author: Gopar
pubDatetime: 2026-02-11
title: How a 20ms Middleware Transaction Turned Into 400ms Latency
slug: how-a-20ms-middleware-transaction-turned-into-400ms-latency
featured: false
draft: true
tags:
  - Django
description:
  Even small transactions can become bottlenecks under concurrency. A look at middleware writes, locking, and scalability in Django.
---

While debugging a frontend issue, we noticed something interesting:
every request — across the entire app — was getting incrementally slower.

## Table of contents

## Following the breadcrumbs

The pattern was consistent. And each refresh added a little more latency with the more requests it had to make.

We started where most people would: the endpoint itself — but nothing.

So what's the next step?

Simple, profile it. We like to use [Django-Silk](https://github.com/jazzband/django-silk) for RESTful APIs.

Once setup, we were able to view each endpoint individually and that's when the problem became clear.

On **every** request, the `User.last_login` field was being updated and costing ~20 ms to do it.

And this was happening in a 3rd party middleware!
Note: This middleware is part of an internal company library, and is used for handling custom authentication flows.

## Reading the source code

Let's look at the bit of code responsible for slowing down (condensed):

```python file=middleware.py
class LoginServiceAuthenticationMiddleware(MiddlewareMixin):
    def process_request(self, request):
        # ....
        local_user, _ = User.objects.update_or_create(
            id=self.pk,
            defaults={
                "username": self.username,
                "email": self.email,
                "first_name": self.first_name,
                "last_name": self.last_name,
                "is_staff": self.is_staff,
                "is_active": self.is_active,
                "is_superuser": self.is_superuser,
                "last_login": timezone.now(),
            },
        )
        # ....
```

It relies on `update_or_create`.

Looking at Django’s implementation, we find:
```python file=django/db/models/query.py
def update_or_create(self, defaults=None, create_defaults=None, **kwargs):
        # .....
        with transaction.atomic(using=self.db):
            # Lock the row so that a concurrent update is blocked until
            # update_or_create() has performed its save.
            obj, created = self.select_for_update().get_or_create(
                create_defaults, **kwargs
            )
        # .....
```

Alright, here's what we need to know about `update_or_create`. It does two things:
1. Create a transaction.
2. Once inside, start a `select_for_update` which is translated as a `SELECT FOR UPDATE` statement.

The issue with `SELECT FOR UPDATE`, is that when PostgreSQL reads that, it acquires a row level lock.

Now ~20 ms sounds harmless, but when you have concurrent requests, each trying to update user info which requires a
lock, you will end up creating a bottleneck. In this case each transaction was lasting ~20 ms, but if you have 20
requests from the same user then that means:

1. Each will lock the user row.
2. Each will wait `[20 ms * (N-1)]` requests ahead of it, so when you are the 20th request in line that means you wait
   ~380 ms (20 * 19 ) just to start processing.

This process **kills concurrency**, since now we have to wait sequentially for updates to finish before proceeding.

## The Solution

There are several ways to approach this. The simplest: don’t update user state on every request.

The solution that we recommended was to add caching (cause hey, we all could use a little more caching), with a
customize-able timeout as to when to update the `last_login` again. Also, flamegraphs collected with
[py-spy](https://github.com/benfred/py-spy) showed **application time spent in this path drop from 34% of samples to
4%**.

Another solution would be to make the writes asynchronously (via celery) to move the act of updating/locking out of the
request/response life cycle. But this brings up an issue such as race conditions (an older task with an old timestamp
might overwrite a newer entry), so making tasks idempotent would be needed.

Now, we investigated whether updating on every request was truly necessary, but couldn’t identify a requirement that
justified it. We can't either easily remove transaction and row locking from the package since we don't know what other
teams/applications might rely on that logic (even if its a bit unorthodox).

## Takeaway
Writes inside middleware run on your hottest path.

When transactions and row-level locks are introduced, even small updates can become bottlenecks under load.

Treat every write in the request lifecycle as a scalability decision.
