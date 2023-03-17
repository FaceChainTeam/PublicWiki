# Установка OGATEControl на Ubuntu Linux

-----------------------------

## Минимальный набор

### Освежаем систему

```
sudo apt update && sudo apt upgrade
```

### Устанавливаем `postgresql`

```
sudo apt install postgresql postgresql-contrib
```

Без базы данных приложение работать не может

### Увеличиваем лимит количества открываемых файлов в системе

```
ulimit -n 100000
sudo nano /etc/security/limits.conf
```

Вторая команда откроет редактор текстового файла с конфигурацией лимитов.

Добавляем такой текст в конец файла:

```
* soft nproc 65535
* hard nproc 65535
* soft nofile 65535
* hard nofile 65535
linuxhint soft nproc 100000
linuxhint hard nproc 100000
linuxhint soft nofile 100000
linuxhint hard nofile 100000
```

### Устанавливаем сами приложения

*предварительно заливаем пакеты с программой*, конечно, а потом:

```
sudo dpkg -i oport_*.deb
sudo dpkg -i kiosk_*.deb
```

Места звездочек нужно дополнить настоящим названием файла.

Можно пользоваться *OGATE Control* по порту `8000`, c "личным кабинетом" на `:8001` и киоском на `:8002` 

## Если мы ставим OGATE Control поверх ранее установленного OPORT

В вашем случае этот пункт, скорее всего, неактуален - ранние версии установлены всего в нескольких местах, но все-же Вы можете столкнуться с проблемой.

Симптомы **\- обновленная программа не дает залогиниться**

#### Решения

**Вариант 1** - исправить в конфигурационном файле */etc/facechain/oport/oport/config.toml*, значение для  `database_url`, в части `user=`**`oport`** на `user=`**`terminal`**

**Вариант 2** - если у вас уже была установлена ранняя версия OGC (до релиза 1.2) - измените вышеозначенный файл любым образом, например, добавив один пробел в произвольном месте. Настройки сохранятся прежние, ничего менять не придется

#### Причина 

Новый скрипт миграции предполагает, что обновление производится поверх прежней версии СКУД *TerminalGo* и подключение к базе данных будет продолжено под именем *terminal*, в то время, как база уже была создана под именем пользователя *oport*

Т.е. в общем случае апргрейда копируется база Terminal и используется тот же пользователь БД, в связи с проблемами переназначения прав после клонирования.

Установщик автоматически заменяет файл конфигурации, и система теряет возможность подключения к БД.

## Если сервер будет доступен в сети интернет

### Стоит установить `nginx` в качестве *reverse proxy*

```
sudo apt install nginx
```

#### Cоздаем файл конфигурации для пользовательской страницы и киоска

```
sudo nano /etc/nginx/sites-available/kiosk.conf
```

и вставляем такой текст (**но заменяем названия** доменов) в `kiosk.conf`

> ```
> #map $sent_http_content_type $expires {
> #    default                    off;
> #    text/html                  epoch; # means no-cache
> #    text/css                   max;
> #    application/javascript     max;
> #    ~image/                    max;
> #}
> 
> server {
>     server_name tk.ovision.ru; #!!!ЗАМЕНИТЬ!!!
>     listen 80;
>     client_max_body_size 20M;
> 
>     location  / {
>             proxy_http_version 1.1;
>             proxy_set_header Upgrade $http_upgrade;
>             proxy_set_header Connection "Upgrade";
>             proxy_pass http://localhost:8002;
>     }
> }
> 
> 
> 
> server {
>     server_name tv.ovision.ru; #!!!ЗАМЕНИТЬ!!!
>     listen 80 default_server;  #будет сайтом по умолчанию
>     client_max_body_size 20M;
> 
> #    expires $expires;
> 
>     location  / {
>             proxy_http_version 1.1;
>             proxy_set_header Upgrade $http_upgrade;
>             proxy_set_header Connection "Upgrade";
>             proxy_pass http://localhost:8001;
>     }
> }
> ```

