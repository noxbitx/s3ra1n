# Fix_Palera1n.md — USB Instability Fix for palera1n on postmarketOS (GT-I9300)

> If palera1n is working fine and then suddenly the iPhone keeps disconnecting,
> the muic keeps switching connector types, or pmOS bootloops after an unclean
> shutdown, this is almost certainly your fix.

---

## The Fix

Boot into TWRP, open terminal, run:
```bash
rm -rf /cache/*
reboot
```

That's it.

---

## Why This Fixes USB Instability

The `/cache` partition stores temporary system state that persists across reboots, things like USB gadget configuration, udev persistent device rules, and usbmuxd
socket state snapshots.

When postmarketOS crashes, loses power mid-session, or gets a dirty shutdown
(kernel panic, battery pull, i2c register abuse ), these cached states get
written in a half-corrupted condition. On the next boot, the system reads them
back and tries to restore to the last known state, but that state is garbage.

Specifically, what goes wrong with USB:

- **udev** reads a corrupted USB gadget config and initializes the dwc2/ehci
  controllers in a wrong or conflicting state
- **Leftover muic state files** tell the driver that a specific connector type
  was previously attached, so the muic starts from a non-clean baseline
- The muic hardware reads the actual physical state (nothing connected, or OTG
  cable) and gets a **different answer** than what the cached state says
- To reconcile the conflict, the muic triggers its re-detection cycle, and
  keeps triggering it every few seconds because the cached state keeps
  disagreeing with reality
- This is what causes the connector type flip-flopping between type 2 (USB host),
  type 5 (slow charging), and type 13 (not charging), the muic wasn't broken,
  it was just perpetually confused

Clearing cache = blank slate = muic and udev initialize purely from hardware
state = no conflict = stable USB host mode = palera1n works first try.

The muic was never broken. It was arguing with a ghost. 💀

---

## When to Apply This Fix

- pmOS bootloops after unclean shutdown or battery pull
- iPhone keeps disconnecting every 1-4 seconds for no apparent reason
- muic cycles between connector type 2, 5, and 13 in dmesg
- palera1n was working fine before and suddenly stopped
- You've been debugging USB for 4 days and nothing makes sense

---

*Part of the s3ra1n project — https://github.com/noxbitx/s3ra1n*
