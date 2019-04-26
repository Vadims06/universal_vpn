# About
This container allows to launch OpenVPN Server ready for connecting Mikrotik router with certificate based authentication.
## Preparation
* Download OVPN Image

```docker pull vadims06/ovpn-cert-based-for-mikrotik```
* Pick a name for the volume container. We use "Mkt-volume" in our example. There are configuration files, CA certificate, DH-keys and clients certificates will be stored in such volume.

```docker volume create Mkt-volume```

## Initialize the container for config generation.
Run the container with mentioned options in order to generate a configuration file for OVPN. The config will be saved in the `Mkt-volume`
#### Config flags
```
-u we set protocol for OVPN. Mikrotik support ONLY TCP!!! Choose URL whatever you want - we will use only IP, not domain names.
-r This will allow Mikrotik's private subnet to access the VPN. By default Mikrotik has 192.168.88.0/24 network for LAN.
-p Routes behind OVPN server container. Push routes to the client to allow them to reach other private subnets behind the server. Network format: x.x.x.x 255.255.255.0. Do not use short version of network notation like x.x.x.x/XX
-s subnet for OVPN tunnel. OVPN server will pick up the first address, all clients will get x.x.x.10 - 50 addresses. In our example we use 192.168.99.0/24
-C Cipher, by default set to AES-256-CBC. Supported variants by Mikrotik [aes128, aes192, aes256, blowfish128]
-n NTP servers, if not specify Google DNS 8.8.8.8 will be assigned automatically. Format -n "1.1.1.1 1.1.1.2"
List of allowed ciphers by OVPN:
* DES-CBC
* RC2-CBC
* DES-EDE-CBC
* DES-EDE3-CBC
* DESX-CBC
* BF-CBC
* RC2-40-CBC
* CAST5-CBC
* RC2-64-CBC
* AES-128-CBC
* AES-192-CBC
* AES-256-CBC
* none
-a Authentication, by default set to SHA1
```
Authenticate packets with HMAC using message digest algorithm alg. (The default is SHA1 ). HMAC is a commonly used message authentication algorithm (MAC) that uses a data string, a secure hash algorithm, and a key, to produce a digital signature.
OpenVPN's usage of HMAC is to first encrypt a packet, then HMAC the resulting ciphertext.
### Config generation
* Change the network, which is located behind OVPN server container. -p flag
```
docker run -v Mkt-volume:/etc/openvpn --rm vadims06/ovpn-cert-based-for-mikrotik ovpn_genconfig -u tcp://Mrt-test.docker-ovpn.com -r "192.168.88.0/24" -p "route x.x.x.x 255.255.255.0" -s "192.168.99.0/24"
```
* Initialize the container for certificate generation. The container will prompt for a passphrase to protect the private key used by the newly generated certificate authority.
```
docker run -v Mkt-volume:/etc/openvpn --rm -it vadims06/ovpn-cert-based-for-mikrotik ovpn_initpki
```
* Generate a client certificate without a passphrase
```
docker run -v Mkt-volume:/etc/openvpn --rm -it vadims06/ovpn-cert-based-for-mikrotik easyrsa build-client-full Mkt-test nopass
```
* Retrieve the client configuration with embedded certificates
```
docker run -v Mkt-volume:/etc/openvpn --rm vadims06/ovpn-cert-based-for-mikrotik ovpn_getclient Mkt-test > Mkt-test.ovpn
```
## Start OpenVPN container with all prepared configuration for OVPN
```
docker run -v Mkt-volume:/etc/openvpn -d -p 1194:1194 --cap-add=NET_ADMIN vadims06/ovpn-cert-based-for-mikrotik
```
## Upload Certificate to the Mikrotik
It's quite easy to do, because ftp service is constantly active on a Mikrotik. For example you can connect to the Mikrotik through WinSCP using login as "Admin" with empty password.
You should upload the following certificates:

* ca.crt
* Mkt-test.crt
* Mkt-test.key
```
[admin@MikroTik] > /certificate import file-name=ca.crt

     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0
[admin@MikroTik] > /certificate import file-name=Mkt-test.crt
     certificates-imported: 1
     private-keys-imported: 0
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0
[admin@MikroTik] > /certificate import file-name=Mkt-test.key
passphrase:
     certificates-imported: 0
     private-keys-imported: 1
            files-imported: 1
       decryption-failures: 0
  keys-with-no-certificate: 0
```
OpenVPN client configuration on Mikrotik
```
/interface ovpn-client
add certificate=Mkt-test.crt_0 cipher=aes256 connect-to=IP_ADDRESS_OF_Container disabled=yes max-mtu=1450 name=\
    OVPN-Docker password=none user=none
```
