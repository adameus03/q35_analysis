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

Read the reset vector (at physical address 0x1FFFF0, 16th byte counting back from the last address of the physical addressing space):
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

As per https://www.intel.com/Assets/PDF/datasheet/316966.pdf section 4.5, 0xcf8 coresponds to the Q35 north bridge I/O mapped register CONFIG_ADDRESS.   
`outl %eax, (dx)` writes 0x8000f8f0 into that register.  
We can also see the `addw $0x4, %dx` and `inl (%dx), %eax` instructions which make us interested in 0xcfc too. The same 4.5 section gives information that this corresponds to reading the CONFIG_DATA register.
