# NextKey Bridge

## Project plan and developer handover

24 November 2026 \| Working title \| Prototype design, not yet
electrically validated

## 1. Objective

Build an externally powered modern keyboard-to-PS/2 adapter for the ZX
Spectrum Next. Prioritise the original 2.4 GHz USB receiver supplied
with the mechanical keyboard. Wired USB HID should follow naturally;
Bluetooth Classic and BLE HID are later extensions. Retain the existing
PS/2 mouse and Y-splitter. Avoid drawing dongle or Pico power from the
Next.

## 2. Hardware inventory

Owned: two Raspberry Pi Pico WH (RP2040), breadboards, jumper wires,
micro-USB OTG cables, USB power supplies, multimeter, donor PS/2
keyboards and splitter cables, FNIRSI DSO152 (single channel, 200 kHz,
2.5 MS/s), Hantek 6022BE (two analogue channels, 20 MHz, up to 48 MS/s),
keyboard and dongle. Logic analyser may already be owned; otherwise a
sigrok-compatible 8-channel 24 MHz unit is optional. Verify input
voltage ratings.

Basket: Adafruit ISO1540 bidirectional isolator, USB-C power breakout,
three-pin JST-SH debug cable, male-ended JST-SH cable(s), Perma-Proto
board. Verify two 5.1 kΩ CC pull-downs on the USB-C board, header
availability and connector pinouts. Two four-pin STEMMA QT leads are
useful if avoiding soldering the isolator headers.

## 3. Development environment

Primary: M4 Mac Mini, macOS, VS Code, official Raspberry Pi Pico
extension, Pico SDK, ARM GCC, CMake, Ninja, Git and optionally Codex.
Intel MacBook Pro with macOS and Windows 10 Boot Camp is a fallback for
instrumentation. Omarchy is not in use for now. Check current tool
compatibility before installation. Use OpenHantek6022 or Hantek software
and PulseView/sigrok where supported. Keep host tests portable.

Use C++17 and PICO_BOARD=pico_w (WH has headers, not a different
software target). Use TinyUSB host and later BTstack. Pin SDK versions
once working. Build with
`cmake -S . -B build -G Ninja -DPICO_BOARD=pico_w` and
`cmake --build build`, after installing/configuring SDK and toolchain.
Keep generated build files out of Git.

## 4. Debugging and power

Flash the official debugprobe_on_pico UF2 onto Pico A using BOOTSEL.
Pico B runs the receiver. Verify current firmware pin assignments and
target JST-SH connector pinout against official documentation before
wiring. Expected connections: A GP2 to B SWCLK, A GP3 to B SWDIO, GND to
GND; A GP4 TX to B GP1 RX, A GP5 RX to B GP0 TX. No power connection
between the boards. A is USB-powered by the development computer. B is
independently powered. Avoid driving an unpowered board.

For USB host operation, a regulated 5 V supply via the USB-C sink
breakout feeds target VBUS (physical pin 40) and GND (pin 38). The
micro-USB OTG adapter connects the dongle. Verify voltage and polarity
before connecting. Never attach the target USB socket to a PC while
externally driving VBUS. Consider current limiting/fusing and strain
relief. USB-C Rd resistors enable 5 V; they do not protect against
overcurrent or backfeed. Do not negotiate higher voltage. Measure actual
current consumption.

## 5. Proposed PS/2 electrical interface --- validation required

Candidate: ISO1540, both channels bidirectional, separate supplies and
grounds. Pico side: 3V3 OUT (pin 36), GND, proposed GP2 clock and GP3
data. Next side: keyboard PS/2 pin 1 data, pin 3 ground, pin 4 +5 V, pin
5 clock; verify donor cable continuity and Next/splitter pinout and
revision before connection. Pins 2 and 6 unused on conventional keyboard
leg. Do not connect the two ground or supply domains. Next powers only
its isolator side, not the Pico or dongle.

This is NOT a verified schematic. Check ISO1540 side-specific
input/output low thresholds, pull-up values, rise times, sink current,
propagation and power-off behaviour against datasheets and actual
measurements. The offset-low side may require a different interface.
Verify the Next voltage and available current budget, including mouse.
Test with independent supplies and simulated host open-drain signals
before connecting the Next. If thresholds or power sequencing are
unsuitable, revise the circuit. Never drive PS/2 high actively; release
the line instead. Pico GPIO is not 5 V tolerant.

