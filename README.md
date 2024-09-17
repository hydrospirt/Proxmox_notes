# Proxmox_notes

## Как у становить на дебиан графический интерфейс? Просто и по VNC

Обновите свой репозиторий и систему, запустив:
```bash
apt update && apt dist-upgrade
```
Я выбираю xfce4 desktop, поскольку это облегченная среда рабочего стола. Если вы предпочитаете другую, просто замените xfce4 на lxde, gnome, icewm, kde, …

```bash
apt install xfce4 chromium lightdm
Перед запуском x-windows создайте обычного пользователя
adduser новое имя пользователя
Запустите диспетчер входа в систему:
systemctl start lightdm
```

### VNC
Установите RealVNC клиент:

Скачайте и установите RealVNC клиент с официального сайта: RealVNC Download.
https://www.realvnc.com/en/connect/download/viewer/
Откройте RealVNC клиент на вашем локальном компьютере.


Установите VNC сервер для удаленного доступа к графическому интерфейсу:
```bash
apt-get install tightvncserver
```
Запустите VNC сервер и настройте пароль:
```bash
vncserver
```
VNC сервер запросит пароль для доступа к сессии.
Создайте файл конфигурации для автозапуска VNC сервера:
```bash
nano ~/.vnc/xstartup
```
Добавьте следующие строки в файл:
```nano
#!/bin/bash
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
```
Сделайте файл исполняемым:
```bash
chmod +x ~/.vnc/xstartup
```
Остановите и перезапустите VNC сервер:
```bash
vncserver -kill :1
vncserver
```
Убедитесь, что разрешение экрана настроено правильно. Вы можете изменить его в файле ~/.vnc/config
```bash
nano ~/.vnc/config
```
Добавьте или измените строки для настройки разрешения:

```nano
geometry=1280x800
depth=24
```
Установите пакет dbus-x11, который содержит dbus-launch, если его нет:
```bash
sudo apt-get update
sudo apt-get install dbus-x11
```
Перезапустите VNC сервер:
```bash
vncserver -kill :1
vncserver
```

Введите IP-адрес вашего сервера Proxmox и номер дисплея VNC (обычно :1) в приложение RealVNC или похожем:
```
your-proxmox-server-ip:1 # или 0.0.0.0:1 - где 0.0.0.0 - ip адрес выданный машрутизатором вашему серверу
```

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
wlp3s0 - ваш интерфейс wi-fi.
Находится должны в разных подсетях, если у роутера подсеть 192.168.1.1, то у вашего DCHP на сервере будет 192.168.56.1
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
