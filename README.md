# Arch Linux ARM on a Casio fx-991ES PLUS with Raspberry Pi 3 B+

![Casio fx-991ES PLUS Linux build](calc.png)

Build the Linux side on a Raspberry Pi 3 Model B+, bring up the calculator interfaces separately, then integrate the assemblies. Follow the sequence below: boot media, first boot, processor checks, keyboard, display, drivers, and mechanical installation.

The Pi runs Linux. The Casio controller is not being reflashed with a PC operating system. Replacing its role means routing the calculator interfaces to the Pi or an auxiliary controller, not soldering a BCM2837B0 onto the Casio chip footprint.

## 1. Lay out the hardware

Put these on the bench before touching the calculator PCB:

- Raspberry Pi 3 Model B+.
- microSD card and a reader connected to a Linux workstation.
- Regulated 5 V micro-USB supply rated for the Pi and attached peripherals.
- HDMI monitor and USB keyboard for initial configuration.
- Ethernet connection for package installation.
- Multimeter, fine probes, and insulated interconnects.
- A documented display and keyboard interface for the final assembly.

The Pi 3 B+ has a Cortex-A53-based BCM2837B0, 1 GB of RAM, a microSD interface, and a 40-pin expansion header. Raspberry Pi specifies a 5 V/2.5 A input. Use its [product specifications and mechanical drawing](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/) when designing the mounting arrangement.

An intact Pi 3 B+ and its stacked connectors need an enlarged enclosure, rear extension, or external mounting arrangement. Do not assume the stock calculator case closes around the board. Preserve access to microSD, power, and recovery connections before fixing anything in place.

Keep the calculator battery circuit separate from Pi power. The LR44 and solar supply are not the Pi's power source.

## 2. Get the correct Arch distribution

Use **Arch Linux ARM**, a separate ARM distribution project. The standard Arch Linux installation ISO targets x86-64 PCs and does not boot this ARM board. Copying that ISO onto the card, or into the calculator, is not an installation method.

For this guide, choose the AArch64 Raspberry Pi root-filesystem archive from the [Arch Linux ARM Raspberry Pi installation page](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3). Follow its current partition, extraction, and boot-file instructions. Its AArch64 route uses U-Boot and the mainline kernel; do not mix it with instructions that depend on the alternative ARMv7 vendor stack.

The media layout has two jobs: a FAT boot partition carries the startup files, while an ext4 root partition carries the Linux filesystem. Extracting the archive preserves ownership and permissions that a normal file-manager copy may lose.

Use the upstream installation page for the destructive partitioning and formatting steps. Identify the card by capacity, model, and connection before writing anything:

```sh
lsblk -o NAME,PATH,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
```

Run that command before and after inserting the reader. Resolve the newly appearing device explicitly. Device names such as `/dev/sdb` are examples of naming conventions, not safe defaults.

Do not write to the workstation's system disk. Finish the upstream card preparation, flush pending writes, and unmount the card before removing it.

## 3. Boot the Pi before connecting calculator wiring

Insert the prepared card. Connect HDMI, a USB keyboard, Ethernet, and then power. Complete the first-login and package-key initialization steps from the distribution's installation page, and change the initial account passwords.

Run the following on the Pi:

```sh
uname -m
uname -r
cat /proc/device-tree/model
printf '\n'
findmnt /
findmnt /boot
```

The assembly examples below require `aarch64`. A 64-bit-capable CPU can run a 32-bit userspace, so the board model alone is not enough to choose an assembler target.

As root, bring the installed system up to date and install the diagnostic tools:

```sh
pacman -Syu
pacman -S --needed binutils gcc make file evtest
```

Reboot if the kernel was updated before building modules against it. Keep an HDMI console available until the calculator display is working.

## 4. Save a repeatable system report

