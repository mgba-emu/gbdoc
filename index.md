Open Game Boy Documentation Project
===

- [Technical specifications](#tech-specs)
	- [Models](#models)
	- [Hardware](#hardware)
- [CPU](#cpu)
- [Memory map](#memory-map)
	- [Memory mapped I/O](#mmio)
	- [CGB-specific memory mapped I/O](#mmio-cgb)
- [Cartridges](#cart)
	- [Memory bank controllers](#mbc)
		- [MBC1](#mbc-1)
		- [MBC2](#mbc-2)
		- [MBC3](#mbc-3)
		- [MBC5](#mbc-5)
		- [MBC6](#mbc-6)
		- [MBC7](#mbc-7)
		- [MMM01](#mbc-mmm01)
		- [Pocket Camera](#mbc-pocketcam)
		- [TAMA5](#mbc-tama5)
		- [HuC-1](#mbc-huc1)
		- [HuC-3](#mbc-huc3)
		- [Unlicensed mappers](#mbc-unlicensed)
- [Picture processing unit](#ppu)
	- [Draw mode](#ppu-mode)
		- [Mode 0](#ppu-mode-0)
		- [Mode 1](#ppu-mode-1)
		- [Mode 2](#ppu-mode-2)
		- [Mode 3](#ppu-mode-3)
	- [Palettes](#ppu-pal)
	- [Background](#ppu-bg)
	- [OAM/OBJs (sprites)](#ppu-obj)
	- [Window](#ppu-win)
	- [Scrolling](#ppu-sc)
- [Audio processing unit](#apu)
	- [Channel 1](#apu-ch1)
	- [Channel 2](#apu-ch2)
	- [Channel 3](#apu-ch3)
	- [Channel 4](#apu-ch4)
- [Timer](#timer)
- [DMA](#dma)
- [Interrupts](#irq)
- [Serial I/O](#sio)
	- [Link cable](#sio-link)
	- [Game Boy Printer](#sio-gbprinter)
	- [Infrared](#sio-ir)
- [Miscellaneous](#misc)
	- [Hardware issues](#errata)
	- [Super Game Boy](#sgb)
	- [Cheat codes](#cheats)
- [About this documentation](#about)

<a id="tech-specs">Technical specifications</a>
===

<a id="models">Models</a>
---

There are three lines of Game Boy models based on CPU iterations, each with its own minor revisions:

- DMG: Game Boy
	- DMG-01
		- Original Game Boy, Dot Matrix Game, "brick"
	- SHVC-027
		- Super Game Boy
		- SNES accessory with DMG CPU
		- Slight clock difference from DMG
- MGB: Game Boy Pocket
	- MGB-001
		- Game Boy Pocket
		- Minor hardware revision
	- MGB-101
		- Game Boy Light
		- Japan only
	- SHVC-042
		- Super Game Boy 2
		- SNES accessory with MGB CPU
		- Same clock as MGB
- CGB: Game Boy Color
	- CGB-001
		- Game Boy Color
		- Major hardware revision
	- AGB-001/AGS-001/AGS-101
		- Game Boy Advance
		- CGB compatibility mode

<a id="hardware">Hardware</a>
---

- [CPU](#cpu): Sharp SM83 clocked at 4.194304 MHz
	- Similar instruction set to Intel 8080 and Zilog Z80, sometimes erroneously referred to as a Z80 or LR35902
	- CGB version is optionally clocked at 8.388608 MHz (2x DMG CPU)
- RAM: 8 kiB (DMG), 32 kiB (CGB)
- [Video](#ppu): 160×144 pixels
	- 2 backgrounds: main and window
	- 40 OBJs (sprites), max 10 per line
	- 8×8 pixel tiles
	- 4 "modes" (0–3), with variable timing
	- 456 cycles per line
	- 70224 cycles per frame
	- VDraw: 144 lines, 65664 cycles
	- VBlank: 10 lines, 4560 cycles
	- DMG mode:
		- 2-bit/4 distinct colors
		- 4 colors per palette
		- 1 background palette
		- 2 object palettes (index 0 is transparent, 3 effective colors per palette)
	- CGB mode:
		- 15-bit/32768 distinct colors
		- 4 colors per palette
		- 8 background palettes
		- 8 object palettes (index 0 is transparent, 3 effective colors per palette)
-  Audio: 4 channels
	- [Channel 1](#apu-ch1): Square, sweep
	- [Channel 2](#apu-ch2): Square
	- [Channel 3](#apu-ch3): "Wave" (4-bit PCM patterns)
	- [Channel 4](#apu-ch4): Noise (LFSR)

<a id="cpu">CPU</a>
===

The Game Boy uses a Sharp SM83, which is similar to Intel 8080, Zilog Z80, and other i8080 knockoffs.

- DMG: clocked at 4.194304 MHz
- CGB: optionally clocked at 8.388608 MHz
- Two types of cycles: T states and M cycles
	- M cycles ("machine" cycles, 1:4 clock) are the base unit of CPU instructions
	- T states or T cycles ("transistor"(?) states, 1:1 clock) are the base unit of system operation and many components are clocked directly on T state
- 8-bit general purpose registers
	- A, B, C, D, E, H, L
- 8-bit flags register (F)
	- Bits 0–3 are grounded to 0
	- Bit 4: C (carry flag)
	- Bit 5: H (half-carry flag)
	- Bit 6: N (negative flag)
	- Bit 7: Z (zero flag)
- 16-bit general purpose register views
	- AF, BC, DE, HL
	- AF does not allow writing F bits 0–3
- 16-bit special purpose registers
	- PC (program counter)
	- SP (stack pointer)

TODO

<a id="memory-map">Memory map</a>
===

- `$0000` – `$7FFF`: [External bus (ROM region)](#mbc)
- `$8000` – `$9FFF`: VRAM
- `$A000` – `$BFFF`: External bus (RAM region)
- `$C000` – `$DFFF`: WRAM
- `$E000` – `$FDFF`: ECHO (WRAM secondary mapping)
- `$FE00` – `$FE9F`: Object Attribute Memory (OAM)
- `$FEA0` – `$FEFF`: Invalid OAM region (behavior varies per revision)
- `$FF00` – `$FF7F`: [Memory mapped I/O](#mmio)
- `$FF80` – `$FFFE`: High RAM (HRAM)
- `$FFFF`: [`IE` register](#mmio-if)

<a id="mmio">Memory mapped I/O</a>
---

- `$FF00` — `P1`/`JOYP`: [Joypad](#mmio-p1)
- `$FF01` — `SB`: [Serial byte](#mmio-sb)
- `$FF02` — `SC`: [Serial control](#mmio-sc)
- `$FF03`: unmapped
- `$FF04` — `DIV`: [Clock divider](#mmio-div)
- `$FF05` — `TIMA`: [Timer value](#mmio-tima)
- `$FF06` — `TMA`: [Timer reload](#mmio-tma)
- `$FF07` — `TAC`: [Timer control](#mmio-tac)
- `$FF08` – `$FF0E`: unmapped
- `$FF0F` — `IF`: [Interrupts asserted](#mmio-if)
- `$FF10` — `NR10`: [Audio channel 1 sweep](#apu-nr10)
- `$FF11` — `NR11`: [Audio channel 1 sound length/wave duty](#apu-nr11)
- `$FF12` — `NR12`: [Audio channel 1 envelope](#apu-nr12)
- `$FF13` — `NR13`: [Audio channel 1 frequency](#apu-nr13)
- `$FF14` — `NR14`: [Audio channel 1 control](#apu-nr14)
- `$FF15` — `NR20`: Unmapped
- `$FF16` — `NR21`: [Audio channel 2 sound length/wave duty](#apu-nr21)
- `$FF17` — `NR22`: [Audio channel 2 envelope](#apu-nr22)
- `$FF18` — `NR23`: [Audio channel 2 frequency](#apu-nr23)
- `$FF19` — `NR24`: [Audio channel 2 control](#apu-nr24)
- `$FF1A` — `NR30`: [Audio channel 3 enable](#apu-nr30)
- `$FF1B` — `NR31`: [Audio channel 3 sound length](#apu-nr31)
- `$FF1C` — `NR32`: [Audio channel 3 volume](#apu-nr32)
- `$FF1D` — `NR33`: [Audio channel 3 frequency](#apu-nr33)
- `$FF1E` — `NR34`: [Audio channel 3 control](#apu-nr34)
- `$FF1F` — `NR40`: Unmapped
- `$FF20` — `NR41`: [Audio channel 4 sound length](#apu-nr41)
- `$FF21` — `NR42`: [Audio channel 4 volume](#apu-nr42)
- `$FF22` — `NR43`: [Audio channel 4 frequency](#apu-nr43)
- `$FF23` — `NR44`: [Audio channel 4 control](#apu-nr44)
- `$FF24` — `NR50`: [Audio output mapping](#apu-nr50)
- `$FF25` — `NR51`: [Audio channel mapping](#apu-nr51)
- `$FF26` — `NR52`: [Audio channel control](#apu-nr52)
- `$FF27` – `$FF2F`: Unmapped
- `$FF30` – `$FF3F`: [Wave pattern](#apu-ch3)
- `$FF40` — `LCDC`: [LCD control](#ppu-lcdc)
- `$FF41` — `STAT`: [LCD status](#ppu-stat)
- `$FF42` — `SCY`: [Background vert. scroll](#ppu-scy)
- `$FF43` — `SCX`: [Background horiz. scroll](#ppu-scx)
- `$FF44` — `LY`: [LCD Y coordinate](#ppu-ly)
- `$FF45` — `LYC`: [LCD Y compare](#ppu-lyc)
- `$FF46` — `DMA`: [OAM DMA source address](#ppu-dma)
- `$FF47` — `BGP`: [Background palette](#ppu-bgp)
- `$FF48` — `OBP0`: [OBJ palette 0](#ppu-obp0)
- `$FF49` — `OBP1`: [OBJ palette 1](#ppu-obp1)
- `$FF4A` — `WY`: [Window Y coord](#ppu-wy)
- `$FF4B` — `WX`: [Window X coord](#ppu-wx)
- `$FF50` — [Boot ROM control](#mmio-bootrom)
- `$FFFF` — `IE`: [Interrupts enabled](#mmio-ie)

See [CGB-specific memory mapped I/O](#mmio-cgb) for additional registers in CGB mode.

This section describes the bit mappings of memory mapped I/O.

Mapping Key:

- `1`: Unmapped (hi-Z), always reads 1
- `I`: Input only (readable by CPU)
- `O`: Output only (writable by CPU)
- `B`: Bidirectional (read/write by CPU)

### <a id="mmio-p1">`$FF00` — `P1`/`JOYP`: Joypad</a>
- Mapping: `11BBIIII`

This register is used for reading joypad input. The hardware multiplexes the D-pad and the face buttons so only four buttons can be read at a time, based on the value of bits 4 and 5 (0 for select, 1 for deselect). Keys are pulled low (to 0) when pressed and high (to 1) when unpressed. If bits 4 and 5 are both low, then the combination of the D-pad and face buttons is ANDed (i.e. a 0 value implies one or both of the associated keys are pressed). If neither bit 4 or 5 is selected then all keys read 1.

| Bit | 4 low | 5 low  |
| --- | ----- | ------ |
| 0   | Right | A      |
| 1   | Left  | B      |
| 2   | Up    | Select |
| 3   | Down  | Start  |

Additionally, bits 4 and 5 are used for communicating with the SGB. See the [Super Game Boy section](#sgb) for more information.

### <a id="mmio-sb">`$FF01` — `SB`: Serial byte</a>
- Mapping: `BBBBBBBB`

TODO

### <a id="mmio-sc">`$FF02` — `SC`: Serial control</a>
- DMG Mapping: `B111111B`
- CGB Mapping: `B11111BB`

TODO

### <a id="mmio-div">`$FF04` — `DIV`: Clock divider</a>
- Mapping: `BBBBBBBB`

This register is incremented by the CPU clock. (TODO: explain at what rate)

Writing to this register sets it to 0.

### <a id="mmio-tima">`$FF05` — `TIMA`: Timer value</a>
- Mapping: `BBBBBBBB`

Increases at a rate specified by [`TAC`](#mmio-tac). When it overflows, a Timer IRQ is asserted, and this register is reloaded with [`TMA`](#mmio-tma)'s value.

The way the increment is performed in hardware causes spurious increments under certain conditions, refer to [Timer](#timer).

### <a id="mmio-tma">`$FF06` — `TMA`: Timer reload</a>
- Mapping: `BBBBBBBB`

When [TIMA](#mmio-tima) overflows, this register's contents are copied to it. Controls timer overflow frequency.

### <a id="mmio-tma">`$FF07` — `TAC`: Timer control</a>
- Mapping: `11111BBB`

Bits:
- 0–1: Select at which frequency [`TIMA`](#mmio-tima) increases
  - 0: CPU clock / 1024
  - 1: CPU clock / 16
  - 2: CPU clock / 64
  - 3: CPU clock / 256
- 2: [`TIMA`](#mmio-tima) doesn't increment when this bit is reset

### <a id="mmio-if">`$FF0F` — `IF`: Interrupts asserted</a>
- Mapping: `111BBBBB`

Bits:

- 0: VBlank
- 1: LCD STAT
- 2: Timer
- 3: Serial
- 4: Joypad

TODO

### <a id="ppu-lcdc">`$FF40` — `LCDC`: LCD control</a>
- Mapping: `BBBBBBBB`

Bits:
- 0: Background enable (on CGB: Background & window enable)
- 1: OBJ enable
- 2: OBJ size
- 3: Background tilemap
- 4: Background tiles
- 5: Window enable
- 6: Window tilemap
- 7: LCD enable

### <a id="ppu-stat">`$FF41` — `STAT`: LCD status</a>
- Mapping: `1BBBIII`

Bits:
- 0–1: LCD mode (see [Draw modes](#draw-modes))
- 2: [`LY`](#ppu-ly) coincidence flag
- 3–6: STAT interrupt selection
  - 3: Mode 0 (HBlank)
  - 4: Mode 1 (VBlank)
  - 5: Mode 2 (OAM scan)
  - 6: LY coincidence

Bits 4–6 select which sources are considered for the STAT interrupt. Selecting more than one source may trigger [STAT IRQ blocking](#ppu-stat-bug).

Note that on DMG, writing to this register may assert a STAT IRQ; refer to [STAT writing IRQ](#stat-write-irq).

### <a id="ppu-scy">`$FF42` — `SCY`: Background vert. scroll</a>
- Mapping: `BBBBBBBB`

TODO

### <a id="ppu-scx">`$FF43` — `SCX`: Background horiz. scroll</a>
- Mapping: `BBBBBBBB`

TODO

### <a id="ppu-ly">`$FF44` — `LY`: LCD Y coordinate</a>
- Mapping: `IIIIIIII`

Indicates which line the LCD is currently processing. Values 0-143 indicate VDraw, values 144-153 indicate VBlank.

Note that the LCD is in VBlank for part of line 0; see (TODO).

### <a id="ppu-lyc">`$FF45` — `LYC`: LCD Y compare</a>
- Mapping: `BBBBBBBB`

As long as [`LY`](#ppu-ly) has the same value as this register, [`STAT`](#ppu-stat) bit 2 is set.

### <a id="ppu-dma">`$FF46` — `DMA`: OAM DMA source address</a>
- Mapping: `BBBBBBBB`

When this register is written to, an OAM DMA transfer starts immediately.

If value $XY is written, the transfer will copy $XY00-$XY9F to $FE00-$FE9F.

### <a id="ppu-bgp">`$FF47` — `BGP`: Background palette</a>
- Mapping: `BBBBBBBB`

Defines how colors of BG (and the window) are displayed.

Bits 0 and 1 define color 0, bits 2 and 3 define color 1, etc. The two bits form a value, where 0 is white, 1 is light gray, 2 is dark gray, and 3 is black.

### <a id="ppu-obp0">`$FF48` — `OBP0`: OBJ palette 0</a>
- Mapping: `BBBBBBBB`

Defines how colors of OBJ using palette 0 are displayed.

The mapping is identical to [`BGP`](#ppu-bgp), however color 0 is never displayed (transparent), so the lower 2 bits are never considered.

### <a id="ppu-obp1">`$FF49` — `OBP1`: OBJ palette 1</a>
- Mapping: `BBBBBBBB`

Defines how colors of OBJ using palette 1 are displayed.

The mapping is identical to [`OBP0`](#ppu-obp0).

### <a id="ppu-wy">`$FF4A` — `WY`: Window Y coord</a>
- Mapping: `BBBBBBBB`

Defines the window's Y coordinate, that is, the first scanline on which the window is displayed.

TODO: what happens when changing mid-frame? Mid-scanline?

### <a id="ppu-wx">`$FF4B` — `WX`: Window X coord</a>
- Mapping: `BBBBBBBB`

Defines the window's X coordinate **plus 7**. (Therefore, a value of 7 will cause the window to span the entire scanline, and a value of 87 will only span half of the screen). Values larger than 167 cause the window to not be displayed.

Values 1-6 act as if the window started to the left of the screen. Value 0 has rather erratic behavior that depends on [`SCX`](#ppu-scx) (TODO: explain how)

TODO: what happens when changing mid-frame? Mid-scanline?

TODO: explain what happens when hiding the window every other scanline, for example

TODO: the official manual states that a `WX` value of $A6 (166) is "prohibited". Why?

### <a id="mmio-ie">`$FFFF` — `IE`: Interrupts enabled</a>
- Mapping: `111BBBBB`

Bits:

- 0: VBlank
- 1: LCD STAT
- 2: Timer
- 3: Serial
- 4: Joypad

TODO

## <a id="mmio-cgb">CGB-specific memory mapped I/O</a>

TODO

<a id="cart">Cartridges</a>
===

TODO

<a id="mbc">Memory bank controllers</a>
---

The Game Boy Advance has a 16-bit address space, the lower half of which (`$0000` – `$7FFF`), is devoted to cartridge ROM.
Further there is an a 8 kiB region (`$A000` – `$BFFF`), which is generally used for external RAM (often backed by a battery).
However, this leaves 32 kiB usable for ROM which is insufficient for most games.
This is where memory bank controllers (MBC for short, sometimes called mappers) come into play: using an MBC the allocated address space can be sliced up and swapped out.

The ROM address space is usually broken into two sections, `$0000` – `$3FFF` being a (usually) fixed mapping to the beginning of a ROM, or "bank 0", and `$4000` – `$7FFF` which can be programmatically swapped to point to other regions of ROM.
How this is handled depends on the specific MBC implementation.

Many MBCs share similar traits, namely:

- Writing a value to an address in the region `$0000` – `$1FFF` controls if external RAM is accessible. `$xA` enables access and other values disable access.
- Writing a value to an address in the region `$2000` – `$3FFF` controls which bank is swapped into the upper half of ROM address space.
- Writing a value to an address in the region `$4000` – `$5FFF` controls which bank is swapped into the external RAM address space.

None of these are universal, however. Please see the sections on the respective MBC for details.

A list of known MBC types is as follows:

- No MBC
	- `$00`
	- `$08`: RAM
	- `$09`: Battery-backed RAM
- [MBC1](#mbc-1):
	- `$01`
	- `$02`: RAM
	- `$03`: Battery-backed RAM
- [MBC2](#mbc-2):
	- `$05`: RAM
	- `$06`: Battery-backed RAM
- [MBC3](#mbc-3):
	- `$0F`: Battery-backed RTC
	- `$10`: Battery-backed RAM and RTC
	- `$11`
	- `$12`: RAM
	- `$13`: Battery-backed RAM
- MBC4 does not exist
- [MBC5](#mbc-5):
	- `$19`
	- `$1A`: RAM
	- `$1B`: Battery-backed RAM
	- `$1C`: Rumble
	- `$1D`: Battery-backed rumble
	- `$1E`: Battery-backed RAM and rumble
- [MBC6](#mbc-6):
	- `$20`: Battery backed RAM, flash memory
- [MBC7](#mbc-7):
	- `$22`: EEPROM, accelerometer
- [MMM01](#mbc-mmm01)
	- `$0B`
	- `$0C`: RAM
	- `$0D`: Battery-backed RAM
- [Pocket Camera](#mbc-pocketcam)
	- `$FC`
- [TAMA5](#mbc-tama5)
	- `$FD`: EEPROM, RTC, and alarm
- [HuC-1](#mbc-huc1)
	- `$FF`: IR sensor
- [HuC-3](#mbc-huc3)
	- `$FE`: IR sensor
- [Unlicensed mappers](#mbc-unlicensed)
	- (?): Sachen
	- (?): Sintax
	- (?): Vast Fame
	- N/A: [Wisdom Tree](#mbc-wisdom-tree)
	- TODO ...

### <a id="mbc-1">MBC1</a>

- Maximum ROM size: 128 banks (2 MiB) (?)
- Maximum RAM size: (?)

MBC1 is a common memory bank controller for the first few years of Game Boy games. It can be wired up to the cartridge differently sometimes giving it the illusion of being a different MBC type, usually referred to as MBC1-M. However, the physical chip is the same.

#### Bank swapping

MBC1 bank swapping is a bit unusual due to the fact that it has a high bank and a low bank that combine to make the actual bank number. The high and low bank are combined as such to arrive at the bank number for the `$4000` region bank:

`LO % MAPPING_SIZE + (HI % 4) * MAPPING_SIZE`

The bank for the `$0000` region can be swapped when the MBC is "external RAM mode" and wired as MBC1-M, which is controlled by writes to the `$6000` region. When in RAM mode, bank "0" is mapped as `(HI % 4) * MAPPING_SIZE`.

The mapping size is 32 for regular MBC1 cartridges and 16 for MBC1-M cartridges.

#### Read from `$0000` – `$3FFF`

Default mapped as bank 0. Can be swapped out with specific other banks based on "mode". See the Mapping section.

#### Read from `$4000` – `$7FFF`

Default mapped as bank 1. Can be swapped out with most other banks. See the Mapping section.

#### Read from `$A000` – `$BFFF`

External RAM. Access is disabled by default. Disabled reads are pulled high (`$FF`).

#### Write to `$0xxx` – `$1xxx`

Enable or disable external RAM access. Writing a value of `$xA` enables access, whereas any other value disables access. The high nybble is ignored.

#### Write to `$2xxx` – `$3xxx`

Set the low bank value; see the Mapping section. It cannot be zero and will be coerced to 1 if a value of 0 is written.

#### Write to `$4xxx` – `$5xxx`

Set the high bank value; see the Mapping section. This instead controls the external RAM bank when in RAM mode.

#### Write to `$6xxx` – `$7xxx`

Adjust mode based on the least significant bit. Writing `%…1` enables "external RAM mode" which allows swapping the RAM bank and the bank at `$0000`. Writing `%…0` disables RAM mode and switches the RAM and `$0000` ROM banks back to 0.

#### Write to `$A000` – `$BFFF`

Write a value to external RAM, if enabled. The value is ignored otherwise.

### <a id="mbc-2">MBC2</a>

TODO

### <a id="mbc-3">MBC3</a>

- Maximum ROM size: 128 banks (2 MiB) / 256 banks (4 MiB)
- Maximum RAM size: 4 banks (32 kB) / 8 banks (64 kB) for MBC30

MBC3 is one of the most common MBCs, along with MBC5. It's especially present in the DMG era. MBC3 improves upon MBC1 by removing the "banking mode", and can function with a RTC (Real-Time Clock, famously used in the second generation Pokémon games). The Japanese version of Pokémon Crystal contains a special MBC3 labelled MBC30, that can address twice the external RAM and twice the external ROM. (This is the only known difference; JP Crystal is also the only known cart to use MBC30)

#### Read from `$0000` – `$3FFF`

Always mapped as bank 0.

#### Read from `$4000` – `$7FFF`

Default mapped as bank 1. Can be swapped out with all other banks except bank 0. See the Mapping section.

#### Read from `$A000` – `$BFFF`

External RAM or RTC register. Access is disabled by default. Disabled reads are pulled high (`$FF`).

#### Write to `$0xxx` – `$1xxx`

Enable or disable external RAM access. Writing a value of `$xA` enables access, whereas any other value disables access. The high nybble is ignored.

#### Write to `$2xxx` – `$3xxx`

Set the ROM bank value; see the Mapping section.

#### Write to `$4xxx` – `$5xxx`

Set the RAM bank value, or select a RTC register; see the Mapping section.

#### Write to `$6xxx` – `$7xxx`

Latch RTC registers when writing $00 then $01. Latching the registers should be done before accessing them, but does not stop the clock.

#### Write to `$A000` – `$BFFF`

Write a value to external RAM or a RTC register, if enabled. The value is otherwise ignored.

### <a id="mbc-5">MBC5</a>

- Maximum ROM size: 512 banks (8 MiB) (?)
- Maximum RAM size: 16 banks (128 kB)

MBC5 is ome of the most common MBCs, along with MBC3. It's especially present in the CGB era, mostly because it is the first MBC to officially support CGB double-speed mode, at least according to Nintendo's manual. MBC5 improves upon MBC3 by expanding ROM and RAM capabilities (although only one game makes use of all 512 ROM banks, and a handful of all 16 RAM banks), as well as the aforementioned CGB double-speed support. MBC5 cannot work with a RTC, unlike MBC3, but can work with a rumble motor.

#### Bank swapping

MBC5 can address up to 512 banks, which doesn't fit the ROM bank value into a single byte. Therefore, that value is split into a low byte, and a high byte, which effectively contains only 1 bit (? If MBC5 can *actually* support more ROM banks, this may even be a full byte). Since most, or rather all but one, games use at most 256 banks, rendering this extra bit (? byte) unused. (TODO: for games not using this extra byte, what does writing to `$3xxx` do?)

#### Read from `$0000` – `$3FFF`

Always mapped as bank 0.

#### Read from `$4000` – `$7FFF`

Default mapped as bank 1. Can be swapped out with all other banks **including** bank 0. See the Mapping section.

#### Read from `$A000` – `$BFFF`

External RAM register. Access is disabled by default. Disabled reads are pulled high (`$FF`).

#### Write to `$0xxx` – `$1xxx`

Enable or disable external RAM access. Writing a value of `$xA` enables access, whereas any other value disables access. The high nybble is ignored (?).

#### Write to `$2xxx`

Set the low ROM bank value; see the Mapping section.

#### Write to `$3xxx`

Set the high ROM bank value; see the Mapping section.

#### Write to `$4xxx` – `$5xxx`

Set the RAM bank value; see the Mapping section. If the cartridge has rumble, bit 3 is mapped to rumble instead of RAM bank, i.e. `bank OR 8` enables rumble.

#### Write to `$A000` – `$BFFF`

Write a value to external RAM, if enabled. The value is otherwise ignored.

### <a id="mbc-6">MBC6</a>

MBC6 is special in that it has a 8 Mbit flash chip that can be mapped into ROM space, as well as having two separate ROM banked regions (`$4xxx` –`$5xxx` and `$6xxx` – `$7xxx`) and two SRAM banked regions (`$Axxx` and `$Bxxx`). It is only used in one game, Net de Get: Minigame @ 100.

TODO

### <a id="mbc-7">MBC7</a>

MBC7 is a special memory bank controller for the Game Boy Color that contains an 2-axis accelerometer (ADXL202E) and and a 256 byte EEPROM ([93LC56](http://www.microchip.com/wwwproducts/en/en010904)). It is notably used in the game Kirby's Tilt 'n' Tumble.

Accelerometer data must be latched before reading. Data is 16-bit and centered at the value `$81D0`. Earth's gravity affects the value by roughly `$70`, with larger acceleration providing a larger range. Maximum range is unknown.

Save data is accessed through a manually clocked shift register.

TODO

### <a id="mbc-mmm01">MMM01</a>

MMM01 is a "metamapper" in that it is used in game collection (Momotarō Collection 2 and Taito Variety Pack) to provide a boot menu before locking itself down into a separate "normal" mapper mode that only exposes certain banks to the game inside the collection.

TODO

### <a id="mbc-pocketcam">Pocket Cam</a>

TODO

### <a id="mbc-tama5">TAMA5</a>

TAMA5 is a custom MBC developed by Bandai and officially licensed by Nintendo that provides a small EEPROM, along with RTC and piezoelectric buzzer components, similar to HuC-3. It is an unusal MBC in that it only has two addresses with which it can communicate with the MBC. There are other TAMA chips on the PCB, which include TAMA6 and TAMA7 (the ROM). It is only used in one game, Game de Hakken!! Tamagotchi - Osutchi to Mesutchi, also known as Tamagotchi 3.

TODO

### <a id="mbc-huc1">HuC-1</a>

HuC-1 is a custom MBC developed by Hudson and officially licensed by Nintendo that provides infrared communication for games that predate the Game Boy Color, such as Pokémon Card GB, the Japanese version of Pokémon Trading Card Game. Oddly, the international versions do not use HuC-1 and instead rely on the GBC's IR port.

TODO

### <a id="mbc-huc3">HuC-3</a>

HuC-3 is a custom MBC developed by Hudson and officially licensed by Nintendo that extends HuC-1 with an RTC and a piezoelectric buzzer. It's notably used in the Robot Poncots (Robopon outside of Japan) and Pocket Family game series.

TODO

### <a id="mbc-unlicensed">Unlicensed mappers</a>

Several unlicensed games use unofficial mappers that are less well documented and may have copy protection.

#### <a id="mbc-wisdom-tree">Wisdom Tree</a>

Wisdom Tree is an American company devoted to very, very Christian games. Notable for unlicensed games including Super 3D Noah's Ark, they released games on a wide array of Nintendo platforms including the Game Boy. The Game Boy version uses a very simple mapper with the ROM region (`$0000` – `$7FFF`) remappable as a single monolithic block.

##### Read from `$0000` – `$7FFF`

Default mapped as bank 0. Can be swapped out with other banks.

##### Write to `$0000` – `$3FFF`

Swap the 32 KiB bank in the whole ROM region. Unlike most mappers the selected bank is based on the address written, not the value written.

<a id="ppu">Picture processing unit</a>
===

The Game Boy's picture processing unit is responsible for creating video frames. One frame takes 70224 cycles (~59.7275 frames per second).

Timing is divided into 154 lines, 144 during VDraw and 10 during VBlank. Each line takes 456 cycles.

The PPU can be disabled entirely during normal operation via the [`LCDC` register](#ppu-lcdc). This prevents normal timing such that re-enabling it can lead to irregular frame rates, at least transiently.

<a name="ppu-mode">Draw modes</a>
---
The PPU has 4 distinct modes:

- Mode 0: HBlank
- Mode 1: VBlank
- Mode 2: OAM scan
- Mode 3: HDraw

Modes 0, 2, and 3 do not occur during VBlank, only during VDraw. Mode transitions occur in this order:

- 1 → 2 (VBlank to VDraw transition)
- 2 → 3 (VDraw only)
- 3 → 0 (VDraw only)
- 0 → 1 (VDraw to VBlank transition)
- 0 → 2 (VDraw only)

**Different transitions can affect timing characteristics.**

### <a name="ppu-mode-0">Mode 0</a>

HBlank occurs between lines and allows a short period of time for adjusting PPU parameters, VRAM, and OAM.

#### Timing characteristics
Base timing: 204 cycles (max)

HDraw (mode 3) can steal cycles from HBlank depending on scrolling parameters (SCX), window parameters (WX), and OAM.

### <a name="ppu-mode-1">Mode 1</a>

VBlank occurs between frames and allows a short period of time for adjusting PPU parameters, VRAM, and OAM.

#### Timing characteristics
Exact timing: 4560 cycles (10 lines)

### <a name="ppu-mode-2">Mode 2</a>

Mode 2 occurs after HBlank and before HDraw. It is used to scan OAM to find which OBJs are active. During this time OAM is locked.

#### Timing characteristics
Exact timing: 80 cycles (2 cycles per OAM entry)

### <a name="ppu-mode-3">Mode 3</a>

HDraw is when pixels are drawn.

#### Timing characteristics
Base timing: 172 cycles (min)

- 1 cycle per BG pixel
- 6–11 cycles per active OBJ
- 0–7 extra cycles: `SCX % 8`
- TODO: describe PPU FIFO
- TODO: describe windows

<a id="apu">Audio processing unit</a>
===

TODO

<a id="timer">Timer</a>
===

TODO

<a id="dma">DMA</a>
===

TODO

<a id="irq">Interrupts</a>
===

The Game Boy contains 5 interrupt types. Each type has a different priority and vector:

| Type      | Priority | Vector  |
| --------- | -------- | ------- |
| VBlank    | 0        | `$0040` |
| LCD STAT  | 1        | `$0048` |
| Timer     | 2        | `$0050` |
| Serial    | 3        | `$0058` |
| Joypad    | 4        | `$0060` |

Interrupts are controlled through three registers:

- [`IF`](#mmio-if): interrupts asserted
- [`IE`](#mmio-ie): interrupts enabled
- `IME`: interrupt master enable

`IF` and `IE` are memory-mapped, whereas `IME` is controlled through the [`DI`](#cpu-di) (disable interrupts) and [`EI`](#cpu-ei) (enable interrupts) instructions.

Interrupt requests are edge triggered(_Is this true in all cases?_): when an IRQ is asserted, it sets the bit corresponding to its priority in the [`IF` register](#mmio-if). Interrupt *dispatch* however is level triggered: if at any time between instructions(_Unclear; which T state?_) while `IME` is enabled `IE AND IF != 0` IRQ dispatch will begin. Note that multi-M cycle instructions effectively block interrupt dispatch from starting until the end of the instruction.

Interrupt dispatch takes 4 M cycles. The interrupt vector is loaded into the program counter based on the lowest significant bit of `IE AND IF` after the program counter's old value has been pushed onto the stack, which takes two M cycles (one per byte, high byte first).

When interrupt dispatch completes the bit associated with the interrupt handled is automatically cleared from `IF` and `IME` is set to 0. Returning from the interrupt is handled by the `reti` instruction, which atomically pops the old program counter from the stack and reenables `IME`.

### <a id="irq-cancel">Interrupt dispatch canceling</a>

Interrupt dispatch is not atomic. If the value of `IE AND IF` changes between the first M cycle and when the vector is loaded, the value of the vector may change. This can happen if `IE` or `IF` is altered in some way, either by a higher priority IRQ asserting, or by another memory write (e.g. the first stack push overwriting `IE` or `IF`).

- If a higher priority IRQ is asserted, it effectively "steals" the interrupt dispatch
- If `IE AND IF` becomes zero, no interrupt vector can be located and instead `$0000` is loaded into the program counter.

<a id="sio">Serial I/O</a>
===

TODO

<a id="misc">Miscellaneous</a>
===

<a id="errata">Hardware issues</a>
---

- [OAM bug](#oam-bug)
- [Interrupt dispatch canceling](#irq-cancel)
- [STAT IRQ blocking](#ppu-stat-bug)
- [STAT writing IRQ](#stat-write-irq)

### <a id="oam-bug">OAM bug</a>

Under some circumstances, OAM may be corrupted.

Triggers are:
- Performing a 16-bit `inc` or `dec` (this includes the implicit `inc hl` performed by `ld [hli], a`, and the implicit `dec sp`s performed by `push de`, for example) with the affected register pointing to OAM (TODO: is this Mode-dependent? Does `ld hl, $FE00 ; dec hl` corrupt? Does `ld hl, $FEFF ; inc hl` corrupt?)
- Writing OAM while the PPU is reading it (TODO: does this include Mode 2, Mode 3, or both?)
TODO: are there other triggers?

TODO: explain how exactly OAM is corrupted.

The OAM bug is fixed on CGB.

### <a id="ppu-stat-bug">STAT IRQ blocking</a>

The STAT IRQ is edge-triggered, based on conditions selected in the [`STAT` register](#ppu-stat). A STAT IRQ is asserted when one or more selected conditions become true when none were true previously. Trouble arises between mode transitions, because the signal doesn't go low then high again, but stays low.

For example, enabling Mode 2 and Mode 0 sources will assert an IRQ on LY 0 Mode 2, then on LY 0 Mode 0, but not on LY 1 Mode 2 since both modes are contiguous.

NB: this does not occur with Mode 0 and LYC, because LYC is slightly late on Mode 2, so the signal does go low then high again. ([Source](https://twitter.com/liji32/status/1002867726476562432))

### <a id="stat-write-irq">STAT writing IRQ</a>

When writing to STAT on DMG, bits 3–6 are considered high for one cycle, which temporarily selects all conditions and may thus assert a STAT IRQ. CGB does not exhibit this bug, even in DMG mode.

<a id="sgb">Super Game Boy</a>
---

TODO

<a id="cheats">Cheat codes</a>
---

TODO

<a id="about">About this document</a>
===

![CC-BY-4.0](https://i.creativecommons.org/l/by/4.0/88x31.png)

This document is licensed under the [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/) (CC-BY 4.0)

- Copyright © 2018 – 2019, Vicki Pfau
