# Challenges

We encountered a couple of major challenges during the implemen-
tation of our project.

## Unable to connect to common HTTPS endpoints

Our initial idea was to have the Launchpad make HTTPS requests to an AWS
Lambda endpoint. AWS Lambda is a "serverless" compute service.
That is, it runs a defined function when an HTTP endpoint is
called, and thus allows us to write API handlers without having to
manually configure and maintain any extra server infrastructure.
However, we encountered major difficulties when attempting to
establish a TLS connection to the Lambda endpoint. At first, we
attempted establishing a TLS connection to the lambda endpoint
using the same TLS root, client, and private certificates that we had
configured from Lab 4. However, we received error code `-6` from
the function `sl_Connect()`, indicating that a connection could not
be established on the configured TLS socket. We then attempted
manually downloading and flashing the AWS CA Root Certificate
onto the Launchpad. We verified that the certificate was in DER
format and also attempted consolidating it with intermediary certificates used on our particular Lambda endpoint into other DER
files. However, none of these certificate chains permitted a successful TLS connection. After reading through AWS Lambda TLS
documentation on supported TLS ciphers, we hypothesized that
perhaps the default directly-exposed Lambda certificate required
the use of recent ciphers that the Launchpad did not support. Thus,
we also tried configuring an API Gateway, which is an AWS service
for a REST API front-end that allows for more fine-grained configuration of API endpoints. In particular, API Gateway supports lower
security levels and older ciphers to be explicitly configured on select endpoint paths to support older clients. However, we could not
establish a TLS connection to the API Gateway either.

We also considered using MQTT and just using the IoT shadow
endpoint as previously used in Lab 4. For MQTT, we read through
the sample client code provided in the TI SDK. However, we realized
that the MQTT code required the use of lightweight operating
systems, namely Ti-OS or FreeRTOS, which use hardware interrupts
to implement pre-emptive scheduling for multitasking. If we were
to adapt either of the recommended MQTT clients, then we would
have to rewrite all of our logic to use those operating systems, and
given that we had around 4 days at that point to get a fully working
project while balancing other classes, we decided we did not have
enough time to experiment with MQTT.

The other option was using IoT shadow. However, we determined
that IoT shadow would add another layer of indirection and thus
complexity, since we would still need some kind of business logic
layer such as a Lambda function to compute operations on the
database, and as well as maintaining the separate `reported` and
`desired` IoT Shadow states.

Thus, we decided instead to simply implement an unsecured
HTTP endpoint on an AWS EC2 VPS instance that we already rent
yearly. This is where we decided to use Express.JS to implement this
HTTP API endpoint, and so then we have this service scheduled to
automatically run using Node.JS on the VPS.

This was a simple solution permitted under the project constraints that allowed us to hit all of our minimal, target, and numerous of our stretch goals on-time.

## Unable to receive a full NMEA sentence

Initially when we attempted reading the GPS module NMEA sentences over UART and
printing these sentences out into the computer UART console, we
discovered that we were never reading a full NMEA sentence. Specifically, our GNGGA sentence would always cap at 16 bytes/ASCII
characters.

We remembered from Lab 3 that this maximum value was suspiciously similar to the buffer size of the UART FIFO buffer. Thus,
we immediately suspected a similar issue as to what we had in Lab
3, where our RX buffer interrupt handler was not reading received
bytes fast enough to clear the buffer and allow for the rest of the
message to be received.

Firstly, we tried setting the baudrate to the minimum allowed
setting. However, for our particular board and chip, the HGLRC
M100-5883 with a UBlox M10 baseband chip, the minimum UART
speed that we were able to observe a valid response was 19200 baud.
Even at this baudrate, we still were only able to read the first 16
characters of any NMEA message.

Then, we decided another technique to also reduce the total
number of interrupts that we would have to trigger. Instead of
triggering the UART RX interrupt handler on every received byte,
we instead configured our RX buffer interrupt handler to run only
after every 8th received byte - a half full buffer:

```
MAP_UARTFIFOLevelSet(UARTA1_BASE, UART_FIFO_TX4_8, UART_FIFO_RX4_8);
```

After setting this new interrupt trigger, we were able to observe
full NMEA sentences for all transmitted sentences. We hypothesize
that this made such a big difference because it takes a significant
amount of time to altogether raise the interrupt line, enter the
interrupt service routine, run the peripheral code we implement to
read the buffer characters and parse them, and then return control
back to the main process. Thus, we divide the total amount of
interrupt handles by a factor of 8, significantly increasing the rate
at which we can handle incoming UART data.

Lastly, although we at first tested this working solution at 19200
baud, we then decided to increase the baudrate back up to 57600
baud, which is the highest baud at which we were able to consistently receive correct NMEA sentences. The reason for attempting
to drive the baudrate back up is because of the high throughput of
data transmitted by the GPS module. Thus, to not hog resources
away from other actions in the game loop, we try to make the transmission and receipt of GPS data as fast as possible. As is discussed
above, we also combine this with the GPS module timeout where
we disable the GPS UART RX interrupt when not in use.