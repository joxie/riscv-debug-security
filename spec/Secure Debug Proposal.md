# RISCV Debug Proposal Draft

# Arch Opens

❗Debug spec requires halt request must be served within one second, while secure debug may need to keep halt request pending for longer than 1 sec. Need to resolve this.
 
❗Add per command/request control and define its granularity.

❗ndmreset could be useful when the hart is required to be halted precisely at the entry of code to be debugged.


# The Proposal

## Problem Statement

[https://docs.google.com/presentation/d/1MBg_zuUHW1GM1VC8hWAxN_Hs4VLzRb24/edit?usp=sharing&ouid=101725320140562793201&rtpof=true&sd=true](https://docs.google.com/presentation/d/1MBg_zuUHW1GM1VC8hWAxN_Hs4VLzRb24/edit?usp=sharing&ouid=101725320140562793201&rtpof=true&sd=true)

## Requirements

- The debug accesses should be regulated according to the privilege level. 
- Less privileged debug access cannot tamper resources belongs to more privileged level (e.g. S mode debug privilege level to access M mode CSR).
- Less privileged debug access cannot peep/disturb the hart when it runs in higher privilege level (e.g. S mode debug privilege cannot witness/affect the trap handling in M mode).
- The ability to lock down debug accesses for ROM and enables it for Non-ROM code 


## DM Changes

### Non-debug-module reset 

ndmreset is a feature to reset the whole system except DM. This may become an attack surface if it is not carefully designed and analyzed in threat model. 

A example is that OEM adversaries can use ndmreset to reset the entire SOC when SOC boot rom is running. SOC boot rom usually interacts with external devices such as SPI, flash and etc. If the boot rom is interrupted and restarted in the middle of execution, it might lead to severe result.

It is recommended that SOC vendors do not implement ndmreset, or use a life-cycle fuse to disable ndmreset. 

### Debugger access memory

Debugger accessing memory via System Bus Access block shall always be checked by IOPMP, WG, or other equivalent hardware protection mechanism.

### Error reporting

- Add a new cause in cmderr (6, previously reserved) to indicate a security fault. 
- Add a new cause in sberror (6, previously reserved) to indicate a security fault in DM system bus. 

Abstract commands, halt reqeusts or reset request that violate debug security protections will set cmderr to 6 (security fault). 

Exceptions due to privilege violation (e.g. accessing M mode resource with S mode privilege) will set cmderr to 3 (exception). 

When the System Bus Access is denied by IOPMP, WG, etc., the sberror is set to 6 (security fault).

## Core changes

### Machine Debug Security Control and Status Register **mdbgsec**

This register is WARL in M mode and RO in debug mode for all privilege levels.

Bit 0~3 of mdbgsec is used to control the maximum privilege level in debug mode which applies abstract commands and program buffer.

> 💡 The hart may be halted in a privilege level which is not the maximum level in debug mode. The CSR is made RO in debug mode for debugger in any privilege levels to determine its maximum privilege level. 

The encoding of dbgprv is shown below. Note that dbgv bit and dbgprv bits follow the same encoding as dcsr.v and dcsr.prv to simplify hardware implementation

| H ext. supported | dbgen (bit3) | dbgv (bit2) | dbgprv (bit0-1) | Max debug prv mode |
| --- | --- | --- | --- | --- |
| No | 0 | 0 | don’t-care | External debug disabled |
| No | 1 | 0 | 0 | U mode external debug enabled |
| No | 1 | 0 | 1 | U/S mode external debug enabled |
| No | 1 | 0 | 3 | U/S/M mode external debug enabled  |
| Yes | 0 | dont’ care | don’t care | External debug disabled |
| Yes | 1 | 0 | 0 | U mode external debug enabled |
| Yes | 1 | 0 | 1 | U/VU/VS/HS mode external debug enabled  |
| Yes | 1 | 0 | 3 | U/VU/VS/HS/M mode external debug enabled  |
| Yes | 1 | 1 | 0 | VU mode external debug enabled  |
| Yes | 1 | 1 | 1 | VS/VU mode external debug enabled |


> 💡 The default value of mdbgsec.dbgprv/mdbgsec.dbgen is implementation specific. Resetting it to 0 (all privilege levels disabled for debug) and letting FW or ROM code enable debug is the most secured way. But it means debugability is lost until it is enabled and halt-after-reset is also useless. A good practice to solve this problem is to use a life-cycle fuse that controls the default value of mdbgsec.dbgprv and mdbgsec.dbgen via an input port to the hart. Developers can debug ROM code in development phase and disable it afterwards.

❓Is there a need to allow S/H mode to enable/disable external debug for less privilege levels? NV’s current preference is no (simplify spec, reduce attack surface)


The following behaviors will be changed with debug security extension

- Writing dcsr.prv/dcsr.v with a value whose corresponding privilege level is disabled for debug will set cmderr to 6 (security fault error).
- hartreset/resethaltreq will get security fault error from selected hart if M mode debug is disabled.
- setkeepalive will get security fault error from selected hart if M mode debug is disabled.
- Abstract commands accesses to memory and registers will be checked as if it is in privilege specified by dcsr.prv/dcsr.v. Exceptions will set cmderr to 3 (exception).
- Programming buffer accesses to memory and registers will work as if the it is running at privilege level specified in dcsr.prv/dcsr.v. Exceptions will set cmderr to 3 (exception).
- Ecall, exceptions or interrupts that land on higher privilege level but disallowed for debug will exit debug mode, continue execution in higher privilege level, and re-enter debug mode immediately after mret/sret.
- Halt request behaviors changes as the following
    - If debug is disabled in all modes, halt request will return security fault error
    - if debug is enabled in any modes, halt request will be pending till the hart can be halted
   
> 💡 The debugger could discover the debugability of the hart by issuing halt request and checking the error status. If the halt request responses with an error, it means the hart is not yet granted for debug. Otherwise, the halt request succeeds immediately or will be pending till the hart enter a debugable privilege level. Since the halt request is asynchronous, the hart must be put in an infinite loop at the entry of code to be debugged. The debugger could afterwards halt the hart precisely and jump out of the loop in debug mode.

Bit 4 of mdbgsec (relaxprivdis) controls whether external debugger can bypass M mode protection by setting abstractcs.relaxedpriv. Value 1 means relaxpriv is prohibited to be set even if the debug privilege level is 3 (M mode). This bit is sticky.


> 💡 relaxpriv may be useful when debugging ROM and M mode FW but it shall be disabled when debugging runtime software such as OS. In this case, we need a new knob to achieve the configuration that M mode debug is enabled but relaxedpriv is prohibited. ROM should make sure relaxprivdis is set to 1 when a PMP entry is locked by setting pmpcfg*.L to 1. 


Bit 5 of mdbgsec (extrigen) controls whether the external triggers of Trigger Module are enabled. Setting up triggers of type 7 takes no effect if the mdbgsec.extrigen is 0.

Bits 8-11 of mdbgsec controls RISC-V trace 

| H ext. supported | trcen (bit3) | trcv (bit2) | trcprv (bit0-1) | Trace enabled |
| --- | --- | --- | --- | --- |
| No | 0 | 0 | don’t-care | Trace disabled |
| No | 1 | 0 | 0 | U mode trace enabled |
| No | 1 | 0 | 1 | U/S mode trace enabled |
| No | 1 | 0 | 3 | U/S/M mode trace enabled |
| Yes | 0 | dont’ care | don’t care | Trace disabled |
| Yes | 1 | 0 | 0 | U mode trace enabled |
| Yes | 1 | 0 | 1 | U/VU/VS/HS mode trace enabled |
| Yes | 1 | 0 | 3 | U/VU/VS/HS/M mode trace enabled |
| Yes | 1 | 1 | 0 | VU mode trace enabled |
| Yes | 1 | 1 | 1 | VU/VS mode trace enabled |

> 💡 Similar to mdbgsec.dbgprv/mdbgsec.dbgen, the default value of mdbgsec.trcprv/mdbgsec.trcen is implementation specific.

### Core Debug Register

Core debug registers are still accessible in debug mode regardless of debug privilege level, with restrictions listed below

| Field | Change |
| --- | --- |
| ebreakvs | Requires VS mode debug and above |
| ebreakvu | Requires VU mode debug and above |
| ebreakm | Requires M mode debug  |
| ebeaks | Requires S mode debug |
| ebreaku | Requires U mode debug |
| stepie | Requires M mode debug |
| stopcount  | Requires mdbgsec.dbgen == 1 |
| stoptime | Requires M mode debug |
| mprven | Requires M mode debug |
| nmip | Requires M mode debug |
| v, prv | Cannot exceed privilege defined in mdbgsec |


### Trigger
 
A subset of trigger programing registers are still accessible in debug mode regardless to debug privilege level control.

| Always allowed in debug mode | Access wtih debug privilege  |
| ---------------------------- | -----------------------------|
| tselect(0x7a0)               | tcontrol(0x7a5)              |
| tdata1(0x7a1)                | scontext(0x5a8)              |
| tdata2(0x7a2)                | hcontext(0x6a8)              |
| tdata3(0x7a3)                | mcontext(0x7a8)              |
| tinfo(0x7a4)                 | mscontext(0x7aa)             |

The trigger module behaves differently with debug security extension as the following

- Triggers with action = 1 (enter debug mode) will only fire if debug is enabled for corresponding privilege level.
- The trigger to start/stop/notify trace module succeeds only if the privilege level of the hart is enable for trace in mdbgctl.trcprv/mdbgctl.trcv.
- In debug mode, triggers cannot be enabled for privilege levels higher than the one specified in mdbgsec.dbgprv/mdbgsec.dbgv.
- If the trigger is already enabled for higher privilege level (by hart) than the ones allowed in mdbgsec, it could not be altered in debug mode and tdata1/2/3 return 0x0 in read accesses.

> 💡 There might be the cases, though rare, that native debugger and external debugger co-exists. The native debugger might set a trigger in high privilege level while external debugger is restricted to relative low privilege level accessibility. This rule avoid that the external debugger corrupts the trigger setting in high privilege level of native debugger. Both debuggers should be orthogonal.


- The trigger chain should be protected properly by either of them
	- The privilege level of the trigger chain is determined by the highest privilege level among the chain. The chain could only be modified if debug is enable for this privilege level.
	- Triggers in a chain can only be modified by privilege level 3 when in debug mode.  

>💡 This is a trade-off between usability and hardware complexity. The integrity of the trigger chain set by hart need to be assured when external debugger is going to leverage triggers. There might be the case that the triggers are chained up across privilege levels (e.g. from S mode to M mode), while the external debugger could only acquire S mode privilege. The external debugger should not twist the chain since it might silence or mis-fire the breakpoint exception in M mode. However, it is hard for implementation the check the whole chain when configuring trigger in a chain. To simply it, only M mode debug privilege can modify triggers in a chain. 

tdata1.m/s/u/vs/vu accessibility in different privilege levels 
---
| dbgen (bit3) | dbgv (bit2) | dbgprv (bit0-1) | configurable fields from debug mode |
| ------------ | ----------- | --------------- | ----------------------------------- |
| 0            | 0           | don't care      | N/A                                 |
| 1            | 0           | 0               | U                                   |
| 1            | 0           | 1               | S+U+VS+VU                           |
| 1            | 0           | 3               | M+S+U+VS+VU                         |
| 1            | 1           | 0               | VU                                  |
| 1            | 1           | 1               | VS+VU          

Trigger to start/stop/notify trace module follows similar rule by honoring mdbgsec.tracectl

tdata3(textra32,textra64) for extra configuration 
---
| dbgen (bit3) | dbgv (bit2) | dbgprv (bit0-1) | configurable fields from debug mode |
| ------------ | ----------- | --------------- | ----------------------------------- |
| 0            | 0           | don't care      | N/A                                 |
| 1            | 0           | 0               | N/A                                 |
| 1            | 0           | 1               | sbytemask+svalue+sselect            |
| 1            | 0           | 3               | mhselect+sbytemask+svalue+sselect   |
| 1            | 1           | 0               | N/A                                 |
| 1            | 1           | 1               | N/A      

❗ This feature is optional and the field mhselect mixes the privilege control of M and H

>💡 The CSR field accessibility is associated with privilege levels. The external debugger could determine the accessibility of the CSR fields by writing a specific value and reading it back. If the value matches, the field is configurable for external debugger 
