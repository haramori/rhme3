# [Race of a Lifetime](https://rhme.riscure.com/3/challenge?id=16)

Miscellaneous - 100pts

## Challenge text

You are participating in a race around the world. The prize would be a personalized flag, together with a brand new car. Who wouldn't want that? You are given some locations during this race, and you need to get there as quick as possible. The race organisation is monitoring your movements using the GPS embedded in the car. However, your car is so old and could never win against those used by the opposition. Time to figure out another way to win this race.

## Solution

### Solution overview

This race involves a series of instructions that include your current location and instructions about where to go next.  The solution below relies on calculating the great circle distance between two points on a globe, best possible speed based on the current mode of transportation, and using these two things to fake the hourly updates based on a reasonable amount of progress.

### Analysis

Each prompt is an hourly check-in.  At each check-in, you have to provide your current lat-long GPS location in the form "0.000000, 0.000000".

You have a time limit for completing each segment between two waypoints, so you have to go as fast as you can.  However, at various times travel is accomplished by car, plane or boat; and you can't go faster than allowed by your current mode of transportation.  Therefore, careful control of speed is called for.

There are two possible approaches for calculating hourly check-ins:
* Calculate each hourly check-in by dividing a segment by the allowable number of hours for completing that segment.
* Calculate each hourly check-in by using the best allowable speed for each mode of transportation.  This may get you there a bit faster, so I chose this method for my solution.

