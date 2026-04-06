---
title: Protected Storage
description: Options for confidential storage
weight: 60
---

By default, CoCo workloads execute in confidential guest memory.
Files written to the workload filesystem will be stored in guest memory
and protected by confidential computing.

Of course, cloud native workloads can leverage a wide variety of external storage options.
Some of these might break the confidential trust model. Some can be used with adaptations
to the workload (e.g. wrapping secrets before storing them).
Carefully consider the trust model of any external storage volumes, services, or paradigms
before attaching them to a confidential workload.

To simplify things, Confidential Containers provides some confidential storage primitives.
