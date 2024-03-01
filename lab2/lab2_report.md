# Отчет по лабораторной работе №2 "Развертывание дополнительного CHR, первый сценарий Ansible"

University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Grudoglo Nikita Olegovich

Lab: Lab2

Date of create: 17.02.2024

Date of finished: ...

**Цель работы:** С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.

**Ход работы:**

1. На локальном компьютере в VirtualBox была добавлена вторая виртуальная машина CHR:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/chrs.png)

2. На chr2 организуем второй Wireguard клиент

На Wireguard сервере создаем еще один приватный и публичный ключ клиента:

```
wg genkey | sudo tee /etc/wireguard/wg0-client-private2.key | wg pubkey | sudo tee /etc/wireguard/wg0-client-public2.key
```

Добавляем еще один пир в конфиг сервера (/etc/wireguard/wg0.conf):

```
[Peer]
PublicKey = публичный ключ 2 клиента
AllowedIPs =  10.2.0.12/32
```

Перезапускаем службу Wireguard:

```
sudo systemctl restart wg-quick@wg0
```

подключаемся в WinBox к chr2. Добавляем интерфейс wireguard1:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/wireguard_interface.png)

Интерфейсу wireguard1 задаем ip-адрес:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/wireguard_interface_ip.png)

Настраиваем пир:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/wireguard_peer.png)

Добавляем правило в firewall - разрешаем трафик Wireguard (action - accept):

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/firewall.png)

Проверяем работу VPN туннеля между Wireguard сервером и Wireguard клиентом на chr2:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/ping1.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/ping2.png)

3. Используя Ansible, настроим сразу на 2 CHR: логин/пароль, NTP Client, OSPF с указанием Router ID, сбор данных по OSPF топологии и полный конфиг устройства.

Cоздаем inventory файл c информацией о chr1 и chr2 и переменными:

```
[chrs]
chr1 ansible_host=10.2.0.11 ip=1.1.1.1
chr2 ansible_host=10.2.0.12 ip=2.2.2.2

[chrs:vars]
ansible_user=admin
ansible_password=admin
ansible_connection=network_cli
ansible_network_os=routeros
```

Создаем playbook - yaml файл с инструкциями для выполнения на chr1 и chr2:

```
---
- name: lab2
  hosts: chrs
  tasks:
    - name: Set login/password
      community.routeros.command:
        commands: "user add name=user1 password=user1pass group=full"
    
    - name: Set NTP client
      community.routeros.command:
        commands: "system ntp client set enabled=yes servers=8.8.8.8"

    - name: Set OSPF
      community.routeros.command:
        commands:
          - /interface bridge add name=Lo
          - /ip address add address="{{ ip }}"/32 interface=Lo
          - /routing ospf instance add name=v2inst version=2 router-id="{{ ip }}"
          - /routing ospf area add name=backbone_v2 area-id=0.0.0.0 instance=v2inst
          - /routing ospf interface-template add network=0.0.0.0/0 area=backbone_v2

    - name: Get OSPF information
      community.routeros.command:
        commands: "/routing ospf neighbor print"
      register: ospf_information

    - name: Get config
      community.routeros.command:
        commands: "/export"
      register: config

    - name: Show OSPF information
      debug:
        msg: "{{ ospf_information }}"

    - name: Show config
      debug:
        msg: "{{ config }}"
```

Настраиваем ssh-соединения с сервера на chr1 и chr2 - генерируем пару ключей и копируем публичный ключ на chr1 и chr2:

```
ssh-keygen
ssh admin@10.2.0.11 "/file print file=mykey; file set mykey contents=\"`cat ~/.ssh/id_rsa.pub`\";/user ssh-keys import public-key-file=mykey.txt;/ip ssh set always-allow-password-login=yes"
ssh admin@10.2.0.12 "/file print file=mykey; file set mykey contents=\"`cat ~/.ssh/id_rsa.pub`\";/user ssh-keys import public-key-file=mykey.txt;/ip ssh set always-allow-password-login=yes"
```

Проверяем доступность chr1 и chr2:

```
ansible -m ping all
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/chrs_hosts_check.png)

Запускаем playbook:

```
ansible-playbook -i /etc/ansible/hosts lab2.yaml
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/playbook.png)

Проверяем OSPF и связь между chr:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/chrs_check.png)

Полный конфиг chr1:

```
/interface bridge
add name=Lo
/interface wireguard
add listen-port=51820 mtu=1420 name=wireguard1
/routing ospf instance
add disabled=no name=v2inst router-id=1.1.1.1
/routing ospf area
add disabled=no instance=v2inst name=backbone_v2
/interface wireguard peers
add allowed-address=10.2.0.1/24 endpoint-address=158.160.131.188 endpoint-port=51820 interface=wireguard1 \
    persistent-keepalive=10s private-key="..." public-key="..."
/ip address
add address=10.2.0.11/24 interface=wireguard1 network=10.2.0.0
add address=1.1.1.1 interface=Lo network=1.1.1.1
/ip dhcp-client
add interface=ether1
/ip firewall filter
add action=accept chain=input dst-port=51820 in-interface=wireguard1 protocol=udp
/ip ssh
set always-allow-password-login=yes
/routing ospf interface-template
add area=backbone_v2 disabled=no networks=0.0.0.0/0
/system note
set show-at-login=no
/system ntp client
set enabled=yes
/system ntp client servers
add address=8.8.8.8
```

Полный конфиг chr2:

```
/interface bridge
add name=Lo
/interface wireguard
add listen-port=51820 mtu=1420 name=wireguard1
/routing ospf instance
add disabled=no name=v2inst router-id=2.2.2.2
/routing ospf area
add disabled=no instance=v2inst name=backbone_v2
/interface wireguard peers
add allowed-address=10.2.0.1/24 endpoint-address=158.160.131.188 endpoint-port=51820 interface=wireguard1 \
    persistent-keepalive=10s private-key="..." public-key="..."
/ip address
add address=10.2.0.12/24 interface=wireguard1 network=10.2.0.0
add address=2.2.2.2 interface=Lo network=2.2.2.2
/ip dhcp-client
add interface=ether1
/ip firewall filter
add action=accept chain=input dst-port=51820 in-interface=wireguard1 protocol=udp
/ip ssh
set always-allow-password-login=yes
/routing ospf interface-template
add area=backbone_v2 disabled=no networks=0.0.0.0/0
/system note
set show-at-login=no
/system ntp client
set enabled=yes
/system ntp client servers
add address=8.8.8.8
```

4. Полученная схема связи:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab2/pictures/diagram.png)

**Вывод:** при выполнении работы была получены навыки по работе с Ansible.
