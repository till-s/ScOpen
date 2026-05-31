# The ScOpen - an Affordable 2-Channel 130MSPS USB Scope

This project's goal was the development of a completely open
(hard-, gate- and software) and affordable oscilloscope for
hobbyists.

## Introduction

### Key Features

 - Two channels, 26mVpp to 20Vpp (~60dB adjustable gain)
 - DC/AC coupled inputs
 - 50-Ohm termination
 - 130MSPS, 10-bit resolution
 - Memory for ~ 13Msamples
 - USB-C for power and high-speed (USB2) data connection
 - External trigger in or out
 - Clock output (e.g., for probe calibration)
 - Half of the FPGA resources are available for users to play
 - GUI scope software (Qt/qwt)
 - Completely open source
 - Component cost (w/o PCB, probes and enclosure) ~ CHF 150 (more or less,
   depending on features + components)

### General System Design

I chose to pick a 130MHz ADC because beyond that range the design
quickly becomes more expensive. Higher-rate ADCs are more expensive,
the processing needs a higher-performance FPGA, potentially with
MGTs (JESD205) and RAM bandwidth also becomes an issue. The ScOpen
is still quite useful for hobby projects with signal bandwidths up to
10-20MHz. The ADC and the amplifiers can handle higher input frequencies
which you can undersample with this device.

The dual-channel MAX195xx ADC can easily be processed by a low-cost
Trion T20 FPGA and an inexpensive SDRAM.

## Hardware Design

### Input Stage

The input stage is built around an OPA859 wideband opamp in a classical
inverting configuration. This has two advantages: the common-mode
voltage remains at zero and the input is inherently protected against
over-voltages since there is always the 900k/11p impedance between
the input and the amplifier.

The input stage features a 0/20 dB switchable attenuator using an analog
switch to multiplex the current through the 100k/100p branch between
ground and the opamp's inverting input. Note that all nodes of this
analog multiplexer are always operating at ground/virtual-ground level.
OTOH, the switch must handle some current (at high input frequencies).

The capacitor providing DC insulation can be bridged with an optical
relay. Neither the resistance of this relay nor any parasitic capacitance
of the switch itself are critical (only the stray capacitance between
the isolated switch and the light emitter but this is very small).

The 50-Ohm termination is switched with a mechanical relay (somewhat
surprisingly these devices have quite good isolation up to ~100Mhz or
so). In this case the resistance of the switch and its capacitance
are much more critical and an SSR would not suffice.

There are two locations where the 50-Ohm resistor can be installed:
the user has the option to install it either in front or "behind"
the AC-coupling capacitor.

### PGA

The AD8370 is one of the few digitally programmable amplifiers I found
that have

 - high gain
 - 40dB range
 - differential output that can deliver 1.5Vpp
 - reasonable cost

