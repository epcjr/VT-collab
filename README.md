# VT-collab
A collection of tools for testing layers 2 and 3 with pSSID.

## Setup
- [Download][ubuntu] and install the Ubuntu 22.04 Raspberry Pi Generic
  (64-bit ARM) preinstalled server image
- Make sure you have ssh access without the default network services
- Disable problematic services (note this will disable DHCP on the wired
  interface):
  ```bash
  systemctl --now disable \
    wpa_supplicant \
    unattended-upgrades
  ```
- Install additional dependencies: `apt install jq dhcpcd5`
- Make sure the `dhcpcd` service is disabled:
  ```bash
  systemctl --now disable dhcpcd
  ```

## `pssid-80211`

This script handles the Wi-Fi connection.

This script must be run as root.

### Connect

Given a [configuration file](#wpa_supplicant-configuration-file) and interface,
it will connect to the Wi-Fi network and print a basic report to `stdout` in
json format.

```bash
pssid-80211 -c wpa_supplicant.conf -i wlan0 > 80211-connect.json
```

80211-connect.json (pretty printed):
```json
{
  "status_code": 0,
  "status_msg": "success",
  "pssid_log": [
    {
      "time": 1716181420.534354,
      "verbosity": 0,
      "msg": "Starting supplicant"
    },
    {
      "time": 1716181427.443442,
      "verbosity": 0,
      "msg": "Connected"
    }
  ]
}
```

### Disconnect
To tear down the connection to cleanup and prepare for the next test, call the
command again with the `-d` flag.
Note that the configuration file and interface name are still required.

```bash
pssid-80211 -c wpa_supplicant.conf -i wlan0 -d > 80211-disconnect.json
```

80211-disconnect.json:
```json
{
  "status_code": 0,
  "status_msg": "success",
  "pssid_log": [
    {
      "time": 1716181449.676054,
      "verbosity": 0,
      "msg": "Disconnecting"
    }
  ],
  "wpa_log": "KLUv/QRYxRsAdny1JvCszAOTDD81rK..."
}
```

The path `.wpa_log` in the json contains the whole `wpa_supplicant` log,
compressed with zstd and encoded in base64.

Reading the supplicant log:
```bash
jq -r .wpa_log 80211-disconnect.json | base64 -d | zstd -d | head -5
```

```
1716181421.242576: Successfully initialized wpa_supplicant
1716181424.165842: wlan0: Trying to associate with SSID 'eduroam'
1716181426.983659: wlan0: Associated with cc:d0:83:ce:71:80
1716181426.983884: wlan0: CTRL-EVENT-SUBNET-STATUS-UPDATE status=0
1716181426.985464: wlan0: CTRL-EVENT-EAP-STARTED EAP authentication started
```

### wpa_supplicant configuration file
You must provide a file configuring wpa_supplicant for the wireless network you
want to test.
It can be named whatever you want, but the contents must conform to
[wpa_supplicant.conf][wpa_supplicant.conf].

Additionally, this **must** include a control interface directive.
Any suitable path will work, but the default is recommended.
```conf
ctrl_interface=/var/run/wpa_supplicant
```

Configuring more than one wireless network may produce unexpected/unpredictable
results.

#### Open network example
```conf
ctrl_interface=/var/run/wpa_supplicant
network={
  ssid="VT Open WiFi"
  key_mgmt=NONE
}
```

#### PEAP-MSCHAPv2 example
> [!NOTE]
> Even if you are connecting to an eduroam network, most of this is institution
> specific.
> Not all eduroam networks even use PEAP-MSCHAPv2!

```conf
ctrl_interface=/var/run/wpa_supplicant
network={
  ssid="eduroam"
  key_mgmt=WPA-EAP
  EAP=PEAP
  anonymous_identity="anonymous@vt.edu"
  identity="hokiebird@vt.edu"
  password="correct horse battery staple"
  #password=hash:1b9d5effd34ac283c8efe2eacaea8bbc
  ca_cert="blob://usertrustrsa"
  domain_match="eduroam.nis.vt.edu"
  phase2="auth=MSCHAPV2"
}

blob-base64-usertrustrsa={
MIIF3jCCA8agAwIBAgIQAf1tMPyjylGoG7xkDjUDLTANBgkqhkiG9w0BAQwFADCB
iDELMAkGA1UEBhMCVVMxEzARBgNVBAgTCk5ldyBKZXJzZXkxFDASBgNVBAcTC0pl
cnNleSBDaXR5MR4wHAYDVQQKExVUaGUgVVNFUlRSVVNUIE5ldHdvcmsxLjAsBgNV
BAMTJVVTRVJUcnVzdCBSU0EgQ2VydGlmaWNhdGlvbiBBdXRob3JpdHkwHhcNMTAw
MjAxMDAwMDAwWhcNMzgwMTE4MjM1OTU5WjCBiDELMAkGA1UEBhMCVVMxEzARBgNV
BAgTCk5ldyBKZXJzZXkxFDASBgNVBAcTC0plcnNleSBDaXR5MR4wHAYDVQQKExVU
aGUgVVNFUlRSVVNUIE5ldHdvcmsxLjAsBgNVBAMTJVVTRVJUcnVzdCBSU0EgQ2Vy
dGlmaWNhdGlvbiBBdXRob3JpdHkwggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIK
AoICAQCAEmUXNg7D2wiz0KxXDXbtzSfTTK1Qg2HiqiBNCS1kCdzOiZ/MPans9s/B
3PHTsdZ7NygRK0faOca8Ohm0X6a9fZ2jY0K2dvKpOyuR+OJv0OwWIJAJPuLodMkY
tJHUYmTbf6MG8YgYapAiPLz+E/CHFHv25B+O1ORRxhFnRghRy4YUVD+8M/5+bJz/
Fp0YvVGONaanZshyZ9shZrHUm3gDwFA66Mzw3LyeTP6vBZY1H1dat//O+T23LLb2
VN3I5xI6Ta5MirdcmrS3ID3KfyI0rn47aGYBROcBTkZTmzNg95S+UzeQc0PzMsNT
79uq/nROacdrjGCT3sTHDN/hMq7MkztReJVni+49Vv4M0GkPGw/zJSZrM233bkf6
c0Plfg6lZrEpfDKEY1WJxA3Bk1QwGROs0303p+tdOmw1XNtB1xLaqUkL39iAigmT
Yo61Zs8liM2EuLE/pDkP2QKe6xJMlXzzawWpXhaDzLhn4ugTncxbgtNMs+1b/97l
c6wjOy0AvzVVdAlJ2ElYGn+SNuZRkg7zJn0cTRe8yexDJtC/QV9AqURE9JnnV4ee
UB9XVKg+/XRjL7FQZQnmWEIuQxpMtPAlR1n6BB6T1CZGSlCBst6+eLf8ZxXhyVeE
Hg9j1uliutZfVS7qXMYoCAQlObgOK6nyTJccBz8NUvXt7y+CDwIDAQABo0IwQDAd
BgNVHQ4EFgQUU3m/WqorSs9UgOHYm8Cd8rIDZsswDgYDVR0PAQH/BAQDAgEGMA8G
A1UdEwEB/wQFMAMBAf8wDQYJKoZIhvcNAQEMBQADggIBAFzUfA3P9wF9QZllDHPF
Up/L+M+ZBn8b2kMVn54CVVeWFPFSPCeHlCjtHzoBN6J2/FNQwISbxmtOuowhT6KO
VWKR82kV2LyI48SqC/3vqOlLVSoGIG1VeCkZ7l8wXEskEVX/JJpuXior7gtNn3/3
ATiUFJVDBwn7YKnuHKsSjKCaXqeYalltiz8I+8jRRa8YFWSQEg9zKC7F4iRO/Fjs
8PRF/iKz6y+O0tlFYQXBl2+odnKPi4w2r78NBc5xjeambx9spnFixdjQg3IM8WcR
iQycE0xyNN+81XHfqnHd4blsjDwSXWXavVcStkNr/+XeTWYRUc+ZruwXtuhxkYze
Sf7dNXGiFSeUHM9h4ya7b6NnJSFd5t0dCy5oGzuCr+yDZ4XUmFF0sbmZgIn/f3gZ
XHlKYC6SQK5MNyosycdiyA5d9zZbyuAlJQG03RoHnHcAP9Dc1ew91Pq7P8yF1m9/
qS3fuQL39ZeatTXaw2ewh0qpKJ4jjv9cJ2vhsE/zB+4ALtRZh8tSQZXq9EfX7mRB
VXyNWQKV3WKdwrnuWih0hKWbt5DHDAff9Yk2dDLWKMGwsAvgnEzDHNb842m1R0aB
L6KCq9NjRHDEjf8tM7qtj3u1cIiuPhnPQCjY/MiQu12ZIvVS5ljFH4gxQ+6IHdfG
jjxDah2nGN59PRbxYvnKkKj9
}
```

Only one of the `password` lines needs to be provided (and uncommented).
Using only the hash is a way to guard against shoulder surfing, but this is the
_only_ protection it provides.
It is as sensitive as the cleartext password.
To derive it, use:
```bash
printf 'correct horse battery staple' \
  | iconv -t UTF-16LE \
  | openssl dgst -provider legacy -md4
```

## `pssid-dhcp`

This script handles obtaining a Layer 3 connection via the usual automatic
methods.
IPv4 and IPv6 are supported.

This script must be run as root.

### Connect

Given an interface, it will connect and print a basic report to `stdout` in json
format.

> [!NOTE]
> There is sometimes a delay between `dhcpcd` completing and when the
> address(es) are applied to the interface.
> Even though the DHCP process is complete, the addresses may not be reflected
> in the output.

```bash
pssid-dhcp -i wlan0 > dhcp-connect.json
```

dhcp-connect.json (pretty printed):
```json
{
  "status_code": 0,
  "status_msg": "success",
  "pssid_log": [
    {
      "time": 1716184175.908605,
      "verbosity": 0,
      "msg": "dhcpcd started"
    },
    {
      "time": 1716184179.407504,
      "verbosity": 0,
      "msg": "dhcpcd exited with status 0"
    }
  ],
  "addr_info": {
    "1716184175.538637": {
      "ifname": "wlan0",
      "address": "d8:3a:dd:5a:07:e8",
      "addr_info": []
      ...
    },
    "1716184179.873960": {
      "ifname": "wlan0",
      "address": "d8:3a:dd:5a:07:e8",
      "addr_info": [ ... ]
      ...
    }
  }
}
```

### Disconnect

Just like with `pssid-80211`, just add the `-d` argument.
```bash
pssid-dhcp -i wlan0 -d > dhcp-disconnect.json
```

dhcp-disconnet.json (pretty printed):
```json
{
  "status_code": 0,
  "status_msg": "success",
  "pssid_log": [
    {
      "time": 1716184175.908605,
      "verbosity": 0,
      "msg": "dhcpcd started"
    },
    {
      "time": 1716184179.407504,
      "verbosity": 0,
      "msg": "dhcpcd exited with status 0"
    }
  ],
  "addr_info": { ... }
  "dhcpcd_log": "KLUv/QRYLR8AtuRnIUBP2+Nnu..."
```

And just like before, we can get the whole `dhcpcd` log:
```bash
jq -r .dhcpcd_log dhcp-disconnect.json | base64 -d | zstd -d
```

[ubuntu]: https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04.4-preinstalled-server-arm64+raspi.img.xz
[wpa_supplicant.conf]: https://w1.fi/cgit/hostap/plain/wpa_supplicant/wpa_supplicant.conf
