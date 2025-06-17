# Actualizar Ubuntu
```
sudo apt update && sudo apt upgrade -y
```
# Servidor DNS – BIND9
```
sudo apt install bind9 dnsutils
```
```
sudo nano /etc/bind/named.conf.options
```
```
options {
    directory "/var/cache/bind";
-   // forwarders {
-   //     0.0.0.0;
-   // };
+   forwarders {
+       8.8.8.8;
+       8.8.4.4;
+   };
    // ...
};
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
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};
```
```
cd /etc/bind
sudo cp db.local db.redes.uca        # Copiar plantilla para zona directa
sudo cp db.127  db.192.168.1        # Copiar plantilla para zona inversa
```
```
sudo nano /etc/bind/db.redes.uca
```
```
;
; BIND data file for redes.uca
;
$TTL    604800
@       IN      SOA     redes.uca. admin.redes.uca. (
                              2         ; Serial (incrementar en cambios)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.redes.uca.
@       IN      A       192.168.1.10
ns      IN      A       192.168.1.10
```
```
sudo nano /etc/bind/db.192.168.1
```
```
;
; BIND reverse data file for local 192.168.1.0/24 net
;
$TTL    604800
@       IN      SOA     ns.redes.uca. admin.redes.uca. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.redes.uca.
10      IN      PTR     ns.redes.uca.
```
```
sudo systemctl restart bind9
```
# Servidor DHCP – ISC Kea
```
sudo apt install kea
```
```
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.original
sudo chmod a-w /etc/kea/kea-dhcp4.conf.original   # proteger copia contra escritura
```
```
sudo nano /etc/kea/kea-dhcp4.conf
```
```
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s8" ]
    },
    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.1.0/24",
        "pools": [
          { "pool": "192.168.1.100 - 192.168.1.200" }
        ],
        "option-data": [
          { "name": "routers", "data": "192.168.1.1" },
          { "name": "domain-name-servers", "data": "192.168.1.10" },
          { "name": "domain-name", "data": "redes.uca" }
        ]
      }
    ]
  }
}
```
```
sudo systemctl restart kea-dhcp4-server
```
# Servidor Web – Nginx
```
sudo apt install nginx
```
```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/intranet.redes.uca
sudo nano /etc/nginx/sites-available/intranet.redes.uca
```
En el nuevo archivo, edite la directiva server_name para que diga server_name intranet.redes.uca; y ajuste la raíz de documentos si lo desea (por ejemplo a /var/www/intranet). Guarde el archivo y luego active el sitio creando un enlace simbólico
```
sudo ln -s /etc/nginx/sites-available/intranet.redes.uca /etc/nginx/sites-enabled/
```
```
sudo nginx -t   # comprobar que la config no tiene errores
```
```
sudo systemctl reload nginx
```
```
sudo systemctl stop|start|restart nginx
```
# Servidor Proxy – Squid
```
sudo apt install squid
```
```
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.original  
sudo chmod a-w /etc/squid/squid.conf.original 
```
```
acl lan src 192.168.1.0/24
```
```
http_access allow lan
http_access allow localhost
```
```
sudo systemctl restart squid
```
# Servidor VPN – OpenVPN
```
sudo apt install openvpn easy-rsa
```
```
sudo make-cadir /etc/openvpn/easy-rsa
cd /etc/openvpn/easy-rsa
```
```
sudo -s         # cambiar a root para facilidad (o use sudo antes de cada ./easyrsa)
cd /etc/openvpn/easy-rsa
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
openvpn --genkey --secret /etc/openvpn/ta.key
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
sudo gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
```
```
ca ca.crt
cert server1.crt
key server1.key
dh dh.pem
```
```
push "route 192.168.1.0 255.255.255.0"
```
```
nano /etc/sysctl.conf
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