Unfortunately the AD8370 does not give us control of its output common
mode voltage (it's always mid-supply). The simplest way to shift the
common-mode voltage to just above 0.4V (which is required by the MAX195xx
ADC) is employing resistors.

Alas, we must pay for this with a 2.5dB loss of sensitivity. Shifting
the entire +/-2.5V analog supply to, e.g., +3.0V/-2.0V (which would situate
the common-mode output of the AD8370 at 0.5V) is not an option because
the output swing of the OPA859 would have to be reduced too much (this
wideband amplifier is *not* rail-to-rail).

The AD8370 also does the single-ended to differential conversion for
us and I connected the otherwise unused complementary input of the PGA
to a DAC which can be used for calibration and compensating offset voltages.

### Calibration DAC

The dual-channel DAC operates in single-supply mode but the dual differential
amplifier takes care of converting the voltage for bipolar operation.
In order to reduce noise into the PGA the differential amplifier's bandwidth
is reduced significantly. The feedback connections are chosen so that
the amplifier remains stable when loaded with the large 1u bypass capacitor.

### I/O Expander

Using an I2C I/O expander on the far side of the board makes routing the
PCB easier and reduces the need for FPGA pins (which are almost all used).
It also allows the use or a higher I/O voltage (4V) which is beneficial
to the analog mux in the front-end.

### ADC

The MAX195xx dual-channel ADC is one of the best I could find in
it's price range. It is available with different resolutions (8- or 10-bit)
and sampling rates (65, 100 and 130 MSPS). The devices are all pin-compatible
and you can pick what best suits your needs + budget.

 - pin-compatible variants exist (8-bit, 10-bit; 65, 100 or 130MHz)
 - supports DDR output with overflow indicator and source-synchronous clock
 - digitally controllable with SPI-like interface (few pins)
 - reasonable supply-voltages; generates analog supply internally
 - differential (or single-ended) clock input
 - AD was very nice and provided free samples, thanks

The DDR output time-multiplexes the two channels over the same parallel
interface which reduces the required connections to the FPGA.

### ADC Clock PLL

I decided to use a Versaclock-6e chip from Renesas. Sadly, Silicon-labs'
documentation and support became abysmal after their clock business was
bought by Skyworks. The versaclock is simple enough and one of the
cheapest options to get a high-quality sampling clock.

Unfortunately, I found that the documentation is incomplete and sometimes
incorrect (settings don't do what they claim to do). It is nice that there
are several pin-compatible options (even a versaclock-5 chip can be used
on our board). I also appreciate very much that Renesas supplied me
with a generous amount (I believe due to a misunderstanding) of 5P49V6975
chips (the one with the built-in crystal) so that I never again will
have to buy a clock for my projects.

Some of the caveats I found:

 - no 'in-lock' or 'input-clock detected' pins; not even register settings
   are documented (apparently, register 0x9d[7] and 0x9d[4] provide
   this functionality).
 - no automatic switch-over between external input clock and crystal.
 - i2c interface *stops working* when using external input and the
   input clock is disconnected - this means that you can't use i2c to
   switch back to the internal clock! You must use the dedicated input.
 - *all outputs* disappear (even reference out!) when certain operations
   (requiring re-locking or re-calibration) are performed. If you do these
   from an FPGA and you have no other clock - then you are screwed.

On the plus side:

 - not too complex to set up (no closed-source vendor wizards required
   to get the thing ticking).
 - wide I/O voltage-ranges and output options (LVDS, CMOS, ..)
 - i2c programmable
 - fractional dividers
 - flexible routing inside the chip
 - perfect for this project

#### Clock Outputs

We use output 3 in a differential configuration to clock the ADC.

Output 2 we route out of the board so that it can, e.g., be used as a
1kHz source for calibrating probes (note that such low frequences require
cascading dividers which can only be done in 'sequence', i.e., output 2 can
cascade dividers 1 and 2 - but not 3 or 4. Thus, no cascading at all would
be possible for output 1.

The external clock output can also be programmed to forward the 25Mhz
reference for synchronizing the clock of a second unit.

Output 4 can optionally be used to clock the USB Phy but that would
require logic in the FPGA to configure the clock as no USB connectivity
is available without the correct USB clock.

Output 0 (the buffered reference) is forwarded to the FPGA (currently
unused in the HDL design).

### FPGA

I had built a first prototype using an Artix-7 FPGA and even managed
to solder the BGA package but then decided it was too expensive and
I even had problems to get timing closure.

In my search for cheaper solutions I came across Efinix' Trion
family which I since have used in several projects. I must say that
I really took a liking to these devices. An excellent option
for simple projects (Artix w/o transceivers costs 3-5 times more).

 - affordable (T20 with 20kLE in fastest speed grade, LQFP-144 @CHF10)
 - LQFP packages available; not only BGA (while I found these are also
   not hard to reflow-solder it is harder to route w/o in-pad vias which
   are still expensive).
 - good documentation, support, forums.
 - usable vendor tools; some parts of the programmer written in python.
 - JTAG with an off-the-shelf FTDI chip; no special vendor driver necessary.

The FPGA configuration is stored in a vanilla SPI flash. When blank,
the configuration must be loaded via JTAG (also required when debugging
a design with the embedded logic analyzer). Once a design is loaded
I usually use it to 'self-program' the flash but Efinix also has
flash-loader examples.

The current logic design uses ~9000LEs, i.e., about half of the T20
is still available for user-enhancements.

### SDRAM

The T20 has fewer block ram resources (1Mbit) than the Artix I had used
before. A maximum of ~49k samples (2 channels @10bits each) can be stored
on the device (no logic-analyzer in use; otherwise the sample memory has
to be reduced).

Because sometimes it is really useful to have a deeper memory I decided
to add a SDRAM to the design. I'm currently using a 256Mbit device but bigger
ones should work, too.

At 10-bits per channel this RAM holds ~13.4Msamples. Because the RAM data
interface has 16-bits but the ADC needs to store 20-bits per clock cycle
the RAM needs to run at a higher frequency than the ADC (the FPGA logic
implements a gear box to shuffle the bits around). Accounting for the additional
overhead the RAM needs for switching rows and refreshing etc. a RAM clock of 166MHz
suffices to maintain the throughput of 130Msamples/s.

When a 8-bit ADC is employed the RAM clock can be reduced and at the same
time more samples can be stored.

### USB

#### Phy

We use a standard ULPI transceiver to provide high-speed USB connectivity.
The 480Mbps are still adequate for this application. The USB device
funcionality is implemented in the FPGA logic. To a host computer the device
will appear as a standard communications class ACM device.

The phy generates a 60Mhz clock which is forwarded to the FPGA where it
clocks all of the 'slower' logic downstream of the raw data acquisition.

#### USB-C Connector

The USB-C connector delivers power and high-speed connectivity. Standard
5k1 pull-down resistors are on the CC-pins.

### Power

A number of SMPS are integrated on the board for generating

 - +1V2 FPGA core voltage
 - +3V3 digital voltage
 - +/- 2V5 analog voltage (this looks 'clean enough'; even better noise
   reduction could probably achieved with +/- 3V or higher with LDOs
   downstream. I found, howver, that it is difficult to find cheap
   LDOs for negative voltages).

The -2V5 are generated by a standard buck converter in a 'boot-strap'
configuration. Since the input voltage that this converter 'sees' is
7.5V we cannot use the simpler and cheaper converters employed for the
other voltages.

For the I/O expander which controls the front end we had originally
used the 5V USB bus voltage directly but then found that this fed
intolerable noise into the input stage so that a 4V LDO was added to
filter that rail.

On the PCB all the SMPS can be isolated from their loads by removing
zero-ohm resistors or ferrite beads. This gives you the option to
initially omit these components. After the board is assembled you may
verify that the voltage levels are correct before proceding with
adding the resistors and beads.

### Connectors

There are three BNC connectors in the front; the LHS and middle
BNC are the two inputs and on the RHS there is the clock IO.

On the back side there is a single BNC which serves as a buffered
GPIO (5 or 3.3V levels are selectable by means of 0-ohm jumpers).
The GPIO is used as an external trigger input or output (to
trigger a cascaded second unit).

The USB-C connector is also situated at the back.

### LEDs

There are four RGB LEDs; two in the front (each one next to one of
the input BNCs) and two in the back. An additional green LED is
located next to the clock output (RHS BNC) in the front.

A bi-color (R/G) LED adjacent to the USB connector serves as a power
and configuration indicator. Red indicates that there is power but the
FPGA is not configured, green indicates successful configuration.

### JTAG and Expansion Header

The FPGA's JTAG pins are routed to a 2mm-pitch header. An additional
(2.54mm-pitch) header is available for user-defined purposes. It
connects to two FPGA pins and provides most of the supply voltages.

### PCB

I went with a 4-layer PCB which JLC offers for ~CHF10 (lead-free
HASL finish). I use their cheapest board constraints (which means
min. hole dia. is 0.3mm, at least 0.5mm apart and vias are 0.3/0.45mm).

They do offer somewhat controlled impedances in their stackups for
no extra money and I designed the differential pairs for their 3313
dielectric. However, the distances are so short that it probably doesn't
matter very much.

The PCB is 100mm long and 60mm wide; components are placed on both
sides of the board.

### Enclosure Option

The board fits perfectly into a BOPLA F720-100 extruded aluminum
enclosure. Other (wider) profiles of course also work but you
have to adjust the width of the board accordingly. Make sure there
is enough space for the bottom cages underneath the PCB.

## Firmware

The firmware is written in VHDL and its main components are

 - ADC DDR input and multi-stage CIC decimation filters (these let us
   trade sampling frequency for improved SNR when looking at lower
   frequency signals).
 - Trigger logic.
 - FIFO and gear-box to feed the SDRAM.
 - SDRAM controller.
 - USB stack; all controls as well as main data readout use this stack.
 - Packet engine (the ACM interface is essentially a bi-directional
   byte stream; the packet engine handles framing) and command handler.
 - Bit-bang engine (all i2c and SPI devices on the board are controlled
   by software via this engine except for the SPI flash).
 - Command interface to control acquisition parameters.
 - SDRAM readout.

### Clock Domains

There are three clock domains

 - ADC sampling clock (130MHz). All the sample processing (triggering
   and CIC filters) run at this rate and end up writing a FIFO.
 - SDRAM clock (166MHz). The write logic reads the ADC fifo and writes
   into RAM via the 20-16 bit gear box. The SDRAM controller also
   operates at this clock, for course.
 - USB/ULPI clock (60MHz). All USB operations happen in this
   domain. This includes the SDRAM readout (via FIFO). Acquisition
   parameters that are needed in the ADC clock domain are forwarded
   through CC logic.

## Software

There are several software components

 - Qt/Qwt GUI written in c++. This is the principal tool to operate the instrument.
 - `bbcli` command-line utility. Mostly for expert use but it is also used
   for programming the SPI flash, computing and storing calibration data
   in flash.
 - python bindings (expert use)

### Scope GUI

Some of the features offered by the GUI application include

 - Live display of scope traces. Zooming and panning.
 - Live display of FFTs. Zooming and panning.
 - Parameter control (triggering, decimation, gain control, input stage, etc.).
 - Cursors for measurements.
 - Data storage in HDF5 files.
 - Save/restore settings in JSON files.
 - Hotkeys (e.g., it is possible to initiate a delayed single-shot acquisition
   with a key. Useful if you only have two hands: hit the key then place the
   probes and wait for the scope to be armed and triggered).
 - Firmware upgrade.
 - Control of the clock output.

The beauty of open-source is that when you find yourself in need of a particular
feature you can simply add it yourself!

## How to Build this Project

### Hardware

I ordered the PCB and stainless-steel stencils from JLC and assembled
the device myself. I populate and reflow the top side first (in a toaster
oven). For the back side I use lower-temperature paste (maybe that's paranoid)
to ensure that the top components don't fall off when they are hanging
upside down when I'm baking the back side.

A few components (most notably the stubborn BNC connectors) need to be
hand soldered. The BNCs need a lot of heat and flux but be careful not
to overheat the insulation.

As mentioned already I usually omit the supply jumpers and ferrite beads
and make sure the voltages are correct before installing them.

Once you have assembled the device and it has passed optical inspection
I proceed with connecting JTAG and using the Efinity software I check
that it recognizes the FPGA.

At this point you can use JTAG and some suitable open-source packages
to perform boundary scan operations which let you verify that you
can talk to the components on the board. Alternatively you can just
cross your fingers and proceed loading the firmware and hope it works.
Usually most of it does and you can use the `bbcli` tool to exercise
the different components. Or you use the GUI to identify problems.

If the scope trace window displays complete garbage then most likely
some of the SDRAM pins are not soldered well. Inspect them and/or
use boundary scan and a multimeter to ping the individual pins.

You can also rebuild the firmware to use the block-ram buffer to make sure
everything else works.

### Firmware

In order to generate the FPGA bitstream you need to download and install
the Efinity software from Efinix website.

#### Preparation

Before you can start with synthesis, however, you need to generate a few
files which are not maintained in git.

     chdir fw
     source <efinity-top-dir>/bin/setup.sh
     ./modules/efx-scripts/generate_project.py

This script - among other files - generates `GitVersionPkg.vhd` which
ensures that the current git hash is embedded in the firmware.

If you make any changes it is recommended that you commit these changes
and then run `module/efx-scripts/update_git_version_pkg.sh` to regenerate
this file so that the embedded hash reflects the correct version.

Unfortunately - unlike vivado - Efinity does not currently support adding
user scripts to the flow so that updating the version has to be done manually.

#### Generating the Bitstream

Now you can start Efinity, navigate to the project's XML file and open it.
Once the project is loaded you can hit the buttons and run the flow which
should produce the bit stream. If you make changes to the design make sure
timing is met. Sometimes you have to try different seeds for PNR until this
succeeds. The `modules/efx-scripts/explore.sh` script helps with this.

#### Upload With JTAG and Writing the Flash

Obvously, for anything to happen the board needs power: connect the board
to a computer using a USB cable (the connection is used for data once the
FPGA is configured),

With a blank flash you have to use JTAG to configure the Trion with your
first design. You have to point the Efinity programmer to the `scope_v3.bit` file
which should be in the `outflow` directory.

If the FPGA configures correctly then your computer should now recognize the
ScOpen as a ACM device on USB (use lsusb or similar).

Check if you get any sign of life with `bbcli` (software build instructions below).
If you have multiple ACM devices then you have to figure out which one corresponds
to the ScOpen.

     bbcli -d /dev/ttyACM0 -V

should now print some version information. You can now use `bbcli` to write
the same bitstream into the configuration flash. Note, however, that the utility
requires a different file format (note the different suffix):

     bbcli -d /dev/ttyACM0 -a0 -f outflow/scope_v3.hex.bin -SWena,Erase,Prog -!

This should show a number of progress marks ('Z' while erasing, '.' while writing and 'v'
while verifying) and be done in 20 seconds. The 'Erase' operation is not necessary
when the flash is blank but once you start overwriting you'll need it. It is also
important to remember that the design just written to flash is *not* yet loaded into
the FPGA! In this particular case it is identical to the one loaded through JTAG
but if you burn an updated bitstream then you have to power-cycle the board or press
the reset button in order to reconfigure the FPGA.

## Software

Make sure you have

 - c and c++ compiler
 - cmake and make or ninja
 - Qt5 and Qwt
 - HDF5 (C, not C++ bindings!)
 - fftw3
 - jansson

make sure you have development packages for these libraries. Cmake will let you know
if something is missing.

### Build

The ususal

     chdir sw
     cmake -B build .
     make -j -C build

The aforementioned `bbcli` utility can be found in `build/usbadc-support/sw/bbcli`. Use
cmake's installation features to install the stuff anywhere you like.
