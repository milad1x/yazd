# YAZD - Yet Another Z80 Disassembler

YAZD is a simple command line disassembler for Z80 binary code files.  It's based on the disassembler in [z80ex](http://z80ex.sourceforge.net/), ported to C#.

YAZD supports the following:

* Disassembly of all Z80 instructions, as supported by z80ex.
* Code path analysis can usually tell the difference between code and data.
* Generates labelled assembly language listings.
* Can also generate more detailed listing files with byte code and assembly source.
* Can generate reference listings to all external addresses and I/O ports.
* Can highlight all word literals (use to help find other memory address references).
* Can generate plain text, or hyperlinked HTML output files.
* Data segments are listed 1 DB byte per line with ASCII character in comments.


## Usage

	yazd source.bin [destination.asm] [options] [@responsefile]

Options:

	--addr:N               Z-80 base address of first byte after header, default=0x0000
	--start:N              Z-80 address to disassemble from, default=addr
	--end:N                Z-80 address to stop at, default=eof
	--len:N                Number of bytes to disassemble (instead of --end)
	--entry:N              Specifies an entry point (see below)
	--xref                 Include referenced locations of labels
	--lst                  Generate a listing file (more detail, can't be assembled)
	--html                 Generates a HTML file, with hyperlinked references
	--open                 Automatically opens the generated file with default associated app
	--lowercase|lc         Render in lowercase
	--markwordrefs|mwr     Highlight with a comment literal word values (as they may be addresses)
	--reloffs              Show the offset of relative address mode instructions
	--header:N             Skip N header bytes at start of file
	--help                 Show these help instruction
	--v                    Show version information

Numeric arguments can be in decimal (no prefix) or hex if prefixed with '0x'.

If one or more `--entry` arguments are specified (recommended), the file is disassembled by
following the code paths from those entry points.  All unvisited regions will be rendered
as 'DB' directives.

If the `--entry` argument is not specified, the file is disassembled from top to bottom.

Output is sent to stdout if no destination file specified.

Response file containing arguments can be specified using the @ prefix

Example:

    yazd --addr:0x0400 --entry:0x1983 -lst robotf.bin robotf.lst

## Example Output

Typical source mode output:

	        ORG     0900h

	        ; Entry Point
	L0900   LD      A,0Ah
	        OUT     (0Ch),A
	        LD      A,2Fh           ; '/'
	        OUT     (0Dh),A
	        LD      HL,0F800h
	        LD      C,80h
	L090D   LD      E,C
	        LD      D,03h
	L0910   XOR     A
	        BIT     0,E
	        JR      Z,L0917 
	        OR      0F0h
	L0917   BIT     1,E
	        JR      Z,L091D 
	        OR      0Fh
	L091D   LD      B,05h
	        LD      (HL),A
	        INC     HL
	        DJNZ    L091F   
	        RRC     E
	        RRC     E
	        DEC     D
	        JR      NZ,L0910 
	        LD      (HL),A
	        INC     HL
	        INC     C
	        JR      NZ,L090D 
	        JP      L1042

	L0932   PUSH    BC
	        LD      C,A
	        LD      B,A
	        LD      A,12h
	        OUT     (0Ch),A
	        LD      A,B
	        RRCA
	        RRCA
	        RRCA
	        RRCA

Unreachable addresses are automatically rendered as DB directives:

	L102E   LD      DE,1037h
	        PUSH    DE
	        PUSH    DE
	        PUSH    DE
	        JP      L1000

	L1037   DB      3Eh             ; '>'
	        DB      01h
	        DB      0F5h
	        DB      0DBh
	        DB      02h
	        DB      0CBh
	        DB      0F7h
	        DB      0D3h
	        DB      02h
	        DB      0F1h
	        DB      0C9h

	L1042   DI
	        LD      SP,8000h
	        CALL    L0BF0
	        CALL    L0A6E
	        CALL    L0A51

The `--xref` option, causes the referencing locations of any label to also be included:


	        ; Referenced from 09BEh
	L09C4   LD      (HL),8Ch
	        JR      L09CE

	        ; Referenced from 09BAh
	L09C8   LD      (HL),0B0h
	        JR      L09CE

	        ; Referenced from 09B6h
	L09CC   LD      (HL),80h

	        ; Referenced from 09C2h, 09CAh, 09C6h
	L09CE   INC     A
	        CALL    L0FB2
	        AND     03h
	        ADD     HL,DE
	        DJNZ    L09B5
	        LD      A,(L0A50)
	        LD      B,40h           ; '@'
	        CP      03h

In listing mode (`--lst`), the address and byte code is included on the left:

	                                        ; Entry Point
	0900: 3E 0A                      L0900  LD      A,0Ah
	0902: D3 0C                             OUT     (0Ch),A
	0904: 3E 2F                             LD      A,2Fh   ; '/'
	0906: D3 0D                             OUT     (0Dh),A
	0908: 21 00 F8                          LD      HL,0F800h
	090B: 0E 80                             LD      C,80h
	090D: 59                         L090D  LD      E,C
	090E: 16 03                             LD      D,03h
	0910: AF                         L0910  XOR     A

Listing mode also causes the generation of reference information, including references to external memory addresses:

	references to external address 8009:
	        0F18 CALL 8009h
	        0F43 CALL 8009h
	        0F66 CALL 8009h
	        21E7 CALL 8009h
	        236E CALL 8009h
	        2373 CALL 8009h

	references to external address F0E2:
	        2354 LD (0F0E2h),A

Possible address references are also listed.  This includes any instruction that uses a literal word value.

	possible references to internal address 0900:
	        200E LD BC,0900h
	        201E LD BC,0900h
	        2037 LD BC,0900h
	        204F LD BC,0900h

	possible references to internal address 0A5F:
	        0A60 LD HL,0A5Fh
	        ----------
	        0A51 LD A,(L0A5F)
	        105B LD A,(L0A5F)

*(Note the instructions after the dashed separator are references to the same literal value, but instances where it's known to be an address.)*

Possible references to external addresses are listed separately since they're less likely to need patching to make a source file for assembly:

	possible references to external address CC00:
	        0DCF LD BC,0CC00h

	possible references to external address F000:
	        096F LD HL,0F000h
	        0A8D LD HL,0F000h
	        0AA2 LD DE,0F000h
	        0BDD LD DE,0F000h
	        0C9C LD HL,0F000h
	        0E17 LD DE,0F000h
	        1485 LD HL,0F000h

And finally, port references are also listed:

	references to port 02h
	        0C83 IN A,(02h)
	        0C95 IN A,(02h)
	        0FCA IN A,(02h)
	        0FDA IN A,(02h)
	        0C87 OUT (02h),A
	        0C99 OUT (02h),A
	        0FCE OUT (02h),A
	        0FDE OUT (02h),A

	references to port 0Bh
	        094F OUT (0Bh),A
	        096A OUT (0Bh),A