Save this as `collect-state.sh` on the Pi. It reads system state and writes a timestamped report under your current directory. Run it once before wiring changes and again afterward.

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077
report=$(mktemp -d ./calc-state.XXXXXXXX)
{
    date -Is
    uname -a
    if [[ -r /proc/device-tree/model ]]; then
        tr -d '\0' < /proc/device-tree/model
        printf '\n'
    fi
    printf '\nMounted filesystems\n'
    findmnt
    printf '\nMemory\n'
    free -h
    printf '\nInput devices\n'
    cat /proc/bus/input/devices
    printf '\nLoaded modules\n'
    lsmod
} > "$report/system.txt"
if ! journalctl -b -k --no-pager > "$report/kernel.txt" 2> "$report/journal-error.txt"; then
    printf 'Kernel journal unavailable; inspect journal-error.txt\n' >&2
fi
printf 'Report saved in %s\n' "$report"
```

Run it with:

```sh
chmod +x collect-state.sh
./collect-state.sh
```

If kernel messages are restricted, run the script from an authorized root shell. A blank journal file is not evidence of a clean boot when access was denied.

## 5. Compile a native ARM assembly check

Create `hello.S` on the Pi:

```asm
.section .rodata
message:
    .ascii "fx-991ES PLUS / Raspberry Pi 3 B+ / AArch64\n"
.equ message_len, . - message

.section .text
.global _start
.type _start, %function
_start:
    // Linux AArch64 write(fd, buffer, length).
    mov x0, #1
    adrp x1, message
    add x1, x1, :lo12:message
    mov x2, #message_len
    mov x8, #64
    svc #0

    // This small smoke test requires one complete write.
    cmp x0, #message_len
    cset x0, ne
    mov x8, #93
    svc #0
.size _start, . - _start
.section .note.GNU-stack,"",%progbits
```

Build and inspect it:

```sh
as -o hello.o hello.S
ld -o hello hello.o
file hello
readelf -h hello
objdump -d hello
./hello
printf 'exit status: %s\n' "$?"
```

The program enters through `_start`, places arguments in `x0` through `x2`, selects a system call in `x8`, and enters the kernel using `svc #0`. There is no C runtime or `main` here.

`adrp` computes the page-relative address component. The `:lo12:` relocation supplies the remaining offset. The linker resolves the pair for this executable layout.

This checks executable format, native instruction execution, and the Linux syscall path. It does not initialize the calculator display or drive GPIO. Run it through the console that already works.

For production output handling, retry interrupted or partial writes; this deliberately small diagnostic returns failure if its single write is incomplete.

## 6. Add an assembly routine with a callable interface

Use a second sample to check register-level arithmetic through the C calling convention. Save this as `matrix-index.S`:

```asm
.text
.global matrix_index
.type matrix_index, %function
matrix_index:
    // x0 = row, x1 = column, x2 = column count
    // Return row * column_count + column in x0.
    madd x0, x0, x2, x1
    ret
.size matrix_index, . - matrix_index
.section .note.GNU-stack,"",%progbits
```

Save the harness as `matrix-check.c`:

```c
#include <stdint.h>
#include <stdio.h>

extern uint64_t matrix_index(uint64_t row, uint64_t col,
                             uint64_t columns);

int main(void)
{
    /* Synthetic dimensions, not a recovered Casio matrix. */
    const uint64_t rows = 4, columns = 5;
    for (uint64_t r = 0; r < rows; ++r) {
        for (uint64_t c = 0; c < columns; ++c) {
            if (matrix_index(r, c, columns) != r * columns + c)
                return 1;
        }
    }
    puts("matrix index check passed");
    return 0;
}
```

Compile and execute:

```sh
gcc -Wall -Wextra -O2 -o matrix-check matrix-check.c matrix-index.S
./matrix-check
```

`madd` performs multiplication and addition in one instruction. The function is a leaf routine: it calls nothing, needs no stack frame, and changes only caller-saved state. Bounds checking belongs to its caller. Arithmetic wraps at the register width.

Use the measured matrix dimensions in the actual scanner. This arithmetic sample does not infer electrical connections from key legends.

## 7. Isolate and connect the keyboard interface

Disconnect calculator power, including the solar source where necessary. Map each contact with continuity measurements. Record rows and columns against photographs before attaching an interface.

Choose one of two integration paths:

