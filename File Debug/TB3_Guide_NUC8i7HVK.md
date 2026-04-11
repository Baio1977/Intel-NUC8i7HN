# TB3 Guide for NUC8i7HVK – RP05 / PXSX / Cold Boot / Hotplug

## Goal

Document in a clear and reusable way the debugging path and the final solution adopted for **Thunderbolt 3 on the Intel NUC8i7HVK**, with focus on:

- stable `RP05` population at boot
- the difference between **OEM bring-up** and **PCIe downstream branch stabilization**
- the historical role of `TBON`
- replacing the old `UPV0` check
- controlled use of the OEM `_RMV` method
- the difference between the **cold boot fix** and **real macOS hotplug**
- validation of the **post-fix OEM live hotplug path**
- validation of the **final TINI-integrated readiness check**, including cold boot after AC power removal

---

# Terminal Debugging Commands

- Delete logs

```bash
sudo log erase --all
```

- Dump boot (advanced)

```bash
log show --last boot --predicate 'process == "kernel" AND senderImagePath CONTAINS "AppleACPIPlatform"' --style compact | awk '/ACPI Debug/{getline; getline; print}' | tee ~/Desktop/acpi_debug.txt
```

- Log hotplug live (advanced)

```bash
log stream --info --debug --predicate 'process == "kernel" AND senderImagePath CONTAINS "AppleACPIPlatform"' --style compact | awk '/ACPI Debug/{getline; getline; print; fflush()}' | tee ~/Desktop/acpi_hotplug_live.txt
```

---

# 1. Native OEM baseline

## 1.1 OEM Thunderbolt flow

In the OEM firmware, Thunderbolt 3 bring-up mainly goes through:

```text
PCI0._INI
  -> _GPE.TINI(TBSE, 0)
      -> TBFP
      -> TINO
          -> MMTB
          -> OSUP
      -> (possible notify / hotplug path)
```

### Role of the main OEM methods

- `TBFP`
  - enables **Force Power**
- `TFPS`
  - reads the Force Power state
- `TINO`
  - performs the OEM bring-up path
- `MMTB`
  - rebuilds the Thunderbolt MMIO base
- `OSUP`
  - uses the mailbox for the OS-up / controller handshake
- `XTBT`
  - OEM hotplug handler
- `_E20`
  - GPE path leading to `XTBT`

## 1.2 What RP05._INI does NOT do

The OEM `RP05._INI` method **does not perform Thunderbolt bring-up**.

It mainly initializes a few variables later used by `RP05._DSM`, for example:

- `LTRZ`
- `LMSL`
- `LNSL`
- `OBFZ`

So:

- TB3 controller power-on **does not originate from `RP05._INI`**
- but the `RP05` branch can still be influenced by the context in which `_INI` and `_DSM` are evaluated

---

# 2. Observed problem

## 2.1 Initial symptom

With a pure OEM / incomplete Hackintosh setup:

- the Thunderbolt controller could power up
- but `RP05` / `PXSX` were **not always populated correctly** in `IOService`
- the eGPU / downstream branch were not always visible consistently at boot

## 2.2 Key distinction

During debugging, it became clear that there were **two separate issues**:

### A. TB3 controller bring-up
This involves:
- Force Power
- `TINO`
- `MMTB`
- `OSUP`

### B. Actual population of the downstream PCIe branch
This involves:
- when `PXSX` stops being “empty”
- when the downstream `VDID` changes from `FFFFFFFF` to a valid value
- how macOS evaluates `_RMV`, `_DSM`, and the branch layout

This distinction is fundamental.

---

# 3. Ideas derived from the OSY base

The **OSY / HaC-Mini** base provided several useful references regarding:

- the general TB3 branch structure
- the rename approach
- HPM / I2C methods
- “wait-until-ready” logic
- the importance of real downstream timing, not just controller power-on

## 3.1 What was taken conceptually from OSY

The most useful thing derived from OSY was not a single copied patch, but this principle:

