---
layout: page
title: "Virtual Ghost: Protecting Applications from Hostile Operating Systems"
date: 2014-03-08 16:30
comments: true
sharing: true
footer: true
---

#### Motivation:

protect application from compromised or even hostile OS.

#### Approach:

重新编译OS，对其进行Instrumentation，创建一个Ghost memory，不能被OS所读和做修改。然后通过SVA（Secure Virtual Architecture）来做CFI。

这里SVA会在之后专门来介绍。

我觉得这篇论文最大的收获是它定义了一个hostile OS 都会做那些事:

* Data access in memory: directly read/write, manipulate EPT to read/write, DMA read/write.
* Data access through I/O: directly read/write through File System, read/tamper data transfered via syscall to/from external device, using mmap.
* Code modification attack: modify app's native code directly, load a malicious program file when starting app, transfer control to a malicious handler, link a malicious library.
* Interrupted Program State Attacks: read interrupt state to glean sensitive information, modify the interrupt program state.
* attack through system service: compromise the degree of randomness, return a pointer into the stack via mmap, grant the same lock to two different threads at the same time, introducing data races with unpredictable (and potentially harmful) results.


------
