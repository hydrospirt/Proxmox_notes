# Proxmox_notes
## Как раздать Wi-Fi на proxmox для других устройств?
Для того чтобы раздать интернет через Wi-Fi для физических машин, таких как мобильный телефон, который находится вне моста, вам нужно настроить Wi-Fi адаптер на вашем хосте Proxmox в режиме "хост-моде" (hostapd) и использовать DHCP сервер для назначения IP-адресов подключенным устройствам.
У нас такие интерфейсы:
Фото

Вот шаги, которые вам нужно выполнить:
1. Установите необходимые пакеты:
Установите hostapd (для создания точки доступа) и dnsmasq (для DHCP сервера):
```bash
sudo apt update
sudo apt install hostapd dnsmasq
```
2. Настройте dnsmasq для DHCP:
Откройте файл конфигурации dnsmasq:
```bash
sudo nano /etc/dnsmasq.conf
```
Добавьте следующие строки для настройки DHCP сервера:
wlp3s0 - ваш интерфейс wi-fi
Находится должны в разных подсетях, если у роутера подсеть 192.168.1.1, то у ашего DCHP на сервере будет 192.168.56.1
```bash
interface=wlp3s0
dhcp-range=192.168.56.100,192.168.56.200,255.255.255.0,12h
```
Сохраните и закройте файл. 

3. Настройте hostapd:
Создайте файл конфигурации для hostapd:
```bash
sudo nano /etc/hostapd/hostapd.conf
```
Добавьте следующие строки (замените your_ssid и your_password на ваш SSID и пароль):
```bash
interface=wlp3s0
driver=nl80211
ssid=your_ssid
hw_mode=g
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=your_password
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
Сохраните и закройте файл.

4. Настройте сетевой интерфейс Wi-Fi:

Откройте файл /etc/network/interfaces:
```bash
sudo nano /etc/network/interfaces
```

Добавьте следующие строки для настройки Wi-Fi интерфейса в режиме точки доступа:
```nano
auto wlp3s0
iface wlp3s0 inet static
    address 192.168.56.1
    netmask 255.255.255.0
```
Сохраните и закройте файл.

5. Включите IP forwarding и NAT:

Откройте файл /etc/sysctl.conf:

```bash
sudo nano /etc/sysctl.conf
```
Раскомментируйте или добавьте строку:
```
net.ipv4.ip_forward=1
```
Примените изменения:

```bash
sudo sysctl -p
```

Настройте NAT для трафика, идущего через vmbr0(ваш мост):
```bash
sudo iptables -t nat -A POSTROUTING -o vmbr0 -j MASQUERADE
sudo iptables-save > /etc/iptables.rules
```

Добавьте правило для восстановления правил iptables при загрузке:
```bash
sudo nano /etc/rc.local
```
Добавьте следующую строку:
```
iptables-restore < /etc/iptables.rules
```
Сохраните и закройте файл.

7. Перезапустите службы:
```bash
sudo systemctl restart dnsmasq
sudo systemctl restart hostapd
sudo systemctl restart networking
```

Теперь ваш Proxmox должен раздавать интернет через Wi-Fi интерфейс. Устройства, подключенные к созданной вами Wi-Fi точке доступа, смогут выходить в интернет через ваш хост Proxmox.