1. A keyboard microcontroller scans the contacts and presents USB HID to the Pi.
2. A characterized GPIO matrix connects through an appropriate electrical interface and uses a matching Linux driver and hardware description.

For initial integration, USB HID keeps scan timing and debounce separate from Linux scheduling. The controller still needs firmware matched to the measured contact matrix.

Do not connect Pi outputs to traces still driven by the Casio controller. Establish isolation with continuity checks and powered measurements. A reset signal alone is not proof of electrical isolation.

Pi GPIO uses 3.3 V logic. Use suitable translation if the attached interface requires another voltage domain, and check the interface's power-off behavior.

After attaching the keyboard interface, inspect enumeration:

```sh
cat /proc/bus/input/devices
ls -l /dev/input/by-id/ /dev/input/by-path/
```

Some interfaces will not create both directories. Use the listed identity rather than assuming the keyboard is always `event0`.

From a root shell, run:

```sh
evtest
```

Select the intended device. Press and release every key, then test held keys and simultaneous combinations. Verify that each press has a release. Only after that should you implement calculator-specific shift layers.

## 8. Attach the display and install its driver

Bring up a documented replacement display using its controller-specific instructions. HDMI is the baseline debug display; SPI and DSI integration require a display model, connector wiring, and compatible driver configuration.

**Driver links:** add the repository's display, keyboard-controller, and host-adapter download links here. Match each package to its hardware and kernel version.

Before loading a custom kernel module, inspect its metadata. Save this helper as `inspect-module.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
if [[ $# -ne 1 || ! -f $1 ]]; then
    printf 'Usage: %s path/to/driver.ko\n' "$0" >&2
    exit 2
fi
printf 'Running kernel: %s\n' "$(uname -r)"
modinfo "$1"
printf '\nModule inspection only; nothing was loaded.\n'
```

Run it against the actual module file supplied for the display. Examine architecture, dependencies, license, and `vermagic`. Compatibility depends on the kernel build, not merely the processor name.

Prefer the distribution's existing driver when it supports the controller. Apply only the overlay or hardware-description settings documented for that driver and kernel stack.

Inspect initialization failures with:

```sh
journalctl -b -k --no-pager
ls -l /sys/class/drm/
```

A device file appearing is an enumeration milestone. Confirm visible output, redraws, orientation, and recovery after reboot before treating the display path as finished.

## 9. Replace the controller's role and mount the assemblies

Use the Pi as the application processor. Keep keyboard scanning on the chosen interface and route the replacement display through its documented connection.

If the original controller remains physically present, isolate the nets it no longer owns. If it is removed, preserve the recovered connections through an interposer or replacement contact board. Do not treat bare-die encapsulation as a conventional desolderable package.

For chip-level changes on an auxiliary controller, use its documented programming and recovery interface. Modifying boot straps or firmware is different from replacing the silicon. Neither operation makes the original Casio CPU pin-compatible with the Pi's SoC.

Mount with insulated standoffs and strain relief. Check clearance around solder joints, connector shells, membrane supports, and the microSD slot. Keep heat-producing parts away from the display assembly and provide a route for heat to leave the enclosure.

Power the complete assembly and repeat the keyboard and display tests. Compare the saved state report with the bench baseline.

## 10. Shut down, recover, and keep a known-good card

Before disconnecting power:

```sh
sync
systemctl poweroff
```

Wait for shutdown to complete. `sync` alone does not stop later writes from running applications.

Keep a second card with the last known-good system. When a new driver breaks boot or display output, return to that card and compare kernel logs and configuration rather than changing power, wiring, and software simultaneously.

The sections below expand the electrical and kernel work behind these steps. Use them when a bring-up check fails or when replacing an interface with a custom circuit.

# Detailed integration procedures

## Board investigation

### Establish a baseline

Before disassembly, record normal operation, display behavior, and key response. Photograph the exterior model designation and any revision markings.

After opening the case, photograph both PCB faces with a scale reference. Preserve the orientation of contact membranes and display connectors.

Remove electrical power before continuity measurements. Cover or disconnect the solar source as appropriate; removing the battery alone may not remove every source of power.

