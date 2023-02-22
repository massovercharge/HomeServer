# HomeServer

This repository is my way of documenting strategies for setting up a server using Proxmox

## Configure Proxmox container for docker/portainer
Checklist:
- [ ] Disable the firewall before deploying the Proxmox container (otherwise Portainer templates will not load).
- [ ] Configure static IP, ex. 192.168.1.2/24 and gateway should be the router IP.
- [ ] If router is configured to block ads and trackers it may be a good idea to set alternate DNS servers

### Install Docker in a Proxmox 7 LXC Container
1. The recommended way to install docker is using their official install script:
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
2. Check if docker is running after install:
```
systemctl status docker
```

##### Sources:
- https://thehomelab.wiki/books/promox-ve/page/setup-and-install-docker-in-a-promox-7-lxc-conainer
- https://github.com/docker/docker-install

### Install Portainer in a Proxmox 7 LXC Container
1. Install Portainer with ports 9000 and 8000:
```
docker run -d \
--name="portainer" \
--restart on-failure \
-p 9000:9000 \
-p 8000:8000 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data \
portainer/portainer-ce:latest
```

#####
Sources:
- https://thehomelab.wiki/books/docker/page/installing-docker-and-portainer-to-use-with-the-edge-agent

## Configure WireGuard kill switch

1. Once wireguard has been installed edit the config file, it may have a different name but wg0 is common:
```
sudo nano /etc/wireguard/wg0.conf
```

2. Add the following snippet to the config:
```
PostUp  =  iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
```

3. The config file should resemble the one below after the edit:
```
[Interface]
PrivateKey = abcdefghijklmnopqrstuvwxyz0123456789=
Address = 172.x.y.z/32
DNS = 172.16.0.1
PostUp  =  iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT && ip6tables -D OUTPUT ! -o %i -m mark ! --mark $(wg show  %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
[Peer]
PublicKey = JPT1veXLmasj2uQDstX24mpR7VWD+GmV8JDkidkz91Q=
Endpoint = us-tx1.wg.ivpn.net:2049
AllowedIPs = 0.0.0.0/0
```

##### Sources:
- https://www.ivpn.net/knowledgebase/linux/linux-wireguard-kill-switch/

##### Further resources:
- https://www.linode.com/docs/guides/set-up-wireguard-vpn-on-debian/


## Get public IP from linux terminal

Use one of the following snippets to check the public ip in the terminal:
```
curl ifconfig.me
curl -4/-6 icanhazip.com
curl ipinfo.io/ip
curl api.ipify.org
curl checkip.dyndns.org
dig +short myip.opendns.com @resolver1.opendns.com
host myip.opendns.com resolver1.opendns.com
curl ident.me
curl bot.whatismyipaddress.com
curl ipecho.net/plain
```

##### Sources:
- https://opensource.com/article/18/5/how-find-ip-address-linux

## Shared storage between Proxmox containers using bind mounts

### Mount usb disk by uuid
1. List the uudi of all connected devices:
```
blkid
```
2. Copy the UUID of the disk you want to mount and insert it in a mount command:
```
mount UUID={paste uuid} /mnt/path/to/empty/folder
```
3. To have linux automatically mount the disk on reboot, add the line to the fstab file:
```
UUID={paste uuid} /mnt/path/to/empty/folder ext4 defaults 0 0
```

x. Open the config file of the container you want to add the mount point to:
```
nano /etc/pve/lxc/container-id.conf
```
y. Add a bind mount command to the LXC container config:
```
mp0: /mnt/path/to/empty/folder,mp=/mnt/folder/inside/ct
```

### Remapping uig/gid to have the same user on host and container (example of UID:GID 1000:1000)
1. Allow host to remap the udi/gid by adding the follwing two lines to the subuid and subgid files:
  - Into:
    ```
    nano /etc/subuid
    ```
  - Paste:
    ```
    root:100000:65536
    root:1000:1
    ```
  - Into:
    ```
    nano /etc/subgid
    ```
  - Paste:
    ```
    root:100000:65536
    root:1000:1
    ```
2. Configure the remapping inside the containder config file:
  - From the host, enter:
    ```
    nano /etc/pve/lxc/{container_id}.conf
    ```
  - Paste these lines into the container config file:
    ```
    lxc.idmap: u 0 100000 1000
    lxc.idmap: g 0 100000 1000
    lxc.idmap: u 1000 1000 1
    lxc.idmap: g 1000 1000 1
    lxc.idmap: u 1001 101001 64535
    lxc.idmap: g 1001 101001 64535
    ```
3. This guide was synthesized from https://proxmox-idmap-helper.nieradko.com/ using the following input:
![image](https://user-images.githubusercontent.com/26527393/220623043-825595b2-3849-4287-8bdb-69451ef49967.png)
4. Unwrapping the lxc.idmap commands:
  - "u" means user id mapping
  - "g" means group id mapping
  - first integer is the start of the range of identifiers to remap
  - second integer is the end of the range of identifiers to remap
  - third integer is the number of identifiers to remap (i.e. the size of the range)

    
    
Sources:
- [x] https://itsembedded.com/sysadmin/proxmox_bind_unprivileged_lxc/
- [ ] https://virtualizeeverything.com/2022/05/18/passing-usb-storage-drive-to-proxmox-lxc/
- [x] https://www.youtube.com/watch?v=w9X94bAm3dI
- [ ] https://www.youtube.com/watch?v=iz861jRUujw
- [x] https://proxmox-idmap-helper.nieradko.com/

## Setting up Fileserver turnkey linux setup container
Remember to set permissions of new files and folders as 775

##### Sources:
- [x] https://www.youtube.com/watch?v=UnXxJMjW4LE
