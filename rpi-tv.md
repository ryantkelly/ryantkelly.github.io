# Raspberry Pi 3, kiosk mode, and enterprise WiFi

I recently set up a Rapsberry Pi to display a website on a 60" TV in my office. It is displaying a slideshow of various sales metrics (being served up from another Raspberry Pi, coincidentally). 

I could have used the built-in Samsung browser, but that had some issues - mainly that the TV won't connect to enterprise WiFi. Also, I liked the flexiblity of having full control over what was being displayed on the TV - [RetroPie](https://retropie.org.uk), anyone?

So, the Raspberry Pi:
- Connects to our enterprise WiFi network at boot.
- Boots directly into Chromium in kiosk mode and loads the website that I want to display.

## Connecting to enterprise WiFi

[This guide](https://gist.github.com/chatchavan/3c58511e3d48f478b0c2) from @chatchavan was enormously helpful. Of course, using a Pi 3 I didn't have to worrry about dongles or anything like that.

I am summarizing most of it below, focusing on the enterprise aspect.

### wpa_supplicant.conf edits

In order to connect to enterprise WiFi, you need to make changes to `/etc/wpa_supplicant/wpa_supplicant.conf` and `/etc/network/interface`. Pretty straight forward.

As always, **make a backup** of your configuration files!

**Note that** @chatchavan recommends hashing your password - I tried this, but it did not connect with the hashed password. But it did connect with my password in clear text (eek). I don't know if this is due to the Pi, or my office network. I need to do some more digging. However, the Pi is only accessible via SSH so I'm okay with this for the moment.

Give [hashing it](https://gist.github.com/chatchavan/3c58511e3d48f478b0c2#appendix2) a go on your system - your WiFi might be different than mine.

`echo -n 'YOUR_PASSWORD' | iconv -t utf16le | openssl md4`

The output will be something like `(stdin)= 31d6cfe0d16ae931b73c59d7e0c089c0` - the string after the equals sign is your hash. After hashing your password, clear input history with `history -c`

Use `sudo` to open `/etc/wpa_supplicant/wpa_supplicant.conf` and add the following for enterprise configuration:

```
network={
    ssid="YOUR_NETWORK_NAME"
    proto=RSN
    key_mgmt=WPA-EAP
    pairwise=CCMP TKIP
    group=CCMP TKIP
    identity="YOUR_USER_NAME"
    password=hash:YOUR_PASSWORD_HASH
    phase1="peaplabel=0"
    phase2="auth=MSCHAPV2"
}
```

If you are not hasing your password, the `password` line will read as 

```
    password="YOUR_PASSWORD"
```

### interface edits

`sudo` edit `/etc/network/interface`. In the block for `wlan0`, replace the text there with the following:

```
auto wlan0
allow-hotplug wlan0
iface wlan0 inet dhcp
	pre-up wpa_supplicant -B -Dwext -i wlan0 -c/etc/wpa_supplicant/wpa_supplicant.conf
	post-down killall -q wpa_supplicant
```

### Restart WiFi

Restart `wlan0` with:
```
sudo ifdown wlan0
sudo ifup wlan0
```

...aaaand you should be connected to your enterprise wifi!

## Boot into Chromium in kiosk mode

In order to boot Chromium into kiosk mode, we just need to add a file to `./config/autostart` - which I didn't know existed until I started digging to figure out how to do this! Yay learning! This is summarized from this [StackExchange post](http://raspberrypi.stackexchange.com/questions/38515/auto-start-chromium-on-raspbian-jessie-11-2015).

First, you need Chromium:
```
sudo apt-get install chromium-browser
```

Then, create the Chromium autostart file with your editor of choice:

```
nano ~./config/autostart/autoChromium.desktop
```

And enter the following into the file:
```
[Desktop Entry]
Type=Application
Exec=/usr/bin/chromium-browser --noerrdialogs --disable-session-crashed-bubble --disable-infobars --kiosk http://www.website.com
Hidden=false
X-GNOME-Autostart-enabled=true
Name[en_US]=AutoChromium
Name=AutoChromium
Comment=Start Chromium when GNOME starts
```

## Problems

Despite setting `--disable-session-crashed-bubble`, I have still gotten this error message intermittently. But if I SSH into the Pi and reboot, it boots back into kiosk mode without that message appearing.

Also, getting Chromium *out of* kiosk mode is a bit of a pain - only way I've been able to do it is to remove the `autoChromium.desktop` file, and reboot. However, as this Pi is dedicated to just loading and displaying a webpage, that is not something I need to do...really ever. Only if I want to do something else on a 60" TV in the office!
