# What is this?
This is where I'll document my research for controlling the SoundCore Liberty Air 2 Pro TWS earbuds without using the manufacturer's mobile app.

# First task
The first thing I'll do, and likely the easiest, is figure out how to switch between ANC modes. These earbuds come with Active Noise Cancellation and let you switch between 3 modes: Noise Cancellation, Transparency, and Normal (no ANC applied). The two ANC modes also come with their own submodes.
- Noise Cancellation
  - Transport
  - Indoor
  - Outdoor
  - Custom (lets you spin a slider to adjust some opaque parameter of the ANC algorithm, ignored for now)
- Transparency
  - Fully Transparent (completely passes through all sound)
  - Vocal Mode (only passes through vocals and attempts to block everything else)

In order to figure out what the the SoundCore application is sending to the earbuds over Bluetooth, I have to enable Bluetooth HCI Snoop Log in Android's Developer Options menu. This allows the device to capture and save Bluetooth packets to a log file. That log file can then be accessed by creating a Bug Report in Developer Options and then navigating to ``FS/data/misc/bluetooth/logs/btsnoop_hci.log`` inside the outputted zip file.

But before we open the log, we have to interact with the earbuds first so we have our data. I opened the SoundCore app and toggled between each of the ANC modes, thrn turned off Bluetooth and loaded that log file onto my computer.
We can now open log file Wireshark and analyze the data as seen below
# Analyzing the data in Wireshark
![image](https://github.com/CallMeTak/SoundCoreReversing/assets/104554457/416c1237-8f07-4b64-b1fb-22d2b3832ab2)


There's a lot of data here but the majority of it is actually irrelevant. Take a look here for an overview of Bluetooth architecture: https://www.mathworks.com/help/bluetooth/ug/bluetooth-protocol-stack.html. It's not entirely needed to understand everything on that page, but it helps a bit. The log starts with communication between "host" and "controller", we can ignore this as this is just low level communication between the Android device and the Bluetooth controller; it's info we don't need.

If we scroll down some, we can see the communication between our device and the SoundCore Liberty Air 2 Pro earbuds (assigned the codename Fantasia by the manufacturer, apparently)

![image](https://github.com/CallMeTak/SoundCoreReversing/assets/104554457/53c479d7-0038-4b2f-9c2b-d5f01453bf97)

This is what we're looking for. Now we can filter our Wireshark to display only data sent from my Android device to my earbuds using this display filter: ``bluetooth.dst == ac:12:2f:68:7e:15``
<img width="1280" alt="Wireshark_tn1ikr33o5" src="https://gist.github.com/assets/104554457/3d26b723-fc47-4840-8199-55295e9c06c1">
Okay, we're really narrowing things down now, but  we can go further. We're uninterested in the HFP, AVRCP, and AVDTP protocols. These are audio/playback related (refer back to the Bluetooth reference above). The remaining protocols left would be L2CAP, RFCOMM, and SPP. L2CAP and RFCOMM are transport protocols (with RFCOMM being used on top of L2CAP). We don't care about the transport protocol info, we want the actual data that was sent from the device (which uses the transport protocol), so we are left with SPP: Serial Port Profile. It provides a virtual serial port to send arbitrary data through.

![image](https://github.com/CallMeTak/SoundCoreReversing/assets/104554457/fa3d9f0e-bb6e-44c8-9c4a-a79781d88f19)

Now we have 14 entries in our Wireshark window. Much less than before, and we can see the raw data (in hexadecimal) that was sent. However, I only selected an ANC mode 7 times, so which entries are relevant? Well I can just start from the bottom (I'm fairly confident the last entry is relevant as I turned off bluetooth just as I finished toggling between modes). Then I can just look around for any discrepancies.
![image](https://github.com/CallMeTak/SoundCoreReversing/assets/104554457/2918b6dd-8512-441a-8be2-df8830e61a49)

We can see the data slightly changing at the bottom window when navigating between different entries. Only a few bytes change each time. However, after the 7th entry from the bottom, there's much less data. Coincidence? Nope, whatever those first few entries are, they're irrelevant to us, at least for now. The last 7 entries contain the data needed to switch between each ANC mode (one of them is a duplicate because the app automatically selected an option when I opened it).

We can test this by sending those exact bytes to the earbuds over Bluetooth serial using Python. If you look closely, Wireshark says channel 12 is used for all the data sent. A channel in Bluetooth is similar to a port. So if I send ``08ee00000001010a0002`` over channel 12 using SPP, then my earbuds will switch to normal mode.
