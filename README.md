# Elbrus architecture


## Overview

Elbrus 2000 (Elbrus or e2k for short), is a SPARC-inspired VLIW architecture
developed by the [Moscow Center for SPARC Technology (MCST)](http://mcst.ru/).

Elbrus machine code is organized into [very long instruction words (VLIW)](https://en.wikipedia.org/wiki/Very_long_instruction_word),
which consist of multiple so-called syllables that are executed together.

### References

Several useful documents about Elbrus are available on the internet, albeit
mostly in Russian.

- [Руководство по эффективному программированию на платформе «Эльбрус» (elbrus-prog)](http://ftp.altlinux.org/pub/people/mike/elbrus/docs/elbrus_prog/html/)
- [Микропроцессоры и вычислительные комплексы семейства «Эльбрус»](http://www.mcst.ru/doc/book_121130.pdf)


## Memory organization

Most operations in Elbrus code either:

- Take the values of one or more registers, compute a function, and write the
  result to another register, or
- Load a value from memory into a register or store a value from a
  register into memory.


### Register file, RF (Регистровый файл , РгФ)

The 256 general-purpose registers of the Register File (RF/РгФ) are divided
into two categories:

- 224 registers are part of the _procedure stack_ in a
  [windowed](https://en.wikipedia.org/wiki/Register_window) way. They can
  become available or unavailable during procedure calls and returns.
  (See also [elbrus-prog chapter 9.3.1.1](http://ftp.altlinux.org/pub/people/mike/elbrus/docs/elbrus_prog/html/chapter9.html#mech-register-window))
- 32 registers are global registers. They are available during the whole
  runtime of a program.

 32-bit | 64-bit | description
--------|--------|-----------------------------------
 `%g0`  |`%dg0`  | Global register (0-31)
 `%r0`  |`%dr0`  | Procedure stack register, relative to start of current window
 `%b[0]`|`%db[0]`| [Mobile base registers](http://ftp.altlinux.org/pub/people/mike/elbrus/docs/elbrus_prog/html/chapter9.html#baseregisters), relative to the start of the current window, plus `BR`

TODO: last eight global registers are designated rotating area


#### Changing the register window

The procedure stack contains parameters and local data of procedures. Its top
area is stored in the register file (RF). On overflow or underflow of the
register file, its contents are automatically swapped in/out of memory. Launch
of a new procedure allocates a window on the procedure stack, which may overlap
with the calling procedure's window.


### Procedure chain stack (стек связующей информации)

Stack of return addresses. It can only be manipulated by the operating system
and the hardware. Its top area is stored in CF (chain file) registers.

On this stack the following information is encoded in two quad words:
- return address
- compilation unit index descriptor (CUIR)
- window base (wbs) in the register file
- presence of real 80 (?)
- predicate file
- user stack descriptor
- rotating area base
- processor status register

On overflow or underflow of the chain file, its contents are automatically
swapped in/out of memory.


### Predicate file, PF (Предикатный файл, ПФ)

Comparison operations produce one-bit results (true or false) that can be
stored in the predicate registers.

Predicates can be used in conditional control transfers (jumps/calls), or in
the conditional execution of individual operations.

There are 32 predicate registers in the predicate file, which appear as
`%pred0` to `%pred31` in assembly code.


### Special purpose registers

Special purpose registers can be read using the `rrs` and `rrd` operations, and
writing using the `rws` and `rwd` operations.

 Name   | Description
--------|--------------------------------------------------------------
 CUIR   | compilation unit index register, индекс дескрипторов модуля компиляции
 PSHTP  | procedure stack hardware top pointer
 PSP    | procedure stack pointer - contains the virtual base address of the procedure stack.
 WD     | window descriptor - contains the base and the size of the current procedure's window into the procedure stack.
 PCSHTP | procedure chain stack hardware top pointer
 USBR   | user stack base pointer, РгБСП
 USD    | user stack descriptor, ДСП


## Regular Instructions

Elbrus' wide instructions (широкая команда, ШК) are comprised of a header syllable and zero or more additional syllables. Wide instructions are 8 byte aligned and up to 16 words (64 bytes) long.

### Syllables

Abbreviation | Description
-------------|---------------------------------------------------------
HS           | Header syllable - it encodes length and structure of a wide instruction
SS           | Stubs syllable - short operations that take only a few bits to encode
ALS          | Arithmetic logic channel syllable
CS           | Control syllable
ALES         | Arithmetic logic extension channel semi-syllable. They extend corresponding ALS. ALES2 and ALES5 are only available on Elbrus v4 and higher.
AAS          | Array access semi-syllable
LTS          | Literal syllable - literals to be used as operands
PLS          | Predicate logic syllable - processing of boolean values
CDS          | Conditional syllable - specified which operations are to be executed under which condition

The first syllable is the header syllable. It is always present.
Presence of other syllables depend on the purpose of the command.
Syllables occur in the following order:

- HS
- SS
- ALS0, ALS1, ALS2, ALS3, ALS4, ALS5
- CS0, CS1
- ALES2, ALES5
- ALES0, ALES1, ALES3, ALES4
- AAS0, AAS1, AAS2, AAS3, AAS4, AAS5
- LTS3, LTS2, LTS1, LTS0
- PLS2, PLS1, PLS0
- CDS2, CDS1, CDS0

#### Syllable packing

Semi-syllables ALES and AAS are a half-word (2 bytes) long. All other syllables are one word (4 bytes) long.

Syllables SS, ALS\* and CS\* occur as indicated in the header syllable in the order described above.
They are packed, e.g. if header bits indicate presence of ALS0 and ALS2 but not SS nor ALS1, then the syllable ALS0 follows directly after HS and ALS2 follows directly after ALS0.

If presence of ALES2 or ALES5 is indicated, then a whole word is allocated for them, whether both are present or not.
The first of both to be present occupies the more significant half of the word, the second is encoded in the less significant half.
For example, when looking at the syllables as bytes, if ALES2 and ALES5 are present, then the first two bytes of the little endian word contain ALES5 and the last two bytes contain ALES2.
If only ALES5 is present, the first two bytes are empty and the last two bytes contain ALES5.

ALES{0,1,3,4} and AAS\* start at the word indicated by the "middle pointer" from the header syllable. Their ordering is the same as for ALES2 and ALES5 (high half first, low half second) but they are all packed. This means that any two syllables of ALES{0,1,3,4} and AAS{0,1} may share a word. ALES\* may not share a word with AAS{2,3,4,5} because presence of the latter implies presence of AAS0 and/or AAS1.
For example, if ALES0, ALES1, ALES4, AAS0 and AAS2 are indicated, then they are encoded as ALES1, ALES0, AAS0, ALES4, two bytes left empty, and finally AAS2.

LTS\*, PLS\* and CDS\* are decoded starting from the end of the wide command.
CDS\* and PLS\* are not indicated by individual flags but rather by their number. For example, there cannot be a PLS2 without a PLS0 and PLS1.
LTS take any remaining words between the other syllables. For example, if after the AAS there are five words remaining in the wide command and two CDS and one PLS are indicated, then two words for LTS are left. They would be encoded as LTS1, LTS0, PLS0, CDS1, CDS0.

We do not know what happens if more syllables are indicated than there is space allocated or if syllables are encoded to overlap.

#### HS - Header syllable

Bit     | Name          | Description
------- | ------------- | -----------------------------------------------------
   31   | ALS5          | arithmetic-logic syllable 5 presence
   30   | ALS4          | arithmetic-logic syllable 4 presence
   29   | ALS3          | arithmetic-logic syllable 3 presence
   28   | ALS2          | arithmetic-logic syllable 2 presence
   27   | ALS1          | arithmetic-logic syllable 1 presence
   26   | ALS0          | arithmetic-logic syllable 0 presence
   25   | ALES5         | arithmetic-logic extension syllable 5 presence
   24   | ALES4         | arithmetic-logic extension syllable 4 presence
   23   | ALES3         | arithmetic-logic extension syllable 3 presence
   22   | ALES2         | arithmetic-logic extension syllable 2 presence
   21   | ALES1         | arithmetic-logic extension syllable 1 presence
   20   | ALES0         | arithmetic-logic extension syllable 0 presence
 19:18  | PLS           | number of predicate logic syllables
 17:16  | CDS           | number of conditional execution syllables
   15   | CS1           | control syllable 1 presence
   14   | CS0           | control syllable 0 presence
   13   | set_mark      |
   12   | SS            | stub syllable presence
   11   | --            | unused
   10   | loop_mode     |
   9:7  | nop           |
   6:4  |               | Length of instruction, in multiples of 8 bytes, minus 8 bytes
   3:0  |               | Number of words occupied by SS, ALS, CS, ALES2, ALES5 - called "middle pointer"

#### SS - Stubs syllable

##### Stubs syllable format 1 - SF1

Bit     | Name     | Description
--------|----------|----------------------------------------------
 31:30  | ipd      | instruction prefetch depth
   29   | eap      | end array prefetch
   28   | bap      | begin array prefetch
   27   | srp      |
   26   | vfdi     |
   25   | crp (?)  |
   24   | abgi     |
   23   | abgd     |
   22   | abnf     |
   21   | abnt     |
   20   | type     | type is 0 for SF1
   19   | abpf     |
   18   | abpt     |
   17   | alcf     |
   16   | alct     |
   15   |          | array access syllable 0 and 2 presence
   14   |          | array access syllable 0 and 3 presence
   13   |          | array access syllable 1 and 4 presence
   12   |          | array access syllable 1 and 5 presence
 11:10  | ctop     | `ctpr` number used in control transfer (`ct`) instructions
   9    | ?        |
   8:0  | ctcond   | condition code for control transfers (`ct`)

##### Stubs syllable format 2 - SF2

Bit     | Name     | Description
--------|----------|----------------------------------------------
 31:30  | ipd      | instruction prefetch depth
 29:28  |          | encodes invts and flushts, see below
   27   | srp (?)  |
   26   |          | encodes invts and flushts, see below
   25   | crp (?)  |
   20   | type     | type is 1 for SF2

`(ss >> 27 & 6) \| (ss >> 26 & 1)`   | Description
-----------------------------------|------------
 2 | `invts`
 3 | `flushts`
 6 | `invts ? %predN`, where N is `ss & 0x1f`
 7 | `invts ? ~ %predN`, where N is `ss & 0x1f`


##### `ct` condition codes

The condition code in the stubs syllable controls under which conditions a
control transfer operation is executed.

 Bit    | description
--------|--------------------------------------------------------------
  4:0   | Predicate number (from `pred0` to `pred31`)
  8:5   | Condition type

 Type |  syntax                       | description
------|-------------------------------|---------------------------------
   0  | --                            | never
   1  |                               | always
   2  | `? %pred0`                    | if predicate is true
   3  | `? ~ %pred0`                  | if predicate is false
   4  | `? #LOOP_END`                 |
   5  | `? #NOT_LOOP_END`             |
   6  | `? %pred0 \|\| #LOOP_END`     |
   7  | `? ~ %pred0 && #NOT_LOOP_END` |
   8  | (TODO, depends on syllable)   |
   9  | (TODO, depends on syllable)   |
  10  | (reserved)                    |
  11  | (reserved)                    |
  12  | (reserved)                    |
  13  | (reserved)                    |
  14  | `? ~ %pred0 \|\| #LOOP_END`   |
  15  | `? %pred0 && #NOT_LOOP_END`   |

`#LOOP_END` and `#NOT_LOOP_END` are sometimes spelled as `%LOOP_END` and `%NOT_LOOP_END`.

#### ALS - Arithmetic-logical syllables

Bit     | Description
------- | -------------------------------------------------------------
   31   | Speculative mode
 30:24  | Opcode
 23:16  | Operand 1
 15:8   | Operand 2
  7:0   | Operand 3

The formats and roles of the operands depends on the opcode.

The following formats are known:

 Operand format | description
----------------|-------------------------------------------------------
  src1          | source operand 1
  src2          | source operand 2 - can encode access to literal syllables (LTS)
  src3          | source operand 3
  dst           | destination - where to store the result of an operation
  dst2          | destination
  regnum        | register number
  opext         | opcode extension


Several combinations of operand formats are known:

 Format      | operand 1 | operand 2 | operand 3 | description
-------------|-----------|-----------|-----------|----------------------
 ALOPF1      | src1      | src2      | dst       | Arithmethic-logical operation format 1
 ALOPF2      | opext     | src2      | dst       | Arithmethic-logical operation format 2
 ALOPF3      | src1      | src2      | src3      | Arithmethic-logical operation format 3
 ALOPF5      | opext     | src2      | regnum    | Arithmethic-logical operation format 5
 ALOPF6      | regnum    | --        | dst       | Arithmethic-logical operation format 6
 ALOPF7      | src1      | src2      | dst2      | Arithmethic-logical operation format 7
 ALOPF8      | opext     | src2      | dst2      | Arithmethic-logical operation format 8
 ALOPF9      | opext     | opext     | dst       | Arithmethic-logical operation format 9
 ALOPF10     | opext     | opext     | src3      | Arithmethic-logical operation format 10

TODO: ALOPF11, ALOPF12, ALOPF13, ALOPF15, ALOPF16, ALOPF17, ALOPF21, ALOPF22


#### ALES - Arithmetic-logical extension syllables

Bit     | Description
------- | -------------------------------------------------------------
  15:8  | Opcode 2
   7:0  | Extension or src3

#### CS - Control syllables

CS0 and CS1 encode different operations.

 Syllable | pattern   | name   | description
----------|-----------|--------|----------------------------------------
  CS0     |`0xxxxxxx` | set\*  | setwd/setbn/setbp/settr
  CS0     |`300000xx` | wait   | wait for specified kinds of operations to complete
  CS0     |`4xxxxxxx` | disp   | prepare a relative jump in `ctpr1`
  CS0     |`5xxxxxxx` | ldisp  | prepare an array prefetch program (?) in `ctpr1`
  CS0     |`6xxxxxxx` | sdisp  | prepare a system call in `ctpr1`
  CS0     |`70000000` | return | prepare to return from procedure in `ctpr1`
  CS0     |`8xxxxxxx+`| --     | disp/ldisp/sdisp/return with ctpr2
  CS0     |`cxxxxxxx+`| --     | disp/ldisp/sdisp/return with ctpr3



##### set\*

The set\* operation sets several parameters related to register windows.
Most bits are encoded in the CS0 syllable itself, but some are also read from
the LTS0 syllable.

According to `ldis`, setwd is always performed, but settr, setbn, and setbp
have to be enabled by setting the corrsponding bits in CS0.

 Syl. | bit    | name        | description
------|--------|-------------|-----------------------------------------
 CS0  |     27 |enable settr |
 CS0  |     26 |enable setbn |
 CS0  |     25 |enable setbp |
 CS0  |  22:18 | setbp psz=x |
 CS0  |  17:12 | setbn rcur=x|
 CS0  |  11:6  | setbn rsz=x |
 CS0  |   5:0  | setbn rbs=x |
 LTS0 |     4  | setwd nfx=x |
 LTS0 |  11:5  | setwd wsz=x |


##### wait

 Bit    | name  | description
--------|-------|------------------------------------------------------
  5     |`ma_c` | wait for all previous memory access operations to complete
  4     |`fl_c` | wait for all previous cache flush operations to complete
  3     |`ls_c` | wait for all previous load operations to complete
  2     |`st_c` | wait for all previous store operations to complete
  1     |`all_e`| wait for all previous operations to issue all possible exceptions
  0     |`all_c`| wait for all previous operations to complete

##### disp/ldisp/sdisp/return

The `disp` operation prepares a jump to a different location by using one of
the control transfer preparation registers (`ctpr1` to `ctpr3`).

 bit    | description
--------|--------------------------------------------------------------
  31:30 | can be 1, 2, or 3 for `ctpr1`, `ctpr2`, or `ctpr3` respectively
  29:28 | can be 0, 1, 2, or 3, for `disp`, `ldisp`, `sdisp`, or `return` respectively
  27:0  | offset or system call number

For `disp` and `ldisp`, the offset is relative to the start of the current
instruction, and in multiples of eight bytes. For example, in an instruction at
`0x1000`, with CS0=`40000042`, we get `disp %ctpr1, 0x1210`.

`ldisp` is only allowed with `ctpr2`.

For `sdisp`, the system call number is not shifted. `CS0=6000001a` is
`sdisp %ctpr1, 0x1a`.

The `return` operation doesn't take an offset. The offset field should be zero
in this case.



### Operands

Operands to arithmetic-logical operations can encode different kinds of
registers and literals.

Pattern   | Range | Applicability | Description
----------|-------|---------------|------------------------------------
0xxx xxxx | 00-7f |               | Rotatable area procedure stack register
10xx xxxx | 80-bf |               | procedure stack register
1100 xxxx | c0-cf |               | constant between 0 and 15
1101 xxxx | d0-df | not in src2   | constant between 16 and 31
1101 0xxx | d0-d7 | only in src2  | reference to 16 bit literal semi-syllable; (1<<2) indicates high half of a LTS; Only in lts0 and lts1.
1101 10xx | d8-db | only in src2  | reference to 32 bit literal syllable LTS0, LTS1, LTS2, or LTS3
1101 11xx | dc-df | only in src2  | reference to 64 bit literal syllable pair (LTS0 and LTS1, LTS1 and LTS2, LTS2 and LTS3)
111x xxxx | e0-ff |               | global register
1111 1xxx | f8-ff |               | Rotatable area global register

## Array Prefetch Instructions

Array prefetch instructions are run asynchronously on the array access unit.
They are always 16 bytes long.
To write array prefetch instructions, the mnemonic `fapb` is used.
To call an array prefetch program, load its address with ldisp to %ctpr2 (no need to call or ct).
Even though array prefetch instructions should only ever be called by ldisp and are not processed using the same facilities as
regular instructions, they always seem to be terminated by a regular branch instruction.
The maximum length of an array prefetch program is 32 instructions.