### Build a net inventory

Assign local identifiers to accessible pads rather than guessing their purpose.

For each pad, record:

- Position on the board photograph.
- Continuity to known supply terminals.
- Resistance to reference ground with power absent.
- Observed voltage during operation.
- Whether activity changes when a key is pressed.
- Instrument loading and probe configuration.

A repeating waveform is not automatically UART. It could be a clock, a matrix scan, an LCD waveform, or multiplexed test activity.

### Instrument limitations

A logic analyzer reports transitions relative to its threshold. It does not prove that the waveform has valid analog margins.

An oscilloscope probe adds capacitance. A signal that changes when probed may be sensitive to that loading.

A grounded bench instrument can connect circuit nodes through protective earth. Establish the measurement reference before attaching probe grounds.

### Identifying serial traffic

A UART hypothesis requires consistent bit timing, plausible framing, repeatability, and a known electrical reference.

A protocol decoder producing readable characters once is insufficient evidence.

Do not drive an unidentified test pad to discover its function. Start with passive observations and documented component behavior.

## Modding a chip

The phrase can mean three substantially different operations.

### Modifying surrounding circuitry

Changing boot straps, reset circuitry, clock sources, or bus connections can alter how a documented chip behaves. This is board-level modification, even when the target is a chip's boot behavior.

It requires a datasheet describing strap sampling, acceptable voltage levels, and reset timing.

For replacement hardware, possible modifications include:

- Selecting a documented recovery boot mode.
- Exposing a debug UART.
- Changing a supported storage selection strap.
- Adding a supervisor to improve reset behavior.
- Isolating a peripheral before connecting a replacement controller.

None of these operations creates missing CPU instructions or additional RAM.

### Modifying firmware

Firmware changes require writable storage and a documented or recovered programming route.

Before altering a programmable device, establish its exact identity, supply voltage, image layout, programming algorithm, and recovery procedure.

If readback is available, repeated identical reads provide stronger evidence than one successful transfer. They do not prove the image has been interpreted correctly.

Do not describe the original Casio firmware as replaceable until actual hardware evidence establishes that property.

### Modifying silicon

Decapsulation, microprobing, and focused-ion-beam edits are specialized semiconductor analysis techniques. They are not normal substitutes for a missing firmware update mechanism.

Even a successful connection change would not supply the complete memory system, peripheral support, and boot infrastructure required by a Linux port.

For the replacement-compute route, concentrate on interface recovery and replacement-system engineering. Original-silicon modification is not a prerequisite.

## Replacing a chip

A Linux processor is not a drop-in substitute for an unidentified calculator controller.

Pin compatibility alone would be insufficient. Supply rails, clocking, boot sequence, memory interfaces, package geometry, and peripheral behavior would also have to match.

### Packaged components

Replacing a socketed or soldered packaged component can be practical when the replacement is electrically and functionally compatible.

For a design change, compare every pin, including reserved pins, power domains, reset states, and exposed thermal pads. Matching package names do not establish matching pinouts.

### Chip-on-board assemblies

If the donor uses a bare die bonded directly to the PCB under encapsulation, there may be no removable package footprint. Confirm the construction on the actual unit.

Removing encapsulation does not reveal a standard socket for an application processor.

The realistic replacement is then a new board or an interposer connected to independently characterized peripheral nets.

### Interposer constraints

An interposer is useful only when the signals it carries tolerate the added routing.

Long flying wires may work for slow key contacts. They are not a reasonable default for external DDR memory.

The interposer design must specify:

- Mechanical attachment and strain relief.
- Reference ground paths.
- Connector orientation and pin numbering.
- Voltage domains and translators.
- Power-off behavior.
- Test access.
- Isolation from the original controller.

### Replacement motherboard

Start with a module carrier rather than a bare application-processor design if the objective is demonstrating Linux inside the enclosure.

A module can keep high-speed RAM routing within an already engineered assembly. It does not remove the need to validate carrier power, interfaces, or thermal behavior.

