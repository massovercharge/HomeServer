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



Sources:
- [x] https://itsembedded.com/sysadmin/proxmox_bind_unprivileged_lxc/
- [ ] https://virtualizeeverything.com/2022/05/18/passing-usb-storage-drive-to-proxmox-lxc/
- [x] https://www.youtube.com/watch?v=w9X94bAm3dI
- [ ] https://www.youtube.com/watch?v=iz861jRUujw

## Setting up Fileserver turnkey linux setup container
Remember to set permissions of new files and folders as 775

##### Sources:
- [x] https://www.youtube.com/watch?v=UnXxJMjW4LE
