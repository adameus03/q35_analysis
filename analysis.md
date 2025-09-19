Use `sudo flashrom -V -r test2 -c MX25L1605 -p ch341a_spi` to dump the bios2 chip of the ga-q35m-s2 board.

```
mundus@pc6:~/base/flashrom$ sudo flashrom -V -r test2 -c MX25L1605 -p ch341a_spi
flashrom unknown on Linux 6.1.0-37-amd64 (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
flashrom was built with GCC 12.2.0, little endian
Command line (7 args): flashrom -V -r test2 -c MX25L1605 -p ch341a_spi
Initializing ch341a_spi programmer
Device revision is 3.0.4
The following protocols are supported: SPI.
Probing for Macronix MX25L1605, 2048 kB: compare_id: id1 0xc2, id2 0x2015
Added layout entry 00000000 - 001fffff named complete flash
Found Macronix flash chip "MX25L1605" (2048 kB, SPI) on ch341a_spi.
Chip status register is 0x00.
Chip status register: Status Register Write Disable (SRWD, SRP, ...) is not set
Chip status register: Bit 6 is not set
Chip status register: Bit 5 is not set
Chip status register: Block Protect 2 (BP2) is not set
Chip status register: Block Protect 1 (BP1) is not set
Chip status register: Block Protect 0 (BP0) is not set
Chip status register: Write Enable Latch (WEL) is not set
Chip status register: Write In Progress (WIP/BUSY) is not set
===
This flash part has status UNTESTED for operations: WP
The test status of this chip may have been updated in the latest development
version of flashrom. If you are running the latest development version,
please email a report to flashrom@flashrom.org if any of the above operations
work correctly for you with this flash chip. Please include the flashrom log
file for all operations you tested (see the man page for details), and mention
which mainboard or programmer you tested in the subject line.
Thanks for your help!
Reading flash... done.
```

Read the reset vector (at bios dump offset 0x1FFFF0, 16th byte counting back from the dump end):
```
mundus@localhost:~/flashrom$ objdump -D -b binary -m i386 -M i8086,suffix --start-address=0x1FFFF0 test2
 
test2:     file format binary


Disassembly of section .data:

001ffff0 <.data+0x1ffff0>:
  1ffff0:	e9 de fa             	jmpw   0x1ffad1
  1ffff3:	00 00                	addb   %al,(%bx,%si)
  1ffff5:	2a 4d 52             	subb   0x52(%di),%cl
  1ffff8:	42                   	incw   %dx
  1ffff9:	2a 02                	subb   (%bp,%si),%al
  1ffffb:	00 00                	addb   %al,(%bx,%si)
  1ffffd:	00 60 e3             	addb   %ah,-0x1d(%bx,%si)

```

Calculate the next instruction address relative to the bios dump file beginning:
```
mundus@localhost:~/flashrom$ cat calc.c 
#include <stdint.h>
#include <stdio.h>

int main() {
  uint32_t reset_vector_addr = 0x1ffff0;
  uint32_t reset_vector_instr_len = 0x3;
  int16_t jump_relative_target = 0xfade;
  uint32_t next_instruction_addr = reset_vector_addr + reset_vector_instr_len + jump_relative_target;
  printf("Reset vector address: 0x%x\n", reset_vector_addr);
  printf("Next instruction address: 0x%x\n", next_instruction_addr);
  return 0;
}
mundus@localhost:~/flashrom$ gcc calc.c -o calc
mundus@localhost:~/flashrom$ ./calc
Reset vector address: 0x1ffff0
Next instruction address: 0x1ffad1
```

See what comes at the next instruction address:
```
mundus@localhost:~/flashrom$ objdump -D -b binary -m i386 -M i8086,suffix --start-address=0x1ffad1 test2 | head -20

test2:     file format binary


Disassembly of section .data:

001ffad1 <.data+0x1ffad1>:
  1ffad1:	8c d9                	movw   %ds,%cx
  1ffad3:	8b fa                	movw.s %dx,%di
  1ffad5:	66 b8 f0 f8 00 80    	movl   $0x8000f8f0,%eax
  1ffadb:	ba f8 0c             	movw   $0xcf8,%dx
  1ffade:	66 ef                	outl   %eax,(%dx)
  1ffae0:	83 c2 04             	addw   $0x4,%dx
  1ffae3:	66 ed                	inl    (%dx),%eax
  1ffae5:	66 8b d8             	movl.s %eax,%ebx
  1ffae8:	66 b8 01 00 0d 00    	movl   $0xd0001,%eax
  1ffaee:	66 ef                	outl   %eax,(%dx)
  1ffaf0:	be 00 00             	movw   $0x0,%si
  1ffaf3:	b8 00 d0             	movw   $0xd000,%ax
  1ffaf6:	8e d8                	movw   %ax,%ds
```

