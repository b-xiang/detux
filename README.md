# Detux: The Multiplatform Linux Sandbox

### Introduction:
Detux is a sandbox developed to do traffic analysis of the Linux malwares and capture the IOCs by doing so. QEMU hypervisor is used to emulate Linux (Debian) for various CPU architectures.

The following CPUs are currently supported:
- x86
- x86-64
- ARM
- MIPS
- MIPSEL

Use the Live version now: http://detux.org

### What's in this release?

This release of Detux contains the script for executing a Linux binary/script in a specified CPU arch. Don't worry if you don't know what platform, it's in the script, the Magic package helps picking up the CPU arch in an automated way. x86 is the default CPU version, this can be tuned to a different one in the config file.

This release gives the analysis report in a DICT format, which can be easily customized to be inserted in to NOSQL dbs. 

An example script has been provided which demonstrates the usage of the sandbox library.

### What's in the report?
    - Static Analysis
        -- Basic strings extracted from binary
        -- ELF information generated by readelf commands
        -- the report.py can be modified to add more 3rd party commands to analyse the binary and add the result to DICT.

    - Dynamic Analysis
        -- The captured pcaps are parsed with DPKT to extract the IOC's and readable info from the packets. 

### Requirements
- System packages
    - python 2.7
    - qemu
    - pcaputils
    - sudo
    - libcap2-bin
    - bridge-utils

- Python libraries (Preferable to use virtual environment)
    - pexpect
    - paramiko 
    - python-magic 

Kindly make sure that the above requirements are met before using Detux. A few dependencies may vary from OS to OS.

### Architecture
- Host ( The host itself can be a VM or a baremetal machine)
    - QEMU
    - dumpcap
    - DETUX Scripts

### Network Arch
    - NIC1 : This interface is for accessing the Host
    - NIC2 : Interface bridged with the the QEMU Sandbox VMs. One can redirect the traffic from the interface to WHONIX or REMNUX or a custom Gateway to filter/allow internet access for the Sandboxed VMs.


### VM Setup:
##### Downloading Linux VM Images
Special thanks to aurel who has uploaded pre built QEMU Debian VM images for all possible CPU architectures.
The VM images are located at : https://people.debian.org/~aurel32/qemu/, the same link contains the command examples for invoking the vm images.

You can use the following script to automatically download the VM images to the "qemu" folder of Detux.

```
#x86
wget https://people.debian.org/~aurel32/qemu/i386/debian_wheezy_i386_standard.qcow2 -P qemu/x86/1/


#x86-64
wget https://people.debian.org/~aurel32/qemu/amd64/debian_wheezy_amd64_standard.qcow2 -P qemu/x86-64/1/

#arm
wget https://people.debian.org/~aurel32/qemu/armel/debian_wheezy_armel_standard.qcow2 -P qemu/arm/1/
wget https://people.debian.org/~aurel32/qemu/armel/initrd.img-3.2.0-4-versatile -P qemu/arm/1/
wget https://people.debian.org/~aurel32/qemu/armel/vmlinuz-3.2.0-4-versatile -P qemu/arm/1/

#mips
wget https://people.debian.org/~aurel32/qemu/mips/vmlinux-3.2.0-4-4kc-malta -P qemu/mips/1/
wget https://people.debian.org/~aurel32/qemu/mips/debian_wheezy_mips_standard.qcow2 -P qemu/mips/1/

#mipsel
wget https://people.debian.org/~aurel32/qemu/mipsel/vmlinux-3.2.0-4-4kc-malta -P qemu/mipsel/1/
wget https://people.debian.org/~aurel32/qemu/mipsel/debian_wheezy_mipsel_standard.qcow2 -P qemu/mipsel/1/
```
##### Setting sudoers for qemu execution
Detux uses SSH to communicate with the VMs and so, this is currently required for the VMs to have networking capability. Considering that the listed binaries are in the same path, you may add the following lines to to /etc/sudoers (only if you are a non-root user):

```
Cmnd_Alias  QEMU_CMD    =   /usr/bin/qemu-*, /sbin/ip, /sbin/ifconfig, /sbin/brctl
<your detux username here> ALL = (ALL) NOPASSWD: QEMU_CMD
```
Change the paths to the binaries if they differ for you.

##### Network setup
Add the following config to /etc/qemu-ifup, backup the original if you already have one:
```
#! /bin/sh
# Script to bring a network (tap) device for qemu up.
# The idea is to add the tap device to the same bridge
# as we have default routing to.

# in order to be able to find brctl
PATH=$PATH:/sbin:/usr/sbin
ip=$(which ip)
ifconfig=$(which ifconfig)

echo "Starting"  $1
if [ -n "$ip" ]; then
   ip link set "$1" up
else
   brctl=$(which brctl)
   if [ ! "$ip" -o ! "$brctl" ]; then
     echo "W: $0: not doing any bridge processing: neither ip nor brctl utility not found" >&2
     exit 0
   fi
   ifconfig "$1" 0.0.0.0 up
fi

switch=$(ip route ls | \
    awk '/^default / {
          for(i=0;i<NF;i++) { if ($i == "dev") { print $(i+1); next; } }
         }'
        )
    if [ -d /sys/class/net/br0/bridge/. ]; then
        if [ -n "$ip" ]; then
          ip link set "$1" master br0
        else
          brctl addif br0 "$1"
        fi
        exit    # exit with status of the previous command
    fi

echo "W: $0: no bridge for guest interface found" >&2
```

