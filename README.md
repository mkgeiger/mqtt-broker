# MQTT broker with Mosquitto and Node-RED

## Overview
This is a instuction how to install and configure a Raspberry Pi as an MQTT broker. The free software packages Mosquitto and Node-RED will be used. This project is intended to install a local MQTT broker in a local network which can also be accessed from Internet over a secure TLS connection. The benefit is to have full control over the broker and its availability and reliability instead of using a public (Chinese) broker.

## Prerequisites
* a DynDns-account (e.g. https://freemyip.com/) needs to be created to be able to access your MQTT broker from Internet. Due to that your Internet provider assigns from time to time a different IP-address to your router, your MQTT broker can be accessed allways under the same domain name. If you intent to use your MQTT broker only in the internal network then you can skip this step. As a DDNS-client I could recommend the `ddclient` daemon which also can run on the Raspberry Pi.
* port forwarding needs to be configured in your router for port 8883. This the secured MQTT TLS port.
* a static IP-address needs to be assigned to the Raspberry Pi instead of DHCP. This is required by the router for port forwarding and by the MQTT clients.

## Setup
![Setup](/mqtt_setup.png)

## Installation
The descision was made on the broker software Mosquitto.
```
sudo apt update
sudo apt-get install mosquitto mosquitto-clients openssl
```
You’ll have now a very basic and working MQTT broker on port 1883 with no user authentication. 

## Mosquitto daemon
Several commands exist to control the daemon.
```
sudo systemctl stop mosquitto
sudo systemctl start mosquitto
sudo systemctl restart mosquitto
sudo systemctl status mosquitto
```

## Create a user 
I locked down my broker so that only allow those clients who know the password can publish to a topic. You can get super granular here where certain usernames can publish to certain topics only. For my sake I only have 1 user who can publish. Create a password for publishing with:
```
sudo mosquitto_passwd -c /etc/mosquitto/passfile <your username>
```

## The Mosquitto configuration file
The Mosquitto configuration file can be edited with following command. Make sure you have no empty spaces at the end of those lines or Mosquitto may give you an error.
```
sudo nano /etc/mosquitto/mosquitto.conf
```
The Mosquitto configuration file used for this setup you see in the following. Short explanation: anonymous users are not allowed to connect, default port 1883 (no TLS) for internal MQTT clients, port 8883 (TLS v1.2 secured) for external MQTT clients with the required crypto material, no client certificates required.
```
pid_file /var/run/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d

allow_anonymous false
password_file /etc/mosquitto/passfile

listener 1883

listener 8883
certfile /etc/mosquitto/certs/server.crt
cafile /etc/mosquitto/ca_certificates/ca.crt
keyfile /etc/mosquitto/certs/server.key
tls_version tlsv1.2
require_certificate false
```

## Creating TLS crypto material with openssl
![TLS handshaking](/tls_handshaking.png)

1. As we are our own CA (certificate authority) we create first our CA key pair. Make sure this CA key is stored secure that it cannot be stolen.
```
sudo openssl genrsa -out ca.key 2048
```
2. Now create a self-signed certificate for the CA using the CA key. Fill in every fields of the certificate request some information.
```
sudo openssl req -new -x509 -days 15000 -key ca.key -out ca.crt
```
3. Now we create a server key pair that will be used by the broker.
```
sudo openssl genrsa -out server.key 2048
```
4. Now we create a server certificate request. When filling out the form the `Common Name` is important and is usually the domain name of the server. In my case it would be `YOUR_DOMAIN.freemyip.com`. Please fill in the other fields with slightly different values as in step 2.
```
sudo openssl req -new -out server.csr -key server.key
```
5. Now we use the CA key to verify and sign the server certificate. This creates the server.crt file.
```
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 15000
```
Now copy server.crt and server.key to /etc/mosquitto/certs/ and ca.crt to /etc/mosquitto/ca_certificates/. Now changing the owner of these files needs to be done.
```
sudo chown -R mosquitto: /etc/mosquitto/certs/
sudo chown -R mosquitto: /etc/mosquitto/ca_certificates/
```

## Testing the broker
### Broker log file
You can see all traces of the MQTT broker in the mosquitto.log file. Best is to execute following command in a separate terminal.
```
sudo tail -f /var/log/mosquitto/mosquitto.log
```
### Local testing on the broker
The package `mosquitto-clients` is used for the following. Here is an example for publishing and subsribing to a topic.
```
mosquitto_pub -t "your topic" -h localhast -u "your username" -P "your password" -m "your value" -p 1883
mosquitto_sub -t "your topic" -h localhost -u "your username" -P "your password" -p 1883
```
### Testing on other MQTT clients
The package `mosquitto-clients` is used for the following and needs to be installed also on the connected MQTT clients. Here is an example for publishing and subsribing to a topic from connected MQTT clients.
```
mosquitto_pub -t "your topic" -h 192.168.1.10 -u "your username" -P "your password" -m "your value" -p 1883
mosquitto_sub -t "your topic" -h 192.168.1.10 -u "your username" -P "your password" -p 1883
mosquitto_pub -t "your topic" -h raspberrypi.local -u "your username" -P "your password" -m "your value" -p 1883
mosquitto_sub -t "your topic" -h raspberrypi.local -u "your username" -P "your password" -p 1883
```

## Node-RED
Node-RED is a programming tool for wiring together hardware devices and APIs. It provides a browser-based editor that makes it easy to wire together flows using the wide range of nodes in the palette that can be deployed to its runtime in a single-click.

### Installation
For installation of Node.js, npm and Node-RED on the Broker you need issue following command.
```
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```
After successful installation you need to start the Node-Red by executing following command. 
```
node-red-start
```
Node-RED as a service can be started with this command.
```
sudo systemctl enable nodered.service
```
Now you can directly access Node-RED over the IP or hostname of the Raspberry Pi and the port 1880. You can configure your flows now. Your local Node-RED server can act as any other MQTT client and communicates directly with the Mosquitto broker.
```
http://raspberrypi.local:1880
```

### Dashboard installation
Node-RED dashboard is a module that provides a set of nodes in Node-RED to quickly create a live data dashboard. To install the Node-RED Dashboard run the following commands.
```
node-red-stop
cd ~/.node-red
npm install node-red-dashboard
```
Then reboot your Raspberry Pi to ensure that all changes take effect on Node-RED software. To open the Node-RED UI, type your Raspberry Pi IP address or hostname in a web browser followed by :1880/ui as shown below.
```
http://raspberrypi.local:1880/ui
```

## DynDns client (ddclient) configuration example
### Installation
First you need to install ddclient.
```
sudo apt-get install ddclient
```

### Configuration
You need to edit following configuration file.
```
sudo nano /etc/ddclient.conf
```
Here is an example of the configuaration to be used with a freemyip-DynDns-account (of course there exist many others). Short explanation: your external WAN-IP address is requested by the server `checkip.feste-ip.net` (of course there exist many others). In this example the deamon would update your WAN-IP address every 600 seconds at the DynDns server.
```
custom=yes
protocol=dyndns2
use=web
server=freemyip.com
web=checkip.feste-ip.net
web-skip='Current IP Address: '
daemon=600
syslog=yes
pid=/var/run/ddclient.pid
login=YOUR_TOKEN
password='YOUR_TOKEN'
YOUR_DOMAIN.freemyip.com
```
