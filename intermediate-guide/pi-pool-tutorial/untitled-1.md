# Environment Setup

| description |
| :--- |
| Install packages needed to run cardano-node and configure our environment |

## Environment Setup

### Install packages

Enable automatic updates.

```text
sudo apt update && sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

Install necessities.

```text
sudo apt install build-essential libssl-dev tcptraceroute chrony python3-pip \
         git jq bc make automake rsync htop curl unzip net-tools build-essential pkg-config \
         libffi-dev libgmp-dev libssl-dev libtinfo-dev libsystemd-dev \
         zlib1g-dev make g++ wget libncursesw5 libtool autoconf -y
```

House cleaning.🧹

```text
sudo apt clean
sudo apt autoremove
sudo apt autoclean
```

### Environment

#### Chrony

We need to get our time synchronization as accurate as possible. Open /etc/chrony/chrony.conf

```text
sudo nano /etc/chrony/chrony.conf
```

Replace the contents of the file with below, Save and exit.

```text
pool time.google.com       iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
#pool time.facebook.com     iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool time.euro.apple.com   iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool time.apple.com        iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3
pool ntp.ubuntu.com        iburst minpoll 2 maxpoll 2 maxsources 3 maxdelay 0.3

# This directive specify the location of the file containing ID/key pairs for
# NTP authentication.
keyfile /etc/chrony/chrony.keys

# This directive specify the file into which chronyd will store the rate
# information.
driftfile /var/lib/chrony/chrony.drift

# Uncomment the following line to turn logging on.
#log tracking measurements statistics

# Log files location.
logdir /var/log/chrony

# Stop bad estimates upsetting machine clock.
maxupdateskew 5.0

# This directive enables kernel synchronisation (every 11 minutes) of the
# real-time clock. Note that it can’t be used along with the 'rtcfile' directive.
rtcsync

# Step the system clock instead of slewing it if the adjustment is larger than
# one second, but only in the first three clock updates.
makestep 0.1 -1

# Get TAI-UTC offset and leap seconds from the system tz database
leapsectz right/UTC

