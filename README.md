# HA-HisenseTVviaMQTT

Good god this took a while to work out.

We will be using the connection scheme and bridge schema  provided by https://github.com/sehaas/ha_hisense_tv/blob/main/README.md?plain=1

Connection shema:
```
+-----------+          +-----------+
| Home      |  client  | Mosquitto |
| Assistant |--------->|           |
+-----------+          +-----------+
                            /\
                     bridge ||
                            \/
                      +-------------+
                      | Hisense TV  |
                      | MQTT Broker |
                      +-------------+
```

## Step 1 - Authorise MQTT Client with Hisense TV
Rather then using the RemoteNow app, and spoofing the MAC address in home assistant, we are just going to authorise MAC address XX:XX:XX:XX:XX and use that going forward.

You will need Cert and Key files, which i obtained from here https://github.com/d3nd3/Hisense-mqtt-keyfiles/tree/main

Use the following
```
Protocol: MQTT://
Host: <Your TV IP>
Port: 36669
Username: hisenseservice
Password: multimqttservice

Under ADVANCED : CERTIFICATES
Attach rcm_certchain_pem.cer as CLIENT CERTIFICATE
and rcm_pem_privkey.pkcs8 as CLIENT KEY
SERVER CERTIFICATE(CA) was not required for me. Encryption and Validate cert do not need to be ticked.
```
Once connected, in MQTT Explorer publish the following:
```
TOPIC: /remoteapp/tv/ui_service/XX:XX:XX:XX:XX/actions/gettvstate
```
A 4 digit code will appear on the TV, with that code publish the following, replacing YYYY with the 4 digit code:
``` 
OPIC: /remoteapp/tv/ui_service/XX:XX:XX:XX:XX:XX$normal/actions/authenticationcode
JSON: {"authNum":"YYYY"} 
```
DO NOT change XX:XX:XX:XX:XX:XX. We have now authorised this MAC address and will use this in any commands 
(Thanks to Rya for this one).

You can test if it works by attempting to launch Netflix and observing the state in MQTT Explorer. It will change to Netflix if successful or say Auth Error if not. If you get an Auth error just retry the above steps. To launch . test Netflix, publish the following:

