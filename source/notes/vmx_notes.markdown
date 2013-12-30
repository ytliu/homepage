---
layout: page
title: "VMX Notes"
date: 2013-11-29 19:50
comments: true
sharing: true
footer: true
---

Notes about Intel VMX study from Intel Manual:

------

### CHAPTER 23 INTRODUCTION TO VIRTUAL-MACHINE EXTENSIONS

There is no software-visible bit whose setting indicates whether a logical processor is in VMX non-root operation. This fact may allow a VMM to prevent guest software from determining that it is running in a virtual machine.

A VMM could use a different VMCS for each virtual machine that it supports. For a virtual machine with multiple logical processors (virtual processors), the VMM could use a different VMCS for each virtual processor.

If CPUID.1:ECX.VMX[bit 5] = 1, then VMX operation is supported.

Before executing VMXON, software should allocate a naturally aligned 4-KByte region of memory that a logical processor may use to support VMX operation.1 This region is called the VMXON region.

In VMX operation, processors may fix certain bits in CR0 and CR4 to specific values and not support other values. Any attempt to set one of these bits to an unsupported value while in VMX operation (including VMX root operation) using any of the CLTS, LMSW, or MOV CR instructions causes a general-protection exception. VM entry or VM exit cannot set any of these bits to an unsupported value.

------

### CHAPTER 24 VIRTUAL-MACHINE CONTROL STRUCTURES

A logical processor may maintain a number of VMCSs that are active.

The launch state of a VMCS determines which VM-entry instruction should be used with that VMCS: the VMLAUNCH instruction requires a VMCS whose launch state is “clear”; the VMRESUME instruction requires a VMCS whose launch state is “launched”. 

There are no other ways to modify the launch state of a VMCS (it cannot be modified using VMWRITE) and there is no direct way to discover it (it cannot be read using VMREAD).

![States of VMCS X](http://ytliu.info/images/vmx-notes-1.png "States of VMCS X")

#### FORMAT OF THE VMCS REGION

![Format of the VMCS Region](http://ytliu.info/images/vmx-notes-2.png "Format of the VMCS Region")

The first 4 bytes of the VMCS region contain the VMCS revision identifier at bits 30:0. Processors that maintain VMCS data in different formats use different VMCS revision identifiers.

VMX-abort indicator bits do not control processor operation in any way. A logical processor writes a non-zero value into these bits if a VMX abort occurs. Software may also write into this field.

 VMCS data control VMX non-root operation and the VMX transitions. To ensure proper behavior in VMX operation, software should maintain the VMCS region and related structures in writeback cacheable memory.

#### VMCS Data

* ORGANIZATION

The VMCS data are organized into six logical groups:

• **Guest-state area**: Processor state is saved into the guest-state area on VM exits and loaded from there on VM entries.

• **Host-state area**: Processor state is loaded from the host-state area on VM exits.

• **VM-execution control fields**: These fields control processor behavior in VMX non-root operation. They
determine in part the causes of VM exits.

• **VM-exit control fields**: These fields control VM exits.

• **VM-entry control fields**: These fields control VM entries.

• **VM-exit information fields**: These fields receive information on VM exits and describe the cause and the nature of VM exits. On some processors, these fields are read-only.

* GUEST-STATE AREA

Processor state is loaded from these fields on every VM entry and stored into these fields on every VM exit.

Guest Register State: 

• Control registers CR0, CR3, and CR4

• Debug register DR7; 

• RSP, RIP, and RFLAGS; 

• CS, SS, DS, ES, FS, GS, LDTR, and TR; 

• GDTR and IDTR; 

• some MSRs; 

• register SMBASE.

Guest Non-Register State: 

• **Activity state**： This field identifies the logical processor’s activity state. 0: Active; 1: HLT; 2: Shutdown; 3: Wait-for-SIPI; 

• **Interruptibility state**: features that permit certain events to be blocked for a period of time. This field contains information about such blocking. 0: Blocking by STI; 1: Blocking by MOV SS; 2: Blocking by SMI; 3: Blocking by NMI; 

• **Pending debug exceptions**: IA-32 processors may recognize one or more debug exceptions without immediately delivering them. This field contains information about such exceptions.; 

• **VMCS link pointer**: This field is included for future expansion; 

• **VMX-preemption timer value**; 

• **Page-directory-pointer-table entries**: PDPTEs; 64 bits each, They correspond to the PDPTEs referenced by CR3 when PAE paging is in use; 

• **Guest interrupt status**: It characterizes part of the guest’s virtual-APIC state and does not correspond to any processor or APIC registers.

* HOST-STATE AREA

Processor state is loaded from these fields on every VM exit.

• CR0, CR3, and CR4.

• RSP and RIP.

• Selector fields (16 bits each) for the segment registers CS, SS, DS, ES, FS, GS, and TR. There is no field in the host-state area for the LDTR selector.

• Base-address fields for FS, GS, TR, GDTR, and IDTR.

• Some MSRs

* VM-EXECUTION CONTROL FIELDS

The VM-execution control fields govern VMX non-root operation.

• Pin-Based VM-Execution Controls: governs the handling of asynchronous events (for example: interrupts)

• Processor-Based VM-Execution Controls: govern the handling of synchronous events, mainly those caused by the execution of specific instructions.

• Exception Bitmap: 32-bit field that contains one bit for each exception.

• I/O-Bitmap Addresses: 64-bit physical addresses of I/O bitmaps A and B (each of which are 4 KBytes in size). I/O bitmap A contains one bit for each I/O port in the range 0000H through 7FFFH; I/O bitmap B contains bits for ports in the range 8000H through FFFFH.

• Time-Stamp Counter Offset: 

• Guest/Host Masks and Read Shadows for CR0 and CR4

• 

• 

• 

• 

* VM-EXIT CONTROL FIELDS

* VM-ENTRY CONTROL FIELDS

* VM-EXIT INFORMATION FIELDS

#### VMCS TYPES: ORDINARY AND SHADOW

有两种VMCS，ordinary和shadow，type由shadow-VMCS indicator in the VMCS region (this is the value of bit 31 of the first 4 bytes of the VMCS region)来决定。Shadow VMCS只在1-setting of the “VMCS shadowing” VM-execution control的CPU上被支持。两种VMCS的区别：

* ordinary VMCS可以用于VMEntry，但是shadow不行。
* 如果“VMCS shadowing” VM-execution control is 1，shadow VMCS可以在non-root中用VMREAD和VMWRITE来访问，这个时候shadow VMCS就是当前ordinary VMCS的link pointer指向的那个VMCS。所以如果“VMCS shadowing” VM-execution control is 1，那么在VM entry的时候要保证current VMCS的link pointer指向的那个VMCS是shadow VMCS。
