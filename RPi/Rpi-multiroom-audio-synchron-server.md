# RPi - Multiroom Audio Server


_(datt Ding, wo die Musik reinkommen tut…)_

## Voraussetzung

* [Volumio](https://volumio.org/) ist installiert

## Software nachinstallieren

```
sudo apt-get update
sudo apt-get install build-essential git
sudo apt-get install libboost-dev libboost-system-dev libboost-program-options-dev libasound2-dev libvorbis-dev libflac-dev alsa-utils libavahi-client-dev avahi-daemon
```

## snapcast ausm git installieren

Sourcen runterladen…

```
git clone https://github.com/badaix/snapcast.git
cd snapcast/externals/
git submodule update --init --recursive
cd ..
```

compilieren…

```
make all
```

Server und Client installieren…

```
sudo make installserver
sudo make installclient
```

## Anpassungen

### ''/etc/mpd.conf'' anpassen

altes Zeux auskommentieren

`/etc/mpd.conf`

```
#audio_output {
#
#       type       "alsa"
#       name       "Output"
#       device    "hw:0,0"
#       dop    "no"
#
#}
```

folgendes hinzufügen

`/etc/mpd.conf`

```
audio_output {
    type            "fifo"
    name            "snapcast"
    path            "/tmp/snapfifo"
#    format          "44100:16:2"
    format          "48000:16:2"
#    codec          "flac"
    mixer_type      "software"
}
```

### ''/etc/default/snapserver'' anpassen

```
SNAPSERVER_OPTS="-d -s pipe:///tmp/snapfifo?name=multiroom&mode=read"
```

Alternativ: bei mehreren Räumen/Streams:

```
SNAPSERVER_OPTS="-d -s pipe:///tmp/snapfifo-room_1?name=room_1&sampleformat=48000:16:2&codec=flac -s pipe:///tmp/snapfifo-room_2?name=room_2&sampleformat=48000:16:2&codec=flac -s pipe:///tmp/snapfifo-room_3?name=room_3&sampleformat=48000:16:2&codec=flac"
```

## 

## SpotifyConnect installieren + einrichten

### Spotify Key beantragen

_(geht nur mir einen Spotify Premiumaccount)_

* einloggen unter: [https://devaccount.spotify.com/my-account/keys/](https://devaccount.spotify.com/my-account/keys/)
* eine „App“ anlegen und App-Key als „**Binary**“ herunterladen (`spotify_appkey.key`)

### Spotify installieren

Skript herunterladen

```
curl -O curl -OL https://github.com/Fornoth/spotify-connect-web/releases/download/0.0.3-alpha/spotify-connect-web.sh
chmod u+x spotify-connect-web.sh
```

pre-built chroot installieren

```
# Download the current chroot (~ 180 MB)
./spotify-connect-web.sh install
```

App-Key kopieren

    # Copy your `spotify_appkey.key` into the app directory. (See below for information on how to get that file.)
    sudo cp spotify_appkey.key spotify-connect-web-chroot/usr/src/app/

testen

```
# Run using normal cmdline options
./spotify-connect-web.sh --username 1234 --password abcd --bitrate 320 --name Volumio --playback_device plughw:CARD=ALSA,DEV=0
```

### Anpassungen

„snapfifo“ löschen & als pipe neu anlegen

```
rm -Rfv /root/spotify-connect-web-chroot/tmp/snapfifo
mknod /root/spotify-connect-web-chroot/tmp/snapfifo p
```

### asound.conf (chroot) anlegen

Spotify wird ja in einer eigenen chroot ausgeführt.

Dort muss ALSA mitgeteilt werden, dass seine Ausgabe in eine Pipe umgeleitet werden soll…

`/root/spotify-connect-web-chroot/etc/asound.conf`

```
# <USER>'s Spotify Connect Einstellung
#######################################################################
pcm.!<USER> {
    type plug
#    slave.pcm rate44100Hz
    slave.pcm rate48000Hz
}

#pcm.rate44100Hz {
pcm.rate48000Hz {
    type rate
    slave {
        pcm writeFile # Direct to the plugin which will write to a file
        format S16_LE
#        rate 44100
        rate 48000
    }
}

pcm.writeFile {
    type file
    slave.pcm null
    file "/tmp/snapfifo-spotify-<USER>"
#    file "/root/spotify-connect-web-chroot/tmp/snapfifo"
    format "raw"
}
#######################################################################
```

### fstab anpassen

Um das_„echte“_`/tmp`-Verzeichnis in der chroot verfügbar zu machen, ergänzen wir noch eine Zeile in der`/etc/fstab`

```
/tmp    /root/spotify-connect-web-chroot/tmp/    none    bind    0    0
```

#### 

### Testen (SpotifyConnect starten)

```
/root/spotify-connect-web.sh --username <NAME> --password <GEHEIM> --bitrate 320 --name <USER>s_Volumio -o <USER>
```

wenn der Dienst gestartet ist sollte in $einer Spotify-Anwendung folgende zusätzliche Wiedergabe-Optionen verfügbar sein

### Systemd Unit anlegen

zuerst eine kleine Config für die Unit in`/etc/default/spotify-conncect`anlegen:

```
# User 1
###################s
USER1=<DEIN_SPOTIFY_USERNAME>
PW1=<DEIN_SPOTIFY_PASSWORT>
BITRATE1=320
NAME1=<NAME_DES_CHANNELS>
ALSA1=<ALSA_OPTIONEN>     # Name der Chain in <CHROOT>/etc/asound.conf
```

dann die Unit selbst unter`/lib/systemd/system/spotify-conncect-<USER>.service`anlegen:

```
[Unit]
Description=Spotify Connect Service von $USER1
After=network.target

[Service]
EnvironmentFile=-/etc/default/spotify-conncect
Type=idle
#ExecStartPre=/bin/sleep 25
ExecStart=/root/spotify-connect-web.sh --username $USER1 --password $PW1 --bitrate $BITRATE1 --name $NAME1 -o $ALSA1
Restart=always
RestartSec=10
StartLimitInterval=30
StartLimitBurst=20


[Install]
WantedBy=multi-user.target
```

Units neu laden & Dienst starten

```
systemctl daemon-reload
systemctl enable spotify-connect-<USER>.service
systemctl restart spotify-connect-<USER>.service
```

## Weblinks

* [http://powerpi.de/spotify-connect-auf-dem-raspberry-pi-2-installieren-dein-smartphone-ist-die-fernbedienung/](http://powerpi.de/spotify-connect-auf-dem-raspberry-pi-2-installieren-dein-smartphone-ist-die-fernbedienung/)
* [http://www.runeaudio.com/forum/spotify-connect-autostart-t3315.html](http://www.runeaudio.com/forum/spotify-connect-autostart-t3315.html)
* [https://discourse.osmc.tv/t/howto-setup-a-spotify-connect-web-server-on-a-raspberry-pi-with-osmc/15818](https://discourse.osmc.tv/t/howto-setup-a-spotify-connect-web-server-on-a-raspberry-pi-with-osmc/15818)
* [https://github.com/Fornoth/spotify-connect-web](https://github.com/Fornoth/spotify-connect-web)
* [https://dennis-henke.de/2016/01/27/spotify-connect-server-raspberry-pi/](https://dennis-henke.de/2016/01/27/spotify-connect-server-raspberry-pi/)
* [https://github.com/badaix/snapcast/issues/75](https://github.com/badaix/snapcast/issues/75)



