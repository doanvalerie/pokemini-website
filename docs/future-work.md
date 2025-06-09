# Future Work

## Create a JSON parser

When the LaunchPad makes an HTTP GET request for user data or the nearest
Pokémon, the API handlers in our Express.JS server return a newline separated
payload. Rather than listing key-value pairs in a JSON format, our response only
returns the values with a newline delimiter. Although this makes it easier to
parse in C, it is not a robust solution. There is a lack of clarity for the
meaning of each line, considering that the key is not present. We implement
numerous heuristics in our parsing to verify validity of each transmission and
whether certain fields are present or absent, rather than adhering to a strict
grammar like JSON. As an example, our parsing logic depends on the order of each
line. Therefore, if we extend our APIs to return additional data in the payload,
we also have to change our C code to ensure that we are reading corresponding
data from the correct line. This contrasts JSON notation, where the keys are not
ordered, and values are instead interpreted according to the exact key they 
correspond to.

Although we initially began writing a JSON parser from scratch in C, we later
found it more important to concentrate our efforts on the goals we set during
our project proposal. Therefore, we converted our payloads from JSON notation to
a newline separated payload. In the future, we would like to create a JSON
parser so that we can return data in a stricter structure that can easily be
extended and correctly read with lower chance of bugs.

## Integrate compass sensor

One of our stretch goals was to use a peripheral compass sensor that would send
magnetic heading data to the LaunchPad. We would then use this compass to
determine the direction to the nearest Pokémon by calculating the vector from
the user’s GPS coordinates to the nearest Pokémon’s GPS coordinates, and then
calculating the angular offset from the direction that the LaunchPad is
pointing. Then, we would render an arrow in a virtual compass on the landing
page to direct the user in the direction toward the nearest Pokémon.

The day before our project verification day, we had finished all of our minimal
and target goals and also numerous of our stretch goals, so we attempted to try
implementing a Bosch BMM150 magnetometer to retrieve magnetic heading data.
However, we quickly found out that integrating a magnetometer without any
existing library code was a monumental challenge.

In particular, we discovered that each individual magnetometer needs to be
calibrated to account for its own imprecisions from its manufacturing, as well
as the particular magnetic field strengths as derived from the Earth’s field in
our particular geographic region. In particular, we noted that we needed to
perform two forms of calibration. A "hard iron calibration" is performed to
compensate for local disturbances in the regional magnetic field caused by
nearby objects such as small magnets. Then, a "soft iron calibration" is
performed to compensate for any inaccuracies in the relative strength of the
magnetic field along any of the euclidean axes when rotating the magnetometer
along any axis.

These calibrations are complicated and are usually performed with existing
calibration software, using better documented boards with existing libraries,
such as an Arduino or a Raspberry Pi. On the LaunchPad CC3200, we lacked any
code to assist us in performing magnetometer calibration. Thus, after
approximately four hours of attempting to perform temporary calibrations, we
decided to move on to abandoning the compass approach and instead polishing the
rest of the project. Without any calibration, the magnetometer would return
widely varying values depending both on its absolute location and rotation even
when moving the magnetometer around just inches at a time. Thus, the
magnetometer was impossible to use in an uncalibrated form.

It was after attempting the compass module that we then implemented one of our
other stretch goals, which was to implement a real-time web map of the Pokémon.
This provided similar functionality of guiding the user to the nearest Pokémon.