Bring out recovery pads before optimizing the board outline. A board that fits perfectly but cannot be recovered after a failed image is an unfinished design.

## Power architecture

The original solar cell and LR44 arrangement must not be assumed capable of powering the replacement computer. Determine requirements from the selected module and measurements.

Use a regulated, current-limited bench supply for initial work.

### Rail definition

For every rail, specify nominal voltage, tolerance, maximum load, startup ramp, sequencing dependencies, and acceptable ripple.

Distinguish continuous current from transient demand. A supply that handles idle current can still collapse during processor startup or storage writes.

### Voltage droop

A first-order capacitor estimate is:

```text
Delta V = I * Delta t / C
```

For an illustrative 0.1 A load step lasting 100 microseconds and a 0.1 V permitted drop, the ideal capacitance is 100 microfarads.

Real behavior also depends on capacitor ESR, ESL, regulator response, layout inductance, and effective capacitance under DC bias. This calculation is not a component selection by itself.

### Power budget

A planning estimate is:

```text
Input power = sum(load rail powers) / conversion efficiency
Runtime = usable stored energy / average input power
```

Use watt-hours for energy and watts for power. Battery capacity stated in ampere-hours requires a voltage assumption before it becomes energy.

Peak demand, temperature, discharge rate, and converter cutoff can reduce usable energy.

### Back-powering

A powered interface may feed an unpowered device through its protection structures. Check the component's partial-power-down specifications.

A series resistor can limit current but does not establish valid power sequencing or guaranteed isolation.

### Portable power

If a rechargeable battery is eventually used, select a protected power system with a charger appropriate to the cell chemistry and a defined load-sharing arrangement.

Charging, system operation, and shutdown are separate operating cases. Verify each rather than assuming the charger output behaves like an unrestricted system supply.

## Memory and signal integrity

A Linux-capable CPU still needs a usable memory subsystem.

### Capacity planning

Budget separately for the kernel, userspace, page cache, graphics buffers, DMA allocations, and temporary startup peaks.

A compressed filesystem image's storage size does not equal its runtime RAM requirement.

An initramfs consumes memory after extraction. Measure the actual booted system before selecting the final capacity.

### DDR layout

For a bare processor design, follow the selected vendor's routing and topology requirements. These can cover controlled impedance, byte-lane relationships, reference planes, termination, and timing budgets.

Do not copy a generic trace-length number from an unrelated processor.

Successful boot once does not establish adequate timing margin over voltage and temperature.

### Return paths

Fast signal edges need continuous return paths. Routing across a plane split can increase loop area and degrade signal integrity even when the nominal bus clock is relatively low.

Edge rate matters independently of transaction frequency.

### Decoupling

Capacitor placement and connection inductance affect high-frequency performance. A large distant capacitor is not equivalent to local decoupling specified by the component manufacturer.

### Clock and reset

Validate reset release against rail stability and clock availability. A reset supervisor can prevent execution during an invalid supply region if its thresholds and timing suit the device.

## Keyboard interface

### Recovering the matrix

With the original circuit unpowered, map which contacts connect when each key is pressed. Avoid treating membrane geometry as a reliable electrical schematic.

Create a row-column table supported by continuity measurements. Include keys that appear to use separate power or reset paths.

### Scanning

A typical matrix scan activates one line and reads the orthogonal lines, then repeats for the remaining positions.

The exact drive states depend on the circuit. Unselected outputs must not fight a selected output through closed contacts.

Allow settling time after changing drive state before sampling.

### Debounce

Mechanical contacts can transition several times around a press or release.

A debounce algorithm should define sampling period, stable-state threshold, release handling, and behavior when several keys change together.

Excessive filtering increases input latency. Insufficient filtering produces duplicate events.

### Ghosting

A matrix without per-key isolation can produce ambiguous multi-key combinations. Software cannot always infer the actual keys from an electrically indistinguishable state.

Document supported combinations. Do not promise arbitrary rollover without evidence.

### Controller protocol

If a microcontroller scans the keyboard, define a versioned event format with key identity, press/release state, and a way to recover from dropped messages.

A periodic full-state report can restore synchronization after an event is lost.

