# Ubuntu 26 MacBook Pro A1706
This repo is dedicated to explanation of the MacBook Pro A1706 config for Ubintu 26. 

## WiFi Fix for Broadcom 43602
Just after installation I have noticed that the wifi does see 2,4 GHz networks, however cannot connect to any of them. 
It did not recognize any of the 5GHz networks. 

Fix:
### copy files to /lib/firmware/brcm
```bash
sudo cp brcmfmac43602-pcie.* /lib/firmware/brcm/
 ```
### reload brcmfmac
```bash
sudo modprobe -r brcmfmac_wcc brmcafmac
sudo modprobe brcmfmac
```
 
### Restart NetworkManager
```bash
sudo systemctl restart NetworkManager
```
### outcome 
Now you should be able to connect to 5GHz networks, still network strength is to be fixed and cannot connect to 2,4GHz, however 5GHz is fully OK for me :)


## Audio fix

Below instruction worked for me in first attempt. After the reboot I could finally liesten to music.

```bash
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
sudo ./install.cirrus.driver.sh
sudo reboot
```
