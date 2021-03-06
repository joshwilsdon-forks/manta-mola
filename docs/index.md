---
title: Mola
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Mola

The Mola Mola, a.k.a "sunfish", a.k.a "a big floating blob"*.

* http://animals.nationalgeographic.com/animals/fish/mola/

In Manta-land Mola can be one of two things:

1. The zone from which Manta "system" jobs are managed.
2. The system of components that takes care of object reconciliation (garbage
   collection, object audit, rebalancing, etc.)

# Mola, System Jobs

The contents of this repo are joined with Mackerel and work with components
running on other systems to produce system-wide "crons".  The overview of all
of these system process can be found here:

[Manta System "Crons"](system-crons.md)

If you want to run a GC manually or want an introduction to what to look for
when GC should be running, here is a brief tutorial:

[Running GC Manually in COAL](gc-manual-coal.md)

# Mola, Object Reconciliation

There are three main processes that Mola manages (or "is" depending on the way
you look at it):

1. [Garbage Collection](gc-overview.md)
2. [Object Auditing](audit-overview.md)
3. [Cruft Tracking and Collection](cruft-overview.md)
4. [Object Rebalancing](rebalancing-objects.md)

Each follow the same pattern for execution:

1. Cron on ops host kicks off node process /opt/smartdc/mola/bin/kick_off_[...]
2. Node process finds all relevant moray/mako dumps.
3. Node process starts marlin job to crunch data.

Depending on which mola job there may be other processes that run:

1. GC: Ops zone: Link creator that checks results from marlin job and will
   create manta object links for moray and mako cleaners.
2. GC: Ops Zone: Moray cleaner that deletes records from Moray.
3. GC: Mako Zone: Mako cleaner that tombstones mako data.
4. Rebalance: Mako Zone: Mako Rebalancer that pulls down objects, updates moray
   records, and tobstones the object at the old location.