Define startup behavior explicitly so a held key does not remain logically stuck after either processor resets.

### Linux mapping

Expose keys through the Linux input subsystem using a suitable existing driver or a documented custom interface.

The physical key legend and the emitted Linux key code are separate decisions. Shift layers, navigation, and text entry require a usable application-level design.

## Display integration

### Reusing the original display

Do not assume that the original LCD accepts a standard SPI framebuffer stream.

First determine whether the accessible interface connects to a controller or directly to glass electrodes.

Direct LCD drive requires the appropriate alternating waveforms, voltage levels, bias, and multiplex timing. Sustained unintended DC across LCD electrodes can damage the display.

### Recovering electrode behavior

If reuse is pursued, measure differential waveforms between relevant electrodes. A waveform relative to board ground alone may not describe the voltage across a pixel.

Record behavior for repeatable display patterns and distinguish measured mappings from inferred ones.

A successful single segment test is not enough to establish full display compatibility.

### Replacement display

A documented replacement display reduces uncertainty but introduces mechanical constraints, power requirements, and a software driver dependency.

Check the visible area, connector bend radius, module thickness, mounting pressure, and any backlight requirement.

### Transfer budget

For an illustrative 320 by 240 display at 16 bits per pixel:

```text
Frame size = 320 * 240 * 2 = 153600 bytes
At 10 full frames/s = 1536000 bytes/s
Payload bitrate = 12288000 bits/s
```

Protocol overhead and bus gaps increase the required link rate. These numbers describe an example replacement display, not the Casio panel.

Dirty-region updates can reduce traffic when only part of the screen changes.

## Boot chain

Trace the boot sequence in this order:

```text
Power valid
  -> Processor boot ROM
  -> Platform-specific early initialization
  -> Bootloader, where required
  -> Linux kernel and hardware description
  -> Root filesystem
  -> Init process
  -> Calculator interface
```

The exact stages depend on the selected platform. Some systems use vendor firmware or combine stages.

### Early initialization

External RAM may need initialization before a large bootloader or kernel can execute. A working UART message from an early stage does not prove RAM is stable.

### Hardware description

On platforms using Device Tree, describe the actual wiring: enabled controllers, pin assignments, interrupts, clocks, regulators, and attached devices.

Device Tree does not implement a missing driver. A node must match the binding expected by the selected kernel driver.

### Memory layout

Boot components, kernel images, device-tree data, and reserved buffers must not overwrite one another during loading and decompression.

Choose addresses from platform documentation and the actual RAM map. No universal load addresses are provided here.

### Recovery

Keep a documented recovery boot mode reachable after assembly. Recovery should not require a functioning graphical interface or an intact writable filesystem.

Record how to identify the target before programming it and how to verify the resulting image.

## Drivers

**Driver downloads:** links are maintained by the repository owner. Select the package for the exact host adapter, target peripheral, and kernel build.

Separate three categories:

| Category | Purpose |
| --- | --- |
| Host driver | Lets the development computer communicate with a programmer or serial adapter |
| Target kernel driver | Controls a peripheral attached to the Linux hardware |
| Controller firmware | Runs on an auxiliary keyboard or power-management microcontroller |

A host driver does not provide Linux support for the original Casio processor.

### Driver manifest

Each driver entry should include its upstream location, revision, license, supported device identifiers, kernel or host compatibility, installation method, and recovery instructions.

For binaries, include a checksum and identify the source or build provenance. A checksum detects an unexpected file change; it does not establish trust by itself.

### Prefer existing support

Check the selected kernel for an existing driver before writing a new one. Board integration may require hardware-description changes rather than new driver code.

