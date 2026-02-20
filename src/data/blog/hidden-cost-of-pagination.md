---
author: Gopar
pubDatetime: 2026-02-11
title: Hidden Cost of Django Rest Framework Pagination
slug: hidden-cost-of-django-rest-framework-pagination
featured: false
draft: true
tags:
  - Django
  - Django Rest Framework
description:
  Django Rest Framework, just like Django, always tries to be helpful, but it might not always be beneficial.
---

Django pagination has 2 problems. Both of which will start to hurt as you scale.

We'll look at these problems through the eyes of Django Rest Framework (DRF) since it also uses Django's pagination
internally. Specifically through:

- CursorPagination
- LimitOffsetPagination
- PageNumberPagination

## Table of Contents

Note: All of code sources are from Django 6.0 and DRF 3.16.1

## A Look At LimitOffsetPagination

Let's start by looking at the source code of `LimitOffsetPagination`:

```python file=rest_framework/pagination.py

class LimitOffsetPagination(BasePagination):
    def paginate_queryset(self, queryset, request, view=None):
        #...
        self.count = self.get_count(queryset)
        self.offset = self.get_offset(request)
        #...
        return list(queryset[self.offset:self.offset + self.limit])

    def get_paginated_response(self, data):
        return Response({
            'count': self.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data
        })

    def get_count(self, queryset):
        try:
            return queryset.count()
        except (AttributeError, TypeError):
            return len(queryset)
```

I've truncated the code to show only relevant parts.

`LimitOffsetPagination` gets it's name from the SQL method it uses to paginate data (via `OFFSET`). It's been well
documented that `OFFSET` is not ideal when there is a large amount of data, because it ends up reading all the data from
the beginning of the query up to where it truncates. For example, if we want to fetch the next 50 entries after 1,000th
place, then the database has to read all 1,000 entries plus the next 50 to return, making it really wasteful once we
reach a heavily populated table.

This alone is rather wasteful and would deter some engineers from even adding this as a pagination option, but that is
not the only issue with this type of pagination (specifically the class implementation).

If we look at what's return from `get_paginated_response`, we can see that the payload has a key of `count`. Now if we
trace where it originates (`get_count`), then we realize another unpleasant truth. And that is, that Django will do a
`.count()` call on the queryset, which means that a sequential scan will have to happen in order to get an accurate
count of rows in the table. On a large table, this might cross the acceptable threshold of a response time, plus it adds
a second query that needs to be executed **every** time we run through this API endpoint.

## A Look At PageNumberPagination

If we look at `PageNumberPagination`, we'll see that it's not much of an improvement:

```python file=rest_framework/pagination.py
class PageNumberPagination(BasePagination):
    def get_paginated_response(self, data):
        return Response({
            'count': self.page.paginator.count,
            'next': self.get_next_link(),
            'previous': self.get_previous_link(),
            'results': data,
        })
```

## A Look At CursorPagination

## Always use CursorPagination?
Analytics, table size, how often ppl go into deeper pages.

Alternative class that hacks `explain` but i dont like it.
