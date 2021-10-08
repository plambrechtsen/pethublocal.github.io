---
layout: page
title: Hub Certificate
permalink: /certificate/
---

**TLDR; Getting the _long_serialnumber_ isn't required but I highly recommend spending the time to get it.**

The physical Hub has a unique per-hub 16 byte password directly after the serial number in the flash of which is referred to as the _long_serialnumber_ in older versions of the flash. This _long_serialnumber_ is used to decrypt the PKCS12 certificate that is returned as part of the [credentials](/internals/#credentials) flow.

If you are feeling handy with a soldering iron or have an older Hub (H001-H009) and you have __NEVER__ connected it to the internet then you can get extract the _long_serialnumber_ as the password to top open the AWS MTLS Certificate so then you can do a few things:

- Just becase you want to.
- Man in the middle the traffic from the Hub to AWS if you are interested in the messages.
- Generate your own PCKS12 certificate.

During the process of re-flashing the firmware the _long_serialnumber_ is also sent so the process below re-applies the same firmware onto the hub using the "hold down the reset button during boot" process but since you are also connecting up a console you catch the _long_serialnumber_ during the firmware flashing process.

### Older Hubs

The older model hubs before the H010-xx revsion they have firmware that send out the _long_serialnumber_ when they first boot up which is handy. If you setup the environment you should see the hub connect to the web server to talk to the __/api/credentials__ endpoint and send a message like:

```
 serial_number=H008-0xxxxxxx&mac_address=0000xxxxxxxxxxxx&product_id=1&long_serial=xxxxxxxxxxxxxxxxxxxxx
```

So... that last value is what you need, it's the password for the AWS Certificates so write it down, then follow the standard instructions to update the hubs firmware buy pressing and holding the reset button when you plug the power in, then the hub will connect to the cloud, or the docker web instance will also cache the firmware locally as that is useful.

### Newer hubs or hubs with latest firmware

If you have a H010 revision hub or have already upgraded the firmware then you need to get a soldering iron out to get the password. First you need a TTL 3.3v Serial Adapter that you solder onto the Hub Mainboard. This will of course void any warranty and don't blame me if you brick your hub, that being said, it isn't that hard.

First I recommend you download the firmware, which is in the [config.ini](https://github.com/plambrechtsen/pethublocal/blob/main/docker/config.ini.sample):

```
#Support downloading the firmware for your hub when it first connects to get the credentials
DOWNLOADFIRMWARE=True
```

When the hub has connected to the docker stack for the first time it should automatically download the current firmware and put it into docker/output/web/H0xx-0xxxxxx-1.177-xx.bin where the last two xx are the 76 pages of the firmware, they are XOR encrypted and decrypted by the hub during the firmware update.

Now to connect up the console. For this you need to connect a TTL serial adapter to the correct header pins. This will void any warranty and you are responsible if you break it... With that out of the way.

I recommend soldering to the side connector to pins 3, 7 & 8 as per: 

https://github.com/plambrechtsen/pethublocal/tree/main/docs/Hub

Then connect the serial console at 57600 8/N/1

If you are using windows then I recommend Putty and you can set the line of scroll back to something larger than 2000 characters under Window -> Lines of Scrollback to something like 2000000
This is because a firmware update generates about 20k lines.

### BE AWARE YOU ARE JUST ABOUT TO FIRMWARE UPDATE YOUR HUB. DO NOT UNPLUG IT WHILE IT IS DOING THE UPDATE AS YOU COULD BRICK YOUR HUB JUST LEAVE IT TO COMPLETE!!!!

Then if you have it working on 57600/8/N/1 you should see the standard boot message when the hub normally. You should save the console log output to a file, as the firmware update generates about 20k lines, so if you are using Windows and Putty change the scroll back to 200000 or some large number.
Make sure to set your TTY to "raw" mode, otherwise the terminal driver might interfere by interpreting control characters. On Linux this can be achieved by running `stty -F /dev/ttyUSB0 raw 57600`.

Lastly unplug the power to the hub and then hold down the "reset" button underneath the hub with a pen or something then plug the power in and you will see the the firmware update process start. This doesn't actually reset the hub, it just causes it to download the latest firmware so you won't lose your cloud configuration

Which takes the output of the firmware update and prints the serial number.

Then there is this python script: [fwlogtopw.py](docker/source/fwlogtopw.py) which takes the output of the firmware update and extracts the long_serial for you.

Then you can use these commands to extract the PKCS12 from the credentials file and test to make sure the password is correct.

```
serialnumber=H0xx-0xxxxxx
macaddress=0000xxxxxxxxxxxx
certpassword=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
awk -F":" '{print $9}' docker/output/web/$serialnumber-$macaddress-2.43.original.bin | base64 -d > $serialnumber.p12
openssl pkcs12 -nodes -passin pass:$certpassword -in $serialnumber.p12
```

### [PolarProxy](PolarProxy)

After you have the password you can use [PolarProxy](PolarProxy) and view wireshark PCAPs of all traffic the hub talks to the cloud service.
