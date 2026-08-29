---
layout: post
title: "Bootloader Design Patterns for Firmware Flashing"
date: 2026-08-29
categories: [Firmware]
tags: [bootloader, flashing, firmware-update, embedded]
---

Every embedded product that ships firmware updates eventually needs an answer to the same question: how do you replace the code running on a microcontroller without ever leaving it in a state where it can't boot? The bootloader is where that answer lives, and its design shapes almost everything downstream — how you handle a failed update, how you roll out security patches, and how much sleep you lose during a field recall.

This post walks through the patterns I keep coming back to when designing or reviewing bootloaders for MCU-based products.

## What the Bootloader Actually Owns

It's easy to think of the bootloader as "the thing that flashes new firmware," but its real job is narrower and more important: **deciding, on every reset, which image is safe to run.** Flashing is just one of the operations it performs in service of that decision. A bootloader that flashes flawlessly but boots a corrupted image after a power loss has failed at its actual job.

That framing changes the design priorities. The flashing routine gets a lot of attention because it's visible and testable; the boot-decision logic gets less, because it only shows its flaws under conditions that are hard to reproduce — power loss mid-write, a reset during a CRC check, a watchdog firing at exactly the wrong instruction.

## Single-Stage vs. Two-Stage Bootloaders

**Single-stage**: one bootloader image handles both the "am I safe to boot" logic and the flashing protocol, and it can update the application but never itself.

**Two-stage**: a minimal, effectively immutable first-stage bootloader (often called an IPL/ROM bootloader) does the bare minimum — verify and jump to stage two — while a second-stage bootloader carries the more complex flashing protocol and can itself be field-updated.

The trade-off is straightforward: single-stage is simpler and has less attack surface, but you can never fix a bug in your update mechanism without a hardware-level recovery path (JTAG/SWD, a factory tool, etc.). Two-stage costs more flash and more complexity, but it means "we shipped a bug in the flashing logic" isn't a fleet-bricking event — you can push a fix through the same channel that pushes everything else.

For anything shipped at volume, I lean two-stage. The cost of the extra few KB of flash for stage one is small compared to the cost of a bootloader bug with no software-only recovery path.

## Memory Layout: Give Yourself an Escape Hatch

A layout I've found reliable, from low to high address:

1. **Stage-1 bootloader** — write-protected after production programming, as small and simple as possible
2. **Stage-2 bootloader** — field-updatable, but only by stage 1
3. **Application slot(s)** — one or two, depending on update strategy (below)
4. **Configuration / metadata region** — boot flags, image headers, rollback counters
5. **Application data / filesystem** — kept separate from anything the bootloader inspects

The metadata region deserves its own attention. It should be small, simple to parse, and ideally power-loss safe on its own (e.g., using a ping-pong pair of sectors so an interrupted metadata write never leaves you with a half-written record). This is a common source of subtle bugs: teams put a lot of rigor into verifying the application image but treat the tiny "which image is active" flag as an afterthought, and that's exactly the write most likely to be interrupted by a badly timed reset.

## Update Strategies: Single-Bank vs. A/B (Dual-Bank)

**Single-bank**: the new image overwrites the old one in place. Minimal flash usage, but there is a window — however short — where neither a complete old image nor a complete new image exists. Any interruption during that window needs a hardware-assisted recovery path.

**A/B (dual-bank)**: two full application slots. The new image is written entirely into the inactive slot while the active slot keeps running (or the device is briefly offline, depending on constraints); only after the new image is verified does the bootloader flip a flag to make it active. The previous image stays in place as an automatic fallback.

A/B roughly doubles the flash budget for application code, which is a real cost on cheaper MCUs. But it buys you two things that are hard to get any other way: updates that are safe against power loss at any point during the write, and instant rollback if the new image fails a post-update health check (a boot loop, a failed self-test, whatever your criteria are). On any product where "bricked in the field" is an expensive outcome, that trade is worth making.

A middle ground I've used on flash-constrained parts: A/B for a **small, critical core** (bootloader stage 2 plus a minimal recovery application) combined with single-bank for the larger main application, accepting a narrower risk window on the piece that's harder to protect fully.

## Verification Before You Trust an Image

Whatever the layout, the bootloader should verify an image before jumping to it — not just after flashing, but on every boot. Two checks that serve different purposes:

- **Integrity** (CRC32 or similar) catches corruption — incomplete writes, flash wear, bit flips.
- **Authenticity** (a signature check against a key baked into stage 1) catches a different problem entirely: someone intentionally flashing firmware you didn't build. These are not interchangeable, and I've seen designs that implement CRC and call the security box checked.

If you're supporting secure boot, the signature check belongs in the immutable stage-1 bootloader, verifying stage 2; stage 2 in turn verifies the application. Each stage should only trust the next one after checking it, not before.

## Designing for the Failure You Didn't Plan For

A few habits that have saved me more than once:

- **Rollback counters, not just rollback flags.** A boolean "did it boot okay" flag can flap forever if the failure is intermittent. A counter with a threshold ("fall back after 3 consecutive failed boots") is more robust and still simple.
- **A watchdog that's armed during flashing, not just during normal operation.** The flashing routine is exactly the code most likely to hang on real hardware — a bad sector, a bus glitch — and it's the code most teams forget to protect with a watchdog because "we're not in the application yet."
- **Make the bootloader's logging survive a reset.** A small ring buffer in a non-volatile region that records the last few boot decisions turns "why did the fleet roll back last Tuesday" from a guessing game into a five-minute log read.
- **Test the interruption, not just the happy path.** Powering off mid-flash, mid-metadata-write, and mid-verification are three different test cases, and each has caught real bugs for me that a normal "flash and reboot" test never would.

## Closing Thought

The flashing protocol is the part of a bootloader that gets a specification, a test plan, and a demo. The boot-decision logic — the part that runs on every single power-up, forever, on every unit in the field — is the part that determines whether a bad update is an inconvenience or an incident. Spend the design budget accordingly.
