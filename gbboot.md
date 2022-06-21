```
; GameBoy boot ROM.
;
; Originally from here: https://gist.github.com/drhelius/6063288

; The last instructions switch to the cartridge ROM and execution falls
; into address 0100.

; The GameBoy (original) simply scrolls the "Nintendo(r)" logo down the screen
; to the center and plays a tone. Later models were more sophisticated.

0000: 31 FE FF     LD    SP,$FFFE                ; Setup stack to end of memory

0003: AF           XOR   A                       ; Zero the VRAM ...
0004: 21 FF 9F     LD    HL,$9FFF                ; ... VRAM is from ...
0007: 32           LD    (HL-),A                 ; ... 8000 ...
0008: CB 7C        BIT   7,H                     ; ... to ...
000A: 20 FB        JR    NZ,$0007                ; ... 9FFF

000C: 21 26 FF     LD    HL,$FF26                ; Setup Audio
000F: 0E 11        LD    C,$11                   ;
0011: 3E 80        LD    A,$80                   ;
0013: 32           LD    (HL-),A                 ;
0014: E2           LD    ($FF00+C),A             ;
0015: 0C           INC   C                       ;
0016: 3E F3        LD    A,$F3                   ;
0018: E2           LD    ($FF00+C),A             ;
0019: 32           LD    (HL-),A                 ;
001A: 3E 77        LD    A,$77                   ;
001C: 77           LD    (HL),A                  ;

001D: 3E FC        LD    A,$FC                   ; 11_11_11_00 : Values 3,2,1 are solid black. Value 0 is white.
001F: E0 47        LD    ($FF00+$47),A           ; Set the backround gray scale values (black and white)
0021: 11 04 01     LD    DE,$0104                ; The 48 byte logo in the cartridge
0024: 21 10 80     LD    HL,$8010                ; Destination tile memory (leave the first tile blank)
;
0027: 1A           LD    A,(DE)                  ; Get the next byte from the source
0028: CD 95 00     CALL  $0095                   ; Convert 4x4 bits ...
002B: CD 96 00     CALL  $0096                   ; ... to 8x8 tile
002E: 13           INC   DE                      ; Next byte from logo
002F: 7B           LD    A,E                     ; Have we ...
0030: FE 34        CP    $34                     ; ... processed all 38 bytes? (04 + 30 = 34)
0032: 20 F3        JR    NZ,$0027                ; No ... go back and do all

0034: 11 D8 00     LD    DE,$00D8                ; The (R) symbol as one 8x8 tile
0037: 06 08        LD    B,$08                   ; 8 rows to copy
0039: 1A           LD    A,(DE)                  ; From ...
003A: 13           INC   DE                      ; ... source
003B: 22           LD    (HL+),A                 ; To destination
003C: 23           INC   HL                      ; Skip the upper bits in the tile data
003D: 05           DEC   B                       ; Moved all 8?
003E: 20 F9        JR    NZ,$0039                ; No ... go back for all

; Tile MAP is 32x32, but the screen is only 20x18 tiles. The map starts at 9800. The whole screen is
; cleared to tile 0, which is blank. The tiles are filled in on the first two rows as shown below
; with "." meaning tile 0. Tile 19 is the (R) tile.
;
;       9800 9801 9802 9803 9804 9805 9806 9807 9808 9809 980A 980B 980C 980D 980E 980F 9910 9911 9912 9913
;         .    .    .    .    1    2    3    4    5    6    7    8    9    A    B    C   19    .    .    .
;         .    .    .    .    D    E    F   10   11   12   13   14   15   16   17   18    .    .    .    .
;       9920 9921 9922 9923 9924 9925 9926 9927 9928 9929 992A 992B 992C 992D 992E 992F 9A20 9A21 9A22 9A23
;
0040: 3E 19        LD    A,$19                   ; The (R) tile (and the number of tiles we made)
0042: EA 10 99     LD    ($9910),A               ; Store the (R) tile
0045: 21 2F 99     LD    HL,$992F                ; Start at bottom right of logo on screen
;
0048: 0E 0C        LD    C,$0C                   ; 12 tiles per row
004A: 3D           DEC   A                       ; Have we moved all tiles?
004B: 28 08        JR    Z,$0055                 ; Yes ... move on
004D: 32           LD    (HL-),A                 ; No ... store the tile number
004E: 0D           DEC   C                       ; Have we finished this row?
004F: 20 F9        JR    NZ,$004A                ; No ... keep going
0051: 2E 0F        LD    L,$0F                   ; End of the top row (980F)
0053: 18 F3        JR    $0048                   ; Do the next row

; Scroll logo on screen, and play logo sound

; The screen is 144 pixels high. The center point is 72.
0055: 67           LD    H,A                     ; Initialize scroll count, H=0
0056: 3E 64        LD    A,$64                   ; Starting Y coordinate for visible window
0058: 57           LD    D,A                     ; Count 64
0059: E0 42        LD    ($FF00+$42),A           ; Set vertical scroll register
005B: 3E 91        LD    A,$91                   ; 10010001 : LCD on, BG set to tiles 8800-97FF, ...
005D: E0 40        LD    ($FF00+$40),A           ; ... BG priority 1
005F: 04           INC   B                       ; Set B=1
;
; Delay between scrolls by counting 2*12 = 24 vertical blanks. Refresh rate is 59.73Hz, so the 
; delay is roughly 1/60 * 24 = 2/5 = 0.4 seconds.
0060: 1E 02        LD    E,$02                   ; Delay counter (MSB=2) for vertical scrolls
0062: 0E 0C        LD    C,$0C                   ; Delay counter (LSB=12) for vertical scrolls
0064: F0 44        LD    A,($FF00+$44)           ; What line is the display drawint?
0066: FE 90        CP    $90                     ; Are we on line 144 (start of vertical blanking)
0068: 20 FA        JR    NZ,$0064                ; No ... wait for VBLANK to start
006A: 0D           DEC   C                       ; LSB delay between scrolls
006B: 20 F7        JR    NZ,$0064                ; Not time ... keep counting
006D: 1D           DEC   E                       ; MSB delay between scrolls
006E: 20 F2        JR    NZ,$0062                ; Not time ... keep counting
;
0070: 0E 13        LD    C,$13                   ;
0072: 24           INC   H                       ; increment scroll count
0073: 7C           LD    A,H                     ;
0074: 1E 83        LD    E,$83                   ;
0076: FE 62        CP    $62                     ; $62 counts in, play sound #1
0078: 28 06        JR    Z,$0080                 ;
007A: 1E C1        LD    E,$C1                   ;
007C: FE 64        CP    $64                     ;
007E: 20 06        JR    NZ,$0086                ; $64 counts in, play sound #2
0080: 7B           LD    A,E                     ; play sound
0081: E2           LD   ($FF00+C),A              ;
0082: 0C           INC   C                       ;
0083: 3E 87        LD    A,$87                   ;
0085: E2           LD    ($FF00+C),A             ;
0086: F0 42        LD    A,($FF00+$42)           ;
0088: 90           SUB   B                       ;
0089: E0 42        LD    ($FF00+$42),A           ; scroll logo up if B=1
008B: 15           DEC   D                       ;
008C: 20 D2        JR    NZ,$0060                ;
008E: 05           DEC   B                       ; set B=0 first time
008F: 20 4F        JR    NZ,$00E0                ;... next time, cause jump to "Nintendo Logo check"
0091: 16 20        LD    D,$20                   ; use scrolling loop to pause
0093: 18 CB        JR    $0060                   ;

MakeOneTile:
;
; This is called twice in a row. First the call to 0095 to roll out the upper 
; nibble. Then the call to 0096 to continue rolling out the lower nibble.
;
; With each pass, the upper nibble of C becomes a byte in A that is stored to the 
; least-significant bits of a tile row. The same value is repeated into the next row 
; of the tile. Thus two bytes of source data become a 4x4 bit pattern that doubles 
; to an 8x8 tile.
;
0095: 4F           LD    C,A                     ; The C register holds the source bits
0096: 06 04        LD    B,$04                   ; Four shifts per nibble
0098: C5           PUSH  BC                      ; Hold the count and the source
0099: CB 11        RL    C                       ; Roll C left ...
009B: 17           RLA                           ; ... into A
009C: C1           POP   BC                      ; Restore the source bits
009D: CB 11        RL    C                       ; Roll C left ...
009F: 17           RLA                           ; ... into A (repeat the bit)
00A0: 05           DEC   B                       ; Have we done all bits?
00A1: 20 F5        JR    NZ,$0098                ; No ... build one byte in A from the upper nibble in C
00A3: 22           LD    (HL+),A                 ; Store the tile row in memory and increment
00A4: 23           INC   HL                      ; Skip over the most significant bits
00A5: 22           LD    (HL+),A                 ; Store the tile to the next row (doubling the rows)
00A6: 23           INC   HL                      ; Skip over the most significant bits
00A7: C9           RET                           ; Done

; Nintendo Logo
00A8: CE ED 66 66 CC 0D 00 0B 03 73 00 83 00 0C 00 0D 00 08 11 1F 88 89 00 0E 
00C0: DC CC 6E E6 DD DD D9 99 BB BB 67 63 6E 0E EC CC DD DC 99 9F BB B9 33 3E
;
; Each row of data above is a row of 12 tiles. Two bytes per tile, one nibble
; per row. 4x4 tiles are doubled into 8x8 tiles. These end up in the tile slots
; shown below.
;
; 1    2    3    4    5    6    7    8    9    A    B    C (Tile number)
; NN.. .NN. ii.. .... .... .... .... .... .... ...d d... ....
; NNN. .NN. ii.. .... ..tt .... .... .... .... ...d d... ....
; NNN. .NN. .... .... .ttt t... .... .... .... ...d d... ....
; NN.N .NN. ii.n n.nn ..tt ..ee ee.. nn.n n... dddd d..o ooo.
;
; NN.N .NN. ii.n nn.n n.tt .ee. .ee. nnn. nn.d d..d d.oo ..oo
; NN.. NNN. ii.n n..n n.tt .eee eee. nn.. nn.d d..d d.oo ..oo
; NN.. NNN. ii.n n..n n.tt .ee. .... nn.. nn.d d..d d.oo ..oo
; NN.. .NN. ii.n n..n n.tt ..ee eee. nn.. nn.. dddd d..o ooo.
; D    E    F    10   11   12   13   14   15   16   17   18 (Tile number)
;
; A single tile for the (R) symbol (letter R with a circle around it)
00D8: 3C 42 B9 A5 B9 A5 42 3C
; Becomes tile number 19
; ..XXXX.. 
; .X....X.
; X.XXX..X
; X.X..X.X
; X.XXX..X
; X.X..X.X
; .X....X.
; ..XXXX..

; Nintendo logo comparison routine

00E0: 21 04 01     LD    HL,$0104                ; HL points to Nintendo logo in cartridge
00E3: 11 A8 00     LD    DE,$00A8                ; DE points to Nintendo logo in DMG rom
;
00E6: 1A           LD    A,(DE)                  ; Get the ...
00E7: 13           INC   DE                      ; ... next byte
00E8: BE           CP    (HL)                    ; Are they the same?
00E9: 20 FE        JR    NZ,$FE                  ; No ... A has some value other than 1 (no 1 in the logo)
00EB: 23           INC   HL                      ; Next in logo
00EC: 7D           LD    A,L                     ; Have we done ...
00ED: FE 34        CP    $34                     ; ... all 30? (remember, we started at 104 ... 04 in L)
00EF: 20 F5        JR    NZ,$00E6                ; No ... keep checking
;
00F1: 06 19        LD    B,$19                   ; Checksum the 25 bytes after logo
00F3: 78           LD    A,B                     ; Start the checksum with 25
;
00F4: 86           ADD   (HL)                    ; Add the ...
00F5: 23           INC   HL                      ; ... next byte
00F6: 05           DEC   B                       ; All done?
00F7: 20 FB        JR    NZ,$00F4                ; No ... go back for all
;
; ?? What if the checksum ends up being 1? Then the check passes.
00F9: 86           ADD   (HL)                    ; Checksum byte (014D)
00FA: 20 FE        JR    NZ,$FE                  ; if $19 + bytes from $0134-$014D don't add to $00 ... lock up
00FC: 3E 01        LD    A,$01                   ; Write 1 to turn off DMG and continue with cartridge ...
00FE: E0 50        LD    ($FF00+$50),A           ; ... or any other value to lock up
```
