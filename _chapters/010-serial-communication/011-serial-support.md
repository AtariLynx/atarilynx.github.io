---
title: Serial support in cc65
abstract: Using cc65 to get basic serial communication in your code
---

Serial support in cc65 is a set of libraries with generic functions to perform serial communication. The libraries are implemented specifically for a variety of devices, such as the Apple II, Atmos and Commodore 64. The library is also available for the Atari Lynx and includes most of the functionality available. 
The serial support includes:
* Loading and unloading serial drivers from the device
* (Un)installing a specific serial driver
* Opening and closing a port for serial communication
* Reading and writing single bytes to the port (referred to as get and put)

# Opening a serial port
Before you can start sending or receiving bytes across the ComLynx serial port, the communication stack needs to be initialized. The Lynx does not require any loading and installation of drivers. It is simply a matter of opening the port. You do need to know how you want to communicate first. The connection details include the following:
* Number of bits per datagram 
* Use a stop bit in transmission
* Parity bit behavior
* Baud rate
* Handshake behavior

The Lynx has a number of fixed or required elements. For example, the datagram is always 11 bits, consisting of the start bit, 8 bits of data, a parity bit and a stop bit. This cannot be changed. The only choice is what the state of the parity bit is, based on the parity calculation of `Odd` and `Even`, or the static value of `Mark` and `Space`. 

The cc65 serial library offers a structure `ser_params` that contains these values in a single memory unit:

``` c
struct ser_params {
    unsigned char       baudrate;       /* Baudrate */
    unsigned char       databits;       /* Number of data bits */
    unsigned char       stopbits;       /* Number of stop bits */
    unsigned char       parity;         /* Parity setting */
    unsigned char       handshake;      /* Type of handshake to use */
};
```
The following baud rates in cc65 are all recognized rates across the various hardware configurations:
`45.5`, `50`, `56.875`, `75`, `110`, `134.5`, `150`, `300`, `600`, `1200`, `1800`, `2400`, `3600`, `4800`, `7200`, `9600`, `19200`, `31250`, `38400`, `57600`, `62500`, `115200`, `230400`.
Each serial library for a particular device implements the relevant and possible baud rates for it. Out of the available baud rates, the Lynx supports these:
`300`, `600`, `1200`, `1800`, `2400`, `4800`, `9600`, `31250` and `62500`. Out of these the rates of `9600` and `62500` are the most common.

``` c
struct ser_params params = {
  SER_BAUD_9600,
  SER_BITS_8, // Only 8 bits are supported
  SER_STOP_1, // Must be 1 stop bit
  SER_PAR_EVEN, // SER_PAR_NONE is not allowed
  SER_HS_SW // No handshake support
};
```

The serial port is opened by calling `ser_open(&params)` while passing in the address of the struct with the details for the connection. The call returns an error code indicating whether the call succeeded or not. On a successful call the result will be `SER_ERR_OK`. All other return values represent a particular error condition.  

The serial implementation for the Lynx will only return four potential error codes. 
* **SER_ERR_NO_DEVICE**: Device not found
* **SER_ERR_BAUD_UNAVAIL**: Baud rate not available
* **SER_ERR_NO_DATA**: Nothing to read
* **SER_ERR_OVERFLOW**: No room in send buffer

The first two are relevant during opening as these mean that the opening of the serial port failed. 

## ComLynx cable detection
Typically, you would use `SER_HS_NONE` to skip any handshake signals (CTS, DTR, CTR or DTS) as the Lynx does not support these. If you want to verify that the ComLynx cable is inserted, you can use the `SER_HS_SW` software handshake to check for the presence of a ComLynx cable. This way it is possible to opt-in on the check. Note that `SER_HS_SW` does not give any additional checks after opening the serial port. In other words, removing the ComLynx cable later will not result in any errors. Also, you can simply ignore the error and continue as if no error occurred.

In this code fragment you can see how to keep checking for a connected ComLynx cable before continuing.

``` c
void open_comlynx()
{
  struct ser_params params = {
    SER_BAUD_300,
    SER_BITS_8, // Only 8 bits are supported
    SER_STOP_1, // Must be 1 stop bit
    SER_PAR_EVEN, // SER_PAR_NONE is not allowed
    SER_HS_SW // No handshake support
  };

  while (ser_open(&params) == SER_ERR_NO_DEVICE)
  {
    if (!tgi_busy())
    {
      tgi_clear();
      tgi_setcolor(COLOR_WHITE);
      tgi_outtextxy(10, 10, "No serial port!");
      tgi_updatedisplay();
    }
  }
}
```

## Closing serial port
After using the connection it is best to call `ser_close()` to properly close the serial port. The serial implementation of the Lynx will reset the serial errors and serial status as well as stopping the serial timer (timer 4). The call always returns error code `SER_ERR_OK` as nothing can go wrong here.

``` c
if (transmit_done)
{
  ser_close();
}
```

# Sending and receiving data over ComLynx
Once the serial port has been opened you can start sending and receiving data over it. A circular buffer of 256 bytes is used for sending and receiving. You can send and receive individual bytes over the port by using the two functions `ser_get` and `ser_put`. The prototypes of these functions reside in `serial.h` which you should include in your code. 

``` c
unsigned char __fastcall__ ser_get (char* b);
unsigned char __fastcall__ ser_put (char b);
```

Both return a `char` value to indicate whether the call was successful or not. It uses the same values as mentioned before. Getting bytes when there is nothing (more) to receive will give `SER_ERR_NO_DATA`. Similarly, `SER_ERR_OVERFLOW` indicates that bytes were added faster than could be sent, so the circular buffer overflowed.

The following code demonstrates how to keep receiving bytes in a message after having opened the serial port.

``` c
char message[16];

open_comlynx();

while (true)
{
  result = ser_get(&message[index]);
  if (result != SER_ERR_NO_DATA)
  {
    ++index;
    if (index == 16) index = 0;

    // Display new contents of message
    show_message();
  }
};

ser_close();
```

The `ser_get` method accepts the address where the received byte will be stored. As there might be no data available, you need to check the result of the operation for a `SER_ERR_NO_DATA` or `SER_ERR_OK` return value. THe first indicates there was no data, whereas the second implies a byte has been stored in the address passed in.

Typically, one would only use a loop like this if receiving data is intended and expected. An example is receiving data of a custom level in a puzzle-based game after selecting it from a game menu. At the start the port can be opened and closed directly after the operation has completed. This is more efficient than opening the serial port at startup. When the serial port is opened, timer 4 and its corresponding receive interrupt is enabled. This might waste unnecessary cycles if there is not going to be any data sent over the cable or at unexpected moments. It is best to keep an opened serial port for as short as possible, unless there is a compelling reason to do so.

In the next code sample a buffer of 256 bytes is transmitted over ComLynx. It is the same size as the buffer of the serial library, so it should not overwrite in a single iteration. 

``` c
unsigned char buffer[256];
unsigned char byte;
unsigned char index = 0;

open_comlynx();

do {
  byte = buffer[index];
  status = ser_put(byte);
  if (status == SER_ERR_OVERFLOW)
  {
    // Oops, we were too fast
    break;
  }
} while (++index != 255);

ser_close();
```

If there is more data to send, you can load more data in the `buffer` array after a single iteration and repeat. For low baud rate a small wait period might be necessary to prevent overflowing.