Both options require that you know the overall distance of a segment.  The distance between two GPS coordinates can be calculated using the Haversine formula [https://en.wikipedia.org/wiki/Haversine_formula].

For the fastest speed approach, the following speeds seem to work best:

	car: 55 mph
	plane: 350 mph
	boat: 10 mph

To complete this challenge, you also have to manually look up the lat-long coordinates of various landmarks around the world.  Those are included in the script below.

### Script

The following Python3 script completes the race and gets the flag.

	import serial
	import re
	from math import asin,sqrt,sin,cos,radians,floor

	port = 'COM42'
	rate = 115200

	class Coord():
		def __init__(self,lat=0.0,long=0.0):
			self.lat = lat
			self.long = long

	s = serial.Serial(port,rate,timeout=3.0,write_timeout=3.0,inter_byte_timeout=3.0)

	def read_until(match=''):
		match = match.encode(encoding="ascii",errors="ignore")
		message = bytearray()
		while (True):
			data = s.read(1)
			if (len(data) == 0):
				break
			message += data
			if (len(match) > 0 and match in message):
				break
		message = message.decode(encoding='ascii',errors='ignore')
		print(message)
		return message

	def write(line):
		print(line)
		line = bytearray(line.encode(encoding="ascii",errors="ignore"))
		line.extend(b'\n')
		s.write(line)

	def haversine(theta):
		return sin(theta / 2.0)**2

	# Calculate distance between two points on the Earth using the Haversine formula.
	def distance(coord1,coord2):
		R = 3958.8      # Earth's mean radius in miles
		delta_lat = radians(coord2.lat - coord1.lat)
		delta_long = radians(coord2.long - coord1.long)
		lat1 = radians(coord1.lat)
		lat2 = radians(coord2.lat)
		return 2.0 * R * asin(sqrt(haversine(delta_lat) + cos(lat1) * cos(lat2) * haversine(delta_long)))

	# Longitudinal arithmetic should always be checked and corrected for crossing the 180 degree line.
	def normalize_hemisphere(long):
		if (long > 180.0):
			long -= 360.0
		if (long < -180.0):
			long += 360.0
		return long

	# Travel a great circle route between any two points at a given speed in MPH.
	def travel(pos_start,pos_end,speed):
		riscure_hq_welcome = re.compile(r'Welcome to Riscure!')
		paris_welcome = re.compile(r'Welcome to Paris airport')
		shanghai_welcome = re.compile(r'Welcome to Shanghai airport')
		ecs_welcome = re.compile(r'Well done!')
		sanfrancisco_welcome = re.compile(r'Welcome to San Francisco airport')
		riscure_na_welcome = re.compile(r'Unfortunately, the analyst')
		finish_line_welcome = re.compile(r'Congratulations!')

		# Calculate segments.
		miles = distance(pos_start,pos_end)
		hours = miles / speed
		delta = Coord()
		delta.lat = (pos_end.lat - pos_start.lat) / hours
		delta.long = normalize_hemisphere(pos_end.long - pos_start.long) / hours

		# Do all the whole hour segments.
		pos_current = Coord(pos_start.lat,pos_start.long)
		for pos_num in range(floor(hours)):
			pos_current.lat += delta.lat
			pos_current.long = normalize_hemisphere(pos_current.long + delta.long)
			write("{:.6f}, {:.6f}".format(pos_current.lat,pos_current.long))
			message = read_until('\n\n>')

			# It's possible to arrive before the last planned segment.
			if (riscure_hq_welcome.search(message)):
				return message
			if (paris_welcome.search(message)):
				return message
			if (shanghai_welcome.search(message)):
				return message
			if (ecs_welcome.search(message)):
				return message
			if (sanfrancisco_welcome.search(message)):
				return message
			if (riscure_na_welcome.search(message)):
				return message
			if (finish_line_welcome.search(message)):
				return message

		# For the last partial hour segment, just hit the target.
		if (pos_current.lat != pos_end.lat or pos_current.long != pos_end.long):
			write("{:.6f}, {:.6f}".format(pos_end.lat,pos_end.long))
			message = read_until('\n\n>')

		return message

	# Initialize some regexes
	latitude_pattern = re.compile(r'Latitude: (-?\d{1,2}\.\d{6})')
	longitude_pattern = re.compile(r'Longitude: (-?\d{1,3}\.\d{6})')
	location_pattern = re.compile(r'Location: (-?\d{1,2}\.\d{2}) (-?\d{1,3}\.\d{2})')

	# Initialize locations
	location_start = Coord()
	location_ecs = Coord()
	location_riscure_hq = Coord(51.994167,4.382306)
	location_paris_cdg = Coord(49.009556,2.542444)
	location_shanghai_pvg = Coord(31.152389, 121.801583)
	location_sanfrancisco_sfo = Coord(37.616389,-122.386111)
	location_riscure_na = Coord(37.793333,-122.404528)

	# Now that everything is set up, let's race!

	# Take care of preliminaries.
	read_until('Enter your name to start:')
	write('TomHaramori')

	# Get current location as starting point.
	message = read_until('\n\n>')
	result = latitude_pattern.search(message)
	location_start.lat = float(result.group(1))
	result = longitude_pattern.search(message)
	location_start.long = float(result.group(1))

	# Go to Riscure HQ for instructions.
	message = travel(location_start,location_riscure_hq,55)

	# Retrieve ECS location.
	result = location_pattern.search(message)
	location_ecs.lat = float(result.group(1))
	location_ecs.long = float(result.group(2))

	# Now go to each point in turn.
	# The speeds that work best are: 55 MPH driving, 350 MPH flying, 10 MPH sailing.
	message = travel(location_riscure_hq,location_paris_cdg,55)
	message = travel(location_paris_cdg,location_shanghai_pvg,350)
	message = travel(location_shanghai_pvg,location_ecs,10)
	message = travel(location_ecs,location_shanghai_pvg,10)
	message = travel(location_shanghai_pvg,location_sanfrancisco_sfo,350)
	message = travel(location_sanfrancisco_sfo,location_riscure_na,55)
	message = travel(location_riscure_na,location_sanfrancisco_sfo,55)
	message = travel(location_sanfrancisco_sfo,location_paris_cdg,350)
	message = travel(location_paris_cdg,location_riscure_hq,55)

### Race log

The following is my completed race log.  The race map and hourly check-ins have been removed for clarity.  Actions are described in brackets.

	Enter your name to start:
	TomHaramori
	X   -       Your location
	Latitude: 35.709999     Longitude: 32.790001
	F   -       Final destination
	Head to the Riscure office to get the directions.

	[Travel from starting point to Riscure HQ at 55 MPH]

	Welcome to Riscure!
	There is a new prototype Edison Model L sunk in the East China Sea while it was being transferred to a secret research facility.
	Reach the destination within the next 49 hours in order to be the first at the site.
	Location: 31.36 122.86
	You can take a flight if you need one at the airports of Paris (CDG), Shanghai (PVG) and San Francisco (SFO).

	[Retrieve ECS location]
	[Travel from Riscure HQ to Paris CDG at 55 MPH]

	Welcome to Paris airport
	Take off...

	[Travel from Paris CDG to Shanghai PVG at 350 MPH]

	Welcome to Shanghai airport
	Landed...

	[Travel from Shanghai PVG to ECS location at 10 MPH]

	Well done!
	Bring the ECU to Riscure North America where it can be analysed.
	Try to be there within 44 hours.
	Location: 550 Kearny St, San Francisco, CA, United States

	[Travel from ECS location to Shanghai PVG at 10 MPH]

	Welcome to Shanghai airport
	Take off...

	[Travel from Shanghai PVG to San Francisco SFO at 350 MPH]

	Welcome to San Francisco airport
	Landed...

	[Travel from San Francisco SFO to Riscure North America at 55 MPH]

	Unfortunately, the analyst who would study the ECU did not get the visa to work at the office.
	Hurry up and find him in the office in Delft in order to recover the flag.
	You have no more than 21 hours before it is too late.

	[Travel from Riscure North America to San Francisco SFO at 55 MPH]

	Welcome to San Francisco airport
	Take off...

	[Travel from San Francisco SFO to Paris CDG at 350 MPH]

	Welcome to Paris airport
	Landed...

	[Travel from Paris CDG to Riscure HQ at 55 MPH]

	Congratulations! You managed to deliver the ECU in time.
	Your flag is:
	147bcde28498b31d5c4d46970e86025e
	==END==
