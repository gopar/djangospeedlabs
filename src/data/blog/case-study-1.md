---
author: Gopar
pubDatetime: 2026-02-11
title: A Case Study of Simple Optimizations
slug: a-case-study-of-simple-optimizations
featured: false
draft: true
tags:
  - Django
description:
  How basic optimizations unlock 9x speed boosts
---
This needs some text maybe?

# Table of Contents

1.  [From 3.5 Minutes to 2.5 Seconds](#orgdb7cb87)
2.  [Versions Used](#org4733c74)
3.  [First pass](#orgd8dd447)
4.  [Profiling & Identifying Bottlenecks](#orgae77caf)
5.  [Tools & Techniques Used](#orgb155ac0)
6.  [The Optimization Process](#org9ebe27d)
7.  [The Results](#org483b4b5)
8.  [Lessons Learned](#org74fb4b3)
9.  [Conclusion](#org9fe1353)

clear;git log &#x2013;reverse &#x2013;no-patch -L 203,223:backend/api<sub>v1</sub>/wells/views.py | grep -i gopar -A 4 -B 1

-   <https://github.com/SummitESP/summit-wells/pull/170/files>
-   <https://github.com/SummitESP/summit-wells/pull/186/files>
-   <https://github.com/SummitESP/summit-wells/pull/287/files>
-   <https://github.com/SummitESP/summit-wells/pull/356/files>
-   <https://github.com/SummitESP/summit-wells/pull/435/files>
-   <https://github.com/SummitESP/summit-wells/pull/449/files>


<a id="orgdb7cb87"></a>

# From 3.5 Minutes to 2.5 Seconds

A client had an application that was taking, on average, 3.5 minutes to load. The main culprit was an API endpoint,
that would be in charge of searching and fetching related data for each record. We'll talk about how we took this from
3.5 minutes to an avg of 2.5 seconds. A ~98.8% speed improvement using **only basic optimization techniques** that anyone
can do.

Note: This optimization was done on a single API endpoint, and that's what this blog post will revolve around.


<a id="org4733c74"></a>

# Versions Used

The project stack at the time:

-   Django (4.2)
-   Django Rest Framework (3.13.1)
-   PostgreSQL (12)
    -   Timescale DB Extension (1.7.5)


<a id="orgd8dd447"></a>

# First pass

When looking under the hood at the actual Implementation. I saw a few things:

-   The view had a very basic query
-   Serializer was calling many more relations (N+1)
    -   Not only N+1 calls, but very `expensive` queries to run
-   Lots of extra columns being pulled
-   Overloading the db with multiple un-needed joins
-   Logic spread throughout multiple palces (models, views, and serializers)
    -   This isn't inherently bad but it doesn't make it hard to reason about things


<a id="orgae77caf"></a>

# Profiling & Identifying Bottlenecks


<a id="orgb155ac0"></a>

# Tools & Techniques Used


<a id="org9ebe27d"></a>

# The Optimization Process


<a id="org483b4b5"></a>

# The Results


<a id="org74fb4b3"></a>

# Lessons Learned


<a id="org9fe1353"></a>

# Conclusion
```python file=my/awes/path.py
class WellFilterSet(PropertyFilterSet):
    status = PropertyBaseInFilter(field_name='status', lookup_expr='in')

    class Meta:
        model = Well
        fields = {
            'id': ['exact'],
            'name': ['icontains', 'exact'],
            'display_name': ['icontains', 'exact'],
            'customer__id': ['exact'],
            'customer__operator__id': ['exact'],
            'customer__name': ['icontains'],
            'well_groups__id': ['exact'],
            'api_number': ['icontains', 'exact'],
            'erp_ship_to_id': ['icontains', 'exact'],
        }

from rest_framework.filters import OrderingFilter, SearchFilter
from django_filters.rest_framework import DjangoFilterBackend

class WellViewSet(ModelViewSet):
    serializer_class = WellSerializer
    filter_backends = [SearchFilter, OrderingFilter, DjangoFilterBackend]
    filterset_class = WellFilterSet
    search_fields = ["id", "name", "field__name", "customer__name", "region__name"]
    ordering_fields = "__all__"
    ordering = ["name"]
    def get_queryset(self):
        # We want to filter by customer to prevent customers from seeing each others wells
        filter_info = {"customer__erp_customer_id__in": self.request.user.customers}
        return Well.objects.filter_by_customer(self.request.user, filter_info).filter(deleted=False)

class WellSerializer(ModelSerializer):
    field = FieldSerializer(required=False, allow_null=True)
    region = RegionSerializer(required=False, allow_null=True)
    customer = CustomerSerializer(required=False, allow_null=True)
    streams = DataStreamSerializer(many=True)
    equipment_runs = EquipmentRunNestedSerializer(source="equipmentrun_set", many=True, required=False)
    active_jobs = serializers.SerializerMethodField()

    class Meta:
        model = Well
        fields = [
            'active_jobs',
            'api_number',
            'customer',
            'erp_well_id',
            'erp_ship_to_id',
            'erp_welladdress_id',
            'field',
            'id',
            'latitude',
            'longitude',
            'name',
            'region',
            'runlife',
            'sk_legacy_well_id',
            'status',
            'streams',
            'equipment_runs',
            'last_reading'
        ]
    @transaction.atomic
    def create(self, validated_data):
        customer_data = validated_data.pop('customer')
        region_data = validated_data.pop('region')
        streams_data = validated_data.pop('streams')
        customer = None
        if customer_data:
            try:
                customer = Customer.objects.get(erp_customer_id=customer_data['erp_customer_id'])
            except Customer.DoesNotExist:
                customer = Customer.objects.create(**customer_data)
        try:
            region = Region.objects.get(erp_region_id=region_data['erp_region_id'])
        except Region.DoesNotExist:
            region = Region.objects.create(**region_data)
        new_well = Well.objects.create(customer=customer, region=region, **validated_data)
        for streams_dict in streams_data:
            stream = DataStream.objects.get(id=streams_dict['id'])
            stream.well = new_well
            stream.save()

        return new_well
```
