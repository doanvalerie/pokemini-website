# Implementation

## GPS Driver Code

### Configure UART interrupts
The GPS module uses the UART protocol to transmit data. To set up communication
between the CC3200 LaunchPad and GPS module, we enable UART1 on SysConfig and
configure GPIO 11 (Pin 02) as the RX pin and GPIO 03 (Pin 58) as the TX pin. In
our circuit, we connect UART1 RX and TX pins on the LaunchPad to the TX and RX
pins of the GPS module, respectively. To implement UART interrupts, we closely
reference our implementation from Lab 3 ("IR Remote Control Texting Over a UART
Link") and the [UART API](https://software-dl.ti.com/ecs/cc31xx/APIs/public/cc32xx_peripherals/latest/html/group___u_a_r_t__api.html).
For UART data transmissions, the CC3200 utilizes a 16-byte receive FIFO buffer.
When GPS data is written to the RX pin of UART1, each byte is appended to the
FIFO buffer, from which we read the data. We use `UARTFIFOLevelSet` to assign
the receive FIFO interrupt level as `UART_FIFO_RX4_8`. As a result, an interrupt
is generated for every 8 bytes written to the receive FIFO buffer.

### UART interrupt handler

For every 8 bytes written to the receive FIFO butter, the UART interrupt handler
is executed. This interrupt service routine is responsible for reading the
NMEA-0183 sentences. When a sentence of type "GNGGA" is read, it parses the
latitude and longitude to assign to the game state. [Aceinna OpenRTK Developer manual](https://openrtk.readthedocs.io/en/latest/communication_port/nmea.html)
provides an example of a "GNGGA" sentence from the NMEA-0183 standard. We
include that example for reference below. It is a comma-delimited sentence that
starts with `$` and ends with `\r\n`.

```
$GNGGA,072446.00,3130.5226316,N,12024.0937010,E,4,27,0.5,31.924,M,0.000,M,2.0,*44 
```

For each byte in the receive FIFO buffer, `UARTCharGet` is used to read the
character and remove it from the buffer. We declare a `volatile` array of `char`
named `rx_buffer` that is large enough to store a single NMEA-0183 sentence. We
also declare `rx_buffer_idx` – a variable to store the current length of the
NMEA-0183 sentence that we are reading. When we read `$`, we also write the
following bytes to a second `volatile` buffer, named `sentence_type`, until we
reach the first comma delimiter.

Reading `\n` indicates that we have finished reading an entire NMEA-0183
sentence to `rx_buffer`. We check if `sentence_type` stores the string `GNGGA`.
If so, we parse `rx_buffer` to obtain the latitude and longitude. Afterwards, we
reset `rx_buffer_idx` to zero, so we can read a new NMEA-0183 sentence to
`rx_buffer`.

### Parse `rx_buffer`

When an entire "GNGGA" sentence has been read to `rx_buffer`, we use `strtok` to
parse the comma-delimited string into an array of `tokens`. To check if this is
a valid GPS reading in North America, we confirm that fourth field – direction
of latitude – and sixth field – direction of longitude – are "N" (i.e., north)
and "W" (i.e., west), respectively. We also confirm that the latitude and
longitude are not empty fields, which can occur when the GPS failed at finding a
fix.

According to [Trimble's guide on NMEA-0183 messages](https://receiverhelp.trimble.com/alloy-gnss/en-us/NMEA-0183messages_CommonMessageElements.html#:~:text=NMEA%20Message%20values&text=Latitude%20is%20represented%20as%20ddmm,dd%20or%20ddd%20is%20degrees), an NMEA-0183 sentence formats the latitude as
"ddmm.mmmm" and longitude as "dddmm.mmmm", where "dd" and "ddd" represent
degrees while "mm.mmmm" denote minutes. We reference this
[StackOverflow answer](https://stackoverflow.com/questions/36254363/how-to-convert-latitude-and-longitude-of-nmea-format-data-to-decimal) to convert
the values to decimal degrees – a common way to express
latitude and longitude values. After converting the strings to floats, we divide
the minutes by 60 and add this quotient to the degrees. Since North America is
located in the western hemisphere, we negate the latitude. 

### Set baudrate

By connecting the LaunchPad's UART1 TX pin to the GPS module's RX pin, we can
write commands to the GPS module for configuration. By default, our GPS module
communicates at a baud rate of 115200. Initially when we attempted sending the
GPS module's generated NMEA sentences to the LaunchPad using the default 115200
baud, we noticed that the messages were being received incomplete. Notably,
sentences would be received cut-off such as in the following example:
`$GNGGA,072446.00`. Here, only the timestamp of the GNGGA sentence is being
received, but otherwise the 16 character FIFO buffer is filled up and unable to
receive the rest of the message before the LaunchPad code is fast enough to read
the buffer. By halving the baud to 57600, we were able to slow the message
transmission enough to allow the LaunchPad to read the message faster than it is
transmitted.

Changing the baud is an involved process. Firstly, the specific GPS module that
we use is the [HGLRC M100-5883](https://www.hglrc.com/products/m100-5883-gps?srsltid=AfmBOor17IWhtMt4P6nsx0QI5u1bGGEDSzH--SjuEkOcAcFVDiZUdRY5). 
This package specifically uses the [UBlox M10 series baseband GPS chip](https://content.u-blox.com/sites/default/files/MAX-M10S_DataSheet_UBX-20035208.pdf).
That chip uses the [UBlox Protocol](https://content.u-blox.com/sites/default/files/products/documents/u-blox6_ReceiverDescrProtSpec_%28GPS.G6-SW-10018%29_Public.pdf)
which specifies the supported NMEA output as well as the format for configuring
the chip itself.

For configuring the baud rate on the UBlox chip, first we reference section
21.12 of the [UBlox protocol specification](https://content.u-blox.com/sites/default/files/products/documents/u-blox6_ReceiverDescrProtSpec_%28GPS.G6-SW-10018%29_Public.pdf) 
which specifies sending a UART message to the RX pin of the UBlox chip with
header `$PUBX,41,1`. Then, as shown in the following screenshot from the
documentation, we append the fields `,0007,0003` to update UART baudrate
specifically. 

![Landing Page](./assets/set-baud-rate.png)

Then, we append `,57600` to specify the new baudrate. Next, we append `,0` as
the default option for "autobauding," which we do not concern ourselves with.
Finally, we must calculate and append a checksum value beginning with an
asterisk and then followed by the one-byte checksum value in hexadecimal format.

In this case, the command string is:

```
$PUBX,41,1,0007,0003,57600,0
```

For the UBlox checksum, we calculate the XOR8 checksum. The XOR8 checksum simply
takes each byte of the message and then computes the bitwise exclusive-or
operation on each successive byte, returning a final checksum byte. We use
[this online tool](https://www.convertcase.com/hashing/xor-checksum)
to calculate the checksum for `PUBX,41,1,0007,0003,57600,0` which is the same
command string as above except without the leading $ character.

Thus, the resulting checksum in hexadecimal is `2b`. Finally, this is appended
to the original message to result in a final command message of
`$PUBX,41,1,0007,0003,57600,0*2b`.

After this command is sent to reconfigure the baudrate of the GPS module, we
then change the UART controller configuration for the UART1 interface to use
57600 baud by calling the `UARTConfigSetExpClk()` function as follows:

```
MAP_UARTConfigSetExpClk(
    UARTA1_BASE,
    MAP_PRCMPeripheralClockGet(PRCM_UARTA1),
    57600,
    (UART_CONFIG_WLEN_8 | UART_CONFIG_STOP_ONE |
     UART_CONFIG_PAR_NONE)
);
```

After updating the baudrate value, we restart the program, but while keeping the
LaunchPad and thus the GPS module powered throughout, ensuring not to lose the
temporarily updated baudrate.

We then attach a Saelee logic analyzer to the RX pin of the GPS module and
capture a sample NMEA sentence to ensure that the baudrate remains at 57600.

Next, we construct a command to save the configuration of the GPS module to the
GPS module's onboard non-volatile memory.

In particular, we reference the following [UBlox protocol section 31.2.1](https://content.u-blox.com/sites/default/files/products/documents/u-blox6_ReceiverDescrProtSpec_%28GPS.G6-SW-10018%29_Public.pdf)
which instructs transmitting a UART message to the UBlox chip RX pin with header
bytes `0xB5 0x62 0x06 0x09`. 

![Landing Page](./assets/ublox-cfg-cfg.png)

Then, we append the byte `13` in decimal to indicate a payload length of 13
bytes. Finally, we append the payload of 32 `0` bits for the `clearMask` and
`loadMask` values, and then bytes `0xFF 0xFF 0x00 0x00` for the `saveMask`.
According to the following screenshot from section 31.2.1, setting the lower two
bytes to all ones will cause all of the GPS module's current configuration to be
saved: 

![Landing Page](./assets/ublox-cfg-save.png)

Then, according to the bottom of section 31.2.1, we set the 13th bit of the
payload to `0xF` to flash the configuration to all the available memory sources,
importantly including the non-volatile sources: 

![Landing Page](./assets/ublox-cfg-device.png)

Finally, this command takes two separate checksum values. The two checksums are
likely because the command is critical to receive correctly; a corrupt saved
configuration would essentially brick the GPS module unless the user is
successfully able to reverse engineer and then communicate with the corrupted
configuration.

The two checksums `CK_A` and `CK_B` are computed as follows:

![Landing Page](./assets/ublox_cksum.png)

CK\_A and CK\_B are appended to the payload.

Finally, this entire message is written at once to the UBlox chip over UART. We
remove power from the GPS module for 10 seconds and then re-attach power to
ensure that the new baudrate is permanently stored in non-volatile memory.

### Disable and re-enable interrupts

The HGLRC UBlox GPS module sends approximately 15 NMEA sentences, each 10 times
per second. Each NMEA sentence is around 50 characters. Approximately 7500 bytes
per second of constantly interrupting the LaunchPad causes a resource hogging
problem where other program behavior, such as network requests and OLED screen
updates, either don't run predictably or stop working for long periods of time.

To fix this, we implement a solution where the UART1 RX buffer interrupt handler
is initially disabled, and then remains disabled until approximately 2 seconds
pass. We hand-tune this interval, eventually settling on a constant in our
`gpg.c` file that we define as `const int GPG_READ_LIMIT = 60000`. How this
works is that on every iteration of our main game loop in `game.c`, a function
`PollEnableGPS()` is called which will increment a counter `gpg_read_timeout`.
Then, when `gpg_read_timeout` reaches `GPG_READ_LIMIT` the UART1 RX buffer
interrupt handler is re-enabled, allowing the board to respond to and process
the influx of GPS NMEA sentences. Then, when the GPGGA sentence is fully parsed,
the UART1 RX buffer is disabled again. This strategy allows the board to sample
the GPS NMEA sentences at a much slower rate that is acceptable, allowing other
program processes to have enough time to execute successfully at their own
respective rates.