# Serve time even if not synchronized to a time source.
local stratum 10
```

```text
sudo service chrony restart
```

#### Create a few directories

```text
mkdir -p $HOME/.local/bin
mkdir -p $HOME/pi-pool/files
mkdir $HOME/git
```

#### Create bash variables & add ~/.local/bin to our $PATH

{% hint style="info" %} [Environment Variables in Linux/Unix](https://askubuntu.com/questions/247738/why-is-etc-profile-not-invoked-for-non-login-shells/247769#247769). {% endhint %}

```text
echo PATH="$HOME/.local/bin:$PATH" >> $HOME/.bashrc
echo export NODE_HOME=$HOME/pi-pool >> $HOME/.bashrc
echo export NODE_FILES=$HOME/pi-pool/files >> $HOME/.bashrc
echo export NODE_CONFIG=mainnet>> $HOME/.bashrc
echo export NODE_BUILD_NUM=$(curl https://hydra.iohk.io/job/Cardano/iohk-nix/cardano-deployment/latest-finished/download/1/index.html | grep -e "build" | sed 's/.*build\/\([0-9]*\)\/download.*/\1/g') >> $HOME/.bashrc
echo export CARDANO_NODE_SOCKET_PATH="$NODE_HOME/db/socket" >> $HOME/.bashrc
source $HOME/.bashrc
```

#### Retrieve aarch64 binaries

{% hint style="info" %} It is no longer possible to build GHC for arm out of the box. The **unofficial** cardano-node & cardano-cli binaries available to us are being built by an IOHK engineer in his spare time. Please visit the '[Arming Cardano](https://t.me/joinchat/FeKTCBu-pn5OUZUz4joF2w)' Telegram group for more information. {% endhint %}

```text
mkdir $HOME/tmp
cd $HOME/tmp
wget -O cardano_node_$(date +"%m-%d-%y").zip https://ci.zw3rk.com/job/Tools/master/rpi64-musl.tarball/latest-finished/download
unzip *.zip
mv cardano-node/* $HOME/.local/bin
cd $HOME
rm -r $HOME/tmp
```

Confirm binaries are in ada $PATH.

```text
cardano-node version
cardano-cli version
```

#### Retrieve configuration files

```text
cd $NODE_FILES
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-byron-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-topology.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-shelley-genesis.json
wget -N https://hydra.iohk.io/build/${NODE_BUILD_NUM}/download/1/${NODE_CONFIG}-config.json
```

Set TraceBlockFetchDecisions to "true" & TraceMempool to "false".

{% hint style="warning" %} It is recommended to turn TraceMempool to false for relays as of this writing \(cardano-node version 1.25.1\) as it is using an excessive amount of cpu. A fix will come soon. I keep it running on my core node, it's a valuable metric. {% endhint %}

```text
sed -i ${NODE_CONFIG}-config.json \
    -e "s/TraceBlockFetchDecisions\": false/TraceBlockFetchDecisions\": true/g" \
    -e "s/TraceMempool\": true/TraceMempool\": false/g"
```

### Systemd unit files

Let us now create the systemd unit file and startup script so systemd can manage cardano-node.

```text
sudo nano $HOME/.local/bin/cardano-service
```

Paste the following, save & exit.

```text
#!/bin/bash
DIRECTORY=/home/ada/pi-pool
FILES=${DIRECTORY}/files
PORT=3003
HOSTADDR=0.0.0.0
TOPOLOGY=${FILES}/mainnet-topology.json
DB_PATH=${DIRECTORY}/db
SOCKET_PATH=${DIRECTORY}/db/socket
CONFIG=${FILES}/mainnet-config.json
cardano-node run +RTS -N4 -RTS --topology ${TOPOLOGY} --database-path ${DB_PATH} --socket-path ${SOCKET_PATH} --host-addr ${HOSTADDR} --port ${PORT} --config ${CONFIG}
```

Allow execution of our new startup script.

```text
sudo chmod +x $HOME/.local/bin/cardano-service
```

Open /etc/systemd/system/cardano-node.service

```text
sudo nano /etc/systemd/system/cardano-node.service 
```

Paste the following, save & exit.

```text
# The Cardano Node Service (part of systemd)
# file: /etc/systemd/system/cardano-node.service 

[Unit]
Description     = Cardano node service
Wants           = network-online.target
After           = network-online.target

[Service]
User            = ada
Type            = simple
WorkingDirectory= /home/ada/pi-pool
ExecStart       = /bin/bash -c "PATH=/home/ada/.local/bin:$PATH exec /home/ada/.local/bin/cardano-service"
KillSignal=SIGINT
RestartKillSignal=SIGINT
TimeoutStopSec=3
LimitNOFILE=32768
Restart=always
RestartSec=5

[Install]
WantedBy= multi-user.target
```

Set permissions and reload systemd so it picks up our new service file..

```text
sudo systemctl daemon-reload
```

Let's add a function to the bottom of our .bashrc file to make life a little easier.

```text
cardano-service() {
    #do things with parameters like $1 such as
    sudo systemctl "$1" cardano-node.service
}
```

Save & exit.

What we just did there was add a function to control our cardano-service without having to type out

> > sudo systemctl enable cardano-node.service
> >
> > sudo systemctl start cardano-node.service
> >
> > sudo systemctl stop cardano-node.service
> >
> > sudo systemctl status cardano-node.service

Now we just have to:

* cardano-service enable \(enables cardano-node.service auto start at boot\)
* cardano-service start \(starts cardano-node.service\)
* cardano-service stop \(stops cardano-node.service\)
* cardano-service status \(shows the status of cardano-node.service\)

### Congratulations you are now ready to start cardano-node

{% hint style="danger" %} If you would like to make an image file you can use to quickly write what you have completed thus far to other Raspberry Pi 4's now is the time to do so before you start to sync the chain\(db folder\). {% endhint %}

### ⛓ Syncing the chain⛓

{% hint style="danger" %} Do not attempt this on an 8GB sd card. Not enough space! Create your image file and flash it to your ssd. {% endhint %}

You are now ready to start cardano-node. Doing so will start the process of 'syncing the chain'. This is going to take about 10 hours and the db folder is about 7GB in size right now. We used to have to sync it to one node and copy it from that node to our new ones to save time.

I have started taking snapshots of my backup nodes db folder and hosting it in a web directory. With this service it takes around 15 minutes to pull the latest snapshop and maybe another 30 minutes to sync up to the tip of the chain. This service is provided as is. It is up to you. If you wan't to sync the chain on your own simply:

```text
cardano-service enable
cardano-service start
cardano-service status
```

Status should show as enabled & running.

### Download snapshot

Make sure your node is not running & delete the db folder if it exists

```text
cardano-service stop
cd $NODE_HOME
rm -r db/
wget -r -np -nH -R "index.html*" -e robots=off https://db.adamantium.online/db/
```

