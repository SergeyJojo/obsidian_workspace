
### **Как поднять свой VPN-сервер (на примере WireGuard и OpenVPN)**  

Собственный VPN-сервер позволяет:  
🔒 **Безопасно подключаться к интернету** из любой точки мира.  
🌐 **Обходить блокировки** (если разрешено законом).  
🏠 **Доступ к домашней сети** удалённо.  

Рассмотрим два популярных решения: **WireGuard** (быстрый и современный) и **OpenVPN** (классический, с TLS).  

---

## **1. WireGuard (Рекомендуемый вариант)**
WireGuard — это **лёгкий и быстрый** VPN с криптографией нового поколения.  

### **1.1 Установка на сервер (Ubuntu/Debian)**
```bash
# Установка WireGuard
sudo apt update
sudo apt install wireguard resolvconf

# Генерация ключей
umask 077
wg genkey | tee /etc/wireguard/privatekey | wg pubkey | tee /etc/wireguard/publickey

# Настройка конфига сервера
sudo nano /etc/wireguard/wg0.conf
```
**Пример конфига (`/etc/wireguard/wg0.conf`)**:
```ini
[Interface]
PrivateKey = <СЕРВЕРНЫЙ_ПРИВАТНЫЙ_КЛЮЧ>
Address = 10.0.0.1/24  # Внутренняя подсеть VPN
ListenPort = 51820      # Порт для подключения
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

### **1.2 Включение IP-форвардинга**
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### **1.3 Запуск WireGuard**
```bash
sudo systemctl enable --now wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

### **1.4 Настройка клиента**
Создайте конфиг для клиента (например, `client.conf`):
```ini
[Interface]
PrivateKey = <ПРИВАТНЫЙ_КЛЮЧ_КЛИЕНТА>
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <СЕРВЕРНЫЙ_ПУБЛИЧНЫЙ_КЛЮЧ>
Endpoint = <IP_СЕРВЕРА>:51820
AllowedIPs = 0.0.0.0/0  # Весь трафик через VPN (или 10.0.0.0/24 для локального доступа)
PersistentKeepalive = 25
```
- **На Android/iOS** используйте приложение **WireGuard**.  
- **На Windows/Linux** — официальный клиент.  

---

## **2. OpenVPN (Альтернативный вариант)**
OpenVPN — более старый, но проверенный вариант с поддержкой TLS.  

### **2.1 Установка на сервер**
```bash
sudo apt update
sudo apt install openvpn easy-rsa

# Генерация сертификатов
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key

# Копируем сертификаты
sudo cp ~/openvpn-ca/pki/{ca.crt,issued/server.crt,private/server.key,dh.pem,ta.key} /etc/openvpn/server/
```

### **2.2 Настройка сервера**
```bash
sudo nano /etc/openvpn/server/server.conf
```
**Пример конфига**:
```ini
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
keepalive 10 120
tls-auth ta.key 0
cipher AES-256-GCM
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
```

### **2.3 Запуск OpenVPN**
```bash
sudo systemctl enable --now openvpn-server@server
sudo systemctl status openvpn-server@server
```

### **2.4 Создание клиентских конфигов**
```bash
cd ~/openvpn-ca
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1

# Создаём client.ovpn
cat > client.ovpn <<EOF
client
dev tun
proto udp
remote <IP_СЕРВЕРА> 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-GCM
verb 3
<ca>
$(cat ~/openvpn-ca/pki/ca.crt)
</ca>
<cert>
$(cat ~/openvpn-ca/pki/issued/client1.crt)
</cert>
<key>
$(cat ~/openvpn-ca/pki/private/client1.key)
</key>
<tls-auth>
$(cat ~/openvpn-ca/ta.key)
</tls-auth>
key-direction 1
EOF
```
- **На Android/iOS** используйте **OpenVPN Connect**.  
- **На ПК** — официальный клиент.  

---

## **3. Настройка фаервола (UFW)**
```bash
sudo ufw allow 51820/udp  # Для WireGuard
sudo ufw allow 1194/udp   # Для OpenVPN
sudo ufw enable
```

---

## **4. Что выбрать?**
| **Критерий**       | **WireGuard**                     | **OpenVPN**                     |
|--------------------|----------------------------------|--------------------------------|
| **Скорость**       | ⚡ Быстрее (на уровне ядра)      | 🐢 Медленнее (TLS-шифрование) |
| **Настройка**      | 🛠 Проще (минимум конфигов)      | 🔧 Сложнее (сертификаты)       |
| **Безопасность**   | 🔒 Современная криптография      | 🔐 Проверенный, но медленнее   |
| **Поддержка**      | Linux, Windows, macOS, Android   | Все ОС, но требует клиент      |

