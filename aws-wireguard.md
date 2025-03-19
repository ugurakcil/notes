# WireGuard ile AWS Üzerinde VPN Sunucu ve İstemci (MacOS, Ubuntu) Kurulumu

AWS EC2 üzerinde WireGuard VPN sunucu kuruyoruz, istemcileri (Mac & Ubuntu) bağlıyoruz, IP yerine DNS kullanıyoruz, yaşadığımız hataları debug edip çözüyoruz. Kurulum sonrası güvenlik gruplarını yapılandırıyoruz, değişken ağ arayüzü isimlerini tespit ediyoruz, istemci dosyalarını doğru path’lere yerleştiriyoruz ve tüm adımları detaylandırıyoruz.

---

## **1. AWS EC2 Üzerinde WireGuard Sunucu Kurulumu**

### **1.1. AWS'de Ubuntu 22.04 EC2 Sunucu Başlat**
- **EC2 Dashboard** → **Launch Instance**
- **Ubuntu 22.04 LTS seç**
- **Minimum t4g.micro veya t3.micro önerilir**
- **Elastic IP al ve sunucuya ata (IP değişmesin)**

### **1.2. AWS Security Group Ayarları**
AWS EC2'deki güvenlik gruplarını açalım:
```bash
# Inbound Rules
Port: 22 (SSH) -> Only YOUR_IP/32
Port: 51820 (WireGuard) -> 0.0.0.0/0 (Sadece belirli IP’ler ekleyebilirsin)

# Outbound Rules
Allow ALL Traffic (Varsayılan olarak açık olmalı)
```

### **1.3. Sunucuya Bağlan ve WireGuard Kur**
```bash
ssh -i ~/.ssh/aws-ec2.pem XXXXXXXXXXXXXXXXXXXXXXX.XXXXXXXX.compute.amazonaws.com
sudo apt update && sudo apt install wireguard -y
```

### **1.4. Doğru Ağ Arayüzünü Bul**
AWS’de network interface adı **eth0 yerine farklı olabilir**. Öncelikle bunu bulalım:
```bash
ip link show
```
Örneğin, çıktı şu olabilir:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: enX0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP
```
Burada **enX0** yerine sisteminde ne varsa onu kullan.

### **1.5. WireGuard Konfigürasyonu Oluştur**
```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
cat privatekey  # Çıktıyı kopyala
cat publickey   # Çıktıyı kopyala
```

### **1.6. Sunucu İçin `wg0.conf` Dosyası**
```bash
sudo nano /etc/wireguard/wg0.conf
```
```ini
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = SUNUCUNUN_PRIVATE_KEY
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o NETWORK_INTERFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o NETWORK_INTERFACE -j MASQUERADE
```
**`NETWORK_INTERFACE` kısmına yukarıda bulduğun ismi koy.**

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show  # Kontrol
```

### **1.7. AWS NAT ve Forwarding Ayarları**
```bash
sudo iptables -t nat -A POSTROUTING -o NETWORK_INTERFACE -j MASQUERADE
sudo iptables -A FORWARD -i wg0 -o NETWORK_INTERFACE -j ACCEPT
sudo iptables -A FORWARD -i NETWORK_INTERFACE -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## **2. İstemci Kurulumu**

### **2.1. MacOS İstemci Kurulumu**
```bash
brew install wireguard-tools
mkdir -p ~/.wireguard
cd ~/.wireguard
wg genkey | tee client_privatekey | wg pubkey > client_publickey
cat client_privatekey  # Bir yere kopyala
cat client_publickey   # Bir yere kopyala
```

#### **MacOS `client.conf`**
```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = SUNUCUNUN_PUBLIC_KEY
Endpoint = XXXXXXXXXXXXXXXXXXXXXXX.XXXXXX.compute.amazonaws.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo wg-quick up ~/.wireguard/client.conf
curl ifconfig.me  # AWS IP çıkmalı
```

### **2.2. Ubuntu İstemci Kurulumu**
```bash
sudo apt update && sudo apt install wireguard -y
mkdir -p /etc/wireguard
cd /etc/wireguard
wg genkey | tee client_privatekey | wg pubkey > client_publickey
cat client_privatekey  # Kopyala
cat client_publickey   # Kopyala
```

#### **Ubuntu `wg0.conf`**
```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.3/24
DNS = 8.8.8.8

[Peer]
PublicKey = SUNUCUNUN_PUBLIC_KEY
Endpoint = XXXXXXXXXXXXXXXXXXXXXXX.XXXXXX.compute.amazonaws.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
curl ifconfig.me  # AWS IP çıkmalı
```

---

## **3. Debugging ve Sorun Giderme**

- **AWS NAT ve routing kontrolü:**
```bash
sudo iptables -t nat -L -n -v
ip route show
```
- **Mac & Ubuntu istemcilerde DNS & route kontrolü:**
```bash
ping -c 4 8.8.8.8
netstat -rn
```
- **AWS’de WireGuard trafiğini izleme:**
```bash
sudo tcpdump -i NETWORK_INTERFACE udp port 51820
```

---

## **Sonuç**

WireGuard AWS üzerinde kuruldu, istemciler bağlandı, doğru interface’ler tespit edildi, IP yerine domain kullanıldı, hata ayıklama yapıldı. **Trafik AWS üzerinden akıyorsa her şey tamam!** 🚀
