# W65C51_LudaCris
A hardware workaround for the W65C51 Transmit Data Register Empty (TDRE) Bug

The W65C51N Asynchronous Communications Interface Adapter (ACIA) [datasheet](https://www.westerndesigncenter.com/wdc/documentation/w65c51n.pdf) is a 28-pin UART designed and produced by the Western Design Center.

Like all UARTs, the ACIA's primary function is to provide an interface between a parallel data bus (8 bits wide for the ACIA, with additional lines for chip and I/O register selection) and a serial data connection, under the supervision of a Central Processing Unit ("CPU", or "processor"). Bits are transmitted or received according to a set of possible serial line disciplines, which latter are pre-programmed into the ACIA by the processor. The design allows choice of baud rate, word length, number of stop bits, and parity checking. A design flaw means that in practice, parity checking is not available for either transmission or reception.

The ACIA features a baud rate generator capable of transmitting and receiving at one of 16 configurable standard baud rates, when supplied with an external crystal or (typically crystal-based) oscillator circuit as a time base. The baud rate generator divides the pulses from the externally supplied time base by the appropriate integer divisor to achieve the programmed baud rate. In support of traditional baud rates found in industry, the nominal oscillator frequency is at 1.8432 MHz (the "standard frequency" 

Here is a table of the standard baud rates supported by the ACIA driven by the standard frequency of 1.8432 MHz. Bits 3 to 0 of the Control Register are given, followed by the target baud rate, followed by the necessary integer divisor represented in decimal, and as a 16 bit integer.

Ctrl   Baud   Baud Rate Divisor Table
3210   Rate   Div 15     8 7      0 
---- ------ ----- -------- --------
0000 115200    16 00000000 00010000
0001     50 36864 10010000 00000000
0010     75 24576 01100000 00000000
0011 109.92 16769 01000001 10000001
0100 134.51 13704 00110101 10001000
0101    150 12288 00110000 00000000
0110    300  6144 00011000 00000000
0111    600  3072 00001100 00000000
1000   1200  1536 00000110 00000000
1001   1800  1024 00000100 00000000
1010   2400   768 00000011 00000000
1011   3600   512 00000010 00000000
1100   4800   384 00000001 10000000
1101   7200   256 00000001 00000000
1110   9600   192 00000000 11000000
1111  19200    96 00000000 01100000

Each of the 16 bits of each 16-bit divisor is active (1) for a subset of the 16 available baud rates. If the bits of the divsor are assigned the names Q15 to Q0, each bit should be a 1 if and only if the following choices of baud rate are selected. The second column consolidates similar input values by specifying X ("Don't Care") in bit positions which may be ignored.

;Table of Divisor Bits per Control Register Value
;X = Don't Care
;Quot Baud Rate Selections    
;Bit  3210 Control Register
;Q15  0001                     = 0001
;Q14  0010 0011                = 001X
;Q13  0010 0100 0101           = 0010 010X
;Q12  0001 0100 0101 0110      = 0X01 01X0
;Q11  0110 0111                = 011X
;Q10  0100 0111 1000 1001      = 0100 0111 100X
;Q9   1000 1010 1011           = 1000 101X
;Q8   0011 0100 1010 1100 1101 = 0011 0100 1010 110X
;Q7   0011 0100 1100 1110      = 0011 0100 11X0
;Q6   1110 1111                = 111X
;Q5   1111                     = 1111
;Q4   0000                     = 0000
;Q3   0100                     = 0100
;Q2   (none, always 0)         = Not mapped by PLD
;Q1   (none, always 0)         = Not mapped by PLD
;Q0   0011                     = 0011

Additional bits in the ACIA Control Register control the remaining features of serial line discipline supported by the ACIA, namely, word length (5, 6, 7, or 8 bits) and number of stop bits (1, 2, and in the exceptional case of 5 bit word length, 1.5 bits).


A start bit is always transmitted. Word length is selectable from lengths between 5 to 8 bits, as is the number of stop bits (1, 2, or, for 5 bit word length, 1.5 stop bits).

Transmission Unit Sizes (7-11 bits, 1.5 stop bits = 2 stop bits.)

Ctl
Bit Start   Word Stop Ceil
765  Bits Length Bits Total
--- ----- ------ ---- -----
000     1      8    1    10
001     1      7    1     9
010     1      6    1     8
011     1      5    1     7
100     1      8    2    11
101     1      7    2    10
110     1      6    2     9
111     1      5  1.5     8

Limitations of the W65C51N

The W65C51N has a number of well-documented limitations stemming from design defects that are unlikely ever to be fixed. Chief among these limitations are the following:

* Transmitter Data Register Empty (TDRE) bug. Status Register bit 4, per the original design, can be used to poll the transmit buffer to determine if the most recent transmission unit has completed transmission. This feature does not work in production W65C51N units. Additionally, bits 2-3 of the Command Register (Transmitter Interrupt Control (TIC)), which in the design were used to control whether interrupts were generated on transitions of the Transmitter Data Register Empty (TDRE) Status Register bit, do not meaningfully enable interrupts, as the TDRE never transitions.

* No parity. The W65C51N does not generate or consume parity bits. The original settings for selecting parity in line discipline are. Within the original design, Status Register Bit 0 was allocated to represent the state of the parity check of the most recently received transmission unit. In production, this bit's value must be ignored. Additionally, bits 6-7 of the Command Register were, according to the original design, allocated to control the selection of parity mode: these bits are ignored in production.

Of these two limitations, the latter (lack of parity bit line discipline) is less problematic, as much serial hardware can cope with using no parity. The other limitation, that of the Transmit Data Register Empty (TDRE) bug, prevents the processor from learning, either by interrupt or by active polling of the Status Register, when transmission has concluded. 

The W65C51N datasheet recommends a software workaround for the TDRE bug, namely, that transmission code enter a loop after the transmission of each byte, in an attempt to ensure that sufficient time elapses before the processor attempts to send the next byte. This workaround wasteful of processor cycles. Additionally, since the time required for transmission is variable in baud rate and transmission unit length, and dependent on a different time base than the clock cycle of the typical processor, naive implementations of software delay loops run the risk of not waiting long enough, or wasting processor cycles. 

A better workaround would be hardware based. Fortunately, the W65C51N exposes an active low input, Data Set Ready, (DSRB) which is exposed via the Status Register. Data Set Ready is part of the RS-232 serial standard, indicating that the transmission equipment (typically, a modem) is ready (logic level low) or not ready (logic level high).

Further, the ACIA can be configured to assert an interrupt to the processor on a change of state of DSRB. 



The present design seeks to overcome the TDRE bug by functionally replicating the W65C51N's baud rate generator using a cascading series of binary counters, programmed to count pulses from the external clock source supplied to the ACIA. 

Since word length is variable, the generated baud clock is used to increment an additional binary counter, programmed with the length, in bits, of the ACIA's configured transmission unit size of between 7-11 bits.

