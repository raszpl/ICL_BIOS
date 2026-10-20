# ICL CL386S BIOS Version 1.20 (920525) reverse engineering.
Disassembly to assist with debugging of broken [ICL CL386S](https://theretroweb.com/motherboards/s/icl-cl386s,-cl386s-25,-cs386s,-cs386s-25) board [Vogons: ICL CL386SX w/integrated ATi Wonder XL – No post & BIOS Confusion](https://www.vogons.org/viewtopic.php?t=110628) .

For a 386 from 1993 this board shipped with quite an elaborate and safe firmware update mechanism. There are two ROM chips present and a mechanism to hot switch between them. 32KB M27C256B EPROM containing ~2KB rescue Boot Block Flasher and a 128KB Intel N28F010 EEPROM filled with VGA bios from the top and main BIOS from the bottom. Switching is performed by a PAL16L8 somehow controlled by 'ICL SUPER I/O' chip thru 120/121h IO ports.

# BootBlk Flash Boot Loader
Boot Block loader is build by stripping original bios to the bones, can tell by some remnants like
- IVT int 11h handler int_11h_BIOS_Equipment_Flags_UNUSED left unused and accessing non existent BDA value
- default int_ack_isr trying to update ds:BDA_6Bh_last_irq, except BootBlk firmware doesnt have normal BDA and 0x6A offset is already taken by Root_Dir_Buffer_Segment word. The thing works without crashing and burning only because PIT is carefully initialized to only enable floppy interrupt and nothing else, otherwise one spurious interrupt (yes, those were a thing) could overwrite Root_Dir_Buffer_Segment in the middle of scanning for files :)

BootBlk reports its progress using LPT port like a POST card.
I still havent figured out how it decides if main bios chip is corrupted in need of re-flashing.
I think there is some code detecting jumpers on LPT port allowing external input, not sure about that yet tho.
BootBlk most likely switches which ROM chip is mapped and where using writes to IO ports 120/121h. 100% sure it can switch between mapping 27C256 to F000:8000h and 28F010 to E000:0000h. There is also possibility it can split map starting 32KB of 28F010 to C000:0000h and last 64KB to F000:0000h (maybe even together with E000:8000h for a bigger 96KB Bios), will know for sure when I disassemble main bios.

# BootBlk recovery process
1. Format standard 720KB or 1.4MB floppy. Only 9 and 18 sectors per track supported.
2. Rename VGA_BIOS.BIN to VGA74_vv.LDB or VGA74_.LDB
3. Rename BIOS_ROM.BIN to B22_vvvv.LDB or B22_.LDB
4. Put those two files in floppy root directory
5. Boot with floppy in drive plugged/connected as A/first

There will be some short beeps followed by flashing firmware to Intel chip. Dont have real hardware to verify.  
Sidenote: Anything in filenames between '_' and '.' is ignored, its their magic wildcard for version markers like B58_c.ldb B56_F.ldb.

# BIOS
Havent gotten around to disassembling it yet.

# BIOS lineage
ICL didnt design this system. Afaik they had no x86 PCs before 1991. [ICL merged with Nokia Data in 1991](https://www.latimes.com/archives/la-xpm-1991-06-03-fi-254-story.html). Nokia merged with Ericsson Information Systems (EIS) creating Nokia Data in 1988. Earliest bios sharing codebase with CL386S I could find is in 1988 [Ericsson 6573](https://theretroweb.com/motherboards/s/ericsson-6573).  
History goes deeper with Saab selling Linköping based Datasaab to Ericsson in 1981, but thats pre x86 PC :)

# BIOS EEPROM chip contents
128KB Intel N28F010. ATI VGA bios 'NVGA2i VGABIOS Version K' starts at the top and takes 32KB, main 64KB BIOS 'ROM BIOS #22  Version 1.20' lines up with end of ROM. There is provision for stuffing more main bios code just below VGA bios.  
Special Header, whose address is encoded at Far Pointer stored in Flash offset 1F850h (F000:F850) pointing at E000:8000 in Bios Version 1.20, ZZOVES_header:
- string 'ZZOVES'
- int16 number of times chip was re-flashed
- int16 size of main bios in 16B chunks
- 22 BYTE gap
- string 'BootBlk'
- 7 BYTE gap
- int16 0x107 magic number taken from Flash Boot Loader firmware, maybe BootBlk version 1.07?
- string CCYYMMDDHHmm timestamp 198001010022
- int16 CRC-16 of optional E8040-EFFFF 32KB minus size of ZZOVES_header (64 bytes). In ver 1.20 its unused and filled with FFh. `reveng.exe -m CRC-16/CCITT-FALSE -c -f erased.BIN` = 09f3 match
- int16 CRC-16 of main bios F0000-FFFFF 64KB `reveng.exe -m CRC-16/CCITT-FALSE -c -f BIOS_ROM.BIN` = 6ea2 match

# BootBlk Flash Boot Loader (1.07?) source code
Disassemble progress 100%, annotation ~60%. [Listing](https://raw.githubusercontent.com/raszpl/ICL_BIOS/refs/heads/main/ICL%20CL386SX%20M27C256B%40DIP28.BIN.lst), very incomplete [IDA C style Header file](https://raw.githubusercontent.com/raszpl/ICL_BIOS/refs/heads/main/ICL%20CL386SX%20M27C256B%40DIP28.BIN.h). [IDA 9.3 Free i64 database](https://raw.githubusercontent.com/raszpl/ICL_BIOS/refs/heads/main/ICL%20CL386SX%20M27C256B%40DIP28.BIN.i64).
