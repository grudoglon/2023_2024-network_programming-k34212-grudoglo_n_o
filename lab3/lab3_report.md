# Отчет по лабораторной работе №3 "Развертывание Netbox, сеть связи как источник правды в системе технического учета Netbox"

University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Grudoglo Nikita Olegovich

Lab: Lab3

Date of create: 20.02.2024

Date of finished: ...

**Цель работы:** с помощью Ansible и Netbox собрать всю возможную информацию об устройствах и сохранить их в отдельном файле.

**Ход работы:**

1. Поднимем NetBox на виртуальной машине Ubuntu 22.04 по шагам из [офф.документации](https://docs.netbox.dev/en/stable/installation/):

1.1. Установим PostgreSQL:

```
sudo apt install -y postgresql
```

Заходим в PostgreSQL shell:

```
sudo -u postgres psql
```

Создаем базу данных для NetBox, назначаем ей имя пользователя и пароль для аутентификации:

```
CREATE DATABASE netbox;
postgres=# CREATE USER netbox WITH PASSWORD '***';
postgres=# ALTER DATABASE netbox OWNER TO netbox;
```

Проверяем:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/psql.png)

1.2. Установим Redis:

```
sudo apt install -y redis-server
```

Проверяем:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/redis.png)

1.3. Установим и настроим NetBox.

Установим необходимые пакеты:

```
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev
```

Создаем директорию для установки NetBox и клонируем туда репозиторий:

```
sudo mkdir -p /opt/netbox/
cd /opt/netbox/
sudo git clone -b master --depth 1 https://github.com/netbox-community/netbox.git .
```

Создаем учетную запись пользователя с именем netbox, назначаем ему необходимые права:

```
sudo adduser --system --group netbox
sudo chown --recursive netbox /opt/netbox/netbox/media/
sudo chown --recursive netbox /opt/netbox/netbox/reports/
sudo chown --recursive netbox /opt/netbox/netbox/scripts/
```

Копируем конфигурационный файл:

```
cd /opt/netbox/netbox/netbox/
sudo cp configuration_example.py configuration.py
```

Генерируем секретный ключ: 

```
python3 ../generate_secret_key.py
```

Откроем конфигурационный файл configuration.py для настройки NetBox и настроим параметры ALLOWED_HOSTS, DATABASE, REDIS и SECRET_KEY:

```
ALLOWED_HOSTS = ['*']

DATABASE = {
    'ENGINE': 'django.db.backends.postgresql',  # Database engine
    'NAME': 'netbox',         # Database name
    'USER': 'netbox',         # PostgreSQL username
    'PASSWORD': 'gnnbpass',   # PostgreSQL password
    'HOST': 'localhost',      # Database server
    'PORT': '',               # Database port (leave blank for default)
    'CONN_MAX_AGE': 300,      # Max database connection age
}

REDIS = {
    'tasks': {
        'HOST': 'localhost',
        'PORT': 6379,
        'USERNAME': '',
        'PASSWORD': '',
        'DATABASE': 0,
        'SSL': False,
    },
    'caching': {
        'HOST': 'localhost',
        'PORT': 6379,
        'USERNAME': '',
        'PASSWORD': '',
        'DATABASE': 1,
        'SSL': False,
    }
}

SECRET_KEY = '***'
```

Запускам upgrade-скрипт (он создаст виртуальную среду Python, установит все необходимые пакеты Python, запустит миграцию схемы базы данных):

```
sudo /opt/netbox/upgrade.sh
```

Заходим в созданную виртуальную среду:

```
source /opt/netbox/venv/bin/activate
```

Создаем суперюзера:

```
cd /opt/netbox/netbox
python3 manage.py createsuperuser
```

1.4. Установим Gunicorn. Копируем конфигурационный файл и необходимые файлы NetBox:

```
sudo cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
sudo cp -v /opt/netbox/contrib/*.service /etc/systemd/system/
```

Перезапускаем systemd daemon:

```
sudo systemctl daemon-reload
```

Запускаем сервисы NetBox:

```
sudo systemctl start netbox netbox-rq
sudo systemctl enable netbox netbox-rq
```

Проверяем статус:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/nb_status.png)

1.5. Установим HTTP сервер apache:

```
sudo apt install -y apache2
```

Генерируем самоподписанный сертификат:

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/netbox.key \
-out /etc/ssl/certs/netbox.crt
```

Копируем конфигурационный файл:

```
sudo cp /opt/netbox/contrib/apache.conf /etc/apache2/sites-available/netbox.conf
```

Убеждаемся, что все необходимые модули apache включены, включаем сайт netbox и перезагружаем apache:

```
sudo a2enmod ssl proxy proxy_http headers rewrite
sudo a2ensite netbox
sudo systemctl restart apache2
```

Теперь в браузере по адресу https://158.160.150.209/ (белый ip-адрес виртуальной машины) доступен NetBox:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/netbox.png)


2. Заполним всю возможную информацию о CHR в NetBox. Были добавлены ip-адреса, устройства и интерфейсы:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/nb_ip-addresses.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/nb_devices.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/nb_interfaces.png)

3. Используя Ansible и роли для NetBox, сохраним все данные из NetBox в отдельный файл:

```
plugin: netbox.netbox.nb_inventory
api_endpoint: https://158.160.150.209
token: токен
validate_certs: False
config_context: False
group_by:
  - device_roles
interfaces: True
```

Данные были сохранены в файл [nb_inventory_res.yml](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/nb_inventory_res.yml):

```
ansible-inventory -v --list -i nb_inventory.yml --yaml > nb_inventory_res.yml
```

В файл были также добавлены общие переменные.

4. Напишем сценарий, при котором на основе данных из NetBox (nb_inventory_res.yml) можно настроить 2 CHR (изменить имя устройства, добавить IP-адрес на устройство):

```
- name: Config CHRs
  hosts: device_roles_router
  tasks:
    - name: Set Device Name
      community.routeros.command:
        commands:
          - /system identity set name="{{interfaces[0].device.name}}"
    - name: Add IP
      community.routeros.command:
        commands:
        - /interface bridge add name="{{interfaces[1].name}}"
        - /ip address add address="{{interfaces[1].ip_addresses[0].address}}" interface="{{interfaces[1].name}}"
```

Запускаем плейбук:

```
ansible-playbook -i nb_inventory_res.yml lab31.yml
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/pb1.png)

Проверяем изменения на chr:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/chrs.png)

5. Напишем сценарий, позволяющий собрать серийный номер устройства и вносящий серийный номер в NetBox:

```
- name: Collect Serial Numbers To NetBox
  hosts: device_roles_router
  tasks:
    - name: Collect Serial Numbers
      community.routeros.command:
        commands:
          - /system license print
      register: license
    - name: Collect Names
      community.routeros.command:
        commands:
          - /system identity print
      register: identity
    - name: Add Serial Number to Netbox
      netbox_device:
        netbox_url: https://158.160.150.209
        netbox_token: токен
        data:
          name: "{{identity.stdout_lines[0][0].split(' ').1}}"
          serial: "{{license.stdout_lines[0][0].split(' ').1}}"
        state: present
        validate_certs: False
```

Запускаем плейбук:

```
ansible-playbook -i nb_inventory_res.yml lab32.yml
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/pb2.png)

Проверяем данные в NetBox:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/chr1.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab3/pictures/chr2.png)

**Вывод:** при выполнении работы была получены навыки по работе с Ansible и NetBox.
