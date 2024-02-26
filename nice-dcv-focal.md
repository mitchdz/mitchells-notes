# Setting up nice-dcv on focal

**NOTE:** Make sure TCP over 8443 is in security group
Create AWS EC2 instance, we want t3.large for more RAM or else we will run out with the graphical environment.

copy the following to a script and execute it
```
sudo apt update -y
sudo apt install ubuntu-desktop gdm3 -y
sudo apt upgrade -y

# Disable wayland
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/g' /etc/gdm3/custom.conf
sudo systemctl restart gdm3

# Start the X Server
sudo systemctl isolate graphical.target

# install dummy driver - https://docs.aws.amazon.com/dcv/latest/adminguide/setting-up-installing-linux-prereq.html#linux-prereq-nongpu
sudo apt install xserver-xorg-video-dummy -y
if [ -f /etc/X11/xorg.conf ]; then sudo mv /etc/X11/xorg.conf /etc/X11/xorg.conf.bak; fi
sudo tee /etc/X11/xorg.conf << EOF
Section "Device"
    Identifier "DummyDevice"
    Driver "dummy"
    Option "ConstantDPI" "true"
    Option "IgnoreEDID" "true"
    Option "NoDDC" "true"
    VideoRam 2048000
EndSection

Section "Monitor"
    Identifier "DummyMonitor"
    HorizSync   5.0 - 1000.0
    VertRefresh 5.0 - 200.0
    Modeline "1920x1080" 23.53 1920 1952 2040 2072 1080 1106 1108 1135
    Modeline "1600x900" 33.92 1600 1632 1760 1792 900 921 924 946
    Modeline "1440x900" 30.66 1440 1472 1584 1616 900 921 924 946
    ModeLine "1366x768" 72.00 1366 1414 1446 1494  768 771 777 803
    Modeline "1280x800" 24.15 1280 1312 1400 1432 800 819 822 841
    Modeline "1024x768" 18.71 1024 1056 1120 1152 768 786 789 807
EndSection

Section "Screen"
    Identifier "DummyScreen"
    Device "DummyDevice"
    Monitor "DummyMonitor"
    DefaultDepth 24
    SubSection "Display"
        Viewport 0 0
        Depth 24
        Modes "1920x1080" "1600x900" "1440x900" "1366x768" "1280x800" "1024x768"
        virtual 1920 1080
    EndSubSection
EndSection
EOF
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target


# install nice-dcv-server - https://docs.aws.amazon.com/dcv/latest/adminguide/setting-up-installing-linux-server.html
wget https://d1uj6qtbmh3dt5.cloudfront.net/NICE-GPG-KEY
gpg --import NICE-GPG-KEY
wget https://d1uj6qtbmh3dt5.cloudfront.net/2023.0/Servers/nice-dcv-2023.0-15065-ubuntu2004-x86_64.tgz
tar -xvzf nice-dcv-2023.0-15065-ubuntu2004-x86_64.tgz
cd nice-dcv-2023.0-15065-ubuntu2004-x86_64/
sudo apt install ./nice-dcv-server_2023.0.15065-1_amd64.ubuntu2004.deb -y
sudo usermod -aG video dcv
sudo apt install ./nice-xdcv_2023.0.547-1_amd64.ubuntu2004.deb -y
cd ../
if [ -f /etc/dcv/dcv.conf ]; then sudo mv /etc/dcv/dcv.conf /etc/dcv/dcv.conf.bak; fi
sudo tee /etc/dcv/dcv.conf << EOF
[license]

[log]
directory=/var/log/dcv
level=info
rotation-interval=every-hour
max-file-size=10
rotation-suffix=timestamp

[session-management]
create-session=true

[session-management/defaults]

[session-management/automatic-console-session]
owner=ubuntu
client-eviction-policy=same-user-oldest-connection
max-concurrent-clients=1
#permissions-file=/etc/wspdcvhostadapter/default.perm

[display]
grabber-target-fps=25
target-fps=20
web-client-max-head-resolution=(2560,1440)
enable-yuv444-encoding=always-off
full-frame-threshold=(20, 3)

[connectivity]
#web-port=8220
#quic-port=8220
#enable-quic-frontend=true
#quic-listen-endpoints=['198.19.104.169']
#web-listen-endpoints=['198.19.104.169']
#idle-timeout=0
#idle-timeout-warning=0

[security]
#auth-token-verifier=http://127.0.0.1:8240

[display/linux]
output-refresh-rate=20

[webcam]
enabled-sessions=console

[printer]
file-printer-name=

[audio]
avsync-support=disabled

[display/x264]
x264opts={'me_range':'0'}

[metrics]
reporters=['jsonlogfile']

[smartcard]
library-injection=disabled
EOF

# enable systemd unit so a session is started on boot
sudo systemctl enable dcvserver.service
sudo systemctl start dcvserver.service

# You can check the session is active like so:
# $ dcv list-sessions 
# Session: 'console' (owner:ubuntu type:console)

# setup a password for the user
echo "ubuntu:passw0rd" | sudo chpasswd
```

user/pass will be: ubuntu/passw0rd

```
# Add public PPA
sudo add-apt-repository ppa:cloud-images/amazon-workspaces-focal -y
sudo apt update -y
sudo apt upgrade -y

# restart gnome-shell so upgrade takes affect
sudo systemctl restart gdm3
```
