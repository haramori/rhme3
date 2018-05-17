# RHme3 board analysis and preparation

## Chips

* Atmel XMEGA128A4U
* Microchip MCP25625 (x2): CAN controllers/transcievers
* CH340G: USB-Serial adapter

## Resources

* [ATXMEGA128A4U Support](http://www.microchip.com/wwwproducts/en/ATXMEGA128A4U#1)
* [8/16-bit Atmel XMEGA Microcontroller XMEGA A4U Datasheet, 09/2014](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-8387-8-and16-bit-AVR-Microcontroller-XMEGA-A4U_Datasheet.pdf)
* [8-bit Atmel XMEGA AU Microcontroller XMEGA AU MANUAL, 04/2013](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-8331-8-and-16-bit-AVR-Microcontroller-XMEGA-AU_Manual.pdf)
* [AVR Instruction Set Manual, 11/2016](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-0856-AVR-Instruction-Set-Manual.pdf)
* [MCP25625 Support](http://www.microchip.com/wwwproducts/en/MCP25625#1)
* [MCP25625 CAN Controller with Integrated Transceiver Datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/20005282B.pdf)

## Headers and jumpers

* Arduino Uno R3
* PDI (labeled "ICSP")
* CAN H/L
* SJ1 (solder jumper)
* JP4 (labeled "uC/VCC")

### Arduino pinout

A map of board-level connections is useful when probing and identifying signals.

	IO   XMEGA  Most likely               Internal
	pin  pin    function                  connection
	===  =====  ===========               ==========
	RST    35   RESET/PDI_CLK             3V3 pullup
	A0     41   PA1/ADC1/AC1
	A1     42   PA2/ADC2/AC2
	A2     43   PA3/ADC3/AC3
	A3     44   PA4/ADC4/AC4
	A4      1   PA5/ADC5/AC5
	A5      2   PA6/ADC6/AC6
	D0     12   PC2/USARTC0_RX            USB_TX
	D1     13   PC3/USARTC0_TX            USB_RX
	D2     10   PC0/SDA
	D3     11   PC1/SCL
	D4     22   PD2/USARTD0_RX
	D5     23   PD3/USARTD0_TX
	D6     24   PD4/SPID_SS               CAN1_CS
	D7      5   PB2/ADC10/DAC0
	D8      7   PB3/ADC11/DAC1
	D9     14   PC4/SPIC_SS               CAN2_CS
	D10    15   PC5/SPIC_MOSI             CAN2_MOSI
	D11    25   PD5/SPID_MOSI             CAN1_MOSI
	D12    26   PD6/USARTD1_RX/SPID_MISO  CAN1_MISO
	D13    27   PD7/USARTD1_TX/SPID_SCK   CAN1_SCK
	AREF   40   PA0/ADC0/AC0/AREF
	SDA                                   D2
	SCL                                   D3

## Notes

Most of the board can be powered by a 3.3V supply, but the MCP25625 requires a 5V supply to operate the transciever circuits.  Running the MCP25625 at 3.3V won't damage it, but CAN won't work.

The XMEGA does not use an external crystal/resonator.  The internal oscillator is assumed to be running at 16/32 MHz.

The XMEGA communicates with the CAN controllers using SPI.

XMEGA does not support ICSP.  The port labeled "ICSP" is actually a PDI port.  You will need a programmer/debugger that supports PDI in order to use this port (e.g. Atmel ICE).  Note: Not used for CTF.

## Avrdude preparation

Bridge the gap across SJ1 with a blob of solder.  This will connect the DTR pin on the CH340G to the ~RESET pin on the XMEGA.  This will allow avrdude to automatically reset the board to initiate programming.

![RHme3 board](../Images/rhme3_board.jpg)

## CAN preparation

Solder jumper headers at CAN H and CAN L.

To support independant CAN channels, cut the traces connecting the two CAN channels, as follows:
* Using a small screwdriver, gently remove the black header base from the CAN H jumper pins.
* Attach a multimeter across channels 1 and 2 on CAN H.  Put multimeter into tone continuity mode.
* Gently score the CAN H trace between channels 1 and 2 with a hobby knife until the continuity tone stops.
* Put the black header base back onto the jumper pins.
* Repeat all steps with CAN L.

To operate connected ECUs: Put a jumper on the header pins across each trace that you cut.

To operate disconnected ECUs: Remove the jumpers.

![Cutting CAN trace](../Images/cutting_can_trace.jpg)

## SCA/FI preparation

Remove the three 100 nF capacitors C7, C16 and C17.

Solder a jumper header at JP4.

![RHme3 board](../Images/rhme3_board.jpg)

Cut the trace connecting the two pins of JP4, as follows:
* Attach a multimeter to both pins of JP4.  Put multimeter into tone continuity mode.
* Turn board over.  The trace to cut is on the back of the board.
* Gently score the trace with a hobby knife until the continuity tone stops.

For normal use: Put a jumper onto the header pins at JP4.

For SCA/FI: replace the jumper at JP4 with a shunt circuit.
