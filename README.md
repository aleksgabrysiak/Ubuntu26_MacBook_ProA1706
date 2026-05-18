# Ubuntu 26 MacBook Pro A1706 (13,2)
This repo is dedicated to explanation of the MacBook Pro A1706 config for Ubuntu 26. 

## WiFi Fix for Broadcom 43602
Just after installation I have noticed that the wifi does see 2,4 GHz networks, however cannot connect to any of them. 
It did not recognize any of the 5GHz networks. 

Fix:
### download both files
brcmfmac43602-pcie.bin\
brcmfmac43602-pcie.txt

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

Below instruction worked for me after first attempt. After the reboot I could finally liesten to music.

```bash
git clone https://github.com/davidjo/snd_hda_macbookpro.git
cd snd_hda_macbookpro/
sudo ./install.cirrus.driver.sh
sudo reboot
```
\

## Touchbar drivers - build and install

### Import of Ronald Tschalär's Macbook T1 drivers for the touchbar, ambient light sensor, and webcam, for kernel v7
Download the zip file with the code\
mbp-t1-touchbar-driver-main.zip\
unzip it and then:

### 1. Build

From the repository root:

```bash
cd /mbp-t1-touchbar-driver-main
make
```

### 2. Install

Install the modules to the kernel module tree:

```bash
sudo make install
```

If your system uses initramfs, update it after installation:

```bash
sudo update-initramfs -u
```

### 3. Deploy supporting files

Copy the helper script, service unit, and blacklist to the proper system locations:

```bash
sudo cp apple-ib-touchbar-rebind.sh /usr/local/bin/
sudo cp apple-ib-touchbar.service /etc/systemd/system/
sudo cp apple-ib-touchbar-blacklist.conf /etc/modprobe.d/
sudo chmod +x /usr/local/bin/apple-ib-touchbar-rebind.sh
```


### 4. Configure module blacklist

The blacklist file prevents conflicting HID/ALS drivers from loading automatically (already copied in step 3.):

```conf
blacklist hid_sensor_hub
blacklist hid_sensor_als
blacklist apple_ib_als
```

### 5. Enable the systemd service

Reload systemd, then enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable apple-ib-touchbar.service
sudo systemctl start apple-ib-touchbar.service
```

The service is defined as `Type=oneshot`, so it runs once at boot and finishes.

### 6. How the helper script works

The script performs these steps:

1. Removes conflicting modules if loaded:
   - `hid_sensor_hub`
   - `hid_sensor_als`
   - `apple_ib_als`
2. Loads `apple_ibridge`
3. Loads `apple_ib_tb`
4. Rebinds the USB device on bus `1-3`

This ensures the touch bar receives the correct HID interface and avoids the `0302` conflict.

### 7. Manual testing

Run the helper manually when needed:

```bash
sudo /usr/local/bin/apple-ib-touchbar-rebind.sh
```

If the touch bar is already working, this is useful after rebuilding and reinstalling the modules.

### 8. Verify status

Check service status with:

```bash
systemctl status apple-ib-touchbar.service
```

A healthy state should show:

- `Loaded: loaded ... enabled`
- `Active: inactive (dead)`
- `status=0/SUCCESS`

Also verify the loaded modules:

```bash
lsmod | grep -E 'apple_ib_tb|apple_ibridge|apple_ib_als|hid_sensor_hub|hid_sensor_als'
```

Expected result:

- `apple_ib_tb` loaded
- `apple_ibridge` loaded
- `hid_sensor_hub`, `hid_sensor_als`, `apple_ib_als` not loaded

### 9. Notes

- If you rebuild the drivers, repeat `make`, `sudo make install`, and optionally `sudo update-initramfs -u`.
- If the touch bar stops working after a reboot, run the helper script again.
- If you want to restore ALS support, remove or modify the blacklist file.


## Problems with wakeing up from the suspended state

I have been having an issue with my laptop freshly after installation of the system. 
Whenever I suspended it/closed the lid, it's been taking a lot of time to wake up (2 minutes of black screen) and after the screen went live, I was unable to type my password. I had to force shutdown each time by holding power button.

This was apparently caused by NVME not properly waking up.\ 
Fixed it following this procedure: 
### Create a service which starts up on boot

```bash
sudo nano /etc/systemd/system/fix_sleep.service
```
In the file, add these lines:
```bash
# systemd oneshot service to set sleep boolean on Apple Macbook Pro disks
# Original by Pier Lim. Posted at https://kerpanic.wordpress.com/2018/03/13/apple-keyboard-get-function-keys-working-properly-in-ubuntu/

[Unit]
Description=Job that disables sleep from stopping nvme hardware on MBP
    
[Service]
ExecStart=/sbin/fixsleep
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Save the changes (CTRL+O->Enter) and exit (CTRL+X)

Now, before adding below lines please check your NVMe drive data by running:
```bash
lspci | grep -i nvme
```
I have got something like:\
01:00.0 Mass storage controller: Apple Inc. S3X NVMe Controller (rev 12)\
If your nvme location is different, you need to adjust it in the script below.

Next create /sbin/fixsleep script
```bash
sudo nano /sbin/fixsleep 
```
Content:
```bash
#!/bin/bash
/bin/echo 0 > /sys/bus/pci/devices/0000\:01\:00.0/d3cold_allowed
```

Save the changes (CTRL+O->Enter) and exit (CTRL+X)



Make the file executable 
```bash
sudo chmod +x /sbin/fixsleep 
```

Now you can reload daemon and run your service to test whether everything works as expected:
```bash
sudo systemctl daemon-reload
sudo systemctl start fix_sleep.service
```

Thanks to this now my laptop wakes up from the syspension within 10 seconds and I am able to sign in and continue my work.
