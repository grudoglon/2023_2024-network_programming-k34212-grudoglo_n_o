# Отчет по лабораторной работе №1 "Установка CHR и Ansible, настройка VPN"

University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Grudoglo Nikita Olegovich

Lab: Lab1

Date of create: 14.02.2024

Date of finished: ...

**Цель работы:** развертывание виртуальной машины на базе платформы Yandex Compute Cloud с установленной системой контроля конфигураций Ansible и установка CHR в VirtualBox.

**Ход работы:**

1. С помощью Yandex Compute Cloud была развернута виртуальная машина Ubuntu 22.04:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/vm.png)

2. На виртуальной машине были установлены python3 и Ansible:

```
sudo apt install python3-pip
sudo pip3 install ansible
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/python_ansible.png)

3. На локальном компьютере в VirtualBox была добавлена виртуальная машина CHR с использованием образа диска с [официального сайта](https://mikrotik.com/download):

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/chr.png)

4. Создадим свой Wireguard сервер для организации VPN туннеля между сервером автоматизаци, где был установлен Ansible, и локальным CHR.

На виртуальной машине Ubuntu установим Wireguard:

```
sudo apt-get install wireguard
```

Создаем приватный и публичный ключ сервера:

```
wg genkey | sudo tee /etc/wireguard/wg0-server-private.key | wg pubkey | sudo tee /etc/wireguard/wg0-server-public.key
```

Создаем конфигурационный файла сервера /etc/wireguard/wg0.conf:

```
[Interface]
PrivateKey = приватный ключ сервера
Address = 10.2.0.1/24
ListenPort = 51820
```

Создаем приватный и публичный ключ клиента:

```
wg genkey | sudo tee /etc/wireguard/wg0-client-private.key | wg pubkey | sudo tee /etc/wireguard/wg0-client-public.key
```

Добавляем ключ пользователя в конфиг сервера (/etc/wireguard/wg0.conf):

```
[Peer]
PublicKey = публичный ключ клиента
AllowedIPs =  10.2.0.11/32
```

Разрешим прохождение трафика между интерфейсами - в файле /etc/sysctl.conf раскомментируем строку ```net.ipv4.ip_forward = 1```.

Применим изменения:

```
sudo sysctl -p
```

Разрешим подключение к серверу на порт 51820 по протоколу UPD:

```
sudo iptables -A INPUT -i eth0 -p udp -m udp --dport 51820 -j ACCEPT
sudo ufw allow 51821/udp
```

Запускаем сервер и включаем автозагрузку:

```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/wireguard_status.png)

5. Настроим Wireguard клиента на RouterOS

Для этого подключаемся в WinBox к CHR. Здесь добавляем интерфейс wireguard1:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/wireguard_interface.png)

Интерфейсу wireguard1 задаем ip-адрес 10.2.0.11/24 (как было указано в конфиграционном файле сервера):

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/wireguard_interface_ip.png)

Настраиваем пир:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/wireguard_peer.png)

Добавляем правило в firewall - разрешаем трафик Wireguard (action - accept):

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/firewall.png)

6. Проверяем работу VPN туннеля между Wireguard сервером на Ubuntu и Wireguard клиентом на CHR:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/ping1.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/ping2.png)

7. В результате работы получаем схему связи:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab1/pictures/diagram.png)

**Вывод:** при выполнении работы была получены навыки по настройке Wireguard сервера и клиента, поднятию VPN туннеля.

