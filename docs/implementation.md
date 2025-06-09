# Description

## Features

### Frontend

The user interface to Pokémini Go will be a 128x128 OLED display. We implement
four different OLED pages – a landing page, collection page, fight or flee page,
and fight page. We describe these pages in greater detail below.

We also implement a web app that displays a real-time map of all Pokémon in our
database. In addition to the coordinates displayed on the OLED landing page, the
map serves as another interface to determine the locations at which a user can
battle a Pokémon. The map also displays a marker for the device on which the
user has accessed the website. As the user physically moves, their real-time
location will update on the map. 

### Backend

We implement RESTful APIs on an Express server that is hosted on a VPS, and use
AWS DynamoDB to store Pokémon and user data. We provide the following
description as a brief overview of the high-level functionality that our backend
offers. When a LaunchPad runs our application, its MAC address is obtained to
check whether this is a new or existing user. If this is a new LaunchPad running
our application, the server creates a unique user on AWS DynamoDB, and the new
server starts with an empty collection. Otherwise, if an existing LaunchPad has
already run our application, their collection of Pokémon is retrieved from the
database when the program is initialized. The user's collection of Pokémon will
be rendered on the collection page. As the user physically moves, the LaunchPad
periodically makes HTTP GET requests for the Pokémon closest to them. When the
server makes these distance calculations, it performs additional calculations to
ensure that enough Pokémon have spawned around the user. In particular, the
server will calculate if there are less than 10 existing Pokémon in a 100 meter
radius of the user; if this threshold is not met, the server generates more
Pokémon in the user's 100 meter radius. If the user is within a 10 meter radius
of a Pokémon, the server informs the LaunchPad that a fight should be activated.
The fight or flee page is rendered on the OLED, and the user has the choice to
flee from the Pokémon and fight via a turn-based attack mini game. If the user
fights the Pokémon and wins, the Pokémon is added to the user's collection. Both
a fight or flee will result in the Pokémon being deleted from the database of
available characters to be battled.

## Hardware

We uses two sensing devices in our implementation of Pokémon Go – an IR receiver
and GPS module.

### IR Receiver

An IR remote control is used to send signals to an IR receiver. We incorporate
code from Lab 3 ("IR Remote Control Texting Over a UART Link") to decode these
signals. We press buttons on the IR remote to get user input for the following
functionalities: choosing to fight or flee, switching between the landing page
and collection page, and playing the turn-based attack mini game on the fight
page.