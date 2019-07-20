*Version 2.0 of ALL includes many updates to core functionality and adds machine learning capability to the V1 reference design. The README contains an in depth discussion about the core functionality, lists the major core updates from V1 to V2 and goes into detail regarding the machine learning aspects which should be used to mainly understand how it may be used in conjunction with Alexa. However, it is expected that some users may find that the README can serve as a very basic introduction to Machine Learning itself.*

# What is Alexa Lambda Linux (ALL) Project?
* ALL is an end-to-end HW/SW reference design created to enable quick prototyping and realization of the control and monitoring of things using Amazon’s Alexa.
* ALL includes Amazon’s Alexa Skills Kit and Lambda running in the cloud as well as a local server based on real-time Linux running on the Raspberry Pi.
* A voice-controlled home security system was built from ALL as an early proof of concept.
* ALL was introduced in late 2015 at https://github.com/goruck/all.

# What's new in ALL V2.0?
* Machine learning model generation based on data from security system sensor data.
* Machine learning prediction based on data from the security system sensor data.
* Server optionally sends JSON in addition to default text in response to commands.
* More robust error handling, in particular to the Raspberry PI real-time code.
* Expanded Alexa skills including handling zones that aren’t ready and arming for pets.
* Fixed misc bugs and addressed corner cases.

# ALL V2.0 System Block Diagram

