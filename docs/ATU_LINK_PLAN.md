# ATU-100 link plan (atu-link branch)

This branch replaces the ukoda serial-firmware approach to the ATU-100 link
with the design in the original spec (`spec/Antenna_Switch_Project.pdf`):
a bit-banged link on PIC16F1938 pins RA6 and RA7.

It supersedes decision 3 in `docs/ASSUMPTIONS.md` ("ukoda/ATU-100-remote
serial firmware, not RA6/RA7 bit-bang"). The files that follow the old
decision are `atu100_firmware/README.md`, `docs/ATU_LINK.md`,
`remote/atu_link.py`, and `remote/button_emulation.py`. They will be
updated once the link protocol is defined.

## Why the change

- The ukoda firmware uses RB1 and RB2 for its serial link. Those are the
  A/M and Bypass button pins, so the tuner's front-panel buttons stop working.
- ukoda removes the ATU-100's own display support, and its author describes
  it as a work in progress with limited testing.
- RA6 and RA7 are free for a new purpose (see below), so the stock buttons
  and display can stay.

## Firmware base

Use WA1RCT's MPLAB/XC8 port of N7DDC ATU-100 EXT firmware V3.2:

- Repo: https://github.com/WA1RCT/N7DDC-ATU-100-mini-and-extended-boards
- Folder: `ATU_100_EXT_board/FirmWare_PIC16F1938/1938_EXT_MPLAB_sources_V_3.2`
- Builds with the free MPLAB X IDE and XC8 compiler. Stock V3.2 needs
  the paid mikroC PRO for PIC compiler.

A function-by-function comparison against the stock mikroC V3.2 source
showed the tuning routines (tune, coarse_tune, sub_tune, sharp_ind,
sharp_cap, correction, set_ind, set_cap, set_sw) use the same constants
and logic. The power and SWR measurement routines differ only in syntax.
Buttons stay on RB0 to RB2 and the OLED driver is complete.

### Fix required in the WA1RCT port before flashing

**Configuration bits.** Only `FOSC` is set with `#pragma config`. The rest
   fall to chip defaults, which differ from stock V3.2 (PLL, low-voltage
   programming, brown-out voltage, power-up timer). Set every config bit to
   match the stock V3.2 hex: CONFIG1 = 0x3FC4, CONFIG2 = 0x1233.

## RA6 and RA7

The PIC runs on its internal 16 MHz oscillator with clock-out disabled, so
both pins are usable as general I/O. In stock V3.2 they are the Tx_request
outputs (RA7 = p_Tx active high, RA6 = n_Tx active low), asserted during a
tune to ask the transmitter for a carrier. That handshake is not used in
this station, so:

- Remove the `p_Tx` / `n_Tx` writes in `button_proc()` (tune branch),
  `show_reset()`, and the end of the tune routine.
- The 250 ms delay after raising Tx_request in the tune branch was there
  to let the transmitter key up. It can be removed or shortened.
- Reconfigure RA6/RA7 in `pic_init()` for the link.

The operator supplies a low-power carrier manually during a tune.

## Link design (draft)

- **ATU-100 is the master.** It decides when each exchange happens. The
  remote Pico 2 W listens and answers.
- **Two one-way lines,** so each can pass through an isolator channel:
  - **RA7: PIC to Pico.** The PIC sends its status frame as asynchronous
    serial. V3.2 uses no interrupts, so software delay loops give accurate
    bit timing.
  - **RA6: Pico to PIC.** After each status frame the PIC opens a short
    reply window and reads the Pico's answer, which carries any pending
    command. The PIC is otherwise idle during the window. (Variation to
    evaluate: the PIC pulses RA7 as a clock while the Pico places reply
    bits on RA6, so the PIC needs no receive timing.)
  - Commands are picked up between main-loop passes; during a tune they
    wait until it finishes.
- **Low baud rate** (to be fixed with the frame format), well within the
  isolator's speed and tolerant of feedthrough capacitance.
- **Compact binary frames, not JSON.** Stock V3.2 under mikroC leaves about
  1,100 program words and 156 bytes of RAM free. Each frame carries a
  checksum; the Pico applies timeouts so a hung tuner cannot stall it.
- **Status fields:** forward power, SWR, L, C, LC/CL order, Auto, Bypass,
  and a tuning-in-progress flag. Forward power feeds the remote's
  hot-switch interlock (refuse antenna change above 1 W).
- **Commands:** Tune, Auto on/off, Bypass on/off, Reset.
- **Command hooks in the PIC firmware:**
  - Tune sets `g_b_Soft_tune`, the existing software-tune flag that auto
    mode already uses.
  - Move the Auto and Bypass logic out of `button_proc()` into small
    functions called by both the buttons and the link.
  - The early-exit checks on PORTB in the power display routine also need
    to check for a pending link command.

## Electrical

The PIC runs at 5 V; the Pico's GPIO is 3.3 V. The PIC side shares the
ATU-100's RF-laden ground, which must stay separate from the Pico's ground
(the same principle as the antenna switch box, which isolates the Pico from
the 12 V switching environment with VOM1271 drivers).

- **Final build: ADuM1201 digital isolator.** One channel in each
  direction, matching RA7 to Pico and Pico to RA6. Each side runs from its
  own supply (5 V from the ATU-100 board, 3.3 V from the Pico), so the
  part handles level translation and galvanic isolation together. Mount it
  in the shielded compartment; the ATU-side lines pass through feedthrough
  capacitors (a few hundred pF, so data edges are not rounded off) and
  common-mode chokes. SOIC-8 package; use an adapter board for
  breadboarding.
- **Bench prototype: BSS138 bidirectional level-shifter boards** (already
  on hand). These translate 3.3 V and 5 V but share one ground, so they are
  for bench work only, with no RF present. A 2-channel board covers RA6 and
  RA7.
- Swapping the level shifter for the isolator is hardware only; the
  firmware is the same.
- The VOM1271 is not suitable for the data link: as a photovoltaic MOSFET
  driver it switches far too slowly.

## Open items

1. Build the unmodified WA1RCT project in MPLAB X with free XC8 and record
   the program and data memory used. This decides whether the ATU-100's own
   OLED code can stay. Fallback if space is short: drop the ATU-100 display
   code, since the tuner is remote and the master shows its data.
2. Define the exact frame format, baud rate, reply-window timing, and
   command encoding.
3. Update the Pico side (`remote/atu_link.py`), the docs listed above, and
   the host tests to match.

## Rules for this branch

- Do not flash the ATU-100 until the config-bit fix is in and the build has
  been reviewed.
- Back up the PIC (program and EEPROM) before any flash.
- Keep `main` unchanged until this branch is reviewed.
