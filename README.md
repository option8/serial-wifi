# RetroConnector Serial-Wifi Adapter
A serial-to-wifi adapter for the Apple II

The hardware is basically an RS232 to UART serial converter, connected to an ESP8266 serial-to-wifi board. Not just basically, but literally, that's what it is.

The ESP8266 board I've built and tested with is the Adafruit Huzzah breakout (https://www.adafruit.com/product/2471) along with a MAX232-based RS232-UART adapter. My test connection runs at 9600 baud. Though it should be able to go faster, there are pacing differences between the WiFi "modem" and the Apple II such that other speeds are unstable at best.

The ESP comes flashed with Lua, so I have flashed the boards using esptool.py (https://github.com/espressif/esptool) with the (at time of this writing) latest "No OS" firmware for the ESP8266 (http://bbs.espressif.com/download/file.php?id=1469). Newer versions are available here: (https://github.com/espressif/ESP8266_NONOS_SDK)

On the Apple II side, if you can get your program to send out each command through the built-in serial (//c) or Super Serial Card (IIe) and wait for the correct response before proceeding (or displays the result if not right) you can probably take and run with it to do arbitrary data I/O.

I've shown how to do the various required commands below, as well as check that they have been done properly. I foresee being able to query the current IP or connection status for troubleshooting, etc.


Here's the steps to get things talking. 

## Step 0, determine if we're actually talking to the ESP

Send:

	AT

every command is all uppercase, and followed by both carriage return and linefeed (ctrl-M, ctrl-J)

Rcv:

	OK

to double-check, 
Send:

	AT+GMR

Rcv: a bunch of version information, which should look something like this

	AT version:1.1.0.0(May 11 2016 18:09:56)
	SDK version:1.5.4(baaeaebb)
	compile time:May 20 2016 15:06:44
	OK

As long as all of that comes over clear without any garbage characters, comms between the Apple and the ESP are established and correct speed, stop bits, etc.


## Step 1a, server or client

Next up, are we setting up as server or client? Let's start with a server. This should setup the ESP as a wifi access point that other devices can connect to.

Send:

	AT+CWMODE=2

Rcv:
	
	OK

If the ESP was previously connected to another network automatically, you may also see:
WIFI DISCONNECT

to confirm, you can do:

	AT+CWMODE?

and get back:

	+CWMODE:2

	OK

## Step 2a, set the SSID

Next, we set up as an access point, with SSID "a2wifi" and password "apple24evr". The other parameters are channel=1 and security=WPA (mode 2, mode 1 is open)
Send:

	AT+CWSAP="a2wifi","apple24evr",1,2

Rcv:

	OK


Again, you can query this with:

	AT+CWSAP?

and get:

	+CWSAP:"a2wifi","apple24evr",1,2,4,0

	OK

## Step 3a, set up connections

Finally, we set it up to receive TCP connections. We'll need to know its IP number (for the client side to connect to later). In this case, it's automatically assigned itself 192.168.4.1.

send:

	AT+CIFSR

Rcv:

	+CIFSR:APIP,"192.168.4.1"
	+CIFSR:APMAC,"5e:cf:7f:15:3c:50"
	+CIFSR:STAIP,"0.0.0.0"
	+CIFSR:STAMAC,"5c:cf:7f:15:3c:50"

	OK

And create the server socket, on port 6502, of course.
send (this allows multiple connections to be set up):

	AT+CIPMUX=1

Rcv:

	OK

Send:

	AT+CIPSERVER=1,6502

Rcv:

	OK

Now, it's waiting for connections. You can even test this by connecting to the access point from your computer and trying the IP address and port from a web browser. It'll spin waiting on a response, but you should see a bunch of HTTP connection information come across on the serial console.


___


## Step 1b, set up the client

For the other Apple II and ESP, we set up as a client, and connect to the base station setup above.

send: 

	AT+CWMODE=1

Rcv:

	OK

and to test:

	AT+CWMODE?

Rcv:

	+CWMODE:1

	OK


## Step 2b, connect to the server's SSID

Now, it's possible to get a list of all base stations/networks, and hopefully see the one we want to connect to.

send:

	AT+CWLAP

and receive a potentially long list like so:

	+CWLAP:(4,"a2wifi",-57,"80:00:6e:f3:fb:2e",6,-39,0)
	+CWLAP:(4,"7700-1",-76,"90:3e:ab:f0:da:20",11,-14,0)
	+CWLAP:(3,"VR116-EA88",-61,"84:16:f9:f4:ea:88",11,3,0)
	+CWLAP:(4,"ATTEuztmGs",-95,"3c:36:e4:38:ac:60",1,5,0)

	OK

To join one of those networks, send the SSID and password, quoted:

	AT+CWJAP="a2wifi","apple24evr"

hopefully, after a few seconds, you will see:

	WIFI CONNECTED
	WIFI GOT IP

	OK


And to query the current network:

	at+CWJAP?

Rcv:

	+CWJAP:"a2wifi","80:00:6e:f3:fb:2e",6,-65

## Step 3b, TCP/IP connections

Now the real fun. Create the TCP connection to start slinging bytes. Needs a connection number to reference, the protocol, the IP and port.

	AT+CIPSTART=1,"TCP","192.168.4.1",6502


## Step 4, profit!

Once the connection is established, each station can send data to the other with 

	AT+CIPSEND=X,Y
	
with X being the connection number, and Y being the number of bytes to send, followed by a CR/LF, then the actual bytes. Once the bytes equal the length specified, the connection sends the filled buffer all at once.

