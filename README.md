# Actualizar Ubuntu
```
sudo apt update && sudo apt upgrade -y
```
# Servidor DNS – BIND9
```
sudo apt install bind9 dnsutils -y
```
```
sudo nano /etc/bind/named.conf.options
```
```
options {
        directory "/var/cache/bind";
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
        dnssec-validation auto;
        auth-nxdomain no;
        listen-on-v6 { any; };
};
```
```
sudo su
```
```
nano /etc/bind/named.conf.local
```
```
// Zona directa (forward zone)
zone "redes.uca" {
    type master;
    file "/etc/bind/db.redes.uca";
};
// Zona inversa (reverse zone)
zone "0.0.127.in-addr.arpa" {
    type master;
    file "/etc/bind/db.127.0.0";
};
```
```
exit
```
```
cd /etc/bind
sudo cp db.local db.redes.uca
sudo cp db.127  db.127.0.0
```
```
sudo nano /etc/bind/db.redes.uca
```
```
;
; BIND data file for redes.uca
;
$TTL    604800
@       IN      SOA     redes.uca. root.redes.uca. (
                              2         ; Serial (incrementar en cambios)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      redes.uca.
@       IN      A       127.0.0.1
mail    IN      A       127.0.0.2
www     IN      A       127.0.0.3
```
```
sudo nano /etc/bind/db.127.0.0
```
```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     redes.uca. root.redes.uca. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      redes.uca.
1       IN      PTR     redes.uca.
2       IN      PTR     mail.redes.uca.
3       IN      PTR     www.redes.uca.
```
```
cd ..
sudo su
nano resolv.conf
```
```
nameserver 127.0.0.1
options edns0
search redes.uca
```
```
sudo systemctl restart bind9
```
```
sudo systemctl status bind9
```
```
nslookup
```
```
redes.uca
mail.redes.uca
www.redes.uca
```
# Servidor DHCP – ISC Kea
```
sudo apt install kea -y
```
```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.original
sudo chmod a-w /etc/kea/kea-dhcp4.conf.original
```
```
sudo nano /etc/kea/kea-dhcp4.conf.final
```
```
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [
        "ens33"
      ]
    },
    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.0.0/24",
        "pools": [
          {
            "pool": "192.168.0.100 - 192.168.0.200"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.0.1"
          },
          {
            "name": "domain-name-servers",
            "data": "192.168.0.10"
          },
          {
            "name": "domain-name",
            "data": "redes.uca"
          }
        ]
      }
    ],
    "expired-leases-processing": {
      "reclaim-timer-wait-time": 10,
      "flush-reclaimed-timer-wait-time": 25,
      "hold-reclaimed-time": 3600,
      "max-reclaim-leases": 100,
      "max-reclaim-time": 250,
      "unwarned-reclaim-cycles": 5
    },
    "renew-timer": 900,
    "rebind-timer": 1800,
    "valid-lifetime": 3600,
    "loggers": [
      {
        "name": "kea-dhcp4",
        "output_options": [
          {
            "output": "stdout",
            "pattern": "%-5p %m\n"
          }
        ],
        "severity": "INFO",
        "debuglevel": 0
      }
    ]
  }
}
```
```
sudo cp /etc/kea/kea-dhcp4.conf.final /etc/kea/kea-dhcp4.conf
```
```
sudo systemctl restart kea-dhcp4-server
```
```
sudo systemctl status kea-dhcp4-server
```
# Servidor Web – Nginx
```
sudo apt install nginx -y
```
```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/intranet.redes.uca
sudo nano /etc/nginx/sites-available/intranet.redes.uca
```
```
server {
	listen 80;
	listen [::]:80;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;
	server_name intranet.redes.uca;
	location / {
		try_files $uri $uri/ =404;
	}
}
```
```
sudo ln -s /etc/nginx/sites-available/intranet.redes.uca /etc/nginx/sites-enabled/
```
```
sudo nginx -t
```
```
sudo systemctl reload nginx
```
```
sudo systemctl status nginx
```
# Servidor Proxy – Squid
```
sudo apt install squid -y
```
```
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.original
sudo chmod a-w /etc/squid/squid.conf.original
```
```
sudo nano /etc/squid/squid.conf
```
```
visible_hostname proxy-redes
acl lan src 192.168.1.0/24
http_access allow lan
http_access allow localhost
http_access deny all
```
```
sudo systemctl restart squid
```
```
sudo systemctl status squid
```
# Servidor VPN – OpenVPN
```
sudo apt install openvpn easy-rsa -y
```
```
sudo make-cadir /etc/openvpn/easy-rsa
```
```
sudo su
```
```
cd /etc/openvpn/easy-rsa
```
```
./easyrsa init-pki
./easyrsa build-ca
```
```
./easyrsa gen-req server1 nopass
```
```
./easyrsa sign-req server server1
```
```
./easyrsa gen-dh
```
```
openvpn --genkey secret /etc/openvpn/ta.key
```
```
cp pki/ca.crt pki/issued/server1.crt pki/private/server1.key pki/dh.pem /etc/openvpn/
```
```
cd /etc/openvpn/easy-rsa
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```
```
exit
```
```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server.conf
```
```
sudo nano /etc/openvpn/server.conf
```
```
ca ca.crt
cert server1.crt
key server1.key
dh dh.pem
tls-auth ta.key 0
key-direction 0
push "route 192.168.1.0 255.255.255.0"
client-to-client
```
```
sudo nano /etc/sysctl.conf
```
```
net.ipv4.ip_forward=1
```
```
sudo sysctl -p /etc/sysctl.conf
```
```
sudo systemctl start openvpn@server
```
```
sudo systemctl enable openvpn@server
```