Considering that eth0 is the interface you want your VMs to be bridged with, you may remove the configs for eth0 and use the following configs in /etc/network/interfaces:
```
auto br0
iface br0 inet dhcp
  bridge_ports eth0
  bridge_maxwait 0
```
You can also specify a static address you used for eth0.

##### Setting up your VMs
Traverse to the folder in which your VM images are located for each QEMU Images e.g. for ARM is :
```
<your detux folder>/qemu/arm/1/
````
For each image, follow the VM boot instructions given at "https://people.debian.org/~aurel32/", to start the VM. However, if you are a non-root user, you will have to use sudo. 

Comands for Booting the VMs (Replace <MACADDR> with the MAC you desire):
```
#x86
sudo qemu-system-i386 -hda qemu/x86/1/debian_wheezy_i386_standard.qcow2 -vnc 127.0.0.1:5901 -net nic,macaddr=<MACADDR> -net tap -monitor stdio


#x86-64
sudo qemu-system-x86_64 -hda qemu/x86-64/1/debian_wheezy_amd64_standard.qcow2 -vnc 127.0.0.1:5901 -net nic,macaddr=<MACADDR> -net tap -monitor stdio

#arm
sudo qemu-system-arm -M versatilepb -kernel qemu/arm/1/vmlinuz-3.2.0-4-versatile -initrd qemu/arm/1/initrd.img-3.2.0-4-versatile -hda qemu/arm/1/debian_wheezy_armel_standard.qcow2 -append "root=/dev/sda1" -vnc 127.0.0.1:5901 -net nic,macaddr=<MACADDR> -net tap -monitor stdio

#mips
sudo qemu-system-mips -M malta -kernel qemu/mips/1/vmlinux-3.2.0-4-4kc-malta -hda qemu/mips/1/debian_wheezy_mips_standard.qcow2 -append "root=/dev/sda1 console=tty0" -vnc 127.0.0.1:5901 -net nic,macaddr=<MACADDR> -net tap -monitor stdio

#mipsel
sudo qemu-system-mipsel -M malta -kernel qemu/mipsel/1/vmlinux-3.2.0-4-4kc-malta -hda qemu/mipsel/1/debian_wheezy_mipsel_standard.qcow2 -append "root=/dev/sda1 console=tty0" -vnc 127.0.0.1:5901 -net nic,macaddr=<MACADDR> -net tap -monitor stdio
```

Detux requires a preconfigured VM snapshot with IP addresses and ssh setup.

###### Steps for setting up your snapshot:
- Choose an unconfigured VM image and start it using the above listed command in a Terminal.
- Connect VM monitor. Connect a VNC client to 127.0.0.1:5901 and wait for the VM to boot completely.
- Login with the default root credentials (root/root). 
- Configure the VM's network interface such that it reachable/ accessible to the host.
- Setup SSH server on the VM and anyother configuration if required for you. 
- Once configured, boot to a running state that accepts network connection.
- Switch back to terminal with qemu console on, which should look like:
```
(qemu)
```
- Save the VM state typing the following qemu commands in the qemu console:
```
(qemu) savevm init
```
- Quit the QEMU console:
```
(qemu) q
```
-- Repeat through step 1 for all the VMs


##### Setting capture permisisons
In order for a non-root user to be able to capture packets, Dumpcap needs capture privileges.  (https://wiki.wireshark.org/CaptureSetup/CapturePrivileges).

The following command may enable the same for you:
```
sudo groupadd -g wireshark
sudo usermod -a -G wireshark <your user name>
sudo chmod 750 /usr/bin/dumpcap
sudo etcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
```

If you are unable to capture packets, you need to check if permisions for your users are set correctly and the dumpcap path is right.

### Detux Config
The detux.cfg files in the main directory needs to be configured. Each VM section has to be configured with correct Network Params and SSH credentials. You can choose root/non-root user depending on your need.


### Running Detux
The Detux library located in "core" directory can be used to suit your need of analysis.
The repo contains "detux.py" which analyses the given binary and saves pcap in pcap folder and writes JSON output to a specified filepath.

### Usage
```
usage: detux.py [-h] --sample SAMPLE [--cpu {x86,x86-64,arm,mips,mipsel}]
                [--int {python,perl,sh,bash}] --report REPORT

optional arguments:
  -h, --help            show this help message and exit
  --sample SAMPLE       Sample path (default: None)
  --cpu {x86,x86-64,arm,mips,mipsel}
                        CPU type (default: auto)
  --int {python,perl,sh,bash}
                        Architecture type (default: None)
  --report REPORT       JSON report output path (default: None)
```
Example:
```
python detux.py --sample test_script/example_binary1 --report reports/example_report1.json
```

### Contributers:
- Vikas Iyengar - The brain ( [dudeintheshell](https://github.com/dudeintheshell) , email: iyengar.vikas@gmail.com ) 
- Muslim Koser - Technichal thought works (muslimk@gmail.com)
- Rahul Binjve - Help in pcap parsing ([@c0dist](https://github.com/c0dist), [twitter](https://twitter.com/c0dist))
- Amey Gat - Help in pcap parsing ([ameygat](https://github.com/ameygat), ameygat@gmail.com )

### Thanks
Thanks to Aurélien Jarno (@aurel32) (https://www.aurel32.net/) for the pre-built VM images.
