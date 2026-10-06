# ICL CL386S BIOS Version 1.20 (920525) reverse engineering.
Disassembly to assist with debugging of broken [Vogons: ICL CL386SX w/integrated ATi Wonder XL – No post & BIOS Confusion](https://www.vogons.org/viewtopic.php?t=110628) board [ICL CL386S](https://theretroweb.com/motherboards/s/icl-cl386s,-cl386s-25,-cs386s,-cs386s-25).

For a 386 from 1993 this board shipped with quite an elaborate and safe firmware update mechanism. There are two ROM chips present and a mechanism to hot switch between them, 32KB M27C256B containing ~2KB rescue Boot Block Flasher and a 128KB Intel N28F010 filled with VGA bios from the top and main BIOS from the bottom. Switching is performed by a  PAL16L8 somehow controlled by 'ICL SUPER I/O' chip thru 120/121h IO ports.

# BootBlk Flash Boot Loader
Boot Block loader is build by stripping original bios to the bones, can tell by some remnants like IVT int 11h handler int_11h_BIOS_Equipment_Flags_UNUSED left unused and accessing non existent BDA value, or default int_ack_isr trying to update ds:BDA_6Bh_last_irq except BootBlk firmware doesnt have normal BDA and 0x6A offset is already taken by Root_Dir_Buffer_Segment word. The thing works without crashing and burning only because PIT is carefully initialized to only enable floppy interrupt and nothing else, otherwise one spurious interrupt (yes, those were a thing) could overwrite Root_Dir_Buffer_Segment in the middle of scanning for files :)
BootBlk reports its progress using LPT port like a POST card.
I still havent figured out how it decides if main bios chip is corrupted in need of re-flashing.
I think there is some code detecting jumpers on LPT port allowing external input, not sure about that yet tho.
BootBlk most likely switches which ROM chip is mapped and where using writes to IO ports 120/121h. 100% sure it can map 27C256 to F000:8000h and 28F010 to E000:0000h. There is also possibility it can split map starting 32KB of 28F010 to C000:0000h and last 64KB to F000:0000h, will know for sure when I disassemble main bios.

# BIOS
Havent gotten around to disassembling it yet.
Bios is stored in 128KB Intel N28F010. ATI VGA bios 'NVGA2i VGABIOS Version K' starts at the top and takes 32KB, main 64KB BIOS 'ROM BIOS #22  Version 1.20' lines up with end of ROM. There is also a special marker, whose address is encoded at Far Pointer stored in Flash offset 1F850h (E000:8000), containing:
- string 'ZZOVES'
- word number of times chip was re-flashed
- word size of main bios in 16B chunks
- string 'BootBlk' string
- word 0x107 magic number taken from Flash Boot Loader firmware, maybe BootBlk version 1.07?
- string timestamp CCYYMMDDHHmm
- word CRC-16 of video bios
- word CRC-16 of main bios

