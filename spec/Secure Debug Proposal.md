# RISCV Debug Proposal Draft

# Arch Opens

❗Debug spec requires halt request must be served within one second, while secure debug may need to keep halt request pending for longer than 1 sec. Need to resolve this.
 
❗Add per command/request control and define its granularity.


# The Proposal

## Problem Statement

[https://docs.google.com/presentation/d/1MBg_zuUHW1GM1VC8hWAxN_Hs4VLzRb24/edit?usp=sharing&ouid=101725320140562793201&rtpof=true&sd=true](https://docs.google.com/presentation/d/1MBg_zuUHW1GM1VC8hWAxN_Hs4VLzRb24/edit?usp=sharing&ouid=101725320140562793201&rtpof=true&sd=true)

## DM Changes

### Non-debug-module reset 

ndmreset is a feature to enable DM to reset the whole system except DM. This may become an attack surface if not carefully designed and analyzed in threat model analysis. 

One potential example is that if SOC boot rom usually interacts with external devices such as SPI or flash, if an OEM adversary can use ndmreset to reset the entire SOC when SOC boot rom is running, something bad may happen.

We recommend to SOC vendors to not implement ndmreset, or use a life-cycle fuse to disable ndmreset. 

### Debugger access memory

Debug access memory shall always be checked by IOPMP, WG, or other equivalent hardware protection mechanism.

### Error reporting

- Add a new cause in cmderr (6, previously reserved) to indicate a security fault. 
- Add a new cuase in sberr (6, previously reserved) to indicate a security fault in Debug Module's system bus. 

Abstract commands, halt reqeusts or reset request that violates debug security protections will get the cmderr 6 (secrity fault). 

Exceptions due to privilege violation (e.g. accessing M mode resource with S mode privilege) will get cmderr 3 (exception). 

When the system bus access is denied by IOPMP, WG, etc., the sberr is set to 6 (secrity fault).

## Core changes

### Machine Debug Security Control and Status Register **(mdbgsec)**

This register is WARL in M mode and RO in debug mode and external debugger 
****
Bit0~3 of mdbgsec (dbgprv) is used to control the maximum privilege level of external debugger and programming buffer.

The encoding of dbgprv is shown below, note that v bit and prv bit follows the same encoding as dcsr.v and dcsr.prv, to simplify hardware implementation

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
| Yes | 1 | 1 | 1 | VS mode external debug enabled |


> 💡 The default value of mdbgsec.dbgprv is implementation specific. Reset to reset to 0 (all modes debug disabled) and FW or ROM code to enable debug is most secured, but that means debuggability is lost before debug is enabled and halt-after-reset is also useless. A good practice to solve is problem is to use a lifecycle fuse to control the the default value of mdbgsec.dbgprv via an input port to the hart so developers can still debug ROM code in development phase and disable afterwards

❓Is there a need to allow S/H mode to enable/disable external debug for less privilege modes? NV’s current preference is no (simplify spec, reduce attack surface)


Below behaviors will be changed with debug security extension

- Writing dcsr.prv with a value that corresponding privilege mode debug is disabled will return a security fault error
- hartreset/resethaltreq will get security fault error from selected hart if M debug is disabled
- setkeepalive will get security fault error from selected hart if M debug is disabled
- Abstract cmd access to memory will be checked as if it is in dcsr.prv, illegal access will get security fault error from selected hart
- Programming buffer access to memory and registers will work as if the it is running at privilege mode specified in dcsr.prv
- Ecall or exception or interrupt… that lands on higher privilege mode but debug is DISABLED will exit debug mode, continue execution in higher privilege mode, and re-enter debug mode immediately after return
- Halt request changes to below.
    - If debug is disabled in all modes, halt request will return security fault error
    - if debug is enabled in any modes, halt request will be pending till the hart can be halted


bit4 of mdbgsec (relaxprivdis) controls whether external debugger can bypass M mode protection by setting abstractcs.relaxedpriv. 1 means relaxpriv cmd is prohibited even if dbgpriv is M. This bit is sticky.


> 💡 Rationale of relaxprivdis - relaxpriv maybe useful in debugging ROM and M mode FW but shall be disabled when debugging runtime software such as OS. In that case we need a new nob here to represent the case where M-mode debug is ON but relaxedpriv is prohibited.

ROM should make sure relaxprivdis is disabled when setting PMP.L to 1


Bit5 of mdbgsec (extrigen) controls whether the external triggers are enabled. Setting up triggers of type 7 takes no effect if the mdbgsec.extrigen is 0.

Bit8-11 of mdbgsec (tracectl) controls RISC-V trace 

| H ext. supported | en (bit3) | v (bit2) | prv (bit0-1) | Trace enabled |
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

### Core Debug Register

Core debug registers are still accessible in debug mode regardless of debug privilege mode control, with restrictions listed below

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
 
Trigger programing registers are still accessible in debug mode regardless to debug privilege mode control.

| Always allowed in debug mode | Access wtih debug privilege  |
| ---------------------------- | -----------------------------|
| tselect(0x7a0)               | tcontrol(0x7a5)              |
| tdata1(0x7a1)                | scontext(0x5a8)              |
| tdata2(0x7a2)                | hcontext(0x6a8)              |
| tdata3(0x7a3)                | mcontext(0x7a8)              |
| tinfo(0x7a4)                 | mscontext(0x7aa)             |

The trigger module behaves differently to on one hand leverage the trigger in debug mode, on the other hand protect the secure code from the external debugger. 

- Triggers with action = 1 (enter debug mode) will only fire if external debug is enabled for corresponding privilege mode
- The trigger to start/stop/notify trace module succeeds only if the hart running privilege mode is enable for trace in mdbgctl.tracectl 
- In debug mode, triggers could not be enabled for privileges higher than the one specified in mdbgsec.dbgprv.
- If the trigger is already enabled for higher privilege mode (by hart) than the ones allowed in mdbgsec.dbgprv/mdbgsec.tracectl, it could not be altered in debug mode and tdata1/2/3 return 0x0 in a read access.

> 💡 Rationale: There might be the cases, though rare, that native debugger set trigger in high privilege mode while external debugger is restriced to reletive low privilege mode accessibility. We would like to avoid that external debugger corrupts the trigger setting in high privilege mode of native debbugger, which makes them othogonal.


- If a trigger chain is set up (by hart), the trigger in the chain can only be modified by M mode debug privilege. 

>💡 Rationale: The restriction is a trade-off between usablity and hardware complexity. The integrity of the trigger chain set by hart need to be assured when external debugger is going to leverage triggers. There might be the case that the triggers are chained up across privilege modes (e.g. from S mode to M mode), while the external debugger could only aquire S mode. The external debugger should not twist the chain since it might slience or mis-fire the breakpoint exception in M mode. However, it is hard for implementation the check the whole chain when configuring trigger in a chian. To simply it, only M mode debug privilge can mofify triggers in a chian. 

tdata1.m/s/u/vs/vu enables triggers for privilege modes 
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

>💡 The external debugger could determine the accessiblity of the CSR fields by write a spefic value and read back it. If the value matches, the field is configurable for external debugger 