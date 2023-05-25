## Obecné poznámky

- Pokud se zasekne `apt`, tak použít 
```sh
sudo pkill -9 -f apt
```
- Pro instalaci pak používat
```sh
apt-get install --no-install-recommends <balíčky>
```

## Nastavení SSH klienta a serveru

1. Vytvořit na klientovi nový certifikát
```sh
ssh-keygen -t rsa -b 4096 -C "jindra@spos"
```
- `rsa` - šifra
- `b` - délka
- `C` - komentář
2. Na serveru klíč přidat _pub_ do `~/.ssh/authorized_keys`
3. Přidat _private_ klíč na klientovi do `.ssh/`
4. `chmod 600` na klíč
5. Na __klientovi__ nastavit `.ssh/config`. Upravit jako:
```bash
Host bsa
    HostName <IP>
    User root
    IdentityFile ~/.ssh/id_rsa_bsa
    Port 22
```
6. Základní nastavení serveru
- V `/etc/ssh/sshd_config`
```sh
AllowUsers root # Seznam povolených uživatelů
PasswordAuthentication no # Autentizace pomocí hesla
PermitRootLogin without-password # Root se může přihlásit pouze bez hesla
```
7. Restartovat SSH `systemctl restart sshd`
7. Zkusit přihlášení z klienta
8. Odšifrovat certifikát na straně klienta
9. Nainstalovat _fail2ban_
```sh
apt install fail2ban
```

## LVM

1. Nainstalovat LVM a cryptsetup
```
apt install lvm2 cryptsetup
```
2. Vytvořit physical volume pomocí 2 přítomných disků
```sh
pvcreate /dev/sdb /deb/sdc
```
3. Udělat virtuální skupiny
```sh
vgcreate <název> /dev/sdb /dev/sdc
```
4. Udělat logical volume
```sh
lvcreate -L <velikost>GB -n <jméno encryptovaného> <jméno VG>
```
5. Zašifrovat
```sh
cryptsetup -y -v- luksFormat /dev/<vg>/<encrypted>
```
6. Odšifrovat do svazku
```sh
cryptsetup luksOpen /dev/<vg>/<encrypted> <decrypted>
```
- Vytvoří se svazek `/dev/<vg>/<decrypted>`
7. Vytvoříme file systém 
```sh
mkfs.<fs> /dev/mapper/decrypted
```
8. Mounting
```sh
mount /dev/mapper/decrypted /mnt/<mount>
```

## CA 

1. Nainstalovat easy-rsa
2. Udělat složku jako `/etc/ca`
3. Zkopírovat `/usr/share/easy-rsa/*` do `/etc/ca`
4. V `/etc/ca`: `vars.example` -> `vars` a změnit v něm:
```sh
set_var EASYRSA_REQ_COUNTRY "CS"
set_var EASYRSA_REQ_PROVINCE "Pilsen"
set_var EASYRSA_REQ_CITY "Pilsen"
set_var EASYRSA_REQ_ORG "BSA"
set_var EASYRSA_REQ_EMAIL "<orion_login>@bsa"
set_var EASYRSA_REQ_OU "BSA"
```
5. Spustit `./easyrsa init-pki`
6. Spustit `./easyrsa build-ca`
- Nastavit heslo
- Common name jako `<orion-login>-bsa`
- CA cert najdeme v `/etc/ca/pki/ca.crt`
7. Udělat certifikáty
```sh
./easyesa gen-req private.mhorinek.bsa # Vytvoří request
./easyrsa sign server private.mhorinek.bsa # Podepíše request
```
8. Odheslovat klíče (.key)
```sh
mv key key.bak # Aby pak šel vytvořit správný název
openssl rsa -in <in soubor> -out <out soubor>
```

## NGINX

1. Nainstalovat
2. V `/etc/nginx/sites-enabled` default přejmenovat
3. Přidat tam config
```sh
listen 443 ssl;
listen [::]:443 ssl;
ssl_certificate <cesta k .cert>;
ssl_certificate_key <cesta k .key>;
server_name public.mhorinek.bsa;
```
```sh
listen 80;
listen [::]:80;
server_name private.mhorinek.bsa;
```

## STUNNEL

1. Nainstalovat `stunnel4`
2. V `/etc/stunnel/` vytvořit `stunnel.conf` a do něj:
```sh
[https]
accept = 127.0.0.1:8443
connect = 127.0.0.1:80
cert = /etc/ca/pki/issued/private.mhorinek.bsa.crt
key = /etc/ca/pki/private/private.mhorinek.bsa.key
```
_Mezery tam jsou asi důležité_ 

3. Restartovat stunnel4
```sh
service stunnel4 restart
```
4. Otestovat
```sh
openssl s_client -connect 127.0.0.1:8443
curl -k https://localhost:8443
```

## OpenVPN

1. Nainstalovat přes apt
2. V CA 
```sh
./easyrsa gen-req vpn.mhorinek.bsa
./easyrsa sign server vpn.mhorinek.bsa # Pak ještě odkódovat
./easyrsa gen-dh
mkdir keys
openvpn --genkey --secret keys/ta.key
cp pki/ca.crt pki/issued/vpn.mhorinek.bsa.crt pki/private/vpn.mhorinek.bsa.key pki/dh.pem /etc/openvpn
cp keys/ta.key /etc/openvpn/
```
3. V `/etc/openvpn` vytvořit soubor `server.conf` a do něj:
```sh
port 1194
proto udp
dev tun
ca ca.crt
cert vpn.mhorinek.bsa.crt
key vpn.mhorinek.bsa.key
dh dh.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
tls-auth ta.key 0
comp-lzo
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
```
4. Spustit server 
```sh
systemctl start openvpn@server
```
5. Na straně klienta vytvořit soubor `client.conf`:
```sh
client
dev tun
proto udp
remote sulis216.zcu.cz 1194
remote-cert-tls server
nobind
persist-key
persist-tun
comp-lzo
verb 3
tls-auth ta.key 1

<ca>
# Sem vlořit CA certifikát
</ca>

<cert>
# Sem certifikát
</cert>

<key>
# Tady private key
</key>
```
_Potřebuješ na klienta dostat i ten ta.key_