---

## **5. Где развернуть VPN-сервер?**
1. **VPS (рекомендуется)**  
   - [DigitalOcean](https://www.digitalocean.com/) ($5/мес)  
   - [Linode](https://www.linode.com/) ($5/мес)  
   - [Hetzner](https://www.hetzner.com/) (€4.50/мес)  

2. **Домашний сервер (Raspberry Pi / ПК)**  
   - Требуется настройка проброса портов (NAT).  
   - Может быть медленнее, чем VPS.  

---

## **Вывод**
- **WireGuard** — лучший выбор для скорости и простоты.  
- **OpenVPN** — если нужна максимальная совместимость.  
- **VPS** — самый удобный вариант для 24/7 работы.  

После настройки проверьте VPN через:  
```bash
curl ifconfig.me  # Должен показать IP сервера, а не ваш.
```

### **Установка и настройка WireGuard клиента на Ubuntu**  

WireGuard — это быстрый и безопасный VPN. Вот как установить и настроить его на Ubuntu (22.04 / 20.04).  

---

## **1. Установка WireGuard**  
Откройте терминал и выполните:  

```bash
sudo apt update
sudo apt install wireguard resolvconf
```

---

## **2. Создание ключей (если нет конфига от сервера)**  
Если у вас уже есть `.conf`-файл от администратора VPN, переходите к шагу 3.  

```bash
# Генерация ключей
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

- `privatekey` — ваш приватный ключ (никому не передавать).  
- `publickey` — публичный ключ (нужно отправить администратору VPN).  

---

## **3. Настройка клиента**  
Создайте конфигурационный файл:  

```bash
sudo nano /etc/wireguard/wg0.conf
```

**Пример конфига (подставьте свои данные):**  
```ini
[Interface]
PrivateKey = <ВАШ_ПРИВАТНЫЙ_КЛЮЧ>
Address = 10.0.0.2/24  # IP клиента в VPN-сети
DNS = 8.8.8.8          # Альтернативно: 1.1.1.1

[Peer]
PublicKey = <ПУБЛИЧНЫЙ_КЛЮЧ_СЕРВЕРА>
Endpoint = <IP_СЕРВЕРА>:51820  # Пример: 123.123.123.123:51820
AllowedIPs = 0.0.0.0/0         # Весь трафик через VPN (или 10.0.0.0/24 для локального доступа)
PersistentKeepalive = 25
```

🔹 **Где взять данные?**  
- `PrivateKey` — содержимое файла `privatekey` (из шага 2).  
- `PublicKey` и `Endpoint` — должен предоставить администратор VPN.  

---

## **4. Запуск WireGuard**  
```bash
sudo wg-quick up wg0
```

**Проверка статуса:**  
```bash
sudo wg show
```
Вывод должен содержать `transfer` (переданные данные) и `latest handshake`.  

**Автозапуск при загрузке:**  
```bash
sudo systemctl enable wg-quick@wg0
```

---

## **5. Отключение VPN**  
```bash
sudo wg-quick down wg0
```

---

## **6. Графический интерфейс (опционально)**  
Если нужен GUI, можно установить **NetworkManager-плагин**:  

```bash
sudo apt install network-manager-wireguard
```
После этого WireGuard появится в настройках сети:  
`Настройки → Сеть → VPN → Добавить WireGuard`.  

---

## **7. Проверка работы**  
1. **Проверьте IP:**  
   ```bash
   curl ifconfig.me
   ```
   Должен отображаться IP сервера, а не ваш.  

2. **Проверьте DNS:**  
   ```bash
   nslookup google.com
   ```
   Должен работать через VPN.  

---

## **8. Возможные проблемы**  
### **❌ "Could not resolve host" (DNS не работает)**  
Попробуйте:  
```bash
sudo nano /etc/wireguard/wg0.conf
```
И добавьте:  
```ini
DNS = 8.8.8.8, 1.1.1.1
```
Затем перезапустите VPN:  
```bash
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

### **❌ "Permission denied" при запуске**  
Проверьте права:  
```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

---

## **Вывод**  
- Установка: `sudo apt install wireguard`.  
- Конфиг: `/etc/wireguard/wg0.conf`.  
- Запуск: `sudo wg-quick up wg0`.  
- Автозапуск: `sudo systemctl enable wg-quick@wg0`.  

Теперь ваш Ubuntu защищён через WireGuard! 🚀