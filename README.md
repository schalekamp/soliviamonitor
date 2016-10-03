# soliviamonitor
Documentation and a python-script for monitoring the status of Delta Solivia RPI PV-inverters


Delta Solivia inverters use a simple serial protocol over RS485. The default baud rate seems to be 19200 bits per second, although this can be changed in the inverter settings. You may need to lower it if you have a really long cable, otherwise the default should be fine. The RS485 bus-ID can also be set for each inverter. The default value is 001, but each inverter should be given a unique ID if they're connected to the same bus. The bus needs to have a terminating resistor at each end, the inverters are fitted with a DIP-switch next to the RS485-connector to enable (1) or disable (0) bus termination.

Messages on the RS485-bus look like this:

 - 1 byte STX (0x02), start of message
 - 1 byte ENQ (0x05 for a request) or ACK (0x06 for a response)
 - 1 byte inverter ID on the RS485-bus (0x01 for the first inverter, 0x02 for the second, etc.)
 - 1 byte LEN, number of bytes to follow, excluding CRC and ETX
 - 2 bytes data address, e.g. 0x00 0x00 to request the inverter type, 0x60 0x01 to request a block of data, etc.
 - data bytes, only in case of a reply
 - 2 bytes CRC-16, over preceding bytes excluding STX, LSB first (little-endian)
 - 1 byte ETX (0x03), end of message

Unfortunately, exactly which variables are assigned to which data address seems to vary between inverter models. 
Moreover, a request for data on address 0x60 0x01 will return a block of data, which seems to contain the values of several important variables.

For Delta Solivia RPI Commercial European three-phase inverters, a request and reply pair may look roughly like this, as read on a Linux-machine with an RS485 USB-dongle:

```
# jpnevulator --ascii --timing-print --tty /dev/ttyUSB0 --read
2016-05-03 12:14:12.944206:
02 05 01 02 60 01 85 FC 03 02 06 01 9D 60 01 31	....`........`.1
35 33 46 41 30 45 30 30 30 30 32 34 31 30 35 31	53FA0E0000241051
35 30 35 30 30 33 34 36 33 32 32 33 30 39 30 31	5050034632230901
30 30 02 18 0F 2D 01 3C 0F 0C 02 24 0F 26 0F C1	00...-.<...$.&..
08 BA 14 9C 13 89 0F B7 13 88 0F CB 08 C2 14 B4	................
13 89 0F BC 13 88 0F C4 08 C7 14 D3 13 89 0F BD	................
13 88 1D 1D 03 52 18 C5 19 08 06 29 27 7C 3E 23	.....R.....)'|>#
0E 78 0E A6 00 00 9C 40 00 00 54 4F 00 00 28 22	.x.....@..TO..("
00 95 94 70 00 32 00 00 00 00 00 00 00 00 00 00	...p.2..........
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00	................
00 00 00 00 00 00 00 00 00 00 5B C6 03         	..........[..
```

The data breaks down as follows, for as far as I've been able to figure out:

Request:

 - STX, ENQ, ID: 02 05 01
 - LEN: 02
 - Address: 60 01
 - CRC: 85 FC
 - ETX: 03

Reply:

 - STX, ACK, ID: 02 06 01
 - LEN, e.g. 9D for 157 bytes
 - Address: 60 01
 - 11 bytes part number, e.g. 153FA0E0000 for an RPI M15A
 - 18 bytes serial, e.g. 241051505003463223
 - 6 bytes unknown, e.g. 090100
 - 2 bytes firmware revision power management, e.g. 02 18 for version 2.24
 - 2 bytes unknown, e.g. 0F 2D
 - 2 bytes firmware revision STS, e.g. 01 3C for version 1.60
 - 2 bytes unknown, e.g. 0F 0C
 - 2 bytes firmware revision display, e.g. 02 24 for version 2.36
 - 2 bytes unknown, e.g. 0F 26
 - 2 bytes AC inverter voltage between L1 and L2 (?), e.g. 0F C1 for 403.3 V
 - 2 bytes AC inverter current on L1, e.g. 08 BA for 22.34 A
 - 2 bytes AC inverter power on L1, e.g. 14 9C for 5276 W
 - 2 bytes AC inverter frequency on L1, e.g. 13 89 for 50.01 Hz
 - 2 bytes AC net (?) voltage between L1 and L2 (?), e.g. 0F B7 for 402.3 V
 - 2 bytes AC net (?) frequency on L1, e.g. 13 88 for 50.00 Hz
 - 2 bytes AC inverter voltage between L2 and L3 (?), e.g. 0F CB
 - 2 bytes AC inverter current on L2, e.g. 08 C2
 - 2 bytes AC inverter power on L2, e.g. 14 B4
 - 2 bytes AC inverter frequency on L2, e.g. 13 89
 - 2 bytes AC net (?) voltage between L2 and L3 (?), e.g. 0F BC
 - 2 bytes AC net (?) frequency on L2, e.g. 13 88
 - 2 bytes AC inverter voltage between L3 and L1 (?), e.g. 0F C4
 - 2 bytes AC inverter current on L3, e.g. 08 C7
 - 2 bytes AC inverter power on L3, e.g. 14 D3
 - 2 bytes AC inverter frequency on L3, e.g. 13 89
 - 2 bytes AC net (?) voltage between L3 and L1 (?), e.g. 0F BD
 - 2 bytes AC net (?) frequency on L3, e.g. 13 88
 - 2 bytes DC voltage on string 1, e.g. 1D 1D for 745.3 V
 - 2 bytes DC current on string 1, e.g. 03 52 for 08.50 A
 - 2 bytes DC power on string 1, e.g. 18 C5 for 6341 W
 - 2 bytes DC voltage on string 2, e.g. 19 08 for 640.8 V
 - 2 bytes DC current on string 2, e.g. 06 29 for 15.77 A
 - 2 bytes DC power on string 2, e.g. 27 7C for 10108 W
 - 2 bytes total output power (?), e.g. 3E 23 for 15907 W
 - 2 bytes unknown, e.g. 0E 78
 - 2 bytes unknown, e.g. 0E A6
 - 4 bytes energy output on current day, e.g. 00 00 9C 40 for 40 kWh (40000 Wh)
 - 4 bytes feed-in time on current day, e.g. 00 00 54 4F for 21583 seconds
 - 4 bytes total lifetime energy output, e.g. 00 00 28 22 for 10274 kWh
 - 4 bytes unknown, e.g. 00 95 94 70
 - 2 bytes heat sink temperature, e.g. 00 32 for 50 degrees Celcius
 - 36 zero-bytes
 - CRC: 5B C6
 - ETX: 03

The output is very similar for other European three-phase inverters in the Delta Solivia RPI series, such as an RPI M20A (part number 203FA0E0000), but will undoubtedly be different for other series. 
On a North-American split-phase Delta Solivia 5.0 NA G4 TR inverter for instance, the data block is somewhat shorter (150 bytes), and the communication looks something like this:

```
02 05 01 02 60 01 85 FC 03 02 06 01 96 60 01 45
4F 45 34 36 30 31 30 31 37 35 32 32 30 31 37 35 
31 30 31 32 33 34 30 30 30 35 30 34 31 32 33 34 
31 30 00 2D 2D 00 2D 2D 00 2D 2D 00 2D 2D 01 0A 
00 3E 00 00 00 2B 00 00 00 41 00 F2 06 3E 17 6F 
00 29 62 D4 17 6F 00 01 5E 88 17 6F 00 01 01 41 
00 E9 00 42 00 EC 00 F4 06 43 17 69 17 74 00 00 
AC 9E 00 00 10 9D 00 79 01 6E 0C 3C 00 00 06 D6 
00 00 00 08 00 00 00 00 00 00 00 00 00 00 00 01 
01 01 01 01 01 01 01 01 01 01 01 01 01 01 01 01 
01 01 01 FC C6 03 
```

Some fields seem to be similar (the part number is EOE46010175, the serial is 220175101234000504, etc.), but working out all the details would still require quite a bit of work, which I haven't attempted. This is left as an excercise to the reader. ;-)



