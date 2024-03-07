# Отчет по лабораторной работе №4 "Базовая 'коммутация' и туннелирование используя язык программирования P4"

University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Network programming](https://itmo-ict-faculty.github.io/network-programming/)

Year: 2023/2024

Group: K34212

Author: Grudoglo Nikita Olegovich

Lab: Lab4

Date of create: 26.02.2024

Date of finished: ...

**Цель работы:** Изучить синтаксис языка программирования P4 и выполнить 2 обучающих задания от Open network foundation для ознакомления на практике с P4.

**Ход работы:**

1. Клонируем репозиторий [p4lang/tutorials](https://github.com/p4lang/tutorials):

```
git clone https://github.com/p4lang/tutorials
```

2. Переходим в папку vm-ubuntu-20.04 и, используя Vagrant, развертывем тестовую среду:

```
cd tutorials/vm-ubuntu-20.04
vagrant up
```

3. В результате установки появилась виртуальная машина с аккаунтами login/password vagrant/vagrant и p4/p4:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/vm.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/vm_login.png)

Был выполнен вход в аккаунт p4.

4. Выполним задание Implementing Basic Forwarding - в виртуальной машине перейдем в папку проекта p4\tutorials\exercises\basic. Цель задания - написать программу на языке P4, которая реализует базовую пересылку для IPv4. В папке уже есть скелет программы, basic.p4, который изначально отбрасывает все пакеты.

Проверим исходный файл basic.p4:

```
make run
```

Пробуем пинговаться между хостами:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_ping1.png)

Пинг неуспешен, так как basic.p4 отбрасывает все пакеты.

Выходим из среды Mininet, останавливаем и очищаем ее:

```
exit
make stop
make clean
```

Дополним файл basic.p4 - допишем функции MyParser, MyIngress и MyDeparser.

Функция MyParser - принимает входящий пакет и извлекает заголовки Ethernet и IPv4 из него:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_MyParser.png)

Функция MyIngress - обрабатывает входящий трафик IPv4 (определяет действия для пересылки и отбрасывания пакетов, а также таблицу для принятия решений о пересылке на основе адреса назначения IPv4):

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_MyIngress1.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_MyIngress2.png)

Функция MyDeparser - формирует исходящий пакет, добавляя в него заголовки Ethernet и IPv4 из соответствующих полей.

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_MyDeparser.png)

Готовый файл basic.p4 [здесь](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/basic.p4).

Проверим полученный файл basic.p4:

```
make run
```

Пробуем пинговаться между хостами:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex1_ping2.png)

Пинг успешен.

5. Выполним задание Implementing Basic Tunneling - в виртуальной машине перейдем в папку проекта p4\tutorials\exercises\basic_tunnel. Цель задания - добавить поддержку базового протокола туннелирования к IP-маршрутизатору, который был написан в предыдущем задании. В папке уже есть скелет программы, basic_tunnel.p4, нужно дописать функции MyParser, MyIngress и MyDeparser.

Функция MyParser - добавлена обработка заголовка myTunnel:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_MyParser.png)

Функция MyIngress - добавлена обработка пакетов, проходящих через туннель:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_MyIngress1.png)

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_MyIngress2.png)

Функция MyDeparser - добавлено присоединение заголовка myTunnel:

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_MyDeparser.png)

Готовый файл basic_tunnel.p4 [здесь](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/basic_tunnel.p4).

Проверим полученный файл basic_tunnel.p4:

```
make run
```

Открываем терминалы для h1 и h2:

```
xterm h1 h2
```

В xterm h2 запускаем прием пакетов:

```
./receive.py
```

В xterm h1 отправляем сообщение на h2 без использования туннелирования:

```
./send.py 10.0.2.2 "P4 is cool!"
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_check1.png)

На h2 был получен пакет, он состоит из заголовков Ethernet, IP и TCP и сообщения.

Теперь в xterm h1 отправляем сообщение на h2 с использованием туннелирования (указываем ID назначения):

```
./send.py 10.0.2.2 "P4 is cool!" --dst_id 2
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_check2.png)

На h2 был получен пакет, он состоит из заголовков Ethernet, MyTunnel и IP и сообщения.

Попробуем в xterm h1 отправить сообщение на h2 с использованием туннелирования, указав неправильный IP-адрес, но оставив верный ID назначения:

```
./send.py 10.0.3.3 "P4 is cool!" --dst_id 2
```

![](https://github.com/grudoglon/2023_2024-network_programming-k34212-grudoglo_n_o/blob/main/lab4/pictures/ex2_check3.png)

На h2 все равно был получен пакет, так как коммутатор не использует IP-заголовок, когда в пакете есть заголовок MyTunnel, а ID назначения в нем был указан верный (2 - h2).

**Вывод:** при выполнении работы была получены навыки по работе с языком P4, были выполнены задания Implementing Basic Forwarding и Implementing Basic Tunneling для ознакомления с языком P4.
