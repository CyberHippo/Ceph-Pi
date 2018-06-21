
# CEPH Storage Cluster on Raspberry Pi.
# Installation Sheet

## Prerequisites
* 6 server nodes with Ubuntu 16.04 server installed
* Root privileges on all nodes


### Downloading the image
You can download Ubuntu server classic from the [Ubuntu Pi Flavour Maker website](https://ubuntu-pi-flavour-maker.org/download/). Make sure to choose the Ubuntu Classic Server 16.04 version for the Raspberry Pi 3. This image is a flavour of Ubuntu server, which have been stripped out of everything which is not strictly necessary, hence its small size.

### Flashing the image
Now, you must flash the image on the SD Card. It is recommended to use [Etcher.io](https://etcher.io/), which makes the process incredibly simple, fast and safe. Etcher is open-source and available for Windows, MacOS and Linux.


![Etcher.io](https://etcher.io/static/screenshot.gif)


Now you are all set to configure the nodes and build up your cluster !  


## Configuring the Nodes

### Activate the built-in Wifi

```
mkdir wifi-firmware
cd wifi-firmware
wget https://github.com/RPi-Distro/firmware-nonfree/raw/master/brcm/brcmfmac43430-sdio.bin
wget https://github.com/RPi-Distro/firmware-nonfree/raw/master/brcm/brcmfmac43430-sdio.txt
sudo cp *sdio* /lib/firmware/brcm/
cd ..
```

You must reboot the machine for changes to take effect.

Thanks to this [wiki](https://wiki.ubuntu.com/ARM/RaspberryPi).

### Set up a wireless connection

First, install the wifi support :

```
sudo apt-get install wireless-tools wpasupplicant
```

Reboot the machine since the name of your network interface might change during the process.

```
sudo reboot
```

Get the name of your wireless network interface with :
```
iwconfig
```

Open the network interfaces configuration:

```
sudo nano /etc/network/interfaces
```

Append the following content to the file (make sure to replace wlan0 with the name of your wireless network interface) :

```
	allow-hotplug wlan0
	iface wlan0 inet dhcp
	wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

Now, open the wireless configuration file :

```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Add the following content to the file with the information about your wifi network :

```
    network={
        ssid="NETWORK-SSID"
        psk="NETWORK-PASSWORD"
    }
```
With a WPA2 Entreprise network, you might want to use the following configuration instead :

```
    network={
    	  ssid="NETWORK-SSID"
    	  scan_ssid=1
      	key_mgmt=WPA-EAP
      	identity="YOUR_USERNAME"
      	password="YOUR_PASSWORD"
      	eap=PEAP
      	phase1="peaplabel=0"
        phase2="auth=MSCHAPV2"
    }
```

### Create the Ceph User

```
useradd -m -s /bin/bash cephuser
passwd cephuser
```

### Add the user to the sudoers

```
echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
chmod 0440 /etc/sudoers.d/cephuser
sed -i s'/Defaults requiretty/#Defaults requiretty'/g /etc/sudoers
```

### Install and Configure NTP

Install NTP to synchronize date and time on all nodes. Run the ntpdate command to set the date and time via NTP. We will use the US pool NTP servers. Then start and enable NTP server to run at boot time.
```
sudo apt-get install -y ntp ntpdate ntp-doc
ntpdate 0.us.pool.ntp.org
systemctl enable ntp
systemctl start ntp
```


### Install Python and parted

We will need python packages for building the ceph-cluster and parted to prepare the OSD nodes.
```
sudo apt-get install -y python python-pip parted
```


### Configure the Hosts File

You can edit the hosts file on all nodes with vim editor if the IP addresses don't resolve to their hostnames. Which might be the case with an enterprise network.
```
sudo nano /etc/hosts
```

Paste the configuration below:

```
    192.168.1.50      admin
    192.168.1.61      mon1
    192.168.1.62      mon2
    192.168.1.63      mon3
    192.168.1.71      osd1
    192.168.1.72      osd2
    192.168.1.73      osd3
```

### Adding CEPH repository to the sources

To add the CEPH repository to the sources, you have to create or edit the ``/etc/apt/sources.list.d/ceph.list`` file using :

```
sudo nano /etc/apt/sources.list.d/ceph.list
```

Replace the content of the file with the following line :

```
	deb https://download.ceph.com/debian-jewel/ xenial main
```

## Configure the SSH Server

Our admin node is used for installing and configuring all cluster node remotely. Therefore, we need a user on the ceph-admin node with privileges to connect to all nodes without a password. In other words, we need to configure password-less SSH access for the user *cephuser* on the *admin* node.

### Generate SSH keys

First, generate the ssh keys for *cephuser* :

```
ssh-keygen
```

Leave passphrase is blank/empty.

Next, create a configuration file for the ssh config.
```
sudo nano ~/.ssh/config
```

Paste the configuration below to the file:

```
    Host admin
            Hostname admin
            User cephuser
    Host mon1
            Hostname mon1
            User cephuser
    Host mon2
            Hostname mon2
            User cephuser
    Host mon3
            Hostname mon3
            User cephuser
    Host osd1
            Hostname osd1
            User cephuser
    Host osd2
            Hostname osd2
            User cephuser
    Host osd3
            Hostname osd3
            User cephuser
```

Change the permission of the config file to 644 :

```
sudo chmod 644 ~/.ssh/config
```

### Add the SSH keys to the nodes


Now we will use the ssh-copy-id command to add the key to all nodes :

```
ssh-keyscan osd1 osd2 osd3 mon1 mon2 mon3 >> ~/.ssh/known_hosts
ssh-copy-id osd1
ssh-copy-id osd2
ssh-copy-id osd3
ssh-copy-id mon1
ssh-copy-id mon2
ssh-copy-id mon3
```

## (Optional) Configure the Ubuntu Firewall

For security reasons, we need to turn on the firewall on the servers. Preferably we use Ufw (Uncomplicated Firewall), the default Ubuntu firewall, to protect the system. In this step, we might want to enable ufw on all nodes, then open the ports needed by the admin, the monitors and the osd's.

## Configure OSD Nodes

For this installation, we have 3 OSD nodes. Each of these nodes has two hard disk partitions. The first one on the micro-SD card holding the OS and an empty partition on a storage device attached by USB.
```
    /dev/sda for root partition
    /dev/sdb is empty partition
```

We will use /dev/sdb for the ceph disk. From the ceph-admin node, login to all OSD nodes and format the /dev/sdb partition with XFS file system.
```
ssh osd1
```
Note : repeat the process for osd2 and osd3.

Check the disk partitioning scheme using the *fdisk* command :

```
sudo fdisk -l /dev/sdb
```

### Format the /dev/sdb partition

We will format the ``/dev/sdb`` partition with an XFS filesystem and with a GPT partition table by using the parted command:

```
sudo parted -s /dev/sdb mklabel gpt mkpart primary xfs 0% 100%
```

Next, format the partition in XFS format with the mkfs command.

```
sudo mkfs.xfs -f /dev/sdb
```

You can check the partition status using these commands :

```
sudo fdisk -s /dev/sdb
sudo blkid -o value -s TYPE /dev/sdb
```

## Build the Ceph Cluster

In this section, we will install Ceph on all nodes from the admin node. To get started, login to the admin node.

```
ssh cephuser@dmin
```

### Install ceph-deploy on the admin node

Install *ceph-deploy* on the ceph-admin node with the pip command :

```
sudo pip install ceph-deploy
```

## Create a new Cluster

After the ceph-deploy tool has been installed, create a new directory for the Ceph cluster configuration :

```
mkdir cluster
cd cluster/
```

Next, using the *ceph-deploy* command, create a new cluster by passing the monitor node names as parameters :

```
ceph-deploy new mon1 mon2 mon3
```

The command will generate the Ceph cluster configuration file ``ceph.conf`` in cluster directory.

### Install Ceph on all nodes

Now install Ceph on all nodes from the ceph-admin node with the command below :

```
ceph-deploy install admin mon1 mon2 mon3
ceph-deploy install osd1 osd2 osd3
```

This step might take some time..

Now create a monitor key :
```
ceph-deploy mon create-initial
```

### Adding OSD's to the Cluster

After Ceph has been installed on all nodes, now we can add the OSD daemons to the cluster. OSD Daemons will create the data and journal partition on the disk ``/dev/sdb``.

You can check the available disk ``/dev/sdb`` on all osd nodes and see ``/dev/sdb`` with the XFS format that we created before :
```
ceph-deploy disk list osd1 osd2 osd3
```

Next, erase the device partition table and contents on all OSD nodes with the zap option :

```
ceph-deploy disk zap osd1:/dev/sdb
ceph-deploy disk zap osd2:/dev/sdb
ceph-deploy disk zap osd3:/dev/sdb
```

Now we will prepare all OSD nodes :

```
ceph-deploy osd prepare osd1:/dev/sdb
ceph-deploy osd prepare osd2:/dev/sdb
ceph-deploy osd prepare osd3:/dev/sdb
```

The result of theses steps is that ``/dev/sdb`` has two partitions now:
```
    /dev/sdb1 - Ceph Data
    /dev/sdb2 - Ceph Journal
```

You can check it directly on the OSD node :

```
ssh osd1
sudo fdisk -l /dev/sdb
```


### Pushing the admin settings

Next, deploy the management-key to all associated nodes :
```
ceph-deploy admin mon1 mon2 mon3 osd1 osd2 osd3
```

Change the permission of the key file by running the command below on **all** nodes.
```
sudo chmod 644 /etc/ceph/ceph.client.admin.keyring
```


The Ceph Cluster on Ubuntu 16.04 has been created !

## Testing the Cluster

### Cluster health
Now, to test the cluster and make sure that it works as intended, you can run the following commands :

```
ssh mon1
sudo ceph health
sudo ceph -s
```

\begin{figure}[H]
    \center
    \includegraphics[width=15cm]{__EXTERNAL_ASSETS__/health.png}
    \caption{Cluster Health}
    \label{Cluster Health}
\end{figure}

### Starting over

If at any point you run into trouble and you want to start over, execute the following to purge the Ceph packages, and erase all its data and configuration:
```
ceph-deploy purge {ceph-node} [{ceph-node}]
ceph-deploy purgedata {ceph-node} [{ceph-node}]
ceph-deploy forgetkeys
rm ceph.*
```
