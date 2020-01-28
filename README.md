Настройка протокола PPTP 
Настройка сервера.
Вы можете выполнять команды от root-пользователя, в таком случае не нужно использовать sudo(во всех пунктах). Первым делом установим необходимые пакеты:

```
sudo apt update
sudo apt upgrade 
sudo apt install pptpd ppp
```

Редактируем файл конфигурации:

```
sudo nano /etc/pptpd.conf                                                                                          
```

Его содержание должно быть таким:

```
option /etc/ppp/pptpd-options
bcrelay eth0
logwtmp
localip 172.16.0.1
remoteip 172.16.0.2-254
listen 12.34.15.1
```

Редактируем следующий конфиг:

```
sudo nano /etc/ppp/pptpd-options
```

В нём находим и исправляем эти строки:

```
name pptpd
refuse-pap
refuse-chap
refuse-mschap
require-mschap-v2
require-mppe-128
ms-dns 8.8.8.8
proxyarp
nodefaultroute
lock
nobsdcomp
nologfd
novj
novjccomp
```

Теперь Вы можете запустить PPTP и узнать статус его работы:

```
service pptpd start
service pptpd restart #Если у Вас уже был запущен сервис
service pptpd status
```

Проверьте, что 1723 порт работает и принимает соединения:

```
netstat -alpn | grep 1723
```

Правим конфиг авторизации пользователей:

```
sudo nano /etc/ppp/chap-secrets
```

В нём настроим пользователя с логином и паролем, пример создания пользователей:

```
user1    pptpd   password1     "*"       #Клиент №1
user2    pptpd   password2  "172.16.0.2" #Клиент №2 
```

Создание 2 клиентов необязательно! 
user1 - имя пользователя
password1 - пароль пользователя
"*" - локальный ip будет выдаваться из пула адресов, указанного в файле /etc/pptpd.conf
"172.16.0.2" - пользователю будет присвоен указанный статический ip адрес.

После добавления всех необходимых клиентов, закройте и сохраните файл.
Теперь редактируем другой конфиг:

```
sudo nano /etc/sysctl.conf
```

Настроим форвардинг. Он позволит Вам пересылать пакеты между публичным IP и приватными IP.
В /etc/sysctl.conf нужно раскомментировать строчку, т.е. удалить из неё ‘#’:

```
net.ipv4.ip_forward=1
```

Сохраните и закройте файл.
Для применения настроек к текущей сессии воспользуйтесь командой:

```
sysctl -p
```

Смотрим, через какой интерфейс в интернет смотрит сервер. В моём случае это eth0, его также можно увидеть выше на рис. 1.1.:

```
ifconfig
```

Рис.1.2.- Результат работы команды ifconfig для сервера.
Настроим правила iptables. PPTP работает через TCP-порт 1723, он должен быть открыт. Для этого создадим файл run-iptables.sh:

```
sudo nano run-iptables.sh
```

И вставим в него правила, в итоге файл должен выглядеть так:

```
iptables -A INPUT -p gre -j ACCEPT
iptables -A INPUT -m tcp -p tcp --dport 1723 -j ACCEPT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables --table nat --append POSTROUTING --out-interface ppp0 -j MASQUERADE
iptables -I INPUT -s 172.16.0.0/24 -i ppp0 -j ACCEPT
iptables --append FORWARD --in-interface eth0 -j ACCEPT
```

Сохраняем, выходим.

```
chmod +x run-iptables.sh
./run-iptables.sh
service pptpd restart
rm  ./run-iptables.sh
```

Чтобы правила iptables запускались автоматически при перезагрузке, необходимо установить утилиту netfilter-persistent  и iptables-persistent. Выполним:

```
sudo apt install netfilter-persistent iptables-persistent
```

Для сохранения введённых iptables, воспользуйтесь командой:

```
service netfilter-persistent save
```

Перезапускаем сервис pptpd для применения новых настроек и смотрим состояние сервиса:

```
service pptpd restart
service pptpd status
```
Поздравляем! Сервис настроен и запущен! Включаем на автозагрузку pptpd-сервер: 

```
sudo systemctl enable pptpd
```

16.Настройка клиента. Скачиваем утилиту pptp-linux и обновляем систему. Вся настройка далее выполняется на компьютере-клиенте.

```
apt update
apt install pptp-linux
```
Создадим файл для подключения, называться он будет vpn, Вы можете выбрать любой название.
```
nano /etc/ppp/peers/vpn
```

Его содержание должно быть таким:

```
pty "pptp 12.34.15.1 --nolaunchpppd" # Публичный IP-адрес Вашего сервера
require-mschap-v2
require-mppe-128
user test1 # Имя пользователя. Оно находится в файле /etc/ppp/chap-secrets на PPTP-сервере
password "testtest" # Пароль. Он находится в файле /etc/ppp/chap-secrets на PPTP-сервере
nodeflate
nobsdcomp
noauth
nodefaultroute  # Отключаем маршрут по умолчанию, иначе замените на defaultroute
persist # Переподключение при обрыве
maxfail 10 # Количество попыток переподключения
holdoff 15 # Интервал между подключениями
```

Для того, чтобы трафик “ходил” через наш сервер, необходимо добавить на компьютер-клиент маршрутизацию на Вашу приватную !!!!!! сеть через интерфейс ppp0:

```
ip route add 10.0.0.0/8 dev ppp0
```

Пробуем подключиться. На клиенте вызываем команду:

```
pppd call vpn
```

vpn - название файла, который мы создали в /etc/ppp/peers, это было в шаге 15.
На клиенте:

На скрине видно, что клиент получил первый свободный ip-адрес из пула, который мы создали на сервер. На сервере же:

Выполним на клиенте:

```
traceroute 8.8.8.8
```
Не смотря на то, что тунель между клиентом и сервером есть, трафик между не ходит по нему, решим это добавив маршрут для интерфейса ppp0 на клиенте:

```
ip route add 172.16.0.0/24 dev ppp0

```
