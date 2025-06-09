# Description

To showcase the knowledge we gained from the course EEC 172 – Embedded Systems,
we exercise the skills taught in labs to design and implement a final project
with increased complexity and creativity. Our project is titled Pokémini Go, an
application that is inspired by the popular game Pokémon Go. For context,
Pokémon Go is a mobile application that uses augmented reality and the phone's
GPS sensor to spawn collectible characters, called Pokémon, on a real-world map.
To capture Pokémon, users must physically walk to real-world locations where the
characters have spawned. As the user physically moves, their real-time location
is updated on a map displayed in the application. When the user reaches close
proximity to that Pokémon, they can attempt to catch it and add it to their
collection. Our project Pokémini Go is a simplified version of Pokémon Go.
Pokémini Go implements the core functionality of Pokémon Go by spawning Pokémon
characters at real-world coordinates. Users must walk to these coordinates to
battle the Pokémon and add it to their collection.

## Features

### Frontend

The user interface to Pokémini Go is a 128x128 OLED display. We implement four
different OLED pages – a landing page, collection page, fight or flee page, and
fight page. We describe these pages in greater detail below.

1. When the program starts finishes setup, the user is navigated to the landing
   page. The landing page displays the board's real-time location in latitude
   and longitude. Furthermore, the coordinates of the Pokémon closest to the
   user are displayed. This informs the user of the nearest location they can
   physically approach to fight a Pokémon. The nearest Pokémon is calculated on
   the server, and the client periodically requests that data.
2. To view all the Pokémon that the user has collected, they can navigate to a
   collection page that displays character sprites for each Pokémon in their
   collection. 
3. When the user enters a 10-meter radius of a Pokémon, a "fight or flee" page
   is rendered. The user is informed that a Pokémon nearby has been detected.
   The user utilizes an IR remote to input whether they will flee or fight the
   Pokémon. If the user flees, they are redirected to the landing page.
   Otherwise, the user is redirected to a fight page, where they engage in a
   game to collect that Pokémon.
4. When the user is redirected to the fight page, they participate in a
   turn-based attack mini-game, where they use the IR remote to provide user
   input. Pressing a button from 0 to 9 on the IR remote will randomly harm
   either the user or the enemy. Both the user and the enemy start with four
   hearts. If the enemy is the first to lose all their hearts, the user wins and
   collects the Pokémon that had activated the fight. The user is redirected to
   the collection page, where they can view the new Pokémon in their collection.
   Otherwise, if the user is the first to lose all their hearts, the Pokémon
   "escapes", and the user loses the opportunity to collect that Pokémon. They
   are subsequently redirected to the landing page.

We also deploy a web application that displays a real-time map of all Pokémon in
our database. In addition to the coordinates displayed on the OLED landing page,
the map serves as another interface to determine the locations at which a user
can battle a Pokémon. The map also displays a location marker for the device on
which the user has accessed the website. As the user physically moves, their
real-time location is updated on the map. 

### Backend

We implement REST APIs on an Express server that is hosted on a VPS (Virtual
Private Server), and use AWS DynamoDB to store Pokémon and user data. We provide
the following description as a brief overview of the high-level functionality
that our backend offers. When a LaunchPad runs our application, its MAC address
is obtained to check whether this is a new or existing user. If this is a new
LaunchPad that runs our application, the server creates a unique user on AWS
DynamoDB, and the new user starts with an empty collection. Otherwise, if an
existing LaunchPad has already run our application, their collection of Pokémon
is retrieved from the database when the program is initialized. The user's
collection of Pokémon will be rendered on the collection page. As the user
physically moves, the LaunchPad periodically makes HTTP GET requests for the
Pokémon closest to them. When the server makes these distance calculations, it
performs additional calculations to ensure that enough Pokémon have spawned
around the user. In particular, the server will calculate if there are at least
6 existing Pokémon in a 100 meter radius of the user; if this threshold is not
met, the server generates more Pokémon in the user's 100 meter radius. If the
user is within a 10-meter radius of a Pokémon, the server informs the LaunchPad
that a fight can be activated. The "fight or flee" page is rendered on the OLED,
and the user has the choice to flee from the Pokémon or fight via a turn-based
attack mini-game. If the user fights the Pokémon and wins, the Pokémon is added
to the user's collection. Both a fight and flee will result in the Pokémon being
deleted from the database of available characters to be battled.

## Hardware

We use two sensing devices in our implementation of Pokémini Go – an IR receiver
and GPS module.

### IR receiver

An AT\&T IR universal remote control is used to send signals to an IR receiver.
We incorporate code from Lab 3 ("IR Remote Control Texting Over a UART Link") to
decode signals for identifying specific button presses. To reduce the noise from
the power source, the IR receiver is connected to a low-pass filter composed of
a 100 micro-farad capacitor and 100 ohm resistor. We press buttons on the IR
remote to get user input for the following functionalities: choosing to fight or
flee, switching between the landing page and collection page, and playing the
turn-based attack mini-game on the fight page.

### GPS module