Hantek probe grounds are common; PC-connected analyser/debugger grounds
may also be common. Never bridge isolation domains through instruments.
Use the battery-operated handheld scope disconnected from charging/other
equipment for single-domain checks. Verify probe settings and analyser
voltage ratings.

## 6. Firmware architecture

USB HID input -\> report parser -\> normalised key state/events -\>
scan-code mapping -\> PS/2 queue -\> bidirectional PS/2 transport. Host
command handling feeds back to keyboard state/LED output. Later
Bluetooth input shares the same normalised event interface.

Suggested files: CMakeLists.txt, README.md, tusb_config.h, src/main.cpp,
usb_host.cpp, keyboard.cpp, scancodes.cpp, ps2.cpp, matching headers,
tests/keyboard_tests.cpp and scancode_tests.cpp. Separate
hardware-dependent code from pure C++ translation logic. Use UART
logging while USB is in host mode. Avoid blocking TinyUSB task
servicing.

USB: enumerate dongle, inspect interfaces and HID report descriptors; do
not assume boot keyboard reports. Handle report IDs, modifiers,
rollover, multiple interfaces, disconnect/reconnect and release of held
keys. Rearm receive callbacks. Log raw reports before translating.

PS/2: idle high, open-drain clock and data, device-generated clock
nominally 10--16.7 kHz, 11-bit frame (start 0, 8 data bits LSB first,
odd parity, stop 1). Implement set 2 make/break, extended keys,
modifiers, and special sequences (Pause/Print Screen). A press = 1C,
release = F0 1C. Implement host inhibit, request-to-send, host-to-device
reception and acknowledgement. Start with timer/state machine or simple
test transmitter; consider PIO once requirements are clear. Do not
assume an illustrative bit-bang routine is a complete protocol
implementation.

Compatibility: reset FF -\> FA then AA when ready; identify F2, enable
F4, disable F5, defaults F6, LED ED, typematic F3, scan set F0, echo EE
and resend FE as appropriate. Validate command semantics, timing, queue
priority and actual Next behaviour against protocol references. Avoid
stuck keys after disconnect; define overflow recovery. Support LED
feedback where dongle allows it.

## 7. Suggested schedule (flexible, not a commitment)

Day 1: install SDK, create Git repository, blink, UART logs, debug probe
and breakpoint test. Confirm toolchain reproducibility. Get TinyUSB host
example running and enumerate dongle. Day 2: capture HID reports;
implement normalised key events and scan-code table with host unit
tests. Test modifiers, simultaneous keys and disconnect. Build PS/2
transmit proof of concept, capture A press/release on analyser or scope.
Day 3: implement bidirectional transport and essential host commands.
Bench-test isolator with simulated host and independent supplies.
Compare captures against a donor keyboard. Only after electrical checks
pass, connect to Next and test typing, held keys, menus and mouse
coexistence. Day 4 (optional): improve compatibility, repeat/LED
handling, reconnect and stress tests; document wiring and results.
Consider wired USB and Bluetooth experiments. Transfer to Perma-Proto
only after breadboard design is stable. Enclosure/3D printing later.

## 8. Acceptance criteria

-   Dongle enumerates and normal keyboard events are reliably received.
-   Unit tests cover ordinary keys, modifiers, extended keys, releases
    and disconnect recovery.
-   PS/2 clock, framing, parity and host inhibit/commands verified on
    bench.
-   Both isolator domains meet measured voltage/timing and power-off
    requirements; no unintended power path to Next.
-   Next accepts typing, modifiers and repeated keys without stuck keys;
    existing mouse remains functional.
-   Document actual pinout, current draw, firmware version and known
    limitations before permanent assembly.

## 9. References and handover

Official Pico documentation:
https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html
Pico examples: https://github.com/raspberrypi/pico-examples Debugprobe:
https://github.com/raspberrypi/debugprobe VS Code extension:
https://github.com/raspberrypi/pico-vscode TinyUSB:
https://github.com/hathach/tinyusb ISO1540 datasheet:
https://www.ti.com/lit/ds/symlink/iso1540.pdf OpenHantek:
https://github.com/OpenHantek/OpenHantek6022 PulseView:
https://sigrok.org/wiki/PulseView

Start a Codex session in a new repository with this document as
PROJECT_PLAN.md. Ask it to inspect the plan, confirm the SDK/toolchain,
and implement only the first milestone. Keep a progress log, measured
results and decisions in the repository. No hardware or software has yet
been tested; this plan records proposed decisions rather than completed
work. Do not connect the Spectrum Next until the electrical validation
gate passes.
