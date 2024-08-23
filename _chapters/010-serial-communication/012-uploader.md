---
title: Uploader 
abstract: Add support to upload BLL object files to the Atari Lynx over ComLynx  
---

The Atari Lynx hardware with the original Mikey ROM has a specific startup procedure that requires encrypted headers to be present on the cartridge's content. The encryption process and its encryption keys are well-known at the moment. It is possible to create cartridges with new and old header styles that allow the Lynx hardware to load homebrew, post-Atari cartridges.

Before the reverse-engineering of the encryption process and the discovery of the public-private key pair, it was hard to get a custom game to run on the physical hardware. After the first discovered method to create custom cartridges, the creators of the BLL library included a way to upload game content to the Lynx over the ComLynx serial port. It is quite easy to include this upload capability in your own code. This article describes how this can be done.

# Overview of upload implementation
The upload facility is implemented in both BLL and the Lynx library for cc65. You can find the code in `upload.m65` for BLL and in `uploader.s` for cc65. The implementations are mostly the same and differ only in the way the interrupt routines are setup and handled.
The picture shows the overall setup, with some bias towards cc65 and assumes the TGI library is being used. 
![image](https://github.com/AtariLynx/ProgrammingTutorial/assets/5504642/c089b545-c7c9-4a93-9b0c-c2f29016d7e6)

In the Atari Lynx all interrupts are timer driven, with the exception of timer 4, which is a serial timer, where the interrupt trigger is driven by the arrival or sending of a byte in `SERDAT`.

Whenever an IRQ is triggered, the code execution will jump to the Interrupt Service Routine (ISR) located at the IRQ interrupt vector previously stored in `$FFFE` and `$FFFF`. The ISR will go through all registered IRQ handlers in order and also use the one for the uploader handler. This handler checks whether `INTSET` has timer 4 set, as that indicates an interrupt triggered by serial. If not, it will return immediately, because apparently the IRQ was caused by another timer. Otherwise, if the serial IRQ was detected, it is checking whether the received byte is part of a sequence of two magic bytes (`$81` and `$50` (or '`P`') to be precise). This will take at least two handlings of the serial interrupt, as only one byte is read per interrupt. If the two magic bytes are not received in order, it will keep checking on each new IRQ.

If the magic bytes have been received, the uploader handler passes execution to a different routine that will load and store the actual code of the program that is uploaded. Once this routine is entered, there is no going back. The original code is overwritten and the state of the processor is changed, so a normal return is not possible. During receiving, interrupts are disabled, as everything is focused on loading, storing and finally executing the new code.

After the successful upload of the code it is executed by jumping to the start address. The code should do its own initialization preferably or rely on the already initialized hardware.

## cc65 upload specifics
The cc65 Lynx library has the uploader functionality defined in `uploader.s`. It has the logic and ISR previously discussed compiled into the object file `uploader.o` and included in lynx.lib. The compilation of the assembly file exports a symbol `__UPLOADER__` with an absolute value of `1` to allow it to be included when required.

It is important to note that this implies the upload logic is not included in your program by default. You will need to add this manually and also load the appropriate segment manually before the interrupt logic is needed. In other words, you should load the file segment with the ISR and upload logic before the upload routine is started. If memory is of no concern, you can load the routine during startup at the cost of some pre-allocated RAM segment. Alternatively, you can load it just in time, right before the upload functionality is enabled.

Fortunately, the cc65 has a special configuration file `lynx-uploader.cfg` for the uploader available, referring to the symbol `__UPLOADER__`. It defines a module `UPLOAD` with corresponding segments `UPCODE` and `UPDATA`.
In the `lynx-uploader.cfg` you can find the extra symbols referring to upload:

```
SYMBOLS {
    # Other symbols omitted for brevity
    __BOOTLDR__:          type = import;
    __UPLOADER__:         type = import;
    __UPLOADERSIZE__:     type = export, value = $61;
    __HEADERSIZE__:       type = export, value = 64;
}
```

The `__UPLOADER__` symbol needs to be imported to give the linker access to the uploader part in `lynx.lib` that was added by the `uploader.o` object file. The uploader code has a well known size of $60 = 96 bytes. Together with the single byte of data needed it comes to a total of 97 (`$61`) bytes, which is reserved right as the `UPLOAD` memory block after the available game RAM in `MAIN` and before the video RAM at `$C038`:

```
MEMORY {
    ZP:     file = "", define = yes, start = $0000, size = $0100;
    HEADER: file = %O,               start = $0000, size = __HEADERSIZE__;
    BOOT:   file = %O,               start = $0200, size = __STARTOFDIRECTORY__;
    DIR:    file = %O,               start = $0000, size = 16;
    MAIN:   file = %O, define = yes, start = $0200, size = $C038 - __UPLOADERSIZE__ - $200 - __STACKSIZE__;
    UPLOAD: file = %O, define = yes, start = $C038 - __UPLOADERSIZE__, size = $0061;
}
```

Additionally, you need to specify a file segment in your directory and game binary output that holds these segments. We will discuss this later during the implementation details.

# Uploading a program
The uploader logic and ISR requires you to send specific bytes over the serial connection from your client device (mostly a laptop or desktop computer) to upload the code for a program or game. These bytes include two magic bytes followed by a header with details of the code and finally the actual code. The code is sent byte by byte and is stored as a sequential block in RAM memory, starting at a memory location as specified in the header bytes. The load location is also the start address for execution after loading all bytes.

The documentation for BLL describes the sequence bytes for the uploader implementation:

```
*  start-sequence  : $81,"P"                       ; command : load program
*  init-sequence   : LO(Start),HI(Start)           ; start address = dest.
*  LO((Len-10) XOR $FFFF),HI((Len-10) XOR $FFFF)   ; and len of data
*  Xmit-sequence : ....                            ; data
*  checksum : none at all !! 
```

The serial connection needs to send the bytes from these three sections:
1. **Start**: magic bytes 
2. **Init**: code details for what will be transmitted next
3. **Transmit**: actual code that needs to be uploaded

## Start section debug bytes
The first two bytes are $81 and $50. The first one, $81 is a debug signal byte from the BLL libraries. It comes from a range of $81 to $86 to facilitate debugging, each byte representing a different command. The $81 header byte is followed by `P`, `R` or `S` for uploading, reset and creating a screenshot respectively. `P` has a hexadecimal value of $50, so for uploading this is the second byte to send. 

The two bytes signal a start of the upload sequence. The IRQ handler will pass execution to the uploader routine and will never return. 

## Init section to load code
As soon as the IRQ handler passes control to the uploader routine, it starts accepting 4 bytes in the init sequence. The sequence contains details about the load that will be loaded. It will use this to load, store and execute the uploaded code.

The four required bytes in the init sequence are:
- **Load address** (in RAM)  
This is the first address in RAM memory where the code will be stored after being received over the serial connection. The load address is also the address where execution starts after all code is received and stored. It is sent in high-low byte order for BLL and cc65 (even though the BLL documentation says otherwise).
- **Length of code** (in bytes)  
The length of the code is sent as a 16-bit number that is XOR'ed with `$FFFF` in high-low byte order. The main reason for this is efficiency in the routine for the loop over the low and high byte of the length in the uploader routine.

## Transmit section
The code will be transmitted after the init sequence. The routine receives as many bytes as where indicated in the init bytes. The received bytes need to be sent in order from lowest memory at the load address start in a contiguous block. The total number of bytes in the block must match the length from init. 

During receiving and storing of the uploaded bytes any present code will be overwritten in RAM memory. The loader routine should be in a memory area that is not overlapping with this contiguous block, as it will destroy its own loading logic and halt operations.

After all bytes have been received, execution is passed to the loaded code, by making a `JMP` to the load address. 

## A practical example
An uploadable file must be directly executable from its start and load address. This might be $0200, or any other chosen starting memory address in RAM, typically as low as possible, but after the zeropage and page 1 for the stack. Object files (`.o` or `.com` files) created by BLL and newcc65 are suitable. cc65 generated `.o` files not so much. 

Let's put together all data that needs to be uploaded. The prolog bytes in the start sequence are easy, since they are fixed.
The `init` sequence data is typically determined from the file header that precedes a BLL `.o` file.

Here is an example of the first 10 bytes of some header of a BLL `.o` file:

```
80 08 02 00 BA 00 42 53 39 33              ......BS93
```
The first bytes `80 08` are the header marker bytes. The next two bytes `02 00` indicate the start (and load) address of the code $0200, in high-low order. The following bytes `BA 00` are the length of the code fragment (again in high-low order): $BA00 = 47616 decimal _including the header of 10 bytes total_.
So, the actual length of the code is 47616 - 10 = 47606 = $B9F6. 

The XOR'ed actual length is `$B9F6 ^ $FFFF = $4609`

Taking this example the initial bytes to send for uploading are:

```
81 50         ; start sequence
02 00         ; load address hi-lo
46 09         ; (length - 10) ^ $FFFF hi-lo
...           ; code bytes
```

# Adding upload capabilities to your code
If you want to include the upload capabilities to your code, you need to do several things during development:
- Add a file segment for uploader code in your file directory on the cartridge image
- Load uploader segment before receiving logic is triggered
- Change configuration for your game code to cater for uploader
- Implement logic to set timer 4 of Mikey to correct source period and reload for your desired baud rate
- Clear receive buffer before receive procedure starts
- Enable receive interrupt in serial control bits to start actual receiving

At runtime the Lynx program needs to go through related steps to enable and bootstrap the upload process. This is typically done from a menu option or static screen in the game or program. It is less common to enable it during gameplay.
After activating the uploader in the Lynx you will need to have a program that sends the data over ComLynx from your client device to the Lynx. The uploader logic will take care of the rest.

## Adding file segment for uploader
The uploader logic will be loaded whenever needed by making a call to `lynx_load` with the file number inside your directory. Adding the file segment is similar to any other code and data segment in your cartridge layout.
Inside your `directory.asm` file include the symbols required for upload:

```
.export _UPLOAD_FILENR : absolute
.import __UPCODE_SIZE__
.import __UPCODE_LOAD__
.import __UPDATA_SIZE__
```

These are the export of the file number as created during the directory layout construction, so you can reference it later during `lynx_load`. The imports of the `UPCODE` size and load address as well as the `UPDATA` size is required to build a correct directory entry later:

```
; Uploader segment
_UPLOAD_FILENR=_MAIN_FILENR+1
entry mainoff, mainlen, uploadoff, uploadblock, uploadlen, __UPCODE_SIZE__ + __UPDATA_SIZE__, __UPCODE_LOAD__
```

In the example above, the upload file segment is included right after the main file. You can change it to be anywhere in the file structure by positioning it as such. The resultant file number `_UPLOAD_FILENR` will allow you to load it later when in C code by defining it as an external integer:

``` c
extern int UPLOAD_FILENR;

void initialize()
{
    lynx_load((int)&UPLOAD_FILENR);

    // No ser_install needed
    tgi_install(&tgi_static_stddrv);
    joy_install(&joy_static_stddrv);

    // Rest of initialization code omitted
}
```

The serial driver of cc65 does not need to be included if you only want to use the upload functionality. The ISR of `uploader.s` contain all logic and the other C code uses only the definitions of Mikey for serial control manipulation.

## Changing configuration
The cc65 has a separate configuration `lynx-uploader.cfg` that is ready to apply. You can use it in simple scenarios where you have a standard game with a single main game segment and only upload functionality added.

Your `Makefile` should specify the use of the uploader configuration file instead of the implied `lynx.cfg` by adding an extra argument `-C` to the linker `cl65`.

```
$(target) : $(objects)
	$(CL) -t $(SYS) -o $@ -v -C lynx-uploader.cfg -m uploader.map $(objects) lynx.lib
	$(CP) $@ $(BUILD)-$@
```

The `lynx-uploader.cfg` configuration reserves the uploader segment in RAM memory after `MAIN` and video memory. If you choose to load the uploader segment just-in-time you could alter this to overlap with a `MAIN` memory segment to contiguously connects to video memory. This will work if the loading of the uploader ISR logic will not overwrite an actively used part of `MAIN` memory. It is risky, but will save 97 bytes in cases where saving this memory is required for `MAIN`.

## Enabling serial communication

First, pick your desired baud rate for the uploader. You will need to select the right source period and reload values for timer 4.
Here is a list of calculated reload values and source periods that are common and also used in CC65 serial support for the Lynx.

|Reload   |Source period   |Clock speed   | Frequency   | Baud rate   |
|---|---|---|---|---|
|1   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((1+1)·8) = 62500 Bps   |
|3   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((3+1)·8) = 31250 Bps   |
|12   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((12+1)·8) = 9615 Bps   |
|25   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((25+1)·8) = 4807 Bps   |
|51   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((51+1)·8) = 4807 Bps   |
|68   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((68+1)·8) = 1811 Bps   |
|103   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((103+1)·8) = 1201 Bps   |
|207   | AUD_1   | 1 µs   | 1 MHz   | 1000000/((207+1)·8) = 600 Bps   |
|51   | AUD_8   | 8 µs   | 125 kHz   | 125000/((51+1)·8) = 300 Bps   |

Assuming the highest baud rate of 62500 is chosen, it implies a source period of 1 µs (microsecond) as indicated with `AUD_1` and a reload value of 1 (the lowest possible).
The next code fragment shows how to set timer 4 on Mikey to the corresponding values in cc65. 

``` c
// Set baud rate to 62500

// Turn on serial timer to 1 MHz (1 microsecond source period)
MIKEY.timer4.control = ENABLE_RELOAD | ENABLE_COUNT | AUD_1; // %0001 1000
// Reload is after 1 + 1 = 2 periods, with clock speed of 1 MHz => rate = 1M / (2*8) = 62500
MIKEY.timer4.reload = 1;
MIKEY.serctl = PAREN | TXOPEN | PAREVEN | RESETERR; // %0001 1101
```

The read buffer or receive register might be filled by previous uploads or by connecting the ComLynx cable. To make sure that the buffer and register are empty before we start reading the new incoming uploaded code, some dummy reads of `SERDAT` are appropriate.

``` c
unsigned char data; // At start of routine

// Clear receive buffer
while ((MIKEY.serctl & RXRDY) != 0)
{
    data = MIKEY.serdat; // Dummy read from receive buffer
}
```

Finally, the interrupt for receive needs to be enabled. The write interrupt should not be set, as it would potentially interfere with the receival of data, and we do not have any need to write out data over ComLynx in this period anyway.

``` c
MIKEY.serctl = RXINTEN | PAREN | RESETERR | TXOPEN | PAREVEN; // %0101 1101
```

You can continue normal code execution after this, because the start of the upload is interrupt driven. Once the serial connection receives bytes the ISR will jump into the receiver logic and normal code execution is no longer performed.

The uploader has a visual indication of green bars during upload. It allows the user to see that there is code actively being uploaded. After upload has completed, the new game or program is started automatically. A reset of your Lynx is required to start the original game with the uploader, unless the uploaded game also has this functionality.