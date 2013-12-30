---
layout: page
title: “Out-of-the-box” Monitoring of VM-based High-Interaction Honeypots
date: 2013-09-14 20:50
comments: true
sharing: true
footer: true
---

Software-based VMM Qemu - Binary Translation

honeypot project: http://www.honeynet.org 

VMscope，提供和Sebek相同的语义信息，而且和external monitor（network sniffer）一样安全

Sebek：
- LKM hook system call: sys open, sys read, sys readv, sys pread64, sys write, sys writev, sys pwrite64, sys fork, sys vfork, sys clone, sys socketcall. 

- intercept subsequent invocation, record arguments and context information(UID, PID)

    - send log to remote trusted Sebek server
    "anomalies":
    - modification on the system call table
    - inconsistency in the statistics
    - existence of a hidden Sebek module
    rootkit can re-overwrite those system call entries

    use interpretation code to resolve the semantic gap!

    e.g.: for sys_execve
    - ebx contains process name
    - ecx contains address of argv[]
    - edx contains address of envp[]
    VA -> PA by walking page table

总结：
    
* 感觉就是比较早期的VMI的paper，套了一个HoneyPot的外壳，还用了Qemu这种Binary translation的VMM，使的interception变得更好做了，然后里面提到一些现在用的VMI的技术比如VA到PA的转换之类的，来解决semantic gap的问题。

VM fingerprint detected malware:
    
    - Thwarting Virtual Machine Detection (NDSS'07)
    - Remote Detection of Virtual Machine Monitors with Fuzzy Benchmarking
    - Remote Physical Device Fingerprinting  (S&P'05)
    - Pioneer: Verifying Integrity and Guaranteeing Execution of Code on Legacy Platforms (SOSP'05)


------
