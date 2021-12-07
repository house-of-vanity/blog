# qBittorent via VPN in Linux  network namespace

## Prerequisites
- Ubuntu 20.04
- Wireguard clinet config. **/etc/wireguard/torrent.conf** in this guide.
- qBittorrent v4.3.9+

## Steps
- Script for setting up namespace and run qbittorent-nox into it. Change user in last line to desired on.
	```bash
	#!/bin/bash
	ip netns del torrent
	sleep 2
	ip netns add torrent
	ip link add veth0 type veth peer name veth1 
	ip link set veth1 netns torrent
	ip address add 10.99.99.1/24 dev veth0
	ip netns exec torrent ip address add 10.99.99.2/24 dev veth1
	ip link set dev veth0 up
	ip netns exec torrent ip link set dev veth1 up
	ip netns exec torrent ip route add default via 10.99.99.1
	mkdir -p /etc/netns/torrent
	echo nameserver 8.8.8.8 > /etc/netns/torrent/resolv.conf
	ip netns exec torrent wg-quick up torrent
	sleep 5
	ip netns exec torrent sudo -u ab qbittorrent-nox
	```

- Systemd service unit

		# /etc/systemd/system/qbittorrent.service
		[Unit]
		Description=qBittorrent via vpn
		After=network.target
		StartLimitIntervalSec=0

		[Service]
		Type=simple
		Restart=always
		RestartSec=1
		User=root
		ExecStart=/usr/bin/torrent_ns
		ExecStop=/usr/bin/ip netns del torrent

		[Install]
		WantedBy=multi-user.target

- NGINX proxy config example.

		server {
			listen 443 ssl http2;
			server_name tr.hexor.ru;
			include ssl.conf;
			location / {
				proxy_pass http://10.99.99.2:8080;
				proxy_set_header Host $host;
				proxy_set_header X-Real-IP $remote_addr;
				proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
				proxy_set_header X-Forwarded-Proto $scheme;
				proxy_hide_header   Referer;
				proxy_hide_header   Origin;
				proxy_set_header    Referer                 '';
				proxy_set_header    Origin                  '';
			}
		}
		server {
			listen 80;
			server_name tr.hexor.ru;
			listen [::]:80;
			return 302 https://$host$request_uri;
		}