We utilize a GPS module to get the user's location in latitude and longitude.
The module reads radio signals emitted by satellites, requiring at least three
satellites to calculate a position by trilateration. After the GPS successfully
finds a fix, the module transmits data using the NMEA-0183 format. Each
NMEA-0183 sentence begins with the character "$" followed by the talker
identifier and the sentence type. For our project, we parse data from the
"GNGGA" sentence. The talker identifier "GN" indicates that multiple satellite
systems, particularly from several constellations of the Global Navigation
Satellite Systems (GNSS), are used to produce data (footnote). "GGA" denotes the
sentence type as Global Positioning System Fix Data, which consists of 16
fields. The third and fifth fields are latitude and longitude, respectively.

## Protocols

Two hardware communication protocols are used to transmit information between
the MCU and peripheral devices. We use Serial Peripheral Interface (SPI) to
communicate between the CC3200 LaunchPad and OLED display. Furthermore, we use
UART to read data from the GPS module and debug to a local console.

### Serial Periphal Interface (SPI)

Serial Peripheral Interface (SPI) is a full-duplex synchronous communication
protocol, which is used to transmit data between the CC3200 LaunchPad and OLED
display. The protocol implements a master-slave interface, where the master
(i.e. MCU) initiates communication and the slave (i.e., OLED display) responds
to these requests. SPI data transmissions do not follow a standardized format
but comply with device specifications. The protocol uses four point-to-point
unidirectional wires as described below.

1. Serial Clock (SCK): A master-to-slave wire that transmits a master-generated
   clock signal.
2. Slave Select (SS): A master-to-slave wire that selects the specific slave
   device to enable communication with.
3. Master-Out Slave-In (MOSI): A master-to-slave wire that transmits data from
   the master to a slave device.
4. Master-In Slave-Out (MISO): A slave-to-master wire that transmits data from a
   slave device to the master.

## Universal Asynchronous Receiver/Transmitter (UART)

Universal Asynchronous Receiver/Transmitter (UART) is an asynchronous serial
communication protocol that uses two one-way wires, a connection for
transmitting data (TX) and another for receiving data (RX). As an asynchronous
protocol, the sender and receiver agree on a baud rate to indicate how often
each device should sample from their own local clocks. The sender initiates
transmission by sending a predetermined start symbol, which informs the receiver
to begin sampling data by the baud rate. After transmission of an $n$-symbol
data frame, the sender sends a stop symbol to end communication. We make the
distinction here between symbol and bit because baud rate describes sampled
symbols per second, not necessarily binary bits per second. However, most
electrical standards used in implementing UART use binary voltage symbols.

## Software

### Code Composer Studio and CC3200 SDK

Our project is implemented in the C programming language, which we compile and
load onto the CC3200 via the Code Composer Studio IDE. We build on top of the
starter code provided in lab 4 ("Introduction to AWS and RESTful APIs") and
import code from lab 2 ("Serial Interfacing with SPI and I2C") and lab 3 ("IR
Remote Control Texting Over a UART Link"). In particular, lab 2 provides the
OLED driver code, and our implementation of writing commands and data to the
display. Our lab 3 code contains the logic for decoding IR remote signals to
determine which button has been pressed. Furthermore, it implements UART
interrupts for communicating with another peripheral device. We extend the
application `aws-rest-api-ssl-demo` from lab 4, which contains the driver code
for Texas Instruments SimpleLink CC3200 SoC, and is what we use to network with
the internet using a WLAN adapter.

### cc3200tool

The `cc3200tool` is a serial flash utility that allows us to program flash
memory and open a pyserial terminal for UART communication between the CC3200
SoC and our local machine. The command-line utility can be installed from this
\href{https://github.com/toniebox-reverse-engineering/cc3200tool}{\color{blue}repository}.
To open a serial port on our local machine, we execute the following commands
and specify our baud rate as 115200.

```
conda activate cc3200
cc3200tool term 115200
```

We also use the `cc3200tool` to write to flash memory, ensuring that our project
is stand-alone. As mentioned in Lab 1 ("Development Tools Tutorial and Lab
Introduction"), we run the following command to write `pokemon-go.bin` to
`/sys/mcuimg.bin`. After the program has been written to non-volatile memory,
the application will run once the LaunchPad is connected to power. Pressing the
"Reset" button will restart the application. 

```
conda activate cc3200
cc3200tool --sop2 \~dtr --reset prompt \
    format_flash --size 1M \
    write_file pokemon-go.bin /sys/mcuimg.bin
```

### AWS DynamoDB

AWS DynamoDB is used to store Pokémon and user data on the cloud. With this
NoSQL serverless database, we create two tables – `PokemonTable` and
`UserTable`. `PokemonTable` consists of entries for each Pokémon that has
spawned and is available to be captured. `UserTable` contains an entry for every
unique LaunchPad that has run our application and tracks all Pokémon that each
user has collected. 

### REST APIs and Express

REST (Representational State Transfer) is a common set of guidelines for writing
Web APIs. Specifically, all REST APIs have stateless calls at the API level.
This means that a single call transfers some state either from the server or
from the client, and without the value dependent on the status of any other
call. Furthermore, REST APIs are conventionally implemented using HTTP with
semantically meaningful verbs and query parameters, and with JSON being the most
common payload data format for transmitting structured or relational data.

We use Node.js and TypeScript to create an Express server, where we implement
REST APIs to handle HTTP requests. To retrieve, delete, update, and perform
calculations on data within AWS DynamoDB, we write REST APIs to interface the
LaunchPad with the database. 
