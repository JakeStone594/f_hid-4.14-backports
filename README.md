# f_hid backports for Linux 4.14.190 (SM-A525F)

Three fixes to `drivers/usb/gadget/function/f_hid.c`, carried by hand to a 4.14.190 Android vendor
kernel and verified on the device they run on.

**4.14 is end-of-life.** No stable backport exists for any of these, and none is coming. That is the
whole reason this repository exists: the device is mine, and one of these defects was resetting it.

To be precise about who maintains what: the NetHunter tree this is built from
([`AzeAstro/samsung_kernel_sm7125`](https://github.com/AzeAstro/samsung_kernel_sm7125), branch
`OneUI-6`) *is* actively maintained, and that work is not in question here. But it is device and
NetHunter enablement — the base stays pinned at Samsung's 4.14.190 vendor drop. Nobody is carrying
4.14.y stable security fixes down into it, and Samsung is not shipping them either. That gap is
what these three patches sit in.

## What I did, and what I did not do

I did **not** discover these vulnerabilities. All three are published, with upstream patches written
by other people, credited below. What I did:

1. Established that this tree is in the affected range.
2. Carried each fix to a codebase with no vendor support, where the upstream patch does not apply
   cleanly and the surrounding primitives differ.
3. Verified each one **by effect on the running hardware** where the hardware can observe it — and
   said so plainly where it cannot.

Point 3 is the part I would ask a reader to weigh. Two of these are confirmed by measurement against
a baseline. The third is not, and cannot be, on this device. That distinction is kept throughout
rather than smoothed over.

## The three fixes

| # | Patch | Upstream author | Status on hardware |
|---|---|---|---|
| 1 | `f_hidg` lifetime vs `cdev` (use-after-free) | John Keeping, 2022-11-22 | ✅ efficacy measured |
| 2 | CVE-2026-31721 — object-lifetime state re-initialised in `hidg_bind` | Michael Zimmermann, 2026-03-31 | ✅ efficacy measured |
| 3 | CVE-2026-31606 — `cdev_init` on an in-use `cdev` | Michael Zimmermann (`81ebd43cc0d6d`), 2026-03-27; `-ENOMEM` follow-up by Ethan Tidmore, 2026-04-02 | ⛔ **not observable on this device** |

### 1 — `f_hidg` lifetime vs `cdev`

`f_hidg_open` stored the `f_hidg *` and took no reference. `f_hidg_poll` registered poll waiters on
wait queues embedded in the `kzalloc`'d `f_hidg`. `hidg_unbind` called `cdev_del()`, which does not
invalidate open fds, and `hidg_free` then `kfree`'d the object. Nothing refcounted it against open
files — a whole-file grep for `refcount|kref|atomic_|open_count` found only `opts->lock`.

So a `select()`/`poll()` loop on `/dev/hidg0` that survived Android re-composing the gadget would
call `remove_wait_queue()` on freed slab, take `spin_lock_irqsave` on a garbage ticket word, and spin
with IRQs masked forever. The local timer stops, the watchdog pet thread is never woken, and the
non-secure watchdog barks ~11 s later. In practice: the handset hard-resets. Android's
`UsbDeviceManager` re-composes on its own — measured twice at ~550 ms — with no user action, so this
needed no exotic trigger.

Upstream fix: *"usb: gadget: f_hid: fix f_hidg lifetime vs cdev"*, John Keeping, 2022-11-22,
`Fixes: 71adf1189469`. The backport is clean — every primitive exists at 4.14.190 — except the
patch's `kfree(hidg->set_report_buf)`, a field this version lacks, which is dropped.

### 2 — CVE-2026-31721

CVSS 3.1 **5.5**, vector `AV:L/AC:L/PR:L/UI:N/S:U/C:N/I:N/A:H`. `hidg_bind` re-initialised state
belonging to the object's lifetime, not the bind's: two spinlocks, two wait queue heads, and a list
head, none of which appear in `hidg_alloc`. `init_waitqueue_head` resets the list head to
self-pointing, so a poll/select waiter registered before a rebind is silently orphaned, and its
`poll_freewait` → `remove_wait_queue` then finds `prev->next` wrong.

Upstream moves nine initialisations out of `hidg_bind`; only five of them exist at 4.14.190.

⚠ **Several vendor advisories title this "privilege escalation". The vector is `C:N/I:N/A:H` —
availability only, i.e. a local denial of service.** I flag it because the scarier reading would have
justified a priority the vector does not support, and nobody asks a vendor page to prove itself.

This patch also goes one step past upstream. Moving the initialisations out of `hidg_bind` leaves two
unlocked stores to lock-protected fields, masked today only because the lock was being
re-initialised immediately above them. Once the lock is persistent the race is real: a writer blocked
in `f_hidg_write`'s wait loop, on an fd that survived the unbind, can take the lock, pass the check
and read `hidg->req` while bind stores NULL outside it — the re-check comes after the dereference.
So the patch takes `write_spinlock` around both stores.

The general form is worth stating: **moving an initialisation out of bind can create a race that the
re-initialisation was hiding.** Ask what the removed init was accidentally protecting.

### 3 — CVE-2026-31606

CVSS **5.5**, same vector. `struct cdev cdev` is embedded in `struct f_hidg`, and `hidg_bind` called
`cdev_init()` on it — which is `memset(cdev, 0, sizeof *cdev)` plus `kobject_init` — immediately
before `cdev_device_add`. An open fd holds a reference on that same kobject via `chrdev_open` →
`cdev_get` → `kobject_get_unless_zero`. On unbind→rebind with an fd open, bind wipes live reference
state on a referenced object.

The fix is a struct change rather than a guard: the embedded `struct cdev` becomes a `struct cdev *`
with a per-bind `cdev_alloc()`. `cdev_init` then appears zero times in the file — there is no in-use
object left to re-initialise.

⛔ **This device cannot detect this defect, before or after the fix.** See
`verification/03-cve-2026-31606.md`. That is a positive claim about the device's instrumentation, not
a hedge.

## Verification

Method and per-defect records in [`verification/`](verification/). The short version: fixes 1 and 2
were closed by single-variable regression against the immediately prior build, with positive
controls, and fix 2 was confirmed a second time from a different log source. Fix 3 is evidenced by
source reading only.

## Applying

Patches are `git format-patch` output against a 4.14.190 Samsung vendor tree, in order:

```sh
git am patches/0001-*.patch patches/0002-*.patch patches/0003-*.patch
```

They are cumulative and expect to be applied in sequence — 3 assumes the struct as 1 left it.

## Licence

GPL-2.0, as derivative works of the Linux kernel. See [LICENSE](LICENSE). Upstream authorship of the
original fixes is credited above; the backporting, the added locking in patch 2, and the verification
work are mine.

No compiled artifacts are distributed here — no kernel image, no modules, no flashable package. This
repository is source and evidence only.
