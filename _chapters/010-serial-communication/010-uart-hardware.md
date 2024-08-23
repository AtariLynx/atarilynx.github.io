---
title: UART and serial communication
abstract: Basics of serial communication and UART hardware in Atari Lynx 
---

The Lynx has an interesting feature that sets it apart from other handheld consoles. The ability to connect up to 16 Lynxes and allow them to communicate with one another. ComLynx was the official name for the connection capability of one or more Lynx consoles. We will get started with the basics of serial communication and the hardware inside the Lynx. This helps understand how everything works and gives valuable insights before we move up an abstraction level by using the cc65 serial driver.

# Primer in UART and serial communication
Before we dive into ComLynx we need a good understanding of UART and the serial form of communication that comes with it. [UART](http://en.wikipedia.org/wiki/UART) is short for Universal Asynchronous Receiver/Transmitter and is a piece of hardware (an integrated circuit) that can do serial communication over a small number of data lines. Usually, the physical communication interface between two such hardware components consists of a cable that has a couple of wires, each with a dedicated purpose. Typically, these are a ground (GND) and a receive (RX) and transmit (TX) line at a minimum. Other lines (CTS, RTS, DSR, DTR) can help create a more robust communication allowing for handshakes and transmission control. The Lynx does not have these, so let’s steer clear of those.

The RX and TX line can have a high (e.g. +5V, although the exact voltage may vary) voltage and low voltage (0V and again low might be a different voltage, potentially negative). Using the two voltage levels the lines can send bits across the line by alternating high and low to indicate 1 and 0 respectively. It is similar to morse code where the short and long beeps are also two distinct signals that allow you to build characters. Where it differs is that in this serial communication it is customary to have a particular transmission protocol.

The most common protocol used for communication defines a way to send/receive data and check that the data has arrived completely and without error. The terminology includes Mark (for the high signal, or 1) and Space (for the low signal, or 0). The idea is that each piece of data is surrounded by a start bit and stop bits. The start bit signals the start of the data to follow and serves as a sync point for the receiver to begin reading the data that follows using its internal clock. When the data has been sent, one or two stop bits follow to signal the end. The stop bits might be preceded with a [parity bit](http://en.wikipedia.org/wiki/Parity_bit) that helps determine errors.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/3a2edee9-dd1b-4c17-bdc0-0bb7e18100cf)
[source](http://www.eeherald.com/section/design-guide/esmod7.html)

The parity bit is something special. It can be used by the hardware to detect errors in some cases. The parity bit helps to determine and check the parity of the data. It works like this: the bits in the data that have a high value `1` are counted. The parity bit is chosen to be `0` or `1` depending on the kind of parity check. With 'Even' parity the parity bit is determined to result in an even number of 1 bits. Similarly, 'Odd' parity is used when the number must be odd, not even. For example, the data transmitted was `10011001` in binary. That’s four ones, so that is an even number. With even parity the parity bit should be zero, otherwise the total number of ones including the parity bit will be odd. For odd parity it would be necessary to choose the parity bit as one resulting in 5 ones which is odd. Otherwise the total number would remain an even number. Whenever the parity bit is incorrect for the chosen convention (odd or even) it means an error condition has been detected. Since the check is based on a single bit with two values of which one is correct, it means that you will only find errors in 50% of the cases. Not perfect, but still better than nothing.
Three other options for the parity bit exist: None, Mark and Space. 'None' means that no parity bit is included. The stop bit(s) will immediately follow the data bits in that case. 'Mark' parity always has a high parity bit, and 'Space' has a low parity bit.

Each bit that is transmitted is defined by the high- or low-ness of the line for a period of time. That time, the bit time, is determined by the clock speed of the UART. You have probably come across the unit of [Baud](http://en.wikipedia.org/wiki/Baud), or 'Bits per second'. It was common to refer to your modem speed in Baud. E.g., my first modem was a Dynalink 14k4 baud modem, capable of transmitting 14400 symbols (or tones) per second. Although Baud and bits per second are not necessarily the same, it does hold true for the UART in the Lynx. For digital devices the symbols are bits, hence the reason that Baud and bits per second are equivalent.

# UART in the Lynx
The UART of the Lynx is a circuit that lives inside of Mikey. The UART supports various baud rates and has several settings for the parity bit. The baud rate is governed by the countdown value and frequency settings you provide to timer 4. The timer governs the pace in which the bits are transferred. All Lynx consoles should use the same baud rate to have the same bit rate and understand the data being transmitted.

The data that is transmitted over the wire has the following fixed 11-bits format:
![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/ccd7a1c7-2c2d-4779-89f5-cefe81a59471)

The start bit and stop bit are always present and have a values of `0` and `1` respectively. This is a common choice as it will make sure that there is always a transition between the stop and start bit (`1` to `0`). The data is sent with the least significant bit first (LSB).

The parity bit is set depending on a chosen setting. The Lynx supports the variations of `Odd`, `Even`, `Mark` and `Space`. Since `None` is not supported the 9th bit is always sent. For odd and even parity the parity bit is set depending on the parity calculation. The Epyx documentation mentions a hardware flaw that results in the parity calculation to include the parity bit itself. The parity bit is actually set correctly. The Lynx can communicate with another Lynx and non-Lynx devices (such as a PC with a serial port) perfectly, regardless of the parity setting. You only need to make sure to use the same parity check on both sides.

The Lynx’s UART has a TX, RX and GND line for send, receive and ground. The peculiar thing about the Lynx wiring of the cable is that the RX and TX are connected together inside the ComLynx cable.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/d7e0830e-e454-4893-81c9-06ca030c6c16)

The design choice to connect RX and TX together has a lot of consequences:

* The hardware now has the simplest setup.
* Whatever a Lynx transmits is also received by itself.
* The Lynx can detect when there is something connected to the ComLynx port, because it will be 'short-circuited'.
* No hardware or software handshakes are possible.
* When one Lynx talks, all others must listen to avoid transmission errors.

Inside the UART there is transmitter and receiver hardware. For transmitting the UART provides a holding register and a shift register. The holding register can hold the next byte that must be sent. The shift register pushes the actual 8 data bits over the wire, wrapping it with the start, parity and stop bits.

To send something you first put a byte in the holding register. The hardware transfers the byte to the shift register when it is empty and ready to accept the next byte. At this point the shift register starts transmitting all 11 bits and effectively sending it out the entire byte of data. In the meantime you can put the next byte in the holding register, because that has become empty after the byte was transferred to the shift register.

The receiving of data is slightly different. The 11 incoming bits are received and the single byte of payload data between the start, parity and stop are stored in a receive holding register. Data will keep coming in from the external source at the baud rate. It is your task to read the data inside the register before it is overwritten by the next byte of data received. If you do not read in time, the Lynx will report a register overrun error.

# Serial interrupts

The UART send and receive mechanism uses timer 4 as its clock to generate the bit rate. In addition it can trigger interrupts (IRQ) when actually sending or transmitting. Indirectly the interrupts are a way of notifying you of a newly received byte or an empty holding register. In the interrupt handler you can check flags (TXRDY and RXRDY as we’ll see in a moment) to see whether a new byte is ready for sending or when a byte has been received. If so, you should put a new byte in the holding register to keep the outbound dataflow going and you should read received bytes quick enough before the next one arrives. Should you be slow to put in a new byte to transmit, you’re wasting time sending all data. If you are too late to pick up a byte from the receive holding register, you will get an overrun because the receiver cannot place the new byte and you lose the data. 

It’s your choice whether you want to use interrupts for sending and receiving or not. Using interrupts you can continue doing other stuff, so this is a reasonable option when you are performing other tasks (such as gameplay) and want to send and receive data in the meantime during interrupt handling. Interrupts will help you do these tasks just in time with dedicated handlers. On the other hand, should you have a dedicated part of your program or game that does send or receive only, you can use the send and receive flags to check whether you should use read or write data. In such cases it offers no benefit to have the overhead of IRQ handling routines

# Baud rate and serial timer
The Lynx has various hardware registers inside Mikey that have a memory address so you can change settings for the UART hardware and help send and receive data.

First of all, the UART uses the serial timer, which is timer 4, to serve as the baud rate generator. So, by setting the time of each timer tick and the countdown period you can indirectly specify the baud rate. The baud rate is calculated by determining the time it takes to send 8 bits when timer 4 has a particular speed. The baud rate calculation is as follows:

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/26974d37-1939-4531-bd66-2302c7f95bbe)

where r<sub>timer4</sub> and f<sub>timer4</sub> are the reload (countdown number) value and frequency of timer 4. The countdown value is set to a minimum of 1, but the timer will trigger when it underflows (from 0 to a virtual -1). It will need a minimum of two timer periods to trigger. The frequency inverted gives the number of clock ticks per seconds. The end result is the baud rate as it is frequently used (bytes per second) for devices, but not the official definition of bits per second.

Some examples of baud rate calculations:

|Reload   |Clock speed   | Frequency  | Baud rate  |
|---|---|---|---|
|1   | 1 µs   | 1 MHz  | 1000000/((1+1)·8) = 62500 Bps  |
|3  | 1 µs   | 1 MHz  | 1000000/((3+1)·8) = 31250 Bps  |
|12  | 1 µs   | 1 MHz  | 1000000/((12+1)·8) = 9615 Bps  |
|51   | 8 µs  | 125 kHz  | 125000/((51+1)·8) = 300 Bps  |

# UART hardware registers

The most important two memory locations in the Lynx are `SERCTL (address 0xFD8C)` and `SERDAT (address 0xFD8D)`. The first, `SERCTL`, refers to the serial control register and allows you to change UART settings. `SERDAT` is where you will read or write the serial data being received or sent.

`SERCTL` turns out to be a weird register. The behavior is totally different when writing to or reading from it. In other words, when you write a specific value, and then read it, you will probably get a different value from the one you wrote.

| Bit | Flag| Write | Read | 
|---|---|---|---|
| 7 | `%10000000` | **TXINTEN: Transmitter interrupt enable** <br/>The interrupt bit for timer 4 will correspond to the transmitter ready bit (i.e. you can put a new character in `SERDAT`) | **TXRDY: Transmitter buffer empty** <br/>The buffer to hold data is ready to accept another byte. You can write the byte to transmit to `SERDAT` |
| 6 | `%01000000` | **RXINTEN: Receiver interrupt enable** <br/> With this enabled the interrupt bit for timer 4 will correspond to the receiver ready bit (i.e. a character was received and can be read from `SERDAT`) | **RXRDY: Receive character ready** <br/> A character was received and can be read from `SERDAT`. |
| 5 | `%00100000` | **0 (zero) for future compatibility** No idea what they meant to keep compatible in the future | **TXEMPTY: Transmitter totally done** <br/> The transmitter has both an empty buffer and shift register. All offered data has been sent completely |
| 4 | `%00010000` | **PAREN: Parity bit enabled** <br/> The parity checking is enabled and the parity bit will be calculated according to the odd or even setting (see `PAREVEN`) | **PARERR: Received parity error** <br/> The data that was received had a parity error, so the parity bit did not match according to the parity setting |
| 3 | `%00001000` | **RESETERR: Resets all errors** <br/> Writing a value with this bit set will clear all three errors (parity, framing and overrun) should they be set | **OVERRUN: Received overrun error** <br/> The data in the receive holding register was not read quickly enough. The new data could not be delivered |
| 2 | `%00000100` | **TXOPEN** <br/> `1`: Open Collector <br/> `0`: TTL driver <br/> Choose between these two modes of transmission. A bug in the hardware causes the state of the output to be high after power up. The advice is to set the bit to Open Collector (`1`) to fix the problem | **FRAMERR: Received framing error** <br/> There was an error in the frame. That probably means that from the suspected start bit the stop bit was not received at the expected moment. It usually means that more than one Lynx is sending at the same time | 
| 1 | `%00000010` | **TXBRK: Sends a break** <br/> For as long as this bit remains `1`. It should be set at least for a 24 bit period according to the current baud rate. The specification mentions that a break is a start bit followed by 8 zero data bits, a parity bit and the absence of a stop bit at the expected time | **RXBRK: Break received** <br/> A break was received because for at least 24 bits of transmit time, there was a high signal |
| 0 | `%00000001` | **PAREVEN** <br/> `1`: Even parity (or Mark) <br/> `0`: Odd parity (or Space) <br/> This parity is used for both sending and receiving. When `PAREN` is `1` these parity values are used. If `PAREN` is `0` then the value in parentheses is used (Mark or Space) as the value of the 9th bit | **PARBIT: 9th bit** <br/> This bit reflects the parity bit of a received frame. It is set to the parity calculation when `PAREN` is `1`. Or it is whatever `PAREVEN` is at sender when `PAREN` is `0` | 

There are several other registers in the Lynx that behave this way. If you want to refer to the value written to `SERCTL` you need to maintain a 'shadow register' in memory yourself. Every time a value is written to `SERCTL`, the same value must be written to the shadow register as well. Reading back your previously written value is possible by evaluating the shadow register, not the `SERCTL` register itself, as this gives different values.

# Inside UART transmitter

A deeper look at the way the data is being transmitted will explain how the byte travels through the UART transmitter. It is important to realize that the `TXRDY` bit refers to the holding register, while the `TXEMPTY` bit represents both empty state of the holding and the shift register.

Here’s how the various states of the `TXRDY` and `TXEMPTY` bits change throughout the lifecycle of sending. The scenario shows how two bytes are sent in sequence from the start when no transmission has been done yet.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/711e229c-b559-4d58-a995-c6a0e42778f0)

The holding register and sending shift register are both empty at first. TX is ready and empty.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/e29a1734-16c6-4f9b-a787-392593102c96)

A byte is loaded into the holding buffer by a write to `SERDAT`. `TX` is no longer ready nor empty.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/0fc57807-67fc-4d32-b959-ed5ae0ac5881)

The byte is transferred from the holding to the shift register. `TX` is ready for new input in the holding register now.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/29266f7d-cf3f-492c-94c1-5b86ec3deae9)

The shift register is pushing out the bits of the data. In the meantime, new data is loaded into the holding register.

![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/fbf1cd0e-8e36-4fbe-ba11-e89fba226204)

When all data has been loaded into the shift register and transmitted both register will be empty again as in the starting situation. 

The reverse is true for receiving data, except that the data is eventually put into the receive holding register, which can be read from `SERDAT`.

The UART will always check the parity bit on received data whether `PAREN` is set to enabled or not. It does look at the `PAREVEN` bit, so for `Even` and `Odd` a calculation is done correctly, but for `Mark` and `Space` it only compares the value of the bit to the setting. Any parity errors are always reported through the `PARERR` bit in a read from `SERCTL`. You can inspect the value of the parity bit through `PARBIT` in the `SERCTL` register by reading from it. You should check `Mark` and `Space` manually and ignore the `PARERR` bit.