![mall blockdia](https://cloud.githubusercontent.com/assets/12125472/18031080/8919d892-6c84-11e6-85c8-293c239eb325.png)

# ALL Overview
Using voice to interface with devices and services around the home enables a rich and intuitive experience as shown by Amazon's huge successes with FireTV and Echo. However, with the exception of a few 3rd party point solutions such as WeMo, the voice user interface is generally not available in the home. One reason for this is the difficulty and unfamiliarity of the technologies required to enable voice control. 

Alexa Lambda Linux (ALL) was developed to help accelerate this learning curve. ALL is a HW/SW reference design meant to enable quick prototyping and realization of the control and monitoring of things using Amazon's Alexa. A voice-controlled home security system was first built from the reference design as proof of concept which was later extended to support machine learning. 

The README below describes the system, the main components, its design, and implementation. It is expected that people will find it useful in creating voice user interfaces of their own using Alexa, Lambda, Linux, and the Raspberry Pi.

Feature | Benefit
------------ | -------------
Integrated with Lambda and ASK | Quick bring-up of new voice controls
Real-time Linux / user space app dev model | Low effort to control fast real-world events
Raspberry PI, open source, AWS services | Low cost and quick deployment
End to end SSL/TLS integration | Customer data security

# Table of Contents
1. [Requirements and System Architecture](https://github.com/goruck/all#requirements-and-system-architecture)
2. [Design and Implementation of the Main Components](https://github.com/goruck/all#design-and-implementation-of-the-components)
   1. [Alexa Intent Schema / Utterance database](https://github.com/goruck/all#alexa-intent-schema--utterance-database)
   2. [AWS Lambda function](https://github.com/goruck/all#aws-lambda-function)
   3. [Raspberry Pi Controller / Server](https://github.com/goruck/all#raspberry-pi-controller--server)
   4. [Keybus to GPIO Interface Unit](https://github.com/goruck/all#keybus-to-gpio-interface-unit)
3. [Machine Learning with ALL](https://github.com/goruck/all/blob/master/README.md#machine-learning-with-all)
   1. [Background Information](https://github.com/goruck/all/blob/master/README.md#background-information)
   2. [Problem Definition](https://github.com/goruck/all/blob/master/README.md#problem-definition)
   3. [Data Preparation](https://github.com/goruck/all/blob/master/README.md#data-preparation)
   4. [Data Analysis](https://github.com/goruck/all/blob/master/README.md#data-analysis)
   5. [Algorithm Evaluation](https://github.com/goruck/all/blob/master/README.md#algorithm-evaluation)
   6. [Implementation](https://github.com/goruck/all/blob/master/README.md#implementation)
   7. [Results](https://github.com/goruck/all/blob/master/README.md#results)
4. [Development and Test environment](https://github.com/goruck/all#development-and-test-environment)
5. [Overall Hardware Design and Considerations](https://github.com/goruck/all#overall-hardware-design-and-considerations)
6. [Bill of Materials and Service Cost Considerations](https://github.com/goruck/all#bill-of-materials-and-service-cost-considerations)
7. [Licensing](https://github.com/goruck/all/blob/master/README.md#licensing)
8. [Contact Information](https://github.com/goruck/all/blob/master/README.md#contact-information)
9. [Appendix](https://github.com/goruck/all#appendix)

# Requirements and System Architecture
Please see below the high-level requirements that the project had to meet.

* Low cost
* Extensible / reusable with low effort
* Secure
* Enable fast prototyping and development
* Include at least one real world application

Meeting these requirements would prove this project useful to a very wide variety of voice interface applications. In addition, implementing a non-trivial "real-world" application would show that the design is robust and capable. Hence the last requirement, which drove the implementation of a voice user interface on a standard home security system, the DSC Power832.

The system needs both cloud and (home-side) device components. Amazon's Alexa was selected as the cloud speech service and Amazon Web Services' Lambda was selected to handle the cloud side processing required to interface between Alexa and the home-side devices. Alexa is a good choice given that its already integrated with Lambda and has a variety of voice endpoints including Echo and FireTV and it costs nothing to develop voice applications via the [Alexa Skills Kit](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit). [Lambda](https://aws.amazon.com/lambda/) is ideal for quickly handling bursty processing loads, which is exactly what is needed to control things with voice. It also has a free tier under a certain amount of processing and above that its still very inexpensive. So, Alexa and Lambda are reasonable cloud choices given the requirements above.

The [Raspberry Pi 2](https://www.raspberrypi.org/blog/raspberry-pi-2-on-sale/) was designated as the platform to develop the home-side device components. The platform has a powerful CPU, plenty RAM, a wide variety of physical interfaces, has support for many OSs, and is inexpensive. It is possible to use an even more inexpensive platform like [Arduino](https://www.arduino.cc/) but given its lower capabilities vis-a-vis the Raspberry PI this would limit the types of home-side applications that could be developed. For example, use of GNU/Linux is desirable in this project for extensibility and rapid development reasons. Arduino isn't really capable of running Linux but the Pi is. The downside of using the Pi plus a high-level OS like vanilla Linux is that the system cannot process quickly changing events (i.e., in "real-time"). On the other hand, the Arduino running bare metal code is a very capable real-time machine. To be as extensible as possible, the system needs to support the development of real-time voice controlled applications and without complex device side architectures like using an Arduino to handle the fast events connected to a Pi to handle the complex events. Such an architecture would be inconsistent with the project requirements. Therefore [real-time Linux](https://rt.wiki.kernel.org/index.php/Main_Page) was chosen as the OS on the Pi. However, this comes with the downsides of using a non-standard kernel and real-time programming is less straightforward than standard application development in Linux userspace.

In the reference design, the Pi's GPIOs are the primary physical interface to the devices around the home that are enabled with a voice UI. This allows maximum interface flexibility, and with using real-time Linux, the GPIO interface runs fast. The reference design enables GPIO reads and writes with less than 60 us latency, vanilla Linux at best can do about 20 ms. Of course, all the other physical interfaces (SPI, I2C, etc.) on the Pi are accessible in the reference design though the normal Linux methods.

The requirements and the analysis above drove a system architecture with the following components.

* Alexa Skills developed using the Alexa Skills Kit
* An AWS Lambda function in Node.js to handle the intent triggers from Alexa and send it back responses from the home device 
* A home device built on Raspberry PI running real-time Linux with a server application developed in C running in userspace
* A hardware interface unit that handled the translation of the electrical signals between the Pi and the security system
* The DSC Power832 security system connected via its Keybus interface to the Pi's GPIOs via the interface unit.

Note that although the development of the architecture described above appears very waterfall-ish, the reality is that it took many iterations of architecture / design / test to arrive at the final system solution.

# Design and Implementation of the Components

## Alexa Skills
Creating Alexa Skills (which are essentially voice apps) is done by using the Alexa Skills Kit. An Amazon applications developer account is required to get access to the [Alexa Skills Kit (ASK)](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit), one can be created one at https://developer.amazon.com/appsandservices. There's a [getting started guide](https://developer.amazon.com/appsandservices/solutions/alexa/alexa-skills-kit/getting-started-guide) on the ASK site on how to create a new skill. The skill developed to control the alarm panel, named *panel*, used the example skill *color* as a starting point. Amazon makes the creation of a skill relatively easy but careful thinking through the voice interaction is required. The *panel* skill uses a mental model of attaching a voice command to every button on the alarm's keypad and an extra command to give the status of the system. The alarm system status is the state of the lights on the keypad (e.g., armed, bypass, etc). Four-digit code input is also supported, which is useful for a PIN. The Amazon skill development tool takes you through the following steps in creating the skill:

1. Skill Information - Invocation Name, Version, and Service Endpoint (the Lambda ARN in this project)
2. Interaction Model - Intent Schema, Custom Slot Types, and Sample Utterances
3. Test
4. Description
5. Publishing Information (the *panel* skill is not currently published)

The creation of the *panel* skill required inputting of an [Intent Schema](https://github.com/goruck/all/blob/master/ask/IntentSchema.txt) (which defines the voice interaction model), [sample utterances](https://github.com/goruck/all/blob/master/ask/SampleUtterances.txt), and [custom slot definitions](https://github.com/goruck/all/blob/master/ask/CustomSlotTypes.txt) which are essentially elaborations of the values of parameters used in the Intent Schema. The ASK service will generate a voice model from these inputs. 

The table below shows some of the mapping between example user requests and the sample utterance syntax used in the interaction model. 

Example User Request | Sample Utterance
---------------------|------------------
“Alexa, ask Panel for its status” | WhatsMyStatusIntent its status
“Alexa, ask Panel about its status” |WhatsMyStatusIntent its status
“Alexa, ask Panel for the status” | WhatsMyStatusIntent the status
“Alexa, ask Panel about the status” | WhatsMyStatusIntent the status
“Alexa, ask Panel for status” | WhatsMyStatusIntent status
“Alexa, ask Panel about status” | WhatsMyStatusIntent status
“Alexa, ask Panel for the status” | WhatsMyStatusIntent the status
“Alexa, ask Panel about the status” | WhatsMyStatusIntent the status
“Alexa, ask Panel what is the status” | WhatsMyStatusIntent what is the status
“Alexa, ask Panel what the status is” | WhatsMyStatusIntent what the status is
“Alexa, ask Panel what's the status” | WhatsMyStatusIntent what's the status
“Alexa, ask Panel to give me the status” | WhatsMyStatusIntent give me the status
“Alexa, ask Panel to give me its status” | WhatsMyStatusIntent give me its status
“Alexa, ask Panel to tell me the status” | WhatsMyStatusIntent tell me the status
“Alexa, ask Panel to tell me its status” | WhatsMyStatusIntent tell me its status
"Alexa, ask Panel for key pound" | MyNumIsIntent key {Keys}
"Alexa, ask Panel for button star" | MyNumIsIntent button {Keys}
"Alexa, ask Panel for key pad button 0" | MyNumIsIntent key pad button {Keys}
"Alexa, ask Panel for 8" | MyNumIsIntent {Keys}
"Alexa, ask Panel for pin 1234" | MyCodeIsIntent pin {Code}
"Alexa, ask Panel for code 5678" | MyCodeIsIntent code {Code}
"Alexa, ask Panel for 4545" | MyCodeIsIntent {Code}

The "LIST_OF_KEYS" slot enables the intent "MyNumIsIntent" to activate when Alexa hears the name of the buttons on the keypad defined by the slot. The intent "WhatsMyStatusIntent" activates when Alexa hears any of the status related utterances listed above. The intent "MyCodeIsIntent" activates when Alexa hears a 4-digit code. When the intents activate, they cause the service endpoint Lambda function to run and perform specific processing depending on the user intent.

The skill is invoked by saying to Alexa "ask panel" or "tell panel".

## AWS Lambda function
An Amazon Web Services account is needed to use Lambda, one can be created at at https://aws.amazon.com/. Like all AWS products, there is good [documentation](https://aws.amazon.com/lambda/) available on the AWS site that should be read to come up to speed, if not already familiar. Still it can be challenging to use the product, if unfamiliar with javascript, Node, and asynchronous programming which are all essential to using Lambda. Several books and articles including [*Sams Teach Yourself Node.js in 24 Hours*](http://smile.amazon.com/dp/0672335956) by Ornbo, [*JavaScript: The Good Parts*](http://smile.amazon.com/dp/0596517742) by Crockford, and [*Asynchronous programming and continuation-passing style in JavaScript*](http://www.2ality.com/2012/06/continuation-passing-style.html) by Rauschmayer can help to accelerate the learning curve. Lambda and Node.js is very well-suited to scale cloud side processing associated with applications like voice control, given that Lambda was designed to handle bursty applications and the non-blocking I/O benefits of Node.js.

The *color* skill and Lambda function example provided by ASK was used as a template to create the Lambda function for *panel*. Since the Lambda function needed to talk to a remote Raspberry Pi server, that functionality was added as well as modifying the logic and speech responses to suit the alarm application. The Lambda code uses the Node.js tls method to open, read, and write a TCP socket that connects to the remote Pi server which sends commands to and reads status from the panel. The tls method provides both authentication and encryption between the Lambda client and the Pi server. One of the biggest challenges in this development is the placement of the  callbacks that returned responses back to Alexa due to the async nature of Node.js. The rest of the Lambda function development was straightforward. 

The Lambda code is divided up into three parts, [all.js](https://github.com/goruck/all/blob/master/lambda/all.js) contains most of the functions required to process the speech intents from the Alexa service and interface to the local Pi in a client - server model. However, some Lambda code requires it to be released under the Amazon License which is in the [amzn](https://github.com/goruck/all/tree/master/lambda/amzn) directory. Lastly, the Lambda code common to all.js and the code in the amzn directory is in [shared.js](https://github.com/goruck/all/blob/master/lambda/shared.js).

## Raspberry Pi Controller / Server

### Real-time Linux
The Linux disto on the Pi is based on Debian 7.0 "Wheezy" with a fork of the [Raspberry Pi Linux kernel](https://github.com/raspberrypi/linux) patched with rt-patch and configured as a fully preemptible kernel. The patched and configured source code for the Raspberry Pi real-time kernel is available on [GitHub](https://github.com/emlid/linux-rt-rpi). Another how-to for building real-time Linux for the Raspberry Pi can be found on Frank Dürr's blog, [*"Networked and Mobile Systems"*](http://www.frank-durr.de/?p=203). The wiki [real-time Linux](https://rt.wiki.kernel.org/index.php/Main_Page) is an invaluable source of information on the rt-patch and how to write real-time code.

The patched version of the kernel includes the following changes.
* Replaced default kernel with PREEMPT_RT kernel 3.18.9-rt5-v7+
* Enabled non-FIQ USB driver (currently FIQ driver is not compatible with RT-patch on RPi2)
* Enabled camera, SPI, I2C and set its speed to 1MHz
* Disabled serial console
* Changed default WiFi network parameters

The latency of the patched kernel was measured using [Cyclictest](https://rt.wiki.kernel.org/index.php/Cyclictest) which showed a worse case of less than 60 us. So, that seemed to indicate that applications could respond to events at up to about 15 kHz. The typical latency of a vanilla 3.18 kernel on the Pi was measured at 20 ms.

The distribution of the latency on the patched kernel as measure by Cyclictest is shown below.

![pi latency](https://cloud.githubusercontent.com/assets/12125472/11770022/4d3df3fe-a1a9-11e5-91b7-281b4b064da6.gif)

### Controller / Server Application
The application code on the Pi emulates a DSC keypad controller running in the Linux system's userspace. The application code in its entirety can be found [here](https://github.com/goruck/all/blob/master/rpi/kprw-server.c), this section captures important design considerations that are not obvious from the code itself or the comments therein.

The DSC system is closed and proprietary and besides the installation and programming information in the DSC manuals and what hackers have managed to figured out and posted on the Internet and GitHub, there isn't much information available about the protocol it uses to communicate with its keypads. Also, much of of what is on the Internet relies on using an IP interface bridge like this [one](http://www.eyezon.com/?page_id=176) from Eyezon. A bridge like this instead of the code on the Raspberry Pi could have been used in this project but that would have made the reference design much less generic (and much less fun to develop). A lot of useful information on how to hack the DSC system can be found in the [Arduino-Keybus](https://github.com/emcniece/Arduino-Keybus) GitHub repo, but this project uses an Arduino which was ruled out (see above) and it was read-only. Still, it is very educational. Although these sources helped, a lot of reverse engineering was required to figure out the protocol used on the DSC Keybus. The reverse engineering mainly consisted of using an early version of the application code to examine the bits sent from the panel to the keypads and the keypads to the panel in response to keypad button presses and other events, like motion and door / window openings and closings. It was determined that the first two bytes of each message indicates its type, followed by a variable number of data bits including error protection. For example a message with its first two bytes equal to 0x05 is telling the keypad to illuminate specific status LEDs, such as BYPASS, READY, etc and a message type equal to 0xFF indicates keypad to panel data is being sent. See the *decode()* function in the application code for details regarding the various message types and their contents.

The application code consists of three main parts:

* A thread called *panel_io()* that handles the bit-level processing needed to assemble messages from the keybus serial data
* A thread called *msg_io()* that handles the message-level output processing
* A server called *panserv()* running in the main thread that communicates with an external client for commands and status

The application code directly accesses the GPIO's registers for the fastest possible reads and writes. The direct access code is based on [this](http://elinux.org/RPi_GPIO_Code_Samples#Direct_register_access) information from the Embedded Linux Wiki at elinux.org. The function *setup_io()* in the application code sets up a memory regions to access the GPIOs. The information in the Broadcom BCM2835 ARM Peripherals document is very useful to understand how to safely access the Pi's processor peripherals. It is [here](http://elinux.org/RPi_Documentation) on the Embedded Linux Wiki site.

Inter-thread communication is handled safely via read and write FIFOs without any synchronization (e.g., using a mutext). Its not desirable to use conventional thread synchronization methods since they would block the *panel_io()* thread. There is no way (to my knowledge) to tell the panel not to send data on the keybus, so if *panel_io()* was blocked by another thread accessing a shared FIFO, the application code would drop messages. A good solution to this problem was found in the article [Creating a Thread Safe Producer Consumer Queue in C++ Without Using Locks](http://blogs.msmvps.com/vandooren/2007/01/05/creating-a-thread-safe-producer-consumer-queue-in-c-without-using-locks/) by Vandooren, which is what was implemented with a few modifications.

Running code under real-time Linux requires a few special considerations. Again, the [real-time Linux](https://rt.wiki.kernel.org/index.php/Main_Page) wiki was used extensively to guide the efforts in this regard. The excellent book [The Linux Programming Interface](http://man7.org/tlpi/) by Kerrisk also was very helpful to understand more deeply how the Linux kernel schedules tasks. There are three things needed to be set by a task in order to provide deterministic real time behavior:

1. Setting a real time scheduling policy and priority by use of the *sched_setscheduler()* system call and its pthread cousins. Note that *panel_io()*, *message_io()*, and *main()* were all given the same scheduling policy (SCHED_FIFO) but *panel_io()* has a higher priority than *message_io()* and *main()*. This was done to ensure that the bit-level processing would have enough time to complete. 
2. Locking memory so that page faults caused by virtual memory will not undermine deterministic behavior. This is done with the *mlockall()* system call. 
3. Pre-faulting the stack, so that a future stack fault will not undermine deterministic behavior. This is done by simply accessing each element of the program's stack by use of the *memset()* system call. 

Extensive use of the *clock_nanosleep()* function from librt is used in the code. For example, its use is shown in the *panel_io()* thread where it awakens the thread every INTERVAL (10 us) to read and write to the GPIOs driving the keybus clock and data lines (via the interface unit). The thread contains logic to detect the start of word marker, to detect high and low clock periods, and to control precisely when the GPIOs are read and written. The below taken from *panel_io()* causes the thread to wait for valid data before reading the GPIO.

```c
t.tv_nsec += KSAMPLE_OFFSET;
tnorm(&t);
clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &t, NULL); // wait KSAMPLE_OFFSET for valid data
wordkr_temp = (GET_GPIO(PI_DATA_IN) == PI_DATA_HI) ? '0' : '1'; // invert
```

Even though the *panel_io()* is given higher priority than the other two threads, other critical threads in the Linux kernel must have higher still priority for the overall system to function. Therefore it is possible that a higher priority kernel task could preempt *panel_io()* and cause it to drop bits. The real-time Linux wiki suggests this solution:

*"In general, I/O is dangerous to keep in an RT code path. This is due to the nature of most filesystems and the fact that I/O devices will have to abide to the laws of physics (mechanical movement, voltage adjustments, <whatever an I/O device does to retrieve the magic bits from cold storage>). For this reason, if you have to access I/O in an RT-application, make sure to wrap it securely in a dedicated thread running on a disjoint CPU from the RT-application."* 

Based on this, the *panel_io()* thread was locked to its own CPU and the other threads are locked to the other three CPUs (its really great to have four CPUs at one's disposal). The application code sets the thread CPU affinity as shown below. 

```c
// CPU(s) for main, predict and message i/o threads
  CPU_ZERO(&cpuset_mio);
  CPU_SET(1, &cpuset_mio);
  CPU_SET(2, &cpuset_mio);
  CPU_SET(3, &cpuset_mio);
  CPU_ZERO(&cpuset_main);
  CPU_SET(1, &cpuset_main);
  CPU_SET(2, &cpuset_main);
  CPU_SET(3, &cpuset_main);
  // CPU(s) for panel i/o thread
  CPU_ZERO(&cpuset_pio);
  CPU_SET(0, &cpuset_pio);
```

With the threads pinning to the CPUs in this way and with the priority of *panel_io()* being greater than the other two threads in the application code (but not greater than the kernel's critical threads), the system exhibits robust real-time performance.

The server uses openSSL for authentication and encryption. For development and test purposes self-signed certificates are used. The cert and key pairs are generated with the following commands.

```bash
# Step 1. Generate ca private key
$ openssl genrsa -out /home/pi/certs/ca/ca.key 4096
# Step 2. Create self-signed ca cert, COMMON_NAME="My CA"
$ openssl req -new -x509 -days 365 -key /home/pi/certs/ca/ca.key -out /home/pi/certs/ca/ca.crt -sha256
# 
# Step 3. Create client private key
$ openssl genrsa -out /home/pi/certs/client/client.key 2048
# Step 4. Create client cert signing request, COMMON_NAME="Client 1"
$ openssl req -new -key /home/pi/certs/client/client.key -out /home/pi/certs/client/client.csr -sha256
# Step 5. Create signed client cert
$ openssl x509 -req -days 365 -in /home/pi/certs/client/client.csr -CA /home/pi/certs/ca/ca.crt -CAkey /home/pi/certs/ca/ca.key -set_serial 01 \
-out /home/pi/certs/client/client.crt -sha256
# 
# Step 6. Create server private key
$ openssl genrsa -out /home/pi/certs/server/server.key 2048
# Step 7. Create server cert signing request, COMMON_NAME="localhost"
$ openssl req -new -key /home/pi/certs/server/server.key -out /home/pi/certs/server/server.csr -sha256
# Step 8. Create signed server cert, where "key.ext" contains "subjectAltName = IP:xxx.xxx.xxx.xxx"
$ openssl x509 -req -days 365 -in /home/pi/certs/server/server.csr -CA /home/pi/certs/ca/ca.crt -CAkey /home/pi/certs/ca/ca.key -set_serial 02 \
-out /home/pi/certs/server/server.crt -sha256 -extfile /home/pi/certs/server/key.ext
# 
# Step 9. Copy client key pair and CA certificate to Lambda
$ cp /home/pi/certs/client/client.crt /home/pi/all/lambda
$ cp /home/pi/certs/client/client.key /home/pi/all/lambda
$ cp /home/pi/certs/ca/ca.crt /home/pi/all/lambda
```

Note: replace the X's in key.ext with the IP address or hostname of your server.

Production code should use certificates signed by a real CA. The server also uses TCP Wrapper daemon for secure access. TCP Wrapper uses the *hosts_ctl()* system call from libwrap library to limit client access via the rules defined in /etc/host.deny and /etc/host.allow files. The rules are set so that only clients with local IP addresses and AWS IP addresses are allowed access to the server.

The application code needs to be compiled with the relevant libraries and executed with su privileges, per the following.

```bash
$ gcc -Wall -o kprw-server kprw-server.c -lrt -lpthread -lwrap -lssl -lcrypto
$ sudo ./kprw-server portnum
```

Where *portnum* is the TCP port number for the server to use.

### Startup
The Raspberry Pi is used here as an embedded system so it needs to come up automatically after power on, including after a possible loss of power. The application code defines the GPIOs as follows:

```c
// GPIO pin mapping.
#define PI_CLOCK_IN	(13) // BRCM GPIO13 / PI J8 Pin 33
#define PI_DATA_IN	(5)  // BRCM GPIO05 / PI J8 Pin 29
#define PI_DATA_OUT	(16) // BRCM GPIO16 / PI J8 Pin 36
``` 

These GPIOs need to be in a safe state after power on and boot up. Per the Broadcom BCM2835 ARM Peripherals document, these GPIOs are configured as inputs at reset and the kernel doesn't change that during boot, so they won't cause any the keybus serial data line to be pulled down before the application code initializes them. The BRCM GPIO16 was selected for the active high signal that drives the keybus data line since that GPIO has the option of being driven low from an external pull down. Other GPIOs have optional pull-ups. Even though the pull resistors are disabled by default, this removes the possibility of the keybus data line becoming active if software somehow enabled the pull resistor by mistake. 

The code below was added to /etc/rc.local so that the application code would automatically run after powering on the Pi.

```bash
sudo /home/pi/all/rpi/kprw-server portnum > /dev/null 2>&1 &
```

Where *portnum* is the TCP port number for the server to use. 

At some point provisions will be added to automatically restart the application code in the event of a crash.

In testing it was observed that the Pi sometimes disconnected from the wireless network and did not automatically reconnect, especially when it was connected to a monitor via HDMI (it may have a desense problem). In the event the Pi's wifi does not automatically reconnect to the network after it loses the connection, a special script is run by cron. The script checks for network connectivity every 5 minutes and if the network is down, the wireless interface is automatically restarted. The script and cron entry are shown below are are based off of [this](http://alexba.in/blog/2015/01/14/automatically-reconnecting-wifi-on-a-raspberrypi/) technical blog. 

/usr/local/bin/wifi_rebooter.sh:
```bash
#!/bin/bash

# IP for the server you wish to ping (8.8.8.8 is a public Google DNS server)
SERVER=8.8.8.8

# Only send two pings, sending output to /dev/null
/bin/ping -c2 ${SERVER} > /dev/null

# If the return code from ping ($?) is not 0 (meaning there was an error)
if [ $? != 0 ]
then
    # Restart the wireless interface
    /sbin/ifdown --force wlan0
    /bin/sleep 5
    /sbin/ifup --force wlan0
fi
```

## Keybus to GPIO Interface Unit

### Keybus Electrical and Timing Characteristics
The keybus is a DSC proprietary serial bus that runs from the panel to the sensors and controller keypads in the house. This bus needs to understood from an electrical, timing, and protocol point of view (see above) before the Pi can be interfaced to the panel. There isn't a lot of information on the keybus other than what's in the DSC installation manual and miscellaneous info on the Internet posted by hackers. Essentially, this is a two wire bus with clock and data, plus ground and the supply from the panel. The supply is 12 V (nominal) and the clock and data transition between zero volts and the supply voltage level (i.e., 0 to 12 V nominally but could be a bit more or less (~ +/-10%) depending on the specific panel). The large voltage swings make sense given that it provides good noise immunity against interference picked up from long runs of the bus through the house. The DSC manual states that the bus supply can source a max of 550 mA. The sum of the current from all the devices presently on the bus was about 400 mA. This meant that the interface unit could draw no more than 150 mA.

An oscilloscope was used to reverse engineer the protocol. Some screen-shots and analysis from them are below.

![whole-word-ann](https://cloud.githubusercontent.com/assets/12125472/11801916/17eb6cf8-a29f-11e5-8d3c-5cd4d39ac7ed.gif)

This shows an entire word with the start of new word marker, which is the clock being held high for a relatively long time. The clock rate when active is 1 KHz. The messages between the panel and keypads are of variable length. The typical message from the panel to the keypads is 43 bits (43 ms in duration) and the start of the new word clock marker is about 15 ms. Thus, a typical message time including start of new marker is about 58 ms. Messages can be up to 62 bits long but they are not frequent. 

![data-clock-close](https://cloud.githubusercontent.com/assets/12125472/11801938/55c9817c-a29f-11e5-82a7-1eac18a5abc4.gif)

This shows a closer view of the clock and data, with data being sent between the panel and keypad and vice-versa. Data from the panel to the keypads transitions on the rising edge of the clock and becomes valid about 120 us later. Data from the keypads to the panel transitions at falling edge of the clock is likely registered in the panel on the next rising edge of the clock. The keypad data needs to be held for at least 25 us after the rising edge of the clock for it to be properly registered. Timers in the Pi's application code must adhere to these values for reliable data transfers to happen. The spikes at 2.5 and 4.5 ms are most likely due to the pull-up resistor on the data line in the panel causing it to be pulled high before the driver on the keypad has a chance to pull it low, which happens when the keypad sends a 0 bit to the panel.

From this data, it is clear that the panel's data line is bidirectional (with direction switched by clock level) and pulled up internally when configured as an input. The strength of the pull-up resister needs to be known in order to design the interface circuit correctly and via experimentation it was determined to be approximately 5 KΩ.

Note that a means to prevent or mitigate keybus data corruption needs to be implemented since ALL may try to write to the panel at the same time as a keypad. This is done in the rpi code by checking for very short words sent by the panel on the keybus. The panel outputs short words (usually 9-bits) which are ignored as invalid since these short words are usually associated with the simultaneous transfer of keypad data to the panel (recall from above the keybus is bidirectional and bits are transferred on the rising and falling edge of the clock). This keypad data is sent in response to a panel keypad query command that was sent previously. These commands include those messages starting with 0x051, 0x0593, 0x11 and possibly others.

### Raspberry Pi GPIO Electrical Characteristics
Data on the electrical characteristics of the Pi's GPIOs is also required to design the interface. There is not really the right level of detail available from the Raspberry Pi standard information. But there is excellent information on the Mosaic Industries' website [here](http://www.mosaic-industries.com/embedded-systems/microcontroller-projects/raspberry-pi/gpio-pin-electrical-specifications). This information is super helpful and detailed. Some of the key points from the article are below. 

* These are 3.3 volt logic pins. A voltage near 3.3 V is interpreted as a logic one while a voltage near zero volts is a logic zero. A GPIO pin should never be connected to a voltage source greater than 3.3V or less than 0V, as prompt damage to the chip may occur as the input pin substrate diodes  conduct. There may be times when you may need to connect them to out ­of ­range voltages – in those cases the input pin current must be limited by an external resistor to a value that prevents harm to the chip. I recommend that you never source or sink more than 0.5 mA into an input pin
* To prevent excessive power dissipation in the chip, you should not source/sink more current from the pin than its programmed limit. So, if you have set the current capability to 2 mA, do not draw more than 2 mA from the pin
* Never demand that any output pin source or sink more than 16 mA
* Current sourced by the outputs is drawn from the 3.3 V supply, which can supply only 50 mA maximum. Consequently, the maximum you can source from all the GPIO outputs simultaneously is less than 50 mA. You may be able to draw transient currents beyond that limit as they are drawn from the bypass capacitors on the 3.3 V rail, but don't push the envelope! There isn't a similar limitation on sink current. You can sink up to 16 mA each into any number of GPIO pins simultaneously. However, transient sink currents do make demands on the board's bypass capacitors, so you may get into trouble if all outputs switch their maximum current synchronously
* Do not drive capacitive loads. Do not place a capacitive load directly across the pin. Limit current into any capacitive load to a maximum transient current of 16 mA. For example, if you use a low pass filter on an output pin, you must provide a series resistance of at least 3.3V/16mA = 200 Ω

### Interface Circuit Design
Its desirable to electrically isolate the panel and Pi to prevent ground loops and optoisolators are an obvious choice. But they do consume a fair amount of current and the keybus data and clock lines cannot supply enough to directly drive them. Therefore, the keybus clock and data lines have buffers (CD4050) between them and the optoisolators. Out of an abundance of caution, buffers (74HC126) were also added between the clock and data optoisolators and the Pi's GPIOs, but they are not really needed since the Pi's GPIOs come out of reset being able to supply or sink 8 mA while the interface circuit only needs a 2 mA supply. The '126 buffers were removed in the final design. A high Current Transfer Ratio (CTR) optoisolator configured to drive a FET with logic-level gate voltage threshold was used to pull down the data line with sufficient strength. The other optoisolators are high CTR as well to minimize current draw from the panel. The interface schematic is shown below.

![interface](./kgiu/kicad_sch.jpg)

# Machine Learning with ALL
Adding machine learning (ML) capabilities to ALL was mainly motivated by a desire to use Alexa to help train a ML algorithm in an intuitive and low-friction manner by employing voice to ground truth (or 'tag') observations. Also since ALL uses a home security system as a proof-of-concept, ML seemed to be a natural fit given the amount of data captured by the security system's sensors. Door, window, and motion data is continually captured which reflects the movement of people into and around the house. Of course, normally this data is used for security monitoring purposes but here the goal of adding machine learning to ALL was to use this data to reliably predict patterns of people movement around the house that would trigger appropriate actions automatically. Using ML as described below can also be used to become familiar with the topic itself since the application is fairly straightforward and tractable.

Its important to note that simple rule based algorithms can be employed instead of ML to trigger actions based on the sensor data. However, this is really only feasible for the most basic patterns of movement around the house. 

## Background Information
### Training ML Models *(This section adapted from [here](http://docs.aws.amazon.com/machine-learning/latest/dg/training-ml-models.html))*
The process of training an ML model involves providing an ML algorithm (that is, the learning algorithm) with training data to learn from. The term ML model refers to the model artifact that is created by the training process.

The training data must contain the correct answer, which is known as a response. The learning algorithm finds patterns in the training data that map the input data predictors to the response (the answer that you want to predict), and it outputs an ML model that captures these patterns.

The ML model is used to get predictions on new data for which a response is not known.

### Steps to build and train a predictive ML model
1. Define the problem that predictive ML model will solve for.
2. Prepare the data that will be used to train the ML model.
3. Analyze the training data to help guide ML algo selection. 
4. Select a suitable algorithm for the ML model. 
5. Evaluate and optimize the ML model’s predictive accuracy.
6. Use the ML model to generate predictions.

### R
The popular open-source software R was selected as the main ML tool and was intended to be used for both modeling and embedded real-time purposes which helps to accelerate development by reusing code. The excellent book [An Introduction to Statistical Learning](http://smile.amazon.com/dp/B01IBM7790) was extensively used as both a learning guide to ML and to R.

The R software package was downloaded from [The R Project for Statistical Computing](https://www.r-project.org/) website and compiled on the Raspberry Pi since there are no recent pre-compiled packages available for Raspbian Wheezy. The following steps are required to install R on the Pi.

```bash
$ wget http://cran.rstudio.com/src/base/R-3/R-3.1.2.tar.gz
$ mkdir R_HOME
$ mv R-3.1.2.tar.gz R_HOME/
$ cd R_HOME/
$ tar zxvf R-3.1.2.tar.gz
$ cd R-3.1.2/
$ sudo apt-get install gfortran libreadline6-dev libx11-dev libxt-dev
$ ./configure
$ make
$ sudo make install
```

Note that this installs R version 3.1.2 which is the latest compatible version for Raspbian Wheezy. 

## Problem Definition
It is important to first understand and clearly describe the problem that is being solved including how training will be done. Here, the problem is to accurately predict a person's movement into and through the house using the security system's motion and door sensors (the window sensors are ignored for the present) and to take action on that prediction.

The proof of concept system is installed in a two story house with a detached playroom / garage in the rear, accessible via a short walk through the backyard. The relevant sensors are as follows.

Zone | Sensor
-----|-------
1  | Front Door
16 | Family Room Slider (rear house exit)
27 | Front Motion
28 | Hall Motion (first floor)
29 | Upstairs Motion (second floor hall)
30 | Playroom Motion (in detached unit behind house)
32 | Playroom Door

Thus, as a person traversed through the zones listed above the problem is to predict the path taken, speed of travel and direction of travel. An example path could be a person arriving home, entering through the front door, walking into the family room via the first floor hall, exiting the house through the rear family room slider door and into the playroom. Patterns can also be time of day dependent (example being a person arriving home around the same time in the evening and following the above path). These sensors and the time of day will be used to fit the model and as inputs to the model to make predictions. 

A related problem is how to train the model in the most intuitive and easiest manner possible for the user. A good solution is tagging the patterns as the occur with voice via Alexa. For example, if a person walked the above pattern at 7:00 PM, she would tell Alexa that ("Alexa, I'm home") which would trigger a sequence of events to capture an observation and update the ML model.

A typical use case is shown in the figure below, where a person arrives home at a specific time and turns on the TV. The person would tell Alexa this happened and after a fashion the TV would automatically be turned on (and perhaps Fire TV would stream a favorite show) when this pattern is detected.

![typ-usecase](https://cloud.githubusercontent.com/assets/12125472/18023973/5f5645ce-6bb5-11e6-8f44-a9162ffa73a8.png)

## Data Preparation
Considerable thought was put into understanding the sensor information required to develop the ML model. The native output from the sensors is binary - the sensor is either activated by movement or its not due to lack of movement. For security monitoring purposes this is normally sufficient but this binary information needs to be transformed into a continuous time series to be useful as inputs to the model described above. This transformation is accomplished by applying a time-stamp to every activation or deactivation of a sensor, which is called the 'absolute' time of an observation. The timestamped sensor data is not used directly to build / update the ML model or predict a pattern. Instead a version of the timestamped data is used which is the last activation / deactivation time of a sensor relative to the current observation time. The relative data is required to ensure that the training data used to build the model are consistent with new data used for prediction. An example transformation for a four sensor zone example is shown in the figure below.

![transform](https://cloud.githubusercontent.com/assets/12125472/17916505/c3ab50ea-6968-11e6-998e-8cf7a93dfb16.png)

Relative activation and deactivation times for each sensor along with the time and date, sample time, and the response ground truth for a specific pattern form a training predictor vector based on a single observation. A collection of these vectors form a dataset from which a model can be generated. Similarly, a test vector is formed from new sensor data (without the ground truth) that is used by the model to make predictions. 

An example training predictor vector is shown below, where 'clock' is the time and date of the observation, 'sample' is a time-stamp value in seconds (derived from Linux system time of the server), 'zaNN' is the sensor activation time in seconds relative to the time-stamp, 'zdNN' is the sensor deactivation time in seconds relative to the time-stamp, and 'patternNN' is the response ground truth for a specific pattern for the given observation.

| clock                    | sample  | za1    | za2      | za3      | za4      | za5      | za6      | za7      | za8      | za9      | za10     | za11     | za12     | za13     | za14     | za15     | za16 | za17     | za18     | za19     | za20     | za21     | za22     | za23     | za24     | za25  | za26  | za27 | za28 | za29 | za30 | za31     | za32 | zd1    | zd2      | zd3      | zd4      | zd5      | zd6      | zd7      | zd8      | zd9      | zd10     | zd11     | zd12     | zd13     | zd14     | zd15     | zd16 | zd17     | zd18     | zd19     | zd20     | zd21     | zd22     | zd23     | zd24     | zd25     | zd26     | zd27 | zd28 | zd29 | zd30 | zd31     | zd32 | pattern1 | pattern2 | pattern3 | pattern4 | pattern5 | pattern6 | pattern7 | pattern8 | pattern9 | pattern10 |
|--------------------------|---------|--------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|------|----------|----------|----------|----------|----------|----------|----------|----------|-------|-------|------|------|------|------|----------|------|--------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|------|----------|----------|----------|----------|----------|----------|----------|----------|----------|----------|------|------|------|------|----------|------|----------|----------|----------|----------|----------|----------|----------|----------|----------|-----------|
| 2016-07-30T21:54:37.950Z | 4134765 | -14172 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -51  | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -8066 | -6541 | -31  | -37  | -12  | -70  | -4134765 | -72  | -14164 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -47  | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -4134765 | -30  | -32  | -6   | -68  | -4134765 | -69  | NA       | NA       | NA       | NA       | NA       | NA       | NA       | TRUE     | NA       | NA        |

## Data Analysis
Once it was understood how to form a relevant sensor dataset for the model the next step was to discover and expose the structure in the dataset. Using the above transformations, a dataset of about 300 observations was collected and visualized in scatterplots using R's graphical capabilities. The R code that generated these plots can be found [here](https://github.com/goruck/all/blob/master/R/genScatterPlots.R) and the observation dataset can be found [here](https://github.com/goruck/all/blob/master/R/panelSimpledb.csv).

Eight unique patterns were used to generate the observations. Each pattern corresponds to a path, direction, and speed of a person walking though the house and the approximate hour of the day this pattern occurred. The training was done by walking the pattern a few times and then invoking an Alexa skill that aquired the sensor data. The patterns and the zones they activate are summarized in the table below.

Pattern Number | Path | z1 | z16 | z27 | z28 | z29 | z30 | z32 | clock
-------|---------|----|-----|-----|-----|-----|-----|-----|-------
0 | virtual pattern that indicates no path has been classified | N | N | N | N | N | N | N | N
1 | from front door into playroom in the evening | Y | Y | Y | Y | N | Y | Y | ~7:00PM
2 | from outside into playroom | N | N | N | N | N | Y | Y | N
3 | from family room into playroom | N | Y | N | N | N | Y | Y | N
4 | slow walk from kid's rooms to playroom | N | Y | Y | Y | Y | Y | Y | N
5 | fast walk from kid's rooms to playroom | N | Y | Y | Y | Y | Y | Y | N
6 | from master bedroom to kitchen in the early morning via living room | N | N | Y | N | Y | N | N | ~4:00AM
7 | reserved | N | N | N | N | N | N | N | N
8 | from playroom to kid's rooms | N | Y | Y | Y | Y | Y | Y | N
9 | reserved | N | N | N | N | N | N | N | N
10 | reserved | N | N | N | N | N | N | N | N

An example R generated scatterplot for the response 'pattern6' (walking from the upstairs to the kitchen via the front hall in the early morning) is shown in the figure below. Note that a temporal filter is always applied to the data that limits sensor times to 120 seconds which is about the maximum time it takes a person to walk a particular path through the house. Sensor data outside this window is not relevant. 

![pattern6scatterplot](https://cloud.githubusercontent.com/assets/12125472/17462392/1b7a6948-5c60-11e6-9c5b-f902c87d2bfa.png)

Another example scatterplot for 'pattern4' (slow walk from the kid's rooms to the playroom via the front hallway) is shown in the figure below.

![pattern4scatterplot](https://cloud.githubusercontent.com/assets/12125472/17462693/556ee17e-5c6a-11e6-806a-65026e8eb40a.png)

The axis for each plot is time in seconds from 0 to -120 with the exception of clock which is in hours from 0 to 23. The red dots are samples where the response is true, the black dots are where the response is false. From the scatterplots, the following can be determined.

* The data is not linearly separable between the true and false responses, so a non-linear model will be required.
* The true responses tend to be tightly clustered and surrounded by false responses, perhaps indicating a higher order non-linear model will be needed.
* The sensor predictors tend to be related by the fact they are offset in time from each other. The offset forms the basic pattern that needs to be classified.
* The data appears relatively unstructured in that there are not well defined regions of true and false responses (the 'decision boundary'). 
* Some patterns will have predictors that are not relevant since in general a path does not involve every sensor.
* An ensemble of two models may be required, one with clock as a predictor and one without it to handle observations taken around a particular time and those at random times.
* The data will need to be normalized given that the clock and sensor activation times are in different units. 

## Algorithm Evaluation
Now that the structure in the dataset is understood candidate model algorithms were evaluated using an R-based test harness.

Given the relatively unstructured decision boundary and non-linear nature of the dataset, the K-Nearest Neighbors (KNN), Random Forests and Support Vector Machine (SVM) algorithms are reasonable choices given that they are well-understood, non-parametric methods and have high flexibility (note that non-parametric models generally have the disadvantage of growing more complex as the number of observations increases). See [An Introduction to Statistical Learning](http://smile.amazon.com/dp/B01IBM7790) for details about the KNN, Random Forests, and SVM algorithms and their relative strengths and weaknesses. In the interest of time, only KNN and SVM approaches were evaluated and overall SVM seemed to offer better over performance (at the expense of more complexity) for the sensor dataset and so therefore was selected. The dataset provides a set of training observations that can be used to build a SVM-based multiple-class classifier. SVM uses the one-versus-one for multiple classification. More information about using SVMs in R can be found [here](https://cran.r-project.org/web/packages/e1071/vignettes/svmdoc.pdf).

An SVM with a radial kernel was selected given the non-linear class boundaries of the datasets. Cross-validation was used to select the best values of the parameters *gamma* and *cost* associated with the model. The R script to generate and test the SVM model can be found [here](https://github.com/goruck/all/blob/master/R/genSvmTest.R) with the dataset [here](https://github.com/goruck/all/blob/master/R/panelSimpledb.csv). The script generates confusion matrices for training and validation data.

The confusion matrix for the training data shown below. The numbers in the row and column headers represent the pattern number classified with 0 being the null case (i.e., no pattern was classified). Note that only patterns 1 through 8 were used and with clock as a factor in these examples. This data indicates that model is trained very well except for pattern 8. This is most likely because pattern does not yet have enough training samples in the current dataset. 

```text
> ### training error of optimal svm model with clock as a predictor
> svmPred = predict(svmOpt, trainData, probability = TRUE)
> table(predict = svmPred, truth = trainLabels)
       truth
predict   0   1   2   3   4   5   6   8
      0 125   0   0   0   0   0   0   5
      1   0   5   0   0   0   0   0   0
      2   0   0  18   0   0   0   0   0
      3   0   0   0  10   0   0   0   0
      4   0   0   0   0  24   0   0   0
      5   0   0   0   0   0   6   0   0
      6   0   0   0   0   0   0  12   0
      8   0   0   0   0   0   0   0   0
```
The confusion matrix for the test data shown below where a number of false negatives are shown particularly for patterns 2, 3, 5, and 8. This could indicate that the model is over-fitted or, as in the case of pattern 8, patterns have insufficient training data. Patterns 2 and 3 also use relatively few factors which likely increases the prediction error. The model will need to be refined as the dataset grows with additional training observations and its prediction performance is evaluated over time. 

```text
> ### test error of optimal svm model with clock as a predictor
> svmPred = predict(svmOpt, testData, probability = TRUE)
> table(predict = svmPred, truth = testLabels)
       truth
predict  0  1  2  3  4  5  6  8
      0 51  0  4  2  0  2  0  3
      1  0  2  0  0  0  0  0  0
      2  0  0  2  0  0  0  0  0
      3  1  0  0  6  0  0  0  0
      4  0  0  0  0 10  0  0  0
      5  0  0  0  0  0  5  0  0
      6  0  0  0  0  0  0  4  0
      8  0  0  0  0  0  0  0  0
```

R's summary function can be used to obtain some information about the SVM fit, the output is shown below. It can be seen that there is a large number of support vectors which indicates a complex decision boundary. This is also evident from the scatterplots above. 

```text
> summary(svmOpt)

Call:
svm(formula = y ~ ., data = trainData, kernel = "radial", gamma = svmTuneOut$best.model$gamma, 
    cost = svmTuneOut$best.model$cost, decision.values = FALSE, probability = TRUE, 
    scale = TRUE)


Parameters:
    SVM-Type: C-classification 
  SVM-Kernel: radial 
        cost: 100 
       gamma: 1 

Number of Support Vectors:  149

 ( 94 16 2 13 4 9 5 6 )


Number of Classes:  8 

Levels: 
 0 1 2 3 4 5 6 8
```

## Implementation
Now that the algorithm was selected, the next step was to implement the real-time prediction of patterns, develop an Alexa skill that performs voice tagging of training observations, and develop a method to periodically re-fit the SVM with new training data.

### Real-time Prediction and Action
A new thread called *predict()* was added to the [Raspberry Pi real-time software](https://github.com/goruck/all/blob/master/rpi/kprw-server.c) which runs periodically and sends sensor data to an R script via the *popen()* Linux system command to make a prediction. The prediction R script is found [here](https://github.com/goruck/all/blob/master/R/predsvm2.R). The thread reads the prediction from R, applies some confidence checking rules and does something if a true prediction is determined. Currently, various WeMo light switches in the house are controlled by the thread in response to predictions in a hardcoded manner but at some point a more flexible and extensible mapping of predictions to actions will be implemented (perhaps by use of another Alexa skill). The WeMo devices are controlled by a bash script called by a *system()* Linux system command in the thread. The WeMo bash script can be found [here](https://github.com/goruck/all/blob/master/wemo/wemo.sh). At some point the capability to automatically take action on a prediction will be added. This can be done by including the state of a WeMo switch as a factor in the model. Obviously, any device around the home that can be monitored and controlled through the LAN or Internet can be used as well.

Two models are used in the prediction R script, one that uses the clock as a prediction and one that does not. If both models predict the same pattern, the higher probability prediction is selected (in the case of both models making the same non-null prediction, a higher probability pattern from the model using clock as a predictor is likely a timed pattern that uses the clock). If one model has not identified any pattern and the other has, then the non-null case is selected.

The thread has the capability to log the output of the prediction R script which includes a timestamp, sensor states, and the prediction with probabilities. A sample from the log is below which correctly identifies the observation as being 'pattern 4' (slow walk from kid's rooms to playroom) with a higher probability that its a non-timed (i.e., not at any specific time of day) path. 

```text
********** New R Run (svm2) **********
timestamp: 2016-08-11T12:56:35Z 
 clock  za1 za16 za27 za28 za29 za30 za32
    12 -120  -56  -85  -74 -103   -1  -26
pred: 4 prob: .74 (w/o clk) | pred: 4 prob: .66 (w/clk)
********** End R Run (svm2) **********
```

There is also a rather naive approach implemented in the thread to predict the number of occupants in the house based on concurrent motion and door sensor activity. However it does provide reasonably accurate results based on subjective testing if there is enough motion in the house. The pattern predictions can probably be used to provide a more accurate occupancy estimate, that work is planned for a future rev of the code. 

The latency between sensor activity and prediction leading to the activation of a WeMo device is less than a second based on subjective testing. This is acceptable for now but as the dataset grows the model will become more complex (given its non-parametric nature) and so the latency will increase. There is also considerable latency added by using the R script via *popen()* since it adds its own overhead. Although using the R script accelerated overall development (since it was easy to reuse much of the R work from earlier stages in the project), at some point a C SVM library will probably be used in the thread instead of calling the R script in order to reduce latency. R itself uses a C/C++ library implementation of the popular *LIBSVM* package so it would be relatively straightforward to use it in the thread. More information on *LIBSVM* can be found [here](https://www.csie.ntu.edu.tw/~cjlin/libsvm/).

A simplified flowchart of the *predict()* thread operation is shown in the figure below.
![predict](https://cloud.githubusercontent.com/assets/12125472/18023995/d5dea556-6bb5-11e6-8df2-95c8a3641897.png)

### Alexa Skill Support for Voice Tagging and Prediction
Firstly, an [AWS SimpleDB](https://aws.amazon.com/simpledb/) database was created to store the observations that the Alexa voice tagging skill generates. The database was created by the code shown below. This assumes that the AWS SDK has been installed on the machine. See [AWS SDK for JavaScript in Node.js](https://aws.amazon.com/sdk-for-node-js/) for how to do this.

```javascript
var AWS = require('/usr/local/lib/node_modules/aws-sdk');

var fs = require('fs');

AWS.config.region = 'us-west-2';

var simpledb = new AWS.SimpleDB({apiVersion: '2009-04-15'});

// create domain
var params = {
  DomainName: 'panelSimpledb' // required
};
simpledb.createDomain(params, function(err, data) {
  if (err) console.log(err, err.stack); // an error occurred
  else     console.log(data);           // successful response
});
```

The existing Alexa skill's schema and utterance database was modified to include three new speech intents, *OccupancyIsIntent*, *PredIsIntent*, and *TrainIsIntent*. This code can be found [here](https://github.com/goruck/all/tree/master/ask). When the user's speech triggers it, *OccupancyIsIntent* causes Alexa to return the occupancy prediction generated by the *predict()* thread as explained above, *PredIsIntent* causes Alexa to return the last true prediction, and *TrainIsIntent* associates a particular observation with a pattern using voice (i.e., voice tagging) and then stores it in SimpleDB.

The existing [Lambda Node.js code](https://github.com/goruck/all/tree/master/lambda) that services the Alexa speech intents was modified to include three new functions to support the new intents. A function called [*trainInSession()*](https://github.com/goruck/all/blob/master/lambda/amzn/trainInSession.js) handles the *TrainIsIntent* intent, the function called *predInSession()* handles the *PredIsIntent* intent, and the function called *anyoneHomeInSession()*, handles the *OccupancyIsIntent* intent.

Currently, *TrainIsIntent* and *trainInSession()* uses 10 fixed patterns with the mapping to specific paths through the home as shown above. This forces the user to remember the mapping during training. Although acceptable for test purposes, a more flexible approach is required whereby the user is asked to provide the path name during training or is offered to select a path from a menu in case the path already exists. These enhancements will be added in a future revision of the skill.

The existing thread *msg_io()* in the [Raspberry Pi real-time software](https://github.com/goruck/all/blob/master/rpi/kprw-server.c) was modified to calculate the timestamped sensor data as described above and the Pi's server was modified to return that along with other information as JSON in response to a command from the Alexa skill running in AWS Lambda. Using JSON instead of raw text greatly simplifies the Node.js code running in Lambda.

A simplified flow diagram of voice tagging using the *TrainIsIntent* and *trainInSession()* functionality is shown in the figure below.
![train-flow-chart](https://cloud.githubusercontent.com/assets/12125472/17685887/e047bf92-631c-11e6-9c81-8d9390074ada.png)

A simplified flow diagram of getting a prediction using *predInSession()* and *PredIsIntent* functionality is shown in the figure below.
![predict-flow-chart](https://cloud.githubusercontent.com/assets/12125472/17686224/9cdcf13e-631f-11e6-89a2-4c164cdfedbd.png)

### Model Retraining
The SVM models needs to be periodically refitted as new observations are taken, ground truth tagged by Alexa, and stored in SimpleDB. This is accomplished by a new Node.js routine locally on the Raspberry Pi that is run as a cron job every day at midnight. When the routine, [*simpledb-read.js*](https://github.com/goruck/all/blob/master/nodejs/simpledb-read.js), is run it reads SimpleDB and compares the latest observation with what was previously read. If there are new observation(s), the data is copied from SimpleDB and appended to a local copy and then the SVM models are refitted with it. The SVM model generation is done using R by a script called [*genSvmModels2.R*](https://github.com/goruck/all/blob/master/R/genSvmModels2.R) which is called from the Node.js routine. From that point forward, the updated models are used for real-time prediction in the *predict()* thread.

## Results
The model was trained using Alexa to voice tag observations for the patterns listed above. About ten true plus a few explicit false observations were required to get good accuracy prediction. The model was typically trained in the following way.

1. Walk the path through the house in a specific direction, approximate pace, and time (if applicable).
2. At the end of the path, tag the path with a pattern number by saying "Alexa, ask panel pattern n true", where n is the pattern number. 
3. Walk the path the opposite direction at the same pace.
4. At the end of the path, tag the path with a pattern number by saying "Alexa, ask panel pattern n false", where n is the pattern number. 

These steps were repeated about ten times. The pattern prediction will become bidirectional if steps 3 and 4 are omitted and in that case observations for the other patterns will form the false data for the specific pattern. It was important to end each pattern at a sensor boundary since data is input to the predictor when motion triggers a change in sensor output.

The model gets refitted with new data everyday at midnight, this process takes less than 2 minutes on the Raspberry Pi with the current dataset consuming about 85% of one of the Pi's CPU cores. 

At this time only subjective real-time prediction performance is available. This indicates less than one second of latency between the end of a path and the activation of a WeMo light. The prediction accuracy is very high (close to 100%) when the only motion in the house is the person walking the pattern. The accuracy degrades when there is other motion in the house at the same time a pattern is being walked as this makes the sensor data noisy. The accuracy degrades more quickly for the patterns that depend on relatively few factors which suggests that one way to make the system more robust is to increase the number of motion sensors in the house. Also, the patterns that use relatively few factors (e.g., #2) it proved infeasible to make them unidirectional given the sparse dataset.

# Development and Test environment
The Raspberry Pi was used headless throughout development and so ssh was used from another Linux machine to edit and debug the application code with gedit. The application code was compiled directly on the Raspberry Pi, given its small size it compiled quickly which obviated the need to setup a cross-compiler on a more capable machine.

Clients written in Node.js and C were used to test the server before integration with Lambda. This code is is in the goruck/all/test repo. 

The ASK and Lambda test tools available in the SDK and Lambda status information from AWS Cloudwatch Logs were used extensively during development and test.

Schematic entry was done using gEDA's Schematic Editor.

# Bill of materials and service cost considerations
The hardware bill of materials total $85 and are as follows.

Item | Qty | Cost
-----|-----|-----
Raspberry Pi 2 Model B Quad-Core 900 MHz 1 GB RAM | 1 | $38
Various ICs and passives | 15 | $15
HighPi Raspberry Pi B+/2 Case | 1 | $14
32GB microSDHC | 1 | $12
JBtek® DIY Raspberry Pi 2 Model B Hat | 1 | $6
Totals | 19 | $85

The Lambda service cost will vary according to how often the Alexa skill is invoked and how much compute time the function consumes per month. A typical Lambda session for the alarm prototype lasted less than 450 ms with AWS typically billing me for 500 ms (they round up to the nearest 100 ms) and was allocated 128 MB memory. The free tier provides provides the first 400,000 GB-s processing and the first one million requests per month at no cost. At 128 MB the Alexa panel skill can use 3,200,000 seconds of free processing per month (6.2 million invocations of the skill) and the first 1 million invocations will be at **no cost**. AWS is truly built for scale. This assumes less than 1 GB per month external data is transfered to/from the Lambda function so that the Free tier is maintained.

See the [AWS website](https://aws.amazon.com/lambda/pricing/) for complete pricing information.

# Overall Hardware Design and Considerations
The interface circuit was first breadboarded and then moved to a prototyping board that fits in an extended Raspberry Pi enclosure. The prototyping board is from [JBtek®](http://smile.amazon.com/dp/B00WPFF9OA) and the enclosure is from [MODMYPI](http://www.modmypi.com/raspberry-pi/cases/highpi/highpi-raspberry-pi-b-plus2-case). The extended Pi enclosure also offers more height for components. However, there isn't a lot of area to fit the components; the interface circuit fit nicely but it would get increasingly difficult to fit additional components if required. There is height in the enclosure to stack another prototyping board but the component heights would have to be kept to a minimum. Photographs of the prototype are shown below.

![img_0619](https://cloud.githubusercontent.com/assets/12125472/12216367/b6de26a4-b693-11e5-8ca5-6b1e528edb9f.JPG)

![img_0620](https://cloud.githubusercontent.com/assets/12125472/12216370/bc4a32cc-b693-11e5-9e1c-312df70cbe27.JPG)

![img_0623](https://cloud.githubusercontent.com/assets/12125472/12216371/c0dbaa28-b693-11e5-8726-cf3ce1a0132b.JPG)

![img_0625](https://cloud.githubusercontent.com/assets/12125472/12216373/c5025aac-b693-11e5-8cd4-05d1c2cce285.JPG)

Note: most components were salvaged from other projects and so the selection and placement is not optimized for either cost or size.

The prototype board was later replaced by a printed circuit board that I designed. A rendering of the front side of the PCB is shown below.

![layout](./kgiu/kicad_pcb_front.jpg)

Photographs of the PCB assembled in the Pi enclosure are shown below.

TBA

# Licensing
The ALL project is almost completely licensed under the [Apache License 2.0](http://choosealicense.com/licenses/apache-2.0/) unless otherwise specified in a LICENSE.txt file in a source code directory. The only exception is the code in the file [trainInSession.js](https://github.com/goruck/all/blob/master/lambda/amzn/trainInSession.js) and related documentation in this README which are licensed under the [Amazon Software License](https://aws.amazon.com/asl/).

# Contact Information
For questions or comments about the ALL project please contact the author goruck (Lindo St. Angel) at \{lindostangel\} AT \{gmail\} DOT \{com\}.

# Appendix

## DSC Power832 Overview
Technical manuals and general overview information about the Power832 can be found on [DSC's](http://www.dsc.com/index.php?n=library&o=view_documents&id=1) website.

## Proof of Concept Output to Terminal
![screenshot from 2015-12-09 19 46 40](https://cloud.githubusercontent.com/assets/12125472/11706514/3819320c-9eae-11e5-95d0-af8bdee6ed24.png)
Note: connections to AWS Lambda triggered by Alexa

## Installed System Photo
![installed system](https://cloud.githubusercontent.com/assets/12125472/21471429/f078da24-ca67-11e6-81bc-dd41e5e70e2d.jpg)

## Early Proof of Concept Photo
![proto pic1](https://cloud.githubusercontent.com/assets/12125472/11706674/a8721fcc-9eaf-11e5-8707-f780ae4ef86a.png)
![proto pic2](https://cloud.githubusercontent.com/assets/12125472/11706679/b0fb6950-9eaf-11e5-95be-3d668412c5e2.png)

## Original schematics with design notes
![keybus-gpio-if1](https://cloud.githubusercontent.com/assets/12125472/11919445/f16955be-a707-11e5-8c5d-1de31212bf9a.png)
![keybus-gpio-if2](https://cloud.githubusercontent.com/assets/12125472/11919447/f453d4d4-a707-11e5-988a-f284c41c4085.png)