> It is not enough to power on the Thunderbolt controller.  
> You also need to wait until the downstream PCIe branch behind `RP05 / PXSX` is actually ready.

That intuition was confirmed by the logs.

---

# 4. What the logs showed

## 4.1 The OEM bring-up path works

Advanced logs confirmed that the OEM path is healthy:

- `TBFP(1)` changes Force Power from OFF to ON
- `TINO` starts correctly
- `MMTB` builds the Thunderbolt base
- `OSUP` sends the mailbox command
- `Cmd acknowledged`
- `OSUP return = 1`

So the real problem was not:  
**“Thunderbolt does not power on”**

## 4.2 The real readiness check is PXSX.VDID

The old OSY-inspired `TBON` method used `UPV0` as a custom sentinel.

In the final work, it became clear that the truly useful and more native signal is:

```text
_SB.PCI0.RP05.PXSX.VDID
```

### Interpretation

- `VDID == 0xFFFFFFFF`
  - downstream branch not ready / not readable yet
- `VDID != 0xFFFFFFFF`
  - downstream device actually visible / enumerable

This was the main turning point.

## 4.3 What happens before global init

The logs showed that, very early in the boot process:

- `RP05.PXSX` is queried
- `WIST/WGST` enter
- but they still read `VDID = 0xFFFFFFFF`

So the downstream branch is **not ready yet** when macOS first probes the node.

This is important because it clearly separates:

- **initial failed probe**
- from
- **successful OEM bring-up later on**

---

# 5. The historical role of TBON

## 5.1 What TBON does NOT do

`TBON` **is not** the method that brings the TB3 controller up.

It does not replace:

- `TBFP`
- `TINO`
- `MMTB`
- `OSUP`

## 5.2 What TBON actually did

`TBON` worked as a:

- synchronization barrier
- bounded wait
- readiness check for the `RP05 / PXSX` branch

Its real purpose was:

> to wait until the downstream branch behind `RP05.PXSX` became actually available

## 5.3 Why TBON is no longer needed

Later tests showed that the same `PXSX.VDID` readiness check can be moved into the Darwin wrapper around `_GPE.TINI`, and that this approach remains stable even on a **cold boot after AC power removal and with no devices connected**.

At that point, `TBON` was no longer needed. `RP05._INI` could return to full OEM behavior, while readiness enforcement stayed inside `TINI`.

---

# 6. Final solution adopted

## 6.1 Renames used

The final, reduced rename set is:

- `_GPE.TINI -> TINO`
- `RP05.PXSX._RMV -> XRMV`

### Important note

`RP05._INI -> XINI` was used only in an intermediate phase when `TBON()` was still called from `RP05._INI`.

That rename was later removed, because the final solution no longer needs a custom `RP05._INI` wrapper.

## 6.2 Final Darwin wrapper around _GPE.TINI

The final solution keeps the wait logic directly inside the Darwin wrapper around `TINI`:

- `TBFP(One)`
- `Sleep(0x0320)` = 800 ms
- call OEM `TINO(Arg0, Arg1)`
- bounded poll on `PXSX.VDID` until it becomes valid

### Final release version of `TINI`

```asl
Scope (\_GPE)
{
    Method (TINI, 2, Serialized)
    {
        Local0 = Zero
        Local1 = Zero

        If ((_OSI ("Darwin") && (Arg0 == 0x05)))
        {
            \_SB.TBFP (One)
            Sleep (0x0320)
            \_GPE.TINO (Arg0, Arg1)

            Local1 = (Timer + 0x02FAF080)

            While ((Timer < Local1))
            {
                Local0 = \_SB.PCI0.RP05.PXSX.VDID

                If ((Local0 != 0xFFFFFFFF))
                {
                    Break
                }

                Sleep (0x64)
            }
        }
        Else
        {
            \_GPE.TINO (Arg0, Arg1)
        }
    }
}
```

### Why this is now preferred

This solution keeps the readiness check close to the actual OEM bring-up path and avoids keeping a separate wait method inside `RP05._INI`.

## 6.3 RP05._INI restored to OEM

