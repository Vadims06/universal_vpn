# About
This container allows launching OpenVPN Server ready for connecting OVPN windows client using login-password credentials.

Be noted, that Windows requires TLS authentication even in using login-password credentials. So you need to launch separate container for TLS authentication turned on.
## Preparation
* Download OVPN Image

```docker pull vadims06/ovpn-psk-based-for-win-clients:latest```
* Pick a name for the volume container. We use "win-volume" in our example. There are configuration files will be stored in such volume.

```docker volume create win-volume```

## Initialize the container for config generation.
Run the container with mentioned options in order to generate a configuration file for OVPN. The config will be saved in the `win-volume`
#### Config flags
```
-r This will allow Mikrotik's private subnet to access the VPN. By default, Mikrotik has 192.168.88.0/24 network for LAN.
-p Routes behind OVPN server container. Push routes to the client to allow them to reach other private subnets behind the server. Network format: x.x.x.x 255.255.255.0. Do not use short version of network notation like x.x.x.x/XX
-s subnet for OVPN tunnel. OVPN server will pick up the first address, all clients will get x.x.x.10 - 50 addresses. In our example, we use 192.168.99.0/24
-L Login
-P Password
-n NTP servers, if not specify Google DNS 8.8.8.8 will be assigned automatically. Format -n "1.1.1.1 1.1.1.2"
```
### Config generation
Change the network, which is located behind OVPN server container. -p flag
```
docker run -v win-volume:/etc/openvpn vadims06/ovpn-psk-based-for-win-clients:latest ovpn_genconfig -r "192.168.88.0/24" -p "route #.#.#.0 255.255.255.0" -s "192.168.99.0/24" -L "user1" -P "pass1" -C "AES-256-CBC" -a "sha1"
```
## Start OpenVPN container with all prepared configuration for OVPN
```
docker run -v win-volume:/etc/openvpn --name ovpn-psk-win -d -p 1194:1194 --cap-add=NET_ADMIN vadims06/ovpn-psk-based-for-win-clients:latest
```
