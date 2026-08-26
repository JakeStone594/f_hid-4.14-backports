# Fix 1 — `f_hidg` lifetime vs `cdev`

**Status: ✅ efficacy measured on hardware, 2026-08-22.**

## Before

On the pre-fix build, a blocked writer or a `select()` waiter on `/dev/hidg0` held across a gadget
re-composition produced a hard reset of the handset: non-secure watchdog bark, reset reason
`DPON`. The walk hit freed slab memory and spun with interrupts masked, so the local timer stopped
and the watchdog pet thread was never scheduled.

## The test

The fix makes an open fd hold the object alive. The natural efficacy test is therefore to do the
thing that used to reset the phone and observe that nothing happens.

1. Park a `select()` waiter on `/dev/hidg0` and **verify it is actually parked** rather than spinning
   or exited — process state `S`, `wchan=do_select`. This check matters: an exited waiter would
   produce a clean result for the wrong reason.
2. Unbind the gadget underneath it.
3. Observe.

## Result

- No reset.
- Taint unchanged at 516 — reported for completeness only; see the method note on why taint is not
  load-bearing here.
- Zero faults in a log ring covering the window.
- The waiter's fd then resolved to **`/dev/hidg0 (deleted)`** — the node gone from `devtmpfs` while
  the object behind it stayed alive. This is the refcount chain visible from userspace, and it is
  the positive signal that distinguishes "the fix worked" from "nothing happened".
- `SIGTERM` afterwards exited cleanly, which is where `close()` finally drives the refcount to zero
  and `hidg_release` runs.

## Why it works

`cdev_device_add` → `cdev_set_parent(cdev, &dev->kobj)` → `cdev_add` takes a reference on the device.
`chrdev_open` → `cdev_get` → `kobject_get_unless_zero` means an open fd holds a reference on the
cdev. `cdev_del` only does `kobject_put` — it does not free. `cdev_default_release`, which finally
drops the device reference, runs only at refcount zero. So while any fd is open, `hidg_free`'s
`put_device()` cannot reach zero and the wait queues inside `struct f_hidg` are not freed.

## A trap encountered while probing

`printf … > /dev/hidg0` opens **blocking**, so with no host attached it sleeps in `f_hidg_write`'s
wait queue indefinitely. Earlier notes claiming `EAGAIN` described the **non-blocking** open only.
Kill such a writer by PID found via `/proc/*/fd` — never by pattern match, which catches the wrong
process.
