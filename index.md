---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

PetHubLocal. The local cloud replacement for your SurePetCare Connect series of IoT enabled Pet Devices.

### Supported SurePetCare Devices

I have added support for all currently available SurePetCare devices I have in my home, as I have them all.

- Internet Hub Connect
- Pet Door Connect
- Feeder Connect
- Cat Flap Connect
- Poseidon (Felaqua) Connect. I'm calling it Poseidon as that is a way cooler name than Felaqua.

## Quick Start

### Create config.ini 

You need to copy the `config.ini.sample` to `config.ini` and modify to get yourself going. The two entries you must modify are the **CLOUDUSERNAME & CLOUDPASSWORD**, technically it can build from current settings but it is just easier if you use the current settings with email address and password. This will make an API call to the SurePetCare Cloud Service to download your current configuration from `https://app.api.surehub.io/api/me/start` and write it to  `start-yymmdd-hhmmss.json` then convert it into `pethubconfig.json` which is the main settings file that holds the hub and device configuration. Previous this was a sqlite database but it was easier to just store it as a json file and update as needed. All config files are stored in the `frontend/data/config` folder.

You need to decide your deployment method, as there are a number which really depend on what works the best for you. There are required components, optional components and different ways to deploy Pet Hub Local. There are pro's and con's of each method described below:

You may also want to modify the WEB settings:

```
	WEBHOSTNAME=0.0.0.0
	WEBHTTPPORT=80
	WEBHTTPSPORT=443
```

If you want the webserver to listen on something other than all interfaces, change the hostname. You can't change the WEBHTTPSPORT as the Hub is hard-coded to talk HTTPS on port 443 and it can't be modified unless you modify the firmware and flash new firmware onto the hub. But if you don't plan to update firmware then the WEBHTTPPORT can be moved to something other than port 80. However, if you ever plan to do a firmware update or grab the _long_serial_ then you will need to have the HTTP port listening on port 80.

```
	MQTTIP=192.168.123.1
	MQTTPORT=1883
```

This is the MQTT Broker that the stack will connect to and if you wanted to keep things simple then a single Mosquitto MQTT Broker is all that is needed as Home Assistant also needs a MQTT Broker and set it to the local IP not localhost as the frontend needs your real IP.

### Local interactive deployment

There are two components you need to deploy
- Poison DNS / update local DNS to point `hub.api.surehub.io` to your local host. Depending on your router you may need to add `hub.api.surehub.io.` with a trailing `.`  then make sure the local DNS server is returning the local IP rather than pointing to the SurePetCare cloud service.
- The TLS endpoint using Mosquitto MQTT Broker. Take the [pethublocal mqtt files](https://github.com/plambrechtsen/pethublocal/tree/main/docker/mqtt/pethublocal) and put them into /etc/mosquitto/conf.d or wherever your configuration directory is and update to point to the correct `aws.pem` `hub.pem` and `hub.key`. Then restart Mosquitto and it should now be listening on port `8883`.
- Update config.ini as per the above and deploy [pethub frontend](https://github.com/plambrechtsen/pethublocal/tree/main/docker/frontend). As it's pure python you need to:

```
cd docker/frontend
pip3 install packages.txt
python3 pethubfrontend.py
```

You can use [tmux](https://github.com/tmux/tmux) if you want to run it as a background process as tmux is superior to [screen](https://www.gnu.org/software/screen/).

### Docker Deployment

Exactly the same first two steps are required to poison DNS, setup Mosquitto and also setup the config.ini settings with your required settings. Then you need to build docker.

_To build_
```
cd docker/frontend
docker build -t pethublocal .
```

_To run_
```
docker run -d --name="pethublocal" -v /data/pethublocal:/app/data -p 80:80 -p 443:443 pethublocal
```

Docker was the original deployment scheme with a MQTT broker, MQTT client to save all hub messages and the Web Server and Pet Hub Local. This has now been collapsed into only needing Pet Hub Local Frontend which runs two web servers, MQTT client and a configuration file monitor to write the 

### Docker Compose

This is a complete solution that will deploy the PetHubLocal Frontend, Mosquitto MQTT, MQTT Client to save every frame and Home Assistant.

All that is required here is to modify config.ini then as long as you have `docker` and `docker-compose` installed you just need to type:

```
docker-compose up
```

## Understanding PetHubLocal

Once it has started two folders are created under frontend/data and under the data folder two sub-folders are created:

- config
	> This is where the configuration files are created including:
	- `pethubconfig.json` - configuration file with hub, devices and pets. This is where the local environment configuration is stored
	- `H0xx-0xxxxxx-0000xxxxxxxxxxxx-2.43.bin` - the credentials file with serial number, hardware MAC address and firmware version which is downloaded from the cloud service calling the API `https://hub.api.surehub.io/api/credentials` each time the hub boots. This file is cached locally on the first boot so it doesn't need to be downloaded each boot. This is altered to point the MQTT endpoint also to `hub.api.surehub.io` as it is assumed that the webserver and MQTT endpoint are the same host. The unmodified version is saved as `.original.bin`.
	- `H0xx-0xxxxxx-1.177-00.bin` to `H0xx-0xxxxxx-1.177-76.bin` the firmware pages / records that are downloaded during the firmware update. Again this is cached on first boot so you have a copy of the firmware locally should you ever want to flash firmware as part of getting the _long_serial_ 

- log
	> Log files created

