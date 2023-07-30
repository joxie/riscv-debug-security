# RISCV Debug Proposal Draft

# Arch Opens

❗Debug spec requires halt request must be served within one second, while secure debug may need to keep halt request pending for longer than 1 sec. Need to resolve this.
 
❗Add per operation control and define its granularity.

❗How to efficiently protect ndmreset.

❗The implementation can optionally add life-cycle fuses to disable debug according to WID

❗Add shadow CSRs for trigger and dcsr, other than expose the existing CSR as WLRL. 

# The Proposal

## Problem Statement

[Problem statement slides](RISCV_Debug_Security_0613.pptx)

## Requirements
- The debug accesses except the ones from System Bus Block should be regulated according to the privilege levels (assigning a privilege level to debug access). 
- Less privileged debug accesses cannot peep/interrupt the hart when it runs in higher privilege level (e.g., S mode debug privilege cannot read/halt the trap handler or context switch in M mode).
- Less privileged debug accesses cannot tamper resources belongs to more privileged level (e.g., S mode debug privilege level to access M mode CSR or memory granted to M mode by PMP).
- The debug access can be conditional enabled for the same privilege level. (e.g., both ROM and Non-ROM can live in M mode, but the debugability should be granted differently).
- Memory accesses from System Bus Block shall be regulated by IOPMP or something equivalent.

## Core changes

### Machine Debug Security Control Register (mdbgsec)

This CSR is WARL in M mode and RO in debug mode for all privilege levels.

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


> 💡 The default value of mdbgsec.dbgprv/mdbgsec.dbgen is implementation specific. Resetting it to 0 (all privilege levels disabled for debug) and letting FW or ROM code enable debug is the most secured way. But it means debugability is lost until it is enabled and halt-after-reset is also useless. A good practice to solve this problem is to use a life-cycle fuse that controls the default value of mdbgsec.dbgprv and mdbgsec.dbgen via an input port to the hart. Developers can debug ROM code in development phase and disable it afterwards. Moreover, when the WG extension is adopted, another set of fuses can be implemented to disable debug according to WID. For example, if the fuse for WID n is burnt, the mdbgsec.dbgen is switched to fixed value 0x0 when WID is set to n. As another WID is assigned, the mdbgsec.dbgen will be switched back.

> 💡 The M mode is responsible to manage its own debugability since it is the most privileged mode. The S mode debugability is granted by M mode. The secure monitor in M mode could enforce different policies for each S mode context during context switch, which constrains debug accesses within the debuggable partition. 

❓Is there a need to allow S/H mode to enable/disable external debug for less privilege levels? NV’s current preference is no (simplify spec, reduce attack surface)

The following behaviors will be changed with debug security extension

- Halt request behaviors changes as the following
    - If debug is disabled in all modes, halt request will return security fault error （cmderr set to 6)
    - if debug is enabled in any modes, halt request will be pending till the hart can be halted
- Abstract commands accesses to memory and registers will be checked as if it is in privilege specified by dcsr.prv/dcsr.v. Exceptions will set cmderr to 3 (exception).
- Programming buffer accesses to memory and registers will work as if the it is running at privilege level specified in dcsr.prv/dcsr.v. Exceptions will set cmderr to 3 (exception).
- Writing dcsr.prv/dcsr.v with a value whose corresponding privilege level is disabled for debug will get security fault error.
- hartreset/resethaltreq will get security fault error from selected hart if M mode debug is disabled.
- setkeepalive will get security fault error from selected hart if M mode debug is disabled.
- If exceptions or interrupts occurs during single stepping that lands on higher privilege level with debug disabled, for example, S mode (debug enabled) trap into M mode (debug disabled), the hart will continue execution in higher privilege level, and re-enter debug mode immediately after returning debuggable privilege level.

> 💡 The halt group will halt all harts in the same group if anyone halts. There might be the case that different harts have varied debug control policies and some harts may not run in debuggable privilege levels. The halt requests issued by halt group will be pending till the hart enter a debuggable privilege level. 
   
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

> 💡 Similar to mdbgsec.dbgprv/mdbgsec.dbgen, the value of mdbgsec.trcprv/mdbgsec.trcen could be enforced by hardware fusing. 
 
Bits 4-17 of mdbgsec:

| Field          | Bit | Description                                                                                                          | Access         | Reset |
| -------------- | --- | -------------------------------------------------------------------------------------------------------------------- | -------------- | ----- |
| trclock        | 7   | 0: trcen, trcv, trcrprv are allowed to program <br> 1: trcen, trcv, trcprv values are locked and no more programable | Write 1 sticky | 0     |
| dbglock        | 6   | 0: dbgen, dbgv, dbgprv are allowed to program <br> 1: dbgen, dbgv, dgbprv values are locked and no more programable  | Write 1 sticky | 0     |
| extrigdis      | 5   | 0: external triggers are enabled <br>1: external triggers are disabled                                               | R/W            | 0     |
| relaxedprivdis | 4   | 0: relaxedpriv performs relaxed set of permission checks <br>1: relaxedpriv takes no effect                          | R/W            | 0     |

> 💡 relaxedpriv may be useful when debugging ROM and M mode FW but it shall be disabled when debugging runtime software such as OS. In this case, we need a new knob to achieve the configuration that M mode debug is enabled but relaxedpriv is prohibited. ROM should make sure relaxedprivdis is set to 1 when a PMP entry is locked by setting pmpcfg*.L to 1. 

### Debug Control and Status (dcsr)

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

### Machine Debug Operation Control Register (MDBGOPCTL)

> TBD

## DM Changes

### Debug Module Status (dmstatus)

| Field      | Bit | Description                                                                                               | Access | Reset |
| ---------- | --- | --------------------------------------------------------------------------------------------------------- | ------ | ----- |
| allsecured | 21  | The field is 1 when all currently selected hart are secured and not running in debuggable privilege levels | R/W    | 0     |
| anysecured | 20  | The field is 1 when any currently selected hart is secured and not running in debuggable privilege levels  | R/W    | 0     |

> 💡 The debugger could discover the debugability of the hart by allsecured/anysecured.

### Non-debug-module reset 

ndmreset is a feature to reset the whole system except DM. This may become an attack surface if it is not carefully designed and analyzed in threat model. 

A example is that OEM adversaries can use ndmreset to reset the entire SOC when SOC boot rom is running. SOC boot rom usually interacts with external devices such as SPI, flash and etc. If the boot rom is interrupted and restarted in the middle of execution, it might lead to severe result.

It is recommended that SOC vendors do not implement ndmreset, or use a life-cycle fuse to disable ndmreset. 

### Debugger memory access

Debugger accessing memory via System Bus Access block shall always be checked by IOPMP, WG, or other equivalent hardware protection mechanism.

### Error reporting

- Add a new cause in cmderr (6, previously reserved) to indicate a security fault. 
- Add a new cause in sberror (6, previously reserved) to indicate a security fault in DM system bus. 

Abstract commands, halt requests or reset request that violate debug security protections will set cmderr to 6 (security fault). 

Exceptions due to privilege violation (e.g., accessing M mode resource with S mode privilege) will set cmderr to 3 (exception). 

When the System Bus Access is denied by IOPMP, WG, etc., the sberror is set to 6 (security fault).

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
