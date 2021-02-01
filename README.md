k3s_build

# Overview

Infrastructure setup and k3s cluster build.  
This reposistory builds and tears down my kubernetes k3s cluster (version v1.19.7+k3s1).

## Build Nodes 
Repeat for each node:
1. Install Ubuntu 20.04 LTS server (in the installer set the hostname, specifiy the user as **ubuntu** and select install OpenSSH Server)
2. [Configure static IP address](https://linuxize.com/post/how-to-configure-static-ip-address-on-ubuntu-20-04/) 
3. Add copy your laptop sshkey to host ``` ssh-copy-id ubuntu@host ip ```. If you haven't already created ssh key on yout laptop than ```ssh-keygen```
4. ``` sudo apt install && sudo apt upgrade ```
5. for ARM64 (RPI4) configure leagacy IP tables and cgroups (https://rancher.com/docs/k3s/latest/en/advanced/#enabling-legacy-iptables-on-raspbian-buster)
6. Add the **ubuntu** to the sudo group
7. Add copy your laptop sshkey to host ``` ssh-copy-id ubuntu@host ip ```. If you haven't already created ssh key on yout laptop than ```ssh-keygen```
8. [Configure automatic security updates](https://askubuntu.com/questions/1266548/how-to-enable-automatic-security-updates-on-ubuntu-20-04) 
9. libnss-mdns package installed (required for NAS name resoution) ```sudo apt install libnss-mdns``` 
10. nfs-Common package installed (requried to mount NFS Storage) ```sudo apt install nfs-common```  
11. Disbale swap
12. Install Docker (is use Docker instead of containerd) ```sudo apt install docker.io```

I've had a couple of attempts at automating the above see rpi4-prep.yml playbook and images folder but currenty I do the above manually. tbh i hardly ever rebuild my nodes so this automation isn't top of my priority list.

Note: there is an known [mmc0 timeout](https://askubuntu.com/questions/1151761/ubuntu-mmc0-timeout-waiting-for-hardware-cmd-interrupt-error-no-sd-card) issue with the Beelink running Ubuntu. To workaround this issue: 
```
Create the file /etc/modprobe.d/sdhci.conf and add the lines:
blacklist sdhci_acpi
blacklist sdhci

sudo update-initramfs -u
```

## Ansible prepare:
Rename the hosts.ini.sample to hosts.ini 
Edit file and update he k3s-master and k3s-worker sections with the host details for your environment
Note: the playbooks uses the user ubuntu. This user should be created on all hosts, a member of the adm and sudo groups.

### Cluster Build:
```
ansible_playbook -K main.yml
kubectl get nodes -o wide
```

### Cluster Teardown:
```
ansible_playbook -K teardown.yaml
```
To wipe the usb sticks (must be blank with no partitions before installing rook-ceph)
```
ansible_playbook -K wipedown.yml
```

notes:
- IMPORTANT: wipe-usbstick playbook all clears the partitons on device /dev/sda. To check run ```sudo fdisk -l```. If this is incorrect for your environment then edit group_vars/all.yml.
- playbooks have been known to hang (usually on rpi's), reboot them which will probably take some time and then re-run 
-

### management machine
- Ansible (2.2.9 or above)
```
sudo apt install python3
sudo apt install python3-pip
pip3 install --user ansible
ansible --version
```
- Kubectl
```
sudo curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/usr/local/bin/kubectl"
sudo chmod +x /usr/local/bin/kubectl
```

### HomeLab Environment

| Model                        | Arch  | Host Name  | IP Address   | Additional Perhiperals                           |
 --- | --- | --- | --- | --- 
| Beeklink U55 Mini PC         | AMD64 | k3s-master | 192.168.1.10 |
| PI 4B 4GB, 64GB MicroSD Card | ARM64 | k3s-node1  | 192.168.1.11 | [PoE Hat](https://thepihut.com/products/raspberry-pi-power-over-ethernet-poe-hat), 32GB USB 3.0 Stick |
| PI 4B 4GB, 64GB MicroSD Card | ARM64 | k3s-node2  | 192.168.1.12 |
| PI 4B 4GB, 64GB MicroSD Card | ARM64 | k3s-node3  | 192.168.1.13 |
| PI 4B 4GB, 64GB MicroSD Card | ARM64 | k3s-node4  | 192.168.1.14 | PoE Hat, 32GB USB 3.0 Stick, [CC2531 Zigbee2MQTT Firmware with Antenna](https://www.zigbee2mqtt.io/information/supported_adapters.html#texas-instruments-cc2531), [Zwave Gen5 USB Stick](https://aeotec.com/z-wave-usb-stick/) |
| Sonology DS218+ NAS          | n/a   | home-nas   | 192.168.1.9  |
| NETGEAR 8-Port Gigabit Ethernet PoE Network Switch |

#### Notes
My PI's are stacked on top of each other in a cluster case and i've attached 12" fan to the side of the case. This really helps with cooling an my PI's run at about 40-50 degrees centigate and thats in the room where my heating system).
The NAS provides NFS volumes for my cluster. The name (home-nas) must resolve on all nodes.

Set Host Name (Control Panel -> General)
Enable NFSv4.1 support (Control Panel -> File Services -> smb/afp/nfs)
Enable Bonjour Discovery Service (Control Panel -> File Services -> advanced)
