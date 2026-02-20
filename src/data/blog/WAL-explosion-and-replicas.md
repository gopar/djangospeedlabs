---
author: Gopar
pubDatetime: 2026-02-11
title: Wal Explosion and Replicas
slug: wal-explosion-and-replicas
featured: false
draft: true
tags:
  - PostgreSQL
description:
  Short story on how we over-filled our disk space by configuring a replica incorrectly
---

We have some documentation:

```
# On primary
SELECT * FROM pg_create_physical_replication_slot('replica_1_slot');

# On replica
sudo pg_basebackup -h <primary-ip> -D data -U repuser -vP -W
Run sudo touch <data_directory>/standby.signal
Run sudo chown -R postgres:postgres <data_directory>
```

this caused disk to fill with WAL files, since we started pg_basebackup in streaming mode vs phsyical. This happened b/c
we didn't specifcy the slot that we just created.

in reality we needed to mianly do:

```
sudo pg_basebackup -h <primary-ip> -D data -U repuser -vP -W --create-slot --slot=replication-slot --checkpoint=fast
```