In the final setup, `RP05._INI` is no longer renamed or overridden.

That means:

- no `XINI`
- no custom `_INI`
- no `TBON()` call from `RP05._INI`

`RP05._INI` stays fully OEM, which makes the final setup more natural and more firmware-aligned.

---

# 7. Validation of the TINI-integrated check

## 7.1 Warm / normal boot validation

With the `PXSX.VDID` poll moved into `TINI`, the logs showed:

- normal OEM bring-up (`TBFP -> TINO -> OSUP`)
- then `TINI -> TBON-style check`
- then a valid `VDID`
- then `TINI ready`

So the downstream branch was already ready **before** any custom `RP05._INI` intervention.

## 7.2 Cold boot validation after AC removal

The strongest validation was the following scenario:

- full shutdown
- AC power physically removed
- wait ~10 seconds
- power restored
- boot with **no devices connected**

Even in that case, the logs showed:

- early `WIST/WGST` still seeing `VDID = FFFFFFFF`
- full OEM bring-up completing correctly
- then the integrated readiness poll inside `TINI`
- then `PXSX.VDID` becoming valid
- then `TINI ready`

This demonstrates that the `TINI`-integrated solution is stable even in the harshest tested cold-boot condition.

---

# 8. Using the OEM _RMV method

## 8.1 Goal

The goal was not to fully replace `_RMV`, but to **keep using the OEM method** while making the hotplug path more favorable.

The OEM method uses logic similar to:

```asl
If (((TBTS == One) && (SBNR == TBUS)))
{
    Local0 = TARS
}
Else
{
    Local0 = HPCE
}
```

### Meaning

- if the TB branch is perfectly aligned with the expected bus -> use `TARS`
- otherwise -> use `HPCE`

## 8.2 Choice made

Instead of falsifying:

- `SBNR`
- `TBUS`
- `TBTS`

only this was forced:

- `HPCE = One`

before calling the renamed OEM method `XRMV()`.

### Final `_RMV` wrapper

```asl
Scope (PXSX)
{
    Method (_RMV, 0, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            \_SB.PCI0.RP05.HPCE = One
        }

        Return (\_SB.PCI0.RP05.PXSX.XRMV ())
    }
}
```

### Benefit

- OEM logic is preserved
- real bus numbering is not falsified
- the fallback branch used by macOS in this scenario becomes favorable

---

# 9. Note about SOHP

Inside the OEM `XTBT` method there is this branch:

```asl
If ((SOHP == One))
{
    // TBT SW SMI
}
```

So:

- `SOHP = 1` -> enter the native firmware `TBT SW SMI` path
- `SOHP = 0` -> that branch is bypassed

## Why this matters

This bypass is relevant because it skips part of the native firmware Thunderbolt logic which, in a Hackintosh context, may be unfavorable to the desired macOS behavior.

In practical terms:

- `SOHP=0` does not solve everything by itself
- but it is consistent with a cleaner macOS / eGPU / IOService flow

---

# 10. Post-fix OEM live hotplug

## 10.1 Path observed in the live log

After the fix, the live hotplug log clearly showed the full OEM path:

```text
_E20
 -> XTBT
    -> RLTR
    -> WWAK
    -> WSUB
    -> GNIS
    -> PGWA
    -> TBFF
    -> NTFY (Notify RP05)
    -> NFYG
```

This confirms that real hotplug starts from **GPE `_E20`** and goes through the OEM `XTBT` handler.

## 10.2 What happens inside XTBT

The live log shows that during hotplug:

- `CF2T = 0`
- `TRDO = 0`
- `TRD3 = 0`
- `SOHP = 0`
- `OHPN = 1`
- `GHPN = 1`

Therefore:

- the path is not blocked by the OEM guards `TRDO/TRD3`
- the `TBT SW SMI` branch is bypassed
- both local and global hotplug notifications remain enabled

## 10.3 GNIS and HPFI

During hotplug, `GNIS`:

- sees a real hotplug event
- reads `HPFI = 1`
- consumes / clears it
- then returns `0`

