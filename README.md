# How to connect the OpenBCI via wired USB and change the sample rate


## I. Introduction

This tutorial shows how a third party USB breakout chip can be configured with OpenBCI (OBCI) to allow for a wired USB connection. I've only verified this on Windows, but there's no reason why it shouldn't work on other OSes.  For extra related information beyond this tutorial, see [this thread](http://openbci.com/forum/index.php?p=/discussion/712/prospects-for-higher-sample-rates/p1) on the forum.

### 1. Why do this?

Reasons one might carry out the modifications herein include:
* Increasing the sample rate of OBCI - The default rates of 250 Hz (8-channel) and 125 Hz (16-channel) can be increased, but need a faster data connection than the 115200 baud throughput that bottlenecks the default setup.
* Causing all channels to sample *sychronously* in the 16 channel OBCI.  Currently, channels 1-8 and 9-16 are sampled in an alternating fashion, every 4 ms
* More reliable packet delivery for timing-sensitive applications

### 2. Caveats

The modified system won't run on the default version of OBCI's software ([the Processing code](https://github.com/OpenBCI/OpenBCI_Processing)).  The change in baud rate and possibly other changes break it.  This might be easy to fix, but I haven't investigated it.  It works just fine with the [OBCI Python code](https://github.com/OpenBCI/OpenBCI_Python). There's a [tutorial](http://docs.openbci.com/software/05-OpenBCI_Python) on how to get that code working and [another tutorial](http://docs.openbci.com/software/06-labstreaminglayer) on how to use it to get Lab Streaming Layer (LSL) output.

### 3. What do I need?

 1. OpenBCI V3 32-bit system.  Simple modifications should also work for the 8-bit board.
 2. UART-capable USB breakout board.  E.g. [Adafruit FT232H](https://www.adafruit.com/product/2264)
 3. USB isolator capable of "full speed" (12.0 Mbps).  E.g. [Adafruit 100mA USB Isolator](https://www.adafruit.com/product/2107)
 4. USB cables to connect the PC-->USB isolator --> USB breakout board. For the models listed, it's 1x Micro USB cable and 1x Mini USB Cable.
 6. Female/Male jumper wires.  Shorter is better, to avoid signal degradation.  E.g. [Adafruit's](https://www.adafruit.com/products/825)
 7. Female/Female jumper wires.  Used to do a basic I/O echo test on the USB breakout board.  E.g. [Adafruit's](https://www.adafruit.com/products/794)
 8. Access to a soldering iron, if using the FT232H USB breakout, in order to solder on the pins.  Your OBCI must also have its J3 and J4 pinblock headers soldered on, if you haven't already done so.

## II. Connecting and testing the USB breakout board

### 1. Test echo IO on the USB breakout board
Connect the USB breakout board to your computer with a USB cable.  If using the FT232H, follow [Adafruit's tutorial](https://learn.adafruit.com/adafruit-ft232h-breakout) up through the [serial UART page](https://learn.adafruit.com/adafruit-ft232h-breakout/serial-uart) to verify that the drivers are working on your system and that a basic IO test succeeeds.

### 2. Solder headers onto OBCI and connect the USB breakout board
Solder headers into the J3 and J4 pin blocks on OBCI (Pins GND, VOD, D12, D13, D11, D10, RST, D17).

Use 3 female/male jumper wires to connect pins GND, TX (D0 on the FT232H), RX (D1 on the FT232H), to GND, D12, and D11 respectively, on OBCI.

## III. Update the OBCI firmware

### 1. Prepare to update the OBCI firmware
We will be replacing OBCI's default firmware with a custom version.  To prepare for the update, follow the procedure [here](http://docs.openbci.com/tutorials/02-Upload_Code_to_OpenBCI_Board#upload-code-to-openbci-board-32bit-upload-how-to) up to the point of putting the board into bootloader mode.  We won't do that until we are ready to upload the new code.

### 2. Set TX/RX pins within the chipkit code
We need to change a few lines of code in the OBCI board definition file to enable pins D11 as TX and D12 as RX.  Find the file whose path should resemble: Documents/Arduino/hardware/chipkit-core/pic32/variants/openbci/Board_Defs.h

In this file make changes to the corresponding code so that it reads:

\#define       _SER1_TX_OUT         PPS_OUT_U2TX     // RPB14R = U2TX = 2  
\#define       _SER1_TX_PIN         11  //7                // RB14 CVREF/AN10/C3INB/RPB14/VBUSON/SCK1/CTED5/RB14  
\#define       _SER1_RX_IN          PPS_IN_U2RX      // U2RXR = RPA1 = 0  
\#define       _SER1_RX_PIN         12  //10               // RA1  PGEC3/VREF-/CVREF-/AN1/RPA1/CTED2/PMD6/RA1  


Save the changes.

### 3. Download the new firmware to the board
The custom firmware and associated libraries are in [this repository](https://github.com/WinslowStrong/OpenBCI_Wired_USB).  Place the folders that are inside OBCI_Wired_USB_Lib into /Arduino/libraries/ and place the OBCI_Wired_USB folder into /Arduino/ (this location is called Arduino's "sketchbook").  

### 4. Optional - change the baud and sample rates
The default OBCI firmware is configured to use a USB serial connection at 115200 baud and run at a 250 Hz sample rate for 8 channels, and 125 Hz alternating sample rate in "daisy mode" (i.e. with 16 channels).  Moreover, in daisy mode, the data from channels 1-8 and 9-16 aren't sampled at the same time, but are 4ms apart.  This results in an effective sampling rate of 125 Hz, and this is implemented via downsampled averaging from the 250 Hz internal sampling rate.  

The [custom firmware](https://github.com/WinslowStrong/OpenBCI_Wired_USB) for this tutorial sets a default baud rate of 230400 and a sample rate of 250 Hz for *both* the 8- and 16-channel OBCI systems.  It also corrects the asynchronous sampling in daisy mode, making it synchronous.  You can change the baud and sample rate parameters by opening the OBCI_USB_Serial.ino file in the Arduino editor. 
* baud - Change the line "Serial1.begin(230400)" by replacing 230400 with the baud rate you want.  I'm not sure that any rates other than factor-of-two multiples/divisors can work.
* Change the sample rate via the variable "Rate_Adj."  The possible values are :
Rate_adj: "0b000" (for 16 kHz), "0b001" (for 8 kHz), "0b010" (for 4 kHz), "0b011" (for 2 kHz), "0b100" (for 1 kHz), "0b101" (for 500 Hz), and "0b110" (for 250 Hz).

##### Limitations
* I haven't been able to get this setup to work above 460800 baud (communication becomes totally garbled at 921600 baud).  
* On my system, 460800 baud can handle a max sample rate of 1 kHz with 8 channels or 500 Hz at 16 channels, with the caveat that I get a few dropped packets here and there (fewer than 1 per minute).  The baud and max sample rate track each other down from here by factors of 2.
* When I unplug my laptop to run on its battery (noise reduction tactic), it seems like power or speed of its USB ports is effectively reduced, and I start to get *a lot* of dropped packets.  Maybe it's possible to change that in a power setting somewhere; I haven't tried yet.
* I tried a different USB breakout board, [Numato's FT2232H](http://numato.com/ft2232h-breakout-module/), in an attempt to overcome the occasional lost packets at 460800 baud and 500 Hz sample rate using the 16 channel OBCI.  The thought was that its larger send/receive buffers (4k vs 1k) might solve the problem.  It does reduce dropped packets, but it caused another issue on my system: approximately 1/3 of sent commands don't get through successfully.


### 5. Upload the new firmware
Resume the [uploading tutorial](http://docs.openbci.com/tutorials/02-Upload_Code_to_OpenBCI_Board#upload-code-to-openbci-board-32bit-upload-how-to/) by putting the OBCI into bootloader mode.  Follow the rest of the tutorial to upload the custom firmware that you downloaded above. In case it doesn't run for you for some reason or you later want to revert, you can follow the same procedure to upload the default firmware, available [here](https://github.com/OpenBCI/OpenBCI_32bit).  

### 6. Set the COM port settings
In Windows (presumably, you can make similar changes on Mac OS), go into device manager-->Ports, and find the COM port of the USB breakout board (NOT the USB dongle!).  Right click on it, select "Properties" and select the "Port Settings" tab.  Make these changes:
* "bits per second" is the baud rate. Set it to whatever you initialized Serial1 to in the new firmware (230400 if you made no change).
* Within "Advanced," set the port latency to 4 ms.  1 ms is too fast for the FT232H, and can lead to dropped packets.
On my system, these changes need to be "OKed" individually, or else the baud rate change doesn't save.  It might tell you that you have to restart your system, but you don't.  Just unplug the USB connection and plug it back in.


### 7. Test the system
As mentioned up front, the default OBCI Processing software won't run as-is with this custom OBCI firmware.  Instead you can run the [OBCI Python code](https://github.com/OpenBCI/OpenBCI_Python). There's a [tutorial](http://docs.openbci.com/software/05-OpenBCI_Python) on how to run it and [another](http://docs.openbci.com/software/06-labstreaminglayer) on how to use it to get Lab Streaming Layer (LSL) output.  When running the Python code, you need to supply the following extra arguments: 
* COM port of the *Serial USB* (not the OBCI dongle!): "-p=COM*#*" where "*#*" is the number of the COM port of the Serial USB breakout board.  
*  baud rate: "-b=*rate*"  where "*rate*" is the baud rate you are using (230400 unless you changed it).
* Iff you are using the 16 channel OBCI, then supply "-d" to force daisy mode.

### 8. Connect the USB isolator before connecting to a human
Use your mini USB cable to connect the USB isolator to your computer.  The USB breakout board then connects to the isolator.  This is a safety precaution so that a human connected to OBCI is electronically isolated from the mains power supply running through the computer and its USB port.  

## III. Concluding remarks

### 1. Firmware modifications

The modifications made to the firmware are:
* Removed the "buger protocol" code (it bracketed all commands sent with "+" signs, to reduce the chance a noise error had an effect).  Not necessary over Wired USB.
* In the OBCI board definition chipkit core file, enabled pins 11 (D11 on the board) and 12 (D12) as UART1 TX and RX, repsectively.
* Changed all "Serial0" --> "Serial1" in the .ino and the OpenBCI_32_Daisy library files.  Serial0 is still the RFduino, while Serial1 is UART1, using OBCI pins D11 and D12 to send and recieve data.
* Changed the way that daisy mode sends data: instead of sending ch. 1-8, waiting 1/SampleRate, then sending ch. 9-16, etc, it sends ch. 1-8 and immediatly sends ch. 9-16 in the next packet.  Also, it used to send the mean of the channel data from the current and previous sample (since samples were only being sent every other 1/SampleRate).  Now it just sends the current sample's data.


### 2. Thanks!

Huge thanks to William Croft (wjcroft on the forums) for figuring out how to make this work and troubleshooting nearly every stage of this process.  This is really more his work than mine.
Much credit also goes to Yannick (yj on the forums).  I built off of [his code](https://github.com/yj-xxxiii/OpenBCI_2kHz) for changing the sample rate.

### 3. Contribute!

If you do some modifications based on this work that you think could be useful for others, please share it.  Posting your code to github and/or updating this document would be very appreciated!