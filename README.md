# Arduino_CANHacker
This repo for using your Arduino with MCP2515 CAN Interface and CANHacker software


## Getting Started
In this repository, I will talk about how to use any Arduino board with MCP2515 & MCP2551 board to make CANHacker sniffer.

### The Wiring
You need to wire the MCP2515 module to the arduino as follows:

<img src="https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/mcp2515_CAN_bus.jpg" height="600" width="600">

| MCP2515 Module  | Arduino Uno / Nano / Micro / Pro-Mini | Arduino Mega  |
| ------------- | ------------- | ------------- |
| VCC  | 5V  | 5V  |
| GND  | GND  | GND  |
| CS  | D10  | D10  |
| SO  | D12  | D50  |
| SI  | D11  | D51  |
| SCK  | D13  | D52  |
| INT  | D2  | D2  |

* You might need to put a jumper link on J1 in order to enable the 120 OHM termination resistor ( This is necessary for reducing the reflections on the CAN lines ).
* You could wire the CAN BUS lines throuth the blue screw terminal block or the J3 using any bin headers ( for laboratory experiments ).

As you can see in the picture, there is a 8MHz crystal oscillator onboard ( It might be 16MHz on some models ), we need to know its value in order to write it in the software.


### Installing the required Arduino libraries & Uploading the CAN Hacker Sketch
We need to install these libraries, open these links and you will find a green button called "Code" on the left, click on it and choose "Download ZIP".

* [arduino-mcp2515](https://github.com/autowp/arduino-mcp2515) library. You could also download it directly from here [arduino-mcp2515-master.zip](https://github.com/autowp/arduino-mcp2515/archive/refs/heads/master.zip)
* [arduino-canhacker](https://github.com/autowp/arduino-canhacker) library. You could also download it directly from here [arduino-canhacker-master.zip](https://github.com/autowp/arduino-canhacker/archive/master.zip)

After downloading both of these libraries, you will need to install both of them in the arduino. Just choose [ Sketch -> Include Library -> Add .ZIP Library... ] and select both of the downloaded files.

After installing these two libraries, you will find examples in [ File -> Examples -> arduino-canhacker-master -> softwareserial_debug ].

we need to add this line:

```C++
canHacker->setClock(MCP_8MHZ);
```
before that line below:
```C++
canHacker = new CanHacker(interfaceStream, debugStream, SPI_CS_PIN);
```

according to the crysyal oscillator that I have mensioned earlier, This line configures the MCP2515 to work with this clock in order to get the correct bit rate. you could use that line if you got 8MHz crystal oscillator, or you could simply change it to 16MHz.

Note: You will need to comment the line down below using the [ // ], This line is responsiple for enabling the Loopback function. Its simply a feature which allows the CAN controller to speak with itself without actually sending anything out on the bus. we could use this feature just for testing the setup. but when working with the CAN bus we will comment this line.

```C++
canHacker->enableLoopback(); // remove to disable loopback test mode
```

after the modifications, the code should look like this:

```C++
#include <can.h>
#include <mcp2515.h>

#include <CanHacker.h>
#include <CanHackerLineReader.h>
#include <lib.h>

#include <SPI.h>
#include <SoftwareSerial.h>

const int SPI_CS_PIN = 10;
const int INT_PIN = 2;

const int SS_RX_PIN = 3;
const int SS_TX_PIN = 4;

CanHackerLineReader *lineReader = NULL;
CanHacker *canHacker = NULL;

SoftwareSerial softwareSerial(SS_RX_PIN, SS_TX_PIN);

void handleError(const CanHacker::ERROR error);

void setup() {
    Serial.begin(115200);
    SPI.begin();
    softwareSerial.begin(115200);

    Stream *interfaceStream = &Serial;
    Stream *debugStream = &softwareSerial;
    
    
    canHacker = new CanHacker(interfaceStream, debugStream, SPI_CS_PIN);
    canHacker->setClock(MCP_8MHZ);    
    canHacker->enableLoopback(); // remove to disable loopback test mode
    lineReader = new CanHackerLineReader(canHacker);
    
    pinMode(INT_PIN, INPUT);
}

void loop() {
    CanHacker::ERROR error;
    
    if (digitalRead(INT_PIN) == LOW) {
        error = canHacker->processInterrupt();
        handleError(error);
    }
    
    // uncomment that lines for Leonardo, Pro Micro or Esplora
    // error = lineReader->process();
    // handleError(error);
}

// serialEvent handler not supported by Leonardo, Pro Micro and Esplora
void serialEvent() {
    CanHacker::ERROR error = lineReader->process();
    handleError(error);
}

void handleError(const CanHacker::ERROR error) {

    switch (error) {
        case CanHacker::ERROR_OK:
        case CanHacker::ERROR_UNKNOWN_COMMAND:
        case CanHacker::ERROR_NOT_CONNECTED:
        case CanHacker::ERROR_MCP2515_ERRIF:
        case CanHacker::ERROR_INVALID_COMMAND:
            return;

        default:
            break;
    }
  
    softwareSerial.print("Failure (code ");
    softwareSerial.print((int)error);
    softwareSerial.println(")");
  
    while (1) {
        delay(2000);
    } ;
}
```

Now we are ready to compile our Arduino sketch and flashing the Arduino board.

https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/CAN_Hacker_Software.jpg

### Using the Arduino with CAN Hacker Software
You could download the CANHacker software from [Here](https://www.mictronics.de/posts/USB-CAN-Bus/) scroll down until you find [CANHacker v2.00.01] and click this link, you also will find useful documentation for this software.

After installing the software and running it, you will see the application like this photo.

<img src="https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/CAN_Hacker_Software.jpg" height="771" width="819">

Just click on the [Settings] menu, you will get a small configuration window like this

<img src="https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/CAN_Hacker_Communication_Settings.jpg" height="216" width="276">

* Choose the Arduino's COM port.
* Choose the Baudrate -> 115200 bit/s/.
* Choose the CAN Baudrate for the CAN Bus and press OK.
* Click on Connect on the main screen.

<img src="https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/SendingMessageLoopback.jpg" height="321" width="819">

By enabling the Loopback function in the Arduino code. If you click on the [Single Shot] button, a CAN Message with ID=000, DLC=8, Data = { 00, 00, 00, 00, 00, 00, 00, 00 } will be sent back to the software and you should find it in the receive section like the picture down below:

<img src="https://github.com/rxtxinv/Arduino_CANHacker/blob/main/Pictures/ReceivingMessageLoopback.jpg" height="337" width="819">


Now after verifying that everything is working, comment the line down below to get the setup working with any CAN Bus.

```C++
canHacker->setClock(MCP_8MHZ);
```

Here is the Full code [softwareserial_debug.ino](https://github.com/rxtxinv/Arduino_CANHacker/blob/main/softwareserial_debug/softwareserial_debug.ino)

Happy Reverse Engineering :upside_down_face::joy:.

### Useful Links
* [arduino-canhacker](https://github.com/autowp/arduino-canhacker).
* [arduino-mcp2515](https://github.com/autowp/arduino-mcp2515).
* [canhack Forum](https://www.canhack.de/).

### Support Me
If you seen my work and it helped you, please support me on LinkedIn by endorsing my skills. It will be appreciated :grinning:.
<p>
  <a href="https://www.linkedin.com/in/omar-mekkawy/" rel="nofollow noreferrer">
    <img src="https://img.shields.io/badge/linkedin%20-%230077B5.svg?&style=for-the-badge&logo=linkedin&logoColor=white"  height="40" width="90" alt="linkedin">
  </a> &nbsp;
</p>