```
TOPIC: /remoteapp/tv/ui_service/XX:XX:XX:XX:XX:XX$normal/actions/launchapp
JOSN:  {"appIcon":"","appId":"1","has_detail_page":0,"isLocalApp":1,"name":"Netflix","storeType":0,"type":0,"url":"netflix","urlType":37}
```
![image](https://github.com/user-attachments/assets/3cc79398-2292-47b2-95eb-a1211aa5827d)


## Step 2 - Integrating with Home Assistant
I am utilising  Mosquitto Broker. I also have Zigbee2MQTT installed, so I have catered for this by definiting a listening port. Otherwise Mosquitto failed to launch due to both services listening to the same port.

To connect, the info was provided here: https://github.com/tiagofreire-pt/Home_Assistant_Hisense_TV. The below info is my working tweaked copy:

First you will need to obtain a CA File. While this wasnt needed for MQTT Explorer, I could not get Mosquitto Broker to work without it.
Run this command inside a linux terminal, to get the certificates needed to connect to the TVs embedded MQTT broker:

```
openssl s_client -host TV_IP_ADDRESS_CHANGE_IT_HERE -port 36669 -showcerts
```

Create a file called `hisense.crt` with this structure.
```
-----BEGIN CERTIFICATE-----
qmierjfpaoisdjmçfaisldjcçfskdjafcaçskdjcçfmasidcf(...)
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
7ferusycedaystraedyasredyatrdsecdtrseydtraESYDTRASCY (...)
-----END CERTIFICATE-----
```


You will need to be able to access the HomeAssistant drive, I used the Samba add-on for this and deactivated it once done.

Navigate to `\ssl\` in the HA Samba share. Place the newly created `hisense.crt` file along with the Cert and Key files obtained earlier.

Now Navigate to `\share\mosquitto\` (you may need to create the mosquitto sub folder). This is where we will be storing our connection details. 
Create a file called `hisense.conf` with the following data

```
connection hisensemqtt
listener 1885
address <yourTCIP>:36669
username hisenseservice
password multimqttservice
clientid homeassistant-hisense
bridge_cafile /ssl/hisense.crt
bridge_certfile /ssl/rcm_certchain_pem.cer
bridge_keyfile /ssl/rcm_pem_privkey.key
bridge_tls_version tlsv1.2 
bridge_insecure true
try_private true
start_type automatic
topic +/remoteapp/# both
```

Save the file, and fire up Mosquitto Broker.

You will need to able Cuustom integrations by changing the active: false to active:true in the Configuration tab:
![image](https://github.com/user-attachments/assets/81a2765f-4840-4af2-89d4-bb4b4532de2e)

Turn your TV on and Restart Mosquitto and head over to the logs. You should see the following
```
2025-01-25 14:29:45: Connecting bridge hisensemqtt (<YourHAIP>:36669)
```

If your TV is OFF you will see connection issues (this is fine as we wake the TV using a magic packet.

If you see this error, check your cert files are correct

```
2025-01-25 14:28:47: OpenSSL Error[0]: error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed
2025-01-25 14:28:47: Bad socket read/write on client local.homeassistant-hisense: A TLS error occurred.
```

You have now connected the TV to Home Assistant

## Step 3 - How do i use it in HA

TaigoFreire-pt has provided a punch of items in their repo along with a example card: https://github.com/tiagofreire-pt/Home_Assistant_Hisense_TV/blob/master/README.md

How have I used it ?

I have used the firemote card, override functions and scripts to get a nice working remote
https://github.com/PRProd/HA-Firemote/

# Step 3.1 - Set up Entities
Initially let's set up some entities and devices so we can see our status

NOTE : I AM USING WIFI AND NOT ETHERNET, SO I AM INTENTIONALLY AVOIDING MAGIC PACKETS AND RELYING SOLEY ON MQTT. Not the best, but im not ready to run an ethernet cable mid guide.

In configuration.yaml dump the following;
```
switch:
  - platform: template
    switches:
      hisense_tv:
        icon_template: >
          {% if is_state('sensor.hisense_tv','fake_sleep_0') %}
            {{ 'mdi:television-classic-off' }}
          {% else %}
            {{ 'mdi:television-classic' }}
          {% endif %}
        friendly_name: 'Hisense TV'
        value_template: >
          {{ not is_state('sensor.hisense_tv', 'fake_sleep_0') }}
        turn_on:
          service: mqtt.publish
          data:
            topic: '/remoteapp/tv/remote_service/XX:XX:XX:XX:XX:XX$normal/actions/sendkey'
            payload: 'KEY_POWER'
        turn_off:
          service: mqtt.publish
          data:
            topic: '/remoteapp/tv/remote_service/XX:XX:XX:XX:XX:XX$normal/actions/sendkey'
            payload: 'KEY_POWER'
mqtt:
    sensor:
     -  name: "Hisense TV"
        state_topic: "/remoteapp/mobile/broadcast/ui_service/state"
        value_template: "{{ value_json.statetype }}"
```

This created 2 things:
1 - A sensor to tell whether the TV is on based on  an MQTT status

2 - A swtich to turn the TV on and Off. If you are using a magic packet, replace the "turn on" in the above script with this

```
        turn_on:
          service: wake_on_lan.send_magic_packet
          data:
            mac: 'TV_MAC_ADDRESS_CHANGE_IT_HERE'
```

# Step 3.2 - Make it friendly in Home Assistant
I will be using the firemote card for this.

Since we are not using a pre-defined TV setup, we will override every code. Firemote requires a script to do this, so we will create a script with "trigger id's". 

Script can be found here: https://github.com/Dsx2/HA-HisenseTVviaMQTT/blob/main/MQTTActions-HAScript