#### Редактируем конфигурацию для **OGATE Control**

```
sudo nano /etc/nginx/sites-available/ocontrol.conf
```

и такой текст внутри (не забываем **заменить название** домена) внутри `ocontrol.conf`

> ```
> server {
>     listen 80;
>     server_name tc.ovision.ru;  #!!! ЗАМЕНИТЬ, это пример !!!!
>     client_max_body_size 20M;
> 
>     location / {
>         proxy_set_header Host $host;
>         proxy_set_header X-Real-IP $remote_addr;
>         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
>         proxy_pass http://localhost:8000;
>         proxy_redirect default;
>     }
>     location /websocket {
>         proxy_http_version 1.1;
>         proxy_set_header Upgrade $http_upgrade;
>         proxy_set_header Connection "Upgrade";
>         proxy_set_header Host $host;
>         proxy_pass http://localhost:8000/websocket;
>     }
> }
> ```

Размещаем эти файлы в доступной для *nginx* папке конфигураций

```
cd /etc/nginx/sites-enabled
sudo rm default
sudo ln -s ../sites-available/ocontrol.conf 
sudo ln -s ../sites-available/kiosk.conf
```

#### Проверяем, рабочая ли конфигурация

```
nginx -t
```

Если ошибка, надо поправить

#### Перезапускаем *nginx*

```
sudo systemctl restart nginx
```

#### Проверяем

```
sudo systemctl status nginx
```

### Устанавливаем `certbot` (*Let's Encrypt*) для SSL (HTTPS доступа к сайту)

```
sudo snap install --classic certbot 
```

Примечание: возможно в вашей системе не установлен менеджер пакетов `snap` и вы получили ошибку *sudo: snap: command not found*

В таком случае нужно его установить: `sudo apt install snapd` и повторить команду по установке *certbot* (выше)

### Запускаем `certbot` *(Let's Encrypt)*

```
sudo certbot --nginx 
```

И там:
* задать *e-mail*
* согласиться с лицензией (yes)
* выбрать домены, для которых нужно добавить *ssl* (цифры списка, через запятую)
* подтвердить, что требуется перенаправление на порт *https*

### `OpenVPN` для доступа к устройствам

Ставим *OpenVPN*

```
sudo apt install openvpn 
```

Переименовываем файл с ключом в `client.conf` и кладем его в папку `/etc/openvpn/`

Перезапускаем `openvpn`

```
sudo systemctl restart openvpn
```

### По безопасности - настраиваем *`ufw`* (ubuntu firewall)

```
sudo ufw allow ssh
sudo ufw delete allow 'Nginx HTTP'
sudo ufw delete allow 'Nginx HTTPS'
sudo ufw allow 'Nginx Full'
sudo ufw allow 1194/udp  #OpenVPN
sudo ufw allow 1194/tcp  #если OpenVPN разрешен для tcp
```

Внимательно проверяем, что есть *ssh* (верный порт!) и все, что нужно в списке разрешенных (желательно открыть еще один терминал с *ssh* сессией на правильном порту, на всякий)

```
sudo ufw app list
```

и только потом включаем *ubuntu firewall* 

```
sudo ufw enable
```

### Для отправки СМС и почтовых сообщений нужно добавить настройки в файл конфигурации

Файл конфигурации находится в `/etc/facechain/oport/config.toml`

```
sms_login = "looogin"
sms_password = "paswooord"

smtp_server = "smtp.gmail.com"
smtp_port = 587
smtp_username = "requestvisits@gmail.com"
smtp_password = "smptp-paswoooord"
smtp_encryption = "TLS"
```

### Настраиваем сообщения для СМС и почты

На вкладке `Администрирование`→`Уведомления` указываем url ([`https://visit.demo.ovision.ru/`](https://visit.demo.ovision.ru/) как пример), на который будет отправлен пользователь для регистрации по приглашению, обязательно со слешем в конце, чтобы не сломать *iphone*.

Настраиваем шаблоны сообщений по своему вкусу.