This proves that the hotplug event is not “simulated” by macOS; it is actually seen and handled by firmware.

## 10.4 TBFF confirms real device presence

Inside `TBFF`, the log confirms:

- `VEDI = 0x15D38086`
- `CMDR = 0x00100007`
- `TWIN = 0`
- `Dev Present`

So during OEM hotplug the downstream device is genuinely present and readable.

## 10.5 Final notify

The OEM hotplug path ends with:

- `NTFY`
- `Notify RP05`
- `NFYG`

This is important because it confirms that the native notify path remains active and working.

---

# 11. Cold boot fix vs real hotplug

## 11.1 Cold boot / population fix
The final ACPI patch solves:

- stable controller bring-up
- downstream readiness behind `PXSX`
- stable `RP05` population at boot

## 11.2 Real hotplug

It was observed that full Thunderbolt hotplug on macOS **does not depend on ACPI alone**.

In particular:

- the ACPI patch solves the boot path
- the correct property under `PXSX` is still important for real macOS hotplug / AppleThunderbolt / IOService behavior

### Conclusion

ACPI and DeviceProperties are not doing the same job:

- **ACPI** -> prepares / stabilizes the branch
- **Device Property** -> enables proper hotplug behavior on the macOS side

---

# 12. Final release reference version

```asl
External (_SB.TBFP, MethodObj, 1)
External (_GPE.TINO, MethodObj, 2)
External (_SB.PCI0.RP05.HPCE, IntObj)
External (_SB.PCI0.RP05.PXSX.VDID, IntObj)
External (_SB.PCI0.RP05.PXSX.XRMV, MethodObj, 0)

Scope (\_GPE)
{
    Method (TINI, 2, Serialized)
    {
        Local0 = Zero
        Local1 = Zero

        If ((_OSI ("Darwin") && (Arg0 == 0x05)))
        {
            \_SB.TBFP (One)
            Sleep (0x0320)
            \_GPE.TINO (Arg0, Arg1)

            Local1 = (Timer + 0x02FAF080)

            While ((Timer < Local1))
            {
                Local0 = \_SB.PCI0.RP05.PXSX.VDID

                If ((Local0 != 0xFFFFFFFF))
                {
                    Break
                }

                Sleep (0x64)
            }
        }
        Else
        {
            \_GPE.TINO (Arg0, Arg1)
        }
    }
}

Scope (\_SB.PCI0.RP05.PXSX)
{
    Method (_RMV, 0, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            \_SB.PCI0.RP05.HPCE = One
        }

        Return (\_SB.PCI0.RP05.PXSX.XRMV ())
    }
}
```

---

# 13. Final conclusion

## Pure OEM
- the TB3 controller may initialize
- but `RP05 / PXSX` are not always stable or ready when needed

## Final approach

The effective solution is now a combination of:

- correct OEM bring-up ordering
- a Darwin wrapper around `TINI`
- real downstream readiness waiting through `PXSX.VDID` directly inside `TINI`
- full OEM `RP05._INI` with no rename and no override
- using the OEM `_RMV` method with `HPCE=1` in the fallback path
- validation of the live OEM hotplug path `_E20 -> XTBT -> ... -> Notify RP05`
- validation of the same boot strategy under cold boot after AC removal

## Key point

The real key was not simply “powering on Thunderbolt”, but:

> synchronizing `RP05` with the moment when the downstream PCIe branch actually becomes visible.

---

# 14. Credits

This solution is the result of:

- analysis of the OEM DSDT from the NUC8i7HVK
- repeated ADBG log-based testing
- comparison between the OEM path and actual macOS behavior
- conceptual ideas derived from the **OSY / HaC-Mini** base and its wait / synchronization approach

The OSY base provided useful guidance for the overall direction of the work, while the final patch was specifically adapted to the observed behavior of this NUC.

# P.S.
All this was made possible thanks to ChatGPT, my stubborn belief that not all OSY code was strictly necessary, and a bit of ACPI knowledge accumulated over time.
