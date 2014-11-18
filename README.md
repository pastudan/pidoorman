## PiDoorMan

Developing a simple access control system using the Raspberry Pi

* * *

  

### Introduction

The premise of proximity card access control is actually very simple:

- present a card to a reader
- door unlocks
The actual mechanics are slightly more complex:

- proximity card reader sends out radio waves
- induces electric current in wire coils contained inside the card
- powers a microchip transmitter inside the card
- reader picks up data
- data transmitted down 2 wires as a series of electrical pulses
- interpreted by computer
- compared with allowed cards
- if recognised a relay switches to unlock the door
Passive RFID cards work in this way, whereas active RFID cards work slightly differently, in that they actually send out the number using battery power.

The [Raspberry Pi](http://raspberrypi.org/) is a perfect candidate for the interface between the hardware of a door and software to control it. GPIO (general purpose input/output) pins which are fully addressable from inside the linux operating system through various programming languages. They are able to detect the presence of an open or closed circuit (as an input) and create or break a connetion to ground (as an output).

### Step 1 - Reading Wiegand

As an industry standard for all access control the [Wiegand protocol](http://en.wikipedia.org/wiki/Wiegand_interface) is universally used to transmit data from the card reader to a decoding device. Luckily it is very easy to get your Raspberry Pi to read Wiegand data from a reader.

### Part A: the software

I'm going to assume you've already got your Raspberry Pi setup running one of the standard liunx distros. I am currently using the 2012-07-15-wheezy-raspbian release. To talk to the GPIO I opted to use the c programming language by way of the [wiringPi](https://projects.drogon.net/raspberry-pi/wiringpi/) library, written by a chap called Gordon.

Installing wiringpi onto your Raspberry Pi is nice and easy. Connect your Pi to the internet, and then at the command line (I use ssh to talk to my Pi from my desktop PC - Google how to do this if you're not sure):

- Download to the temporary directory:  

cd /tmp

wget http://project-downloads.drogon.net/files/wiringPi.tgz
- Extract:  

tar xfz wiringPi.tgz
- Install:  

cd wiringPi/wiringPi

make

sudo make install

cd ../gpio

make

sudo make install

### Part B: the hardware

**IMPORTANT DISCLAIMER: WIEGAND READERS USE 5V DC, WHILE THE RASPBERRY PI IS A 3.3V DEVICE. IF YOU CHOOSE TO FOLLOW THESE INSTRUCTIONS, I CAN TAKE NO RESPONSIBILITY IF YOU BLOW UP YOUR RASPBERRY PI.**

With this in mind, please look at the protection circuit very carefully:   
![Protection circuit](http://tonigor.com/pidoorman/images/ProtectionCircuit.jpg)  
The protection needs to be applied to both the Data0 and Data1 connections. The GPIO pins I use in my example code are pins 0 and 1. Your Wiegand reader will obviously require 12V, for which I use an external power supply. The RFID reader I used is the [Controlsoft Oval proximity reader (AC-1200)](http://controlsoft.com/datasheets_readers.html).

### Part C: the C code

  
To download and compile the program, navigate into the folder you want the program installed, and then:

wget http://pidoorman.com/wiegand.c

cc -o wiegand wiegand.c -L/usr/local/lib -lwiringPi -lpthreadYou will then be able to run the program:

sudo /wiegandWhen you show your card to the reader, the program will output the Site code (if applicable) and ID card number on the screen. An unrecognised format will return the "Unknown Format" message.

### Step 2: Switching Outputs

Steps 2 and 3 are linked - in order to be able to switch outputs, the card number has to be recognised against some pre-defined list; i.e. a database. Setting up GPIO for red (access denied) and green (access allowed) LEDs within the C program is straightforward:

system ("gpio export 17 out");

system ("gpio export 21 out");

Then, under the correct conditions the relevant output can be switched:

digitalWrite (ALLOWED_PIN, 1);

digitalWrite (DENIED_PIN, 1);

### Step 3: Linking to a database

Firstly, there was the toss-up between MySQL and SQLite. As I am not worried about overhead at this stage, and I have plans to make PiDoorMan be able to support multiple doors from a central database, MySQL us the obvious choice:

sudo apt-get install mysql-server

Then, to be able to perform SQL commands on a MySQL database:

sudo apt-get install libmysqlclient-dev

The C program now has to include the MySQL header file that is installed as part of the libraries:

#include &lt;mysql.h&gt;

To connect to a MySQL database with C, this is the syntax:

MYSQL *con = mysql_init(NULL);  

    if (con == NULL)
    {
        fprintf(stderr, "%s\n", mysql_error(con));
	    return;
    }  

    if (mysql_real_connect(con, "host", "user", "password",
        "database", 0, NULL, 0) == NULL)
    {
        fprintf(stderr, "%s\n", mysql_error(con));
        mysql_close(con);
	    return;
    }

An example of running a query:

char sqlselect[80];  

   
    snprintf(sqlselect, sizeof sqlselect, "%s%d%s%d%s", "select id from pdm_cards where faccode='",  

	facilityCode, "' and cardno='", cardCode, "'");
      

    if (mysql_query(con, sqlselect))
    {
        fprintf(stderr, "%s\n", mysql_error(con));
        mysql_close(con);
    }

I modified the main Wiegand test program to query a simple database of card numbers to check if a card number exists, and depending on the result, switch the relevant output as desribed above in step 3.

My modified C program is now called pdm.c, and to compile with the MySQL libraries, the following command is required:

gcc pdm.c -o pdm -std=c99 -L/usr/local/lib -lwiringPi -lpthread `mysql_config --cflags --libs` && ./pdm

### Step 4: User (web) interface

To begin with, PiDoorMan will support only a single door and reader. PHP pages served locally from the Raspberry Pi with Apache server. PHP pages can read from and write to the database, so access events can be read, and access card numbers can be added, removed, and assigned to people.

Install Apache and PHP:

sudo apt-get install apache2

sudo apt-get install php5

### Step 5: Getting the C program to run on boot

The best guide I found on this was [here.](http://www.stuffaboutcode.com/2012/06/raspberry-pi-run-program-at-start-up.html)

In the script file, I had to run the pdm program as su (super user), so that it had the relevant permissions to control the I/O:

su pi -c '/home/pi/CPrograms/pdm &gt;/dev/null 2&gt;&1 &'

### Step 6: Extras

I am going to extend the project when I get time. Some upcoming features:

- Expiry dates on cards
- Restricted access to time schedules
- Multiple doors - multiple Raspberry Pis controlling multiple doors, all linked back to a central database.

* * *

  
© 2013 Cyte Design
