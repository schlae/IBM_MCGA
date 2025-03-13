# IBM MCGA Gate Array Reverse Engineering

IBM's MCGA (Multi-Color Graphics Array) is a low-cost video chipset introduced
with the PS/2 models 25 and 30. The Epson Equity 1e uses MCGA compatible video
but does not use the same chips.

The IBM chipset consists of the memory controller gate array and the video
formatter gate array. Some examples of these were fabricated on an internal
IBM gate array process, while others used an external gate array part by
Seiko.

## Memory Controller Gate Array (72X8300)

This gate array contains an implementation of the MC6845 sync generator IC,
manages the video RAM interface to the ISA bus, manages the character RAM
interface, and a few other miscellaneous functions including clock selection
and monitor ID readback.

The example I have reverse engineered is implemented using a Seiko SLA6430
gate array. It contains 4,342 basic cells (BCs) with 4 transistors each.
The BCs are arranged in 167 rows and 26 columns. This is a 2um CMOS process
with 2 metal layers.

The image is from [72x8300-sla6430j](https://siliconpr0n.org/map/ibm/72x8300-sla6430j/)

The reverse engineered schematic and layout can be found in the mcga72x8300flat
subdirectory.

## Video Formatter Gate Array (72X8205)

The formatter gate array decodes the ISA memory and IO port addresses,
manages the RAMDAC interface, and generates pixel data in both graphics
and text modes.

There are two images of this IC. The first,
[72x8205-gl14105fs](https://siliconpr0n.org/map/ibm/72x8205-g14l05fs/)
appears to be on an internal IBM gate array process. Unfortunately, during
decapping, the top metal layer was removed, so the netlist could not be
extracted. The second,
[72x8205-sla6330j](https://siliconpr0n.org/map/ibm/72x8205-sla6330j/), has
not yet been reverse engineered.

It is a Seiko SLA6330 gate array. It contains 3,312 basic cells with 4
transistors each. The BCs are arranged in 144 rows and 23 columns.

## MCGA Notes

Based on the reverse engineering efforts, new information about MCGA has been
discovered.

* MCGA can genlock to external HSYNC and VSYNC signals. These signals are
brought out to the video connector: pin 12 (ID1) is VSYNC and pin 11 (ID0)
is HSYNC. To enable this mode, write a 1 to bit 3 of register 0x12 (character
generator interface and sync polarity, or display sense). In the technical reference manual for the PS/2 model 30, this bit is listed as "Reserved = 0".
Presumably, this genlock mode would require an external clock PLL connected
to the 25MHz or the 14MHz clock input.

* Register 0x10 (Mode Control) bit 3, "Compatibility", only affects 80x25 text
modes. It causes the horizontal timing registers to be multiplied by 2
(and incremented by one, in the case of 0x00, horizontal total, and subtracted by one, in the case of 0x02, start horizontal sync).

* Register 0x10 (Mode Control) bit 2, "Clock = 1", controls which clock drives
the video circuitry. In the default state, most of the video circuitry uses the
25.175MHz clock. You can set clock frequency to the 14.318MHz input by changing
this bit to a 0.

* Register 0x10 (Mode Control) bit 6, "Reserved = 0", is not yet fully
understood.

* Register 0x20 (Reserved) is a manufacturing test mode register.

| Bit | Function |
|-----|----------|
| 7   | 14.318MHz alternate clock mode (unknown)
| 6   | VCK pin alternate mode (normally VCKIN just goes to VCK)
| 5   | Speedup mode: unknown
| 4   | Speedup mode: Cursor position high counter
| 3   | Speedup mode: Cursor position low/char counter
| 2   | Speedup mode: Vertical total adjust counter
| 1   | Speedup mode: Vertical counter
| 0   | Speedup mode: Horizontal counter

The counter speedup modes basically inject a clock signal into the upper
four bits of each counter as well as the lower four bits, so the counter
runs out quicker. This is an aid for the factory test in the chip tester.

## Reverse Engineering Process Information

The 72x8300 image was scaled from 21808x21778 to 10904x10889. The output
jpg file was set to 85% compression to save space, set to 48DPI, and
imported into KiCAD at a scale factor of 0.103170. This results with a
BC-to-BC spacing of "3mm" in KiCAD units.

Library footprints were created for each basic cell type as they were
identified and associated with schematic symbols. Pads were all placed
in the center so the footprint could be rotated, since many of the BCs in
the original were mirrored.

The gate array uses two layers of metal, and it can be challenging to understand
the connections between layers. In general, there are only a few allowable
contacts:

* Metal 1 to metal 2
* Metal 2 to polysilicon (gate)
* Metal 2 to diffusion

Each column has two parallel wires on metal 2 that carry VCC (on the right)
and GND (on the left). Therefore, the left two transistors in each BC are
NMOS and the right two transistors are PMOS. In a BC, two transistors share
a single gate, and each gate has three connecting pads. Contacts can attach
VCC or GND to diffusion regions on the shared channel connection between
two transistors, or the isolated channel connection on one transistor, the
other, or both.

External signals entering or leaving a logic gate *usually* connect using
horizontal traces that cross the entire cell on metal 1. Wiring internal
to a logic gate is typically done on metal 2.

Besides the vertical parallel wires on metal 2 carrying power and ground,
 there is another set of horizontal wires (on metal 1) that also carry power
and ground. My reverse-engineered layout does not have traces placed on top
of these lines since they are not signal lines.

Traces were placed starting at a footprint pad, following the underlying
metal, and tying to all other connected pads, given a net name, and then
back-propagated to the schematic (the reverse of the usual KiCAD process).