As per https://www.intel.com/Assets/PDF/datasheet/316966.pdf section 4.5, 0xcf8 corresponds to the Q35 north bridge I/O mapped register CONFIG_ADDRESS.   
`outl %eax, (dx)` writes 0x8000f8f0 into that register.  
We can also see the `addw $0x4, %dx` and `inl (%dx), %eax` instructions which make us interested in 0xcfc too. The same 4.5 section gives information that this corresponds to reading the CONFIG_DATA register.  
The CONFIG_ADDRESS and CONFIG_DATA registers are used to implement configuration space access mechanism (section 3.10). Let's find out what is being configured specifically.  

Based on sections 4.5.1 and 4.5.2, we can see that the program obtains the Configuration Data Window (CDW) by doing the following
- Write CONFIG_ADDRESS
  - Configuration Enable (CFGE) = 1
  - Bus Number = 0
  - Device Number = 31
  - Function Number = 0
  - Register Number = 0xf0 
- Read CONFIG_DATA
  - The whole 31:0 value is the CDW window.

Reading from CONFIG_DATA involves the (G)MCH north bridge producing a configuration transaction using the values set in CONFIG_ADDRESS, because the CFGE bit is set to 1 by `movl $0x8000f8f0,%eax` along with other mentioned CONFIG_ADDRESS parameters. Let's find out what particular device and register is involved based on this information.  
Based on section 4.5.1 we know, there's a DMI Type 0 configuration cycle generated because Bus Number = 0 and Device Number >= 2.    
As DMI is constitutes an interface between northbridge and southbridge, we shall check [the Intel's ICH9 datasheet](https://www.intel.com/content/dam/doc/datasheet/io-controller-hub-9-datasheet.pdf), as that's what the ga-q35m-s2 chipset uses. 
Section 13.1 of that datasheet reveals the target register D31:F0 is the Root Complex Base Address (RCBA) register - a register in the ICH9 chip.  

Here's what we know so far:
```
1ffad1:   8c d9                   movw   %ds,%cx
1ffad3:   8b fa                   movw.s %dx,%di
1ffad5:   66 b8 f0 f8 00 80       movl   $0x8000f8f0,%eax
1ffadb:   ba f8 0c                movw   $0xcf8,%dx
1ffade:   66 ef                   outl   %eax,(%dx) # enables the configuration space for D31:F0 (function 0) using the north bridge. Precisely we are targeting the RCBA register of the north bridge
1ffae0:   83 c2 04                addw   $0x4,%dx
1ffae3:   66 ed                   inl    (%dx),%eax # obtain a configuration data window for the RCBA register 
1ffae5:   66 8b d8                movl.s %eax,%ebx # save the CDW for later
1ffae8:   66 b8 01 00 0d 00       movl   $0xd0001,%eax
1ffaee:   66 ef                   outl   %eax,(%dx)                                                   
1ffaf0:   be 00 00                movw   $0x0,%si                                                     
1ffaf3:   b8 00 d0                movw   $0xd000,%ax                                              
1ffaf6:   8e d8                   movw   %ax,%ds
```

Now, what does the next `outl %eax,(%dx)` do?  
Based on section 13.1.35 of the ICH9 datasheet and the fact that we are effectively sending 0xd0001 to the south bridge, the following action is done:
- Write RCBA
  - Base Address = 0xd0000 (the 31:14 bits (18 bits) of 0x000d0001 are left-padded with zeroes to get a 32-bit value left shifted by 14)
  - Enable = 1 

So now, the offsets in the chipset configuration registers memory map (as specified in ICH9 datasheet section 10.1) are prepared to be treated as relative to physical address 0xd0000 (we get a clue about this from the upcoming `movw $0x4,0x3410(%si)` instruction (see below) and seeking the 3410 offset in the ICH9 datasheet).  
What happens next? We'll analyse the further instructions as much as we can with the help of the 10.1 chipset configuration registers documentation in the ICH9 datasheet).

```
1ffad1:	8c d9                	movw   %ds,%cx # Save ds for restoring later
1ffad3:	8b fa                	movw.s %dx,%di # Save dx for restoring later
1ffad5:	66 b8 f0 f8 00 80    	movl   $0x8000f8f0,%eax
1ffadb:	ba f8 0c             	movw   $0xcf8,%dx
1ffade: 66 ef                   outl   %eax,(%dx) # enables the configuration space for D31:F0 (function 0) using the north bridge. Precisely we are targeting the RCBA register of the north bridge
1ffae0: 83 c2 04                addw   $0x4,%dx
1ffae3: 66 ed                   inl    (%dx),%eax # obtain a configuration data window for the RCBA register
1ffae5: 66 8b d8                movl.s %eax,%ebx # save the CDW to enable restoring it later
1ffae8:	66 b8 01 00 0d 00    	movl   $0xd0001,%eax
1ffaee:	66 ef                	outl   %eax,(%dx) # Enable RCBA base address = 0xd0000
1ffaf0:	be 00 00             	movw   $0x0,%si 
1ffaf3:	b8 00 d0             	movw   $0xd000,%ax
1ffaf6:	8e d8                	movw   %ax,%ds
1ffaf8:	80 8c 10 34 04       	orb    $0x4,0x3410(%si) # Set Reserved Page Route (RPR) bit of the General Control and Status Register (GCS) - Configure the reservered page registers to have their writes forwarded to PCI, be shadowed within the ICH, and the reads will be returned from that internal shadow. (see ICH9 datasheet section 10.1.75)
1ffafd:	8a 84 11 34          	movb   0x3411(%si),%al
1ffb01:	24 0c                	andb   $0xc,%al
1ffb03:	3c 08                	cmpb   $0x8,%al # check if Boot BIOS Straps (BBS) bits of the  GCS chipset configuration register are 10 - checks if the destination of accesses to the BIOS memory range is PCI (not SPI and not LPC). See ICH9 datasheet section 10.1.75  
1ffb05:	75 12                	jne    0x1ffb19 # If it's not PCI, we skip the below PCI-specific code that is for disabling legacy ranges decoding (as you can see below).
1ffb07:	66 b8 d8 f8 00 80    	movl   $0x8000f8d8,%eax
1ffb0d:	ba f8 0c             	movw   $0xcf8,%dx
1ffb10:	66 ef                	outl   %eax,(%dx) # enable configuration space for D31:D8 function 0 using the north bridge. We are targetting the Firmware Hub Decode Enable Register (FWH_DEC_EN1)
1ffb12:	83 c2 04             	addw   $0x4,%dx
1ffb15:	ec                   	inb    (%dx),%al
1ffb16:	24 3f                	andb   $0x3f,%al
1ffb18:	ee                   	outb   %al,(%dx) # Disable decoding legacy 64KB ranges at F0000h-FFFFFh and E0000h-EFFFFh by setting FWH_Legacy_F_EN = 0 and FWH_Legacy_E_EN = 0
1ffb19:	66 b8 f0 f8 00 80    	movl   $0x8000f8f0,%eax
1ffb1f:	ba f8 0c             	movw   $0xcf8,%dx
1ffb22:	66 ef                	outl   %eax,(%dx) 
1ffb24:	83 c2 04             	addw   $0x4,%dx
1ffb27:	66 8b c3             	movl.s %ebx,%eax
1ffb2a:	66 ef                	outl   %eax,(%dx) # Reset the Root Complex Base Address Register to the default value of 0x00000000 (disables back the chipset configuration registers memory mapping)
1ffb2c:	8b d7                	movw.s %di,%dx # Restore back dx
1ffb2e:	8e d9                	movw   %cx,%ds # Restore back ds
1ffb30:	ea 5b e0 00 f0       	ljmpw  $0xf000,$0xe05b # Long jump, who knows where?
```

So we've got to a clear milestone, the `ljmpw  $0xf000,$0xe05b` long jump instruction which jumps to the physical address 0xfe05b. What will we find there? 