Linux represents devices and drivers through its device model, including bus-specific matching and lifecycle operations. See the [kernel device model overview](https://docs.kernel.org/driver-api/driver-model/overview.html).

### Probe and dependencies

A peripheral can exist electrically while its driver cannot initialize because a clock, regulator, reset controller, or parent bus is unavailable.

Treat deferred initialization as a dependency problem to investigate, not proof of a broken peripheral.

### Interrupt handling

Define interrupt polarity and triggering from the device specification. An incorrect level configuration can create repeated interrupts; an incorrect edge configuration can lose events.

Operations that may sleep require an appropriate execution context. Do not place arbitrary bus transfers in a hard interrupt handler.

### DMA and cache coherency

If a device uses DMA, CPU virtual addresses and device-visible addresses must not be assumed interchangeable.

Buffer lifetime, ownership transfer, cache synchronization, and mapping failures need explicit handling according to the selected kernel's DMA API and platform constraints.

A correct implementation on one coherent system may expose bugs on a noncoherent system.

### Module compatibility

Build external modules against the exact intended kernel build environment. Matching a broad version family is insufficient evidence of compatibility.

Keep configuration and symbol-version information with the build artifacts where applicable.

### Suspend and resume

A driver that works after cold boot may fail after suspend because registers, clocks, or power rails have lost state.

Test key wake, display restoration, and storage behavior separately. Define what happens to input events arriving during transitions.

## Validation

### Stage 1: compute platform

Boot the selected module on its documented development hardware. Save a complete serial boot log and identify the exact image.

Confirm that storage, RAM, and recovery work before adding calculator-specific wiring.

### Stage 2: carrier power

Check rails before connecting expensive components. Observe startup, steady state, shutdown, and a representative transient load.

Measure at the load as well as at the regulator when investigating droop.

### Stage 3: keyboard

Verify each key independently, then test repeated presses, held keys, simultaneous combinations, reset during a held key, and wake behavior.

Report lost, duplicated, and stuck events separately.

### Stage 4: display

Use patterns that expose orientation, bit order, color ordering, clipping, and partial-update mistakes.

Run repeated redraws while monitoring supply behavior and system logs.

### Stage 5: assembled system

Repeat testing after installation in the enclosure. Cable routing, pressure, restricted airflow, and contact alignment can introduce faults absent on the bench.

### Acceptance criteria

Keep the following with each hardware release:

- Photographs identifying the donor and replacement electronics.
- Reproducible build information.
- A Linux boot log from the installed compute hardware.
- Working input and display evidence.
- A measured power profile.
- A documented recovery procedure.
- Explicit identification of any external boards or supplies.

Identify external supplies and development connections in build photographs so the assembly can be reproduced.

## Failure analysis

| Symptom | Candidate causes | Useful next observation |
| --- | --- | --- |
| No serial output | Wrong voltage, wrong UART, reset, boot mode | Rails, reset waveform, documented console pinout |
| Early boot repeats | Brownout, watchdog, image failure | Reset cause and supply trace |
| Kernel starts then crashes | RAM instability, wrong hardware description | Full log and memory validation |
| Root filesystem unavailable | Wrong storage configuration or image layout | Boot arguments and storage enumeration |
| Display remains blank | Power, reset, driver matching, interface mismatch | Controller initialization and bus activity |
| Duplicate key events | Contact bounce or event handling | Raw scan state and timestamps |
| Keys remain pressed | Lost release event or reset mismatch | Controller state report |
| Works only when probed | Loading, reference, or timing sensitivity | Probe-dependent waveform comparison |
| Works open, fails assembled | Mechanical stress, shorts, heat | Controlled enclosure comparison |

Change one variable at a time and keep the failing image. Otherwise a temporary improvement can erase the evidence needed to explain the fault.

## Contribution requirements

Hardware contributions should identify the PCB revision and distinguish measurements from guesses. Include instrument settings when they affect interpretation.

Driver contributions should identify the supported hardware and kernel configuration, describe failure handling, and provide a reproducible test procedure.

Document the hardware revision and test conditions associated with each supported configuration.

Do not add invented pinouts, unexplained programming voltages, fictional driver names, or screenshots presented as proof of unperformed work.

Third-party software and hardware documentation retain their own licenses and redistribution conditions. Check the applicable terms before redistributing source, binaries, or vendor documents.

For subsequent board revisions, repeat the power, recovery, input, display, and enclosure checks. Preserve the working configuration and boot logs before changing the hardware or system image.
