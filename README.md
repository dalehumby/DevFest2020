# Google DevFest South Africa 2020

Demonstrating the capabilities of the [ESP8266](https://www.espressif.com/en/products/socs/esp8266) WiFi enabled microchip from Espressif Systems for home automation and IoT projects. I will show you how to

- turn an LED on and off
- count pulses by detecting changes in the state of an input pin
- publish those changes to an [MQTT](https://mosquitto.org/) pub/sub broker
- host a simple web page

I am using the [D1 Mini](https://www.wemos.cc/en/latest/tutorials/d1/get_started_with_micropython_d1.html) development board which can be purchased from [Communica](https://www.communica.co.za/products/bmt-d1-mini-pro-esp8266-16m-ant) or [Micro Robotics](https://www.robotics.org.za/MINI-D1-4M) in South Africa.

This board is useful because it has the ESP8266 soldered on to a PCB and includes USB-to-serial converter, and 5 VDC USB voltage is stepped down to 3.3 V that the ESP needs, so it can be plugged directly into a computer's USB port.

It also includes a reset pin, an LED, and header sockets for easily connecting to other equipment.

![ESP8266 D1 Mini dev board](https://i.ibb.co/QYjTPxH/MINI-D1-4-M-008.jpg "D1 Mini")

## Install USB to serial drivers on MacOS
https://learn.sparkfun.com/tutorials/how-to-install-ch340-drivers/all

## Instructions to set up ESP8266

1. Make yourself a development directors, eg. `mkdir espdemo`, and then `cd espdemo`
2. Create a virtual environment called `env`: `python3 -m venv env` 
3. Activate the virtual env: `source env/bin/activate`
4. Install ESP Tool: `pip install esptool`
5. Plug in the ESP8266 dev board to USB
6. Find the ESP dev board: `ls /dev/ | grep usb`, will return a something like `tty.wchusbserial1410`

### Flash MicroPython on to the ESP board
I am using the tutorial from: https://docs.micropython.org/en/latest/esp8266/tutorial/intro.html

1. Download MicroPython firmware for the ESP8266. I am using the [Generic ESP8266 module](http://micropython.org/download/esp8266/) firmware with filename `esp8266-20200911-v1.13.bin`
2. Next erase the ESP8266 flash. **NOTE**: All data on the device will be irrecoverably lost. 
    - `esptool.py --port /dev/tty.wchusbserial1410 erase_flash` (Use the USB device you got in the command on step 6 above.)
    - All data on the device will be erased
3. Flash the MicroPython firmware: `esptool.py --port /dev/tty.wchusbserial1410 --baud 460800 write_flash --flash_size=detect 0 ~/Downloads/esp8266-20200911-v1.13.bin`
    - Use your own USB port and location of the .bin file

### Connect to the REPL using a serial emulator

1. I use `screen` to connect to the serial port, but you can use other serial terminal emulators. `screen /dev/tty.wchusbserial1410 115200`
2. You should see a Python prompt come up, similar to 

```
MicroPython v1.13 on 2020-09-11; ESP module with ESP8266
Type "help()" for more information.
>>> 
```

3. Type `print('Hello ESP')` and press Enter. You should see the text printed.
4. If this worked then you are all set!
5. To exit back to your shell: Type `CTRL+A` and then `CTRL+\` and then `y` to confirm. This will take you out of Python and back to your shell. Note `exit()` does not work.

### Install the file downloader

I've been using Adafruit's [Ampy](https://pypi.org/project/adafruit-ampy/). To install:
1. `pip install adafruit-ampy`
2. To see what files are already on the device: `ampy -p /dev/tty.wchusbserial1410 -b 115200 -d 0.5 ls`
    - I added a 0.5s delay, which is sometimes necessary to give the board time to reset
    - `ls` lists all the files on the device

This command should return
```
/boot.py
```

as the only file currently on your device.


## Let's write something useful
Mostly using information from the [MicroPython ESP8266 Quick Reference](https://docs.micropython.org/en/latest/esp8266/quickref.html) guide.

Once again get back to the Python REPL by connecting to the serial terminal to the ESP board: `screen /dev/tty.wchusbserial1410 115200`

### Connect to your WiFi
ESP's only support 2.4 GHz WiFi, so make sure your WiFi router supports that. (5 GHz Wifi is not yet supported.)

The snippet of code will connect to your wifi. Change `testbench` and `mysecretpassword` to your WiFi access point name and password.

```python
import network

station = network.WLAN(network.STA_IF)
station.active(True)
if not station.isconnected():
    print("connecting to network...")
    station.connect("testbench", "mysecretpassword")
    while not station.isconnected():
        pass
print("Network config:", station.ifconfig())
```

Python's [auto-indent](https://docs.micropython.org/en/latest/reference/repl.html#paste-mode) will mess things up if you paste directly. First `CTRL+E` to switch off auto-indent, paste the code above, then `CTRL+D` to get back to normal mode. Press Enter, and your board should connect to your WiFi.

This will output the tuple:
```
network config: ('10.0.0.154', '255.255.255.0', '10.0.0.254', '8.8.8.8')
```

Where
- `10.0.0.154` is this devices IP address on your LAN
- `255.255.255.0` is your subnet mask
- `10.0.0.254` is the network gateway
- `8.8.8.8` is the DNS server (in this case, one of Google's public DNS servers)

### Synchronise your clock using NTP

```python
from machine import RTC
import ntptime

rtc = RTC()
ntptime.settime() # set the rtc datetime from the remote server
rtc.datetime()    # get the date and time in UTC
```

### Switch an LED on
The [D1 Mini](https://www.robotics.org.za/MINI-D1-4M) board that I am using has an LED on GPIO pin 2.

```python
from machine import Pin

pin2 = Pin(2, Pin.OUT)
```

You will notice that the LED is on as soon as you set the pin to output. The LED is on when the pin value is 0; and the LED is off when the pin value is 1.

Try this yourself:

```python
pin2.value(0)  # LED is on
pin2.value(1)  # LED is off
```

### Set up an input pin
This code sets GPIO pin 5 to be an input, and whenever the input changes from logic low to a logic high (a rising edge) then call the function `pulse()`, which prints out that a pulse was detected.

```python
from machine import Pin

def pulse(pin_id):
    print("Pulse detected", pin_id)

pulse_pin = Pin(5, Pin.IN, Pin.PULL_UP)
pulse_pin.irq(handler=pulse, trigger=Pin.IRQ_FALLING)
```

As an exercise, add a global variable called `count` that is incremented and printed out whenever a pulse is detected.

### Sending data to MQTT
[MQTT](https://mqtt.org/) is a lightweight pub/sub protocol designed for IoT. I am using the open source [Mosquitto](https://mosquitto.org/) broker in my home.

To send an MQTT message whenever a pulse occurs:

```python
import network
from machine import Pin
from micropython import schedule
from umqtt.simple import MQTTClient

def pulse(state):
    global count  # Gloabl scope so count is incremented
    print("Pulse detected")
    count += 1
    schedule(mqtt_pub, count)  # Send the message outside of the interrupt service routine

def mqtt_pub(count):
    print("Pubishing to MQTT")
    mqtt.publish(b"counter", bytes(str(count), "utf-8"))

# Setup Wifi
station = network.WLAN(network.STA_IF)
station.active(True)
station.connect("testbench", "mysecretpassword")
while not station.isconnected():
    pass
print("Connected to wifi")
print("Network config:", station.ifconfig())

count = 0
pulse_pin = Pin(5, Pin.IN, Pin.PULL_UP)
pulse_pin.irq(handler=pulse, trigger=Pin.IRQ_FALLING)
mqtt = MQTTClient("espdevboard", "10.0.0.10")
mqtt.connect()
```

To test, connect to your MQTT server using Mosquitto Client CLI.
1. I am assuming that you already have an MQTT broker up and waiting for clients to publish and subscribe to messages. (If not, try this [tutorial](https://medium.com/@rossdanderson/installing-mosquitto-broker-on-debian-2a341fe88981).)
1. If you don't already have the MQTT client CLI installed, on Debian `sudo apt-get install mosquitto-clients`
2. Subscribe to the topic `counter` on your MQTT server (mine is at `10.0.0.10`)
    - `mosquitto_sub -v -h '10.0.0.10' -t 'counter'`
3. To test, in the MicroPython REPL, paste `mqtt.publish(b"counter", b"hello mqtt")`
    - You should see `counter hello mqtt` printed out at your MQTT client

Now press the button on your dev board. After each button press each MQTT client that is subscribed to the topic `counter` will receive the incrementing number.


### A simple web server
The final step is to create a simple web page that displays the current count. To do this we set up a simple web server that watches for HTTP requests to /metrics and returns a simple text webpage.

Restart your dev board (CTRL+D) and then paste the following:

```python
import usocket as socket
import network
from machine import Pin

def pulse(state):
    global count  # Gloabl scope so count is incremented
    print("Pulse detected")
    count += 1

# Setup Wifi
station = network.WLAN(network.STA_IF)
station.active(True)
station.connect("testbench", "mysecretpassword")
while not station.isconnected():
    pass
print("Connected to wifi")
print("Network config:", station.ifconfig())

count = 0
pulse_pin = Pin(5, Pin.IN, Pin.PULL_UP)
pulse_pin.irq(handler=pulse, trigger=Pin.IRQ_FALLING)

# Setup the web socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(("", 80))
s.listen(10)
print("Ready for http connections")

# Wait for web requests
while True:
    conn, addr = s.accept()
    print("Got a connection from", str(addr))
    request = conn.recv(1024)
    print(request)
    if request.find(b"GET / ") >= 0:
        print("Handle GET")
        conn.send("HTTP/1.1 200 OK\n")
        conn.send("Content-Type: text/html\n")
        conn.send("Connection: close\n\n")
        conn.sendall("""
            <h1>Counter</h1>
            The count is at {count}""".format(count=count))
    conn.close()
```

## Putting it all together
We've seen how, using the ESP8266 microcontroller we were able to 
- turn an LED on and off
- count pulses by detecting changes in the state of an input pin
- publish those changes to a MQTT pub/sub broker
- host a simple web page that displays the count

My [Powermeter](https://github.com/dalehumby/powermeter) repository fleshes out what I have demo'ed here, and incudes
- Better handling of transient network issues
- Periodically storing the `count` value to flash
- Prometheus endpoint so the count value (among other metrics) can be scraped, stored and displayed by Grafana
- Better structured code, including a config file to store settings like wifi passwords and pin numbers, and template files for the web pages.
