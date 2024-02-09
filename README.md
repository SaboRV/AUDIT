## Цель домашнего задания
Научится проектировать централизованный сбор логов. Рассмотреть особенности разных платформ для сбора логов.

### Описание домашнего задания
1. В Vagrant разворачиваем 2 виртуальные машины web и log
2. на web настраиваем nginx
3. на log настраиваем центральный лог сервер на любой системе на выбор
journald;
rsyslog;
elk.
4. настраиваем аудит, следящий за изменением конфигов nginx 

Все критичные логи с web должны собираться и локально и удаленно.
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.

Формат сдачи ДЗ - vagrant + ansible


## Решение


## 1. Создаём виртуальные машины
Создаём каталог, в котором будут храниться настройки виртуальной машины. В каталоге создаём файл с именем Vagrantfile, добавляем в него следующее содержимое: 

Vagrant.configure ("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "centos/7"
  config.vm.box_version = "2004.01"

  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.cpus = 2
  end
  
  # Define two VMs with static private IP addresses.
  boxes = [
    { :name => "web",
      :ip => "192.168.56.10",
    },
    { : name => "log",
      :ip => "192.168.56.15",
  ]
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]
    end
  end
end

Результатом выполнения команды vagrant up станут 2 созданные виртуальные машины
Заходим на web-сервер: vagrant ssh web
Дальнейшие действия выполняются от пользователя root. Переходим в root пользователя: sudo -i
Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время. 
Укажем часовой пояс (Уральское время +5):
[root@web ~]# rm -rf /etc/localtime
[root@web ~]# ln -s /usr/share/zoneinfo/Asia/Yekaterinburg /etc/localtime
[root@web ~]# systemctl restart chronyd^C
[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2024-02-09 14:42:53 +05; 33min ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
 Main PID: 381 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─381 /usr/sbin/chronyd

Feb 09 14:42:52 web systemd[1]: Starting NTP client/server...
Feb 09 14:42:53 web chronyd[381]: chronyd version 3.4 starting (+CMDMON +NTP +REFCLOCK +RTC +PRIV...EBUG)
Feb 09 14:42:53 web chronyd[381]: Frequency -17.707 +/- 1.503 ppm read from /var/lib/chrony/drift
Feb 09 14:42:53 web systemd[1]: Started NTP client/server.
Feb 09 14:43:01 web chronyd[381]: Selected source 91.206.16.3
Feb 09 14:43:01 web chronyd[381]: System clock wrong by 1.392560 seconds, adjustment started
Feb 09 14:43:02 web chronyd[381]: System clock was stepped by 1.392560 seconds
Feb 09 14:44:09 web chronyd[381]: Selected source 162.159.200.123
Feb 09 14:46:18 web chronyd[381]: Selected source 91.206.16.3
Hint: Some lines were ellipsized, use -l to show in full.

Далее проверим, что время и дата указаны правильно: date
[root@web ~]# date
Fri Feb  9 15:16:58 +05 2024

#### Настроить NTP нужно на обоих серверах.

Также, для удобства редактирования конфигурационных файлов можно установить текстовый редактор vim: yum install -y vim

## 2. Установка nginx на виртуальной машине web
Для установки nginx сначала нужно установить epel-release: yum install epel-release 


Установим nginx: yum install -y nginx  

Проверим, что nginx работает корректно:
[root@web ~]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2024-02-09 15:04:28 +05; 13min ago
  Process: 1284 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1281 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 1279 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1286 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1286 nginx: master process /usr/sbin/nginx
           ├─1287 nginx: worker process
           └─1288 nginx: worker process

Feb 09 15:04:28 web systemd[1]: Starting The nginx HTTP and reverse proxy server...
Feb 09 15:04:28 web nginx[1281]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Feb 09 15:04:28 web nginx[1281]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Feb 09 15:04:28 web systemd[1]: Started The nginx HTTP and reverse proxy server.

[root@web ~]# ss -tln | grep 80
LISTEN     0      128          *:80                       *:*                  
LISTEN     0      128       [::]:80                    [::]:*             


Также работу nginx можно проверить на хосте. В браузере ввведем в адерсную строку http://192.168.56.10 


## 3. Настройка центрального сервера сбора логов

Откроем еще одно окно терминала и подключаемся по ssh к ВМ log: vagrant ssh log
Перейдем в пользователя root: sudo -i
rsyslog должен быть установлен по умолчанию в нашей ОС, проверим это:

[root@log ~]# yum list rsyslog
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.fra1.de.leaseweb.net
 * extras: mirror.fra1.de.leaseweb.net
 * updates: centos.bio.lmu.de
Installed Packages
rsyslog.x86_64                                8.24.0-52.el7                                     @anaconda
Available Packages
rsyslog.x86_64                                8.24.0-57.el7_9.3                                 updates 

Все настройки Rsyslog хранятся в файле /etc/rsyslog.conf 
Для того, чтобы наш сервер мог принимать логи, нам необходимо внести следующие изменения в файл: 
Открываем порт 514 (TCP и UDP):
Находим закомментированные строки и приводим их к виду:
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514

В конец файла /etc/rsyslog.conf добавляем правила приёма сообщений от хостов:
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~

Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log
Далее сохраняем файл и перезапускаем службу rsyslog: systemctl restart rsyslog
Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514:

[root@log ~]# ss -tuln
Netid State      Recv-Q Send-Q     Local Address:Port                    Peer Address:Port              
udp   UNCONN     0      0              127.0.0.1:323                                *:*                  
udp   UNCONN     0      0                      *:997                                *:*                  
udp   UNCONN     0      0                      *:514                                *:*                  
udp   UNCONN     0      0                      *:68                                 *:*                  
udp   UNCONN     0      0                      *:111                                *:*                  
udp   UNCONN     0      0                  [::1]:323                             [::]:*                  
udp   UNCONN     0      0                   [::]:997                             [::]:*                  
udp   UNCONN     0      0                   [::]:514                             [::]:*                  
udp   UNCONN     0      0                   [::]:111                             [::]:*                  
tcp   LISTEN     0      128                    *:111                                *:*                  
tcp   LISTEN     0      128                    *:22                                 *:*                  
tcp   LISTEN     0      100            127.0.0.1:25                                 *:*                  
tcp   LISTEN     0      25                     *:514                                *:*                  
tcp   LISTEN     0      128                 [::]:111                             [::]:*                  
tcp   LISTEN     0      128                 [::]:22                              [::]:*                  
tcp   LISTEN     0      100                [::1]:25                              [::]:*                  
tcp   LISTEN     0      25                  [::]:514                             [::]:*   


## Далее настроим отправку логов с web-сервера
Заходим на web сервер: vagrant ssh web
Переходим в root пользователя: sudo -i 
Проверим версию nginx: rpm -qa | grep nginx

[root@web ~]# rpm -qa | grep nginx
nginx-1.20.1-10.el7.x86_64
nginx-filesystem-1.20.1-10.el7.noarch

Версия nginx должна быть 1.7 или выше. В нашем примере используется версия nginx 1.20. 
Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:

error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.56.15:514,tag=nginx_error;

access_log  /var/log/nginx/access.log  main;
access_log syslog:server=192.168.56.15:514,tag=nginx_access,severity=info combined;

Для Access-логов указыаем удаленный сервер и уровень логов, которые нужно отправлять. Для error_log добавляем удаленный сервер. Если требуется чтобы логи хранились локально и отправлялись на удаленный сервер, требуется указать 2 строки. 	
Tag нужен для того, чтобы логи записывались в разные файлы.
По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg. Если трубуется хранили или пересылать логи с другим severity, то это также можно указать в настройках nginx. 
Далее проверяем, что конфигурация nginx указана правильно: nginx -t

[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

Далее перезапустим nginx: systemctl restart nginx
Чтобы проверить, что логи ошибок также улетают на удаленный сервер, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-сраницы: rm /usr/share/nginx/html/img/header-background.png

Попробуем несколько раз зайти по адресу http://192.168.56.10
Далее заходим на log-сервер и смотрим информацию об nginx:
cat /var/log/rsyslog/web/nginx_access.log 
cat /var/log/rsyslog/web/nginx_error.log 

[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Feb  9 15:06:28 web nginx_access: 192.168.56.1 - - [09/Feb/2024:15:06:28 +0500] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:122.0) Gecko/20100101 Firefox/122.0"
Feb  9 15:06:28 web nginx_access: 192.168.56.1 - - [09/Feb/2024:15:06:28 +0500] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:122.0) Gecko/20100101 Firefox/122.0"
Feb  9 15:06:28 web nginx_access: 192.168.56.1 - - [09/Feb/2024:15:06:28 +0500] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:122.0) Gecko/20100101 Firefox/122.0"
Feb  9 15:31:28 web nginx_access: 192.168.56.1 - - [09/Feb/2024:15:31:28 +0500] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:122.0) Gecko/20100101 Firefox/122.0"
[root@log ~]# cat /var/log/rsyslog/web/nginx_error.log
Feb  9 15:06:28 web nginx_error: 2024/02/09 15:06:28 [error] 1287#1287: *4 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"

Видим, что логи отправляются корректно. 

## 4. Настройка аудита, контролирующего изменения конфигурации nginx
За аудит отвечает утилита auditd, в RHEL-based системах обычно он уже предустановлен. Проверим это: rpm -qa | grep audit

[root@web ~]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64

Настроим аудит изменения конфигурации nginx:
Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец файла /etc/audit/rules.d/audit.rules добавим следующие строки:

-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf

Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:
/etc/nginx/nginx.conf
Всех файлов каталога /etc/nginx/default.d/
Для более удобного поиска к событиям добавляется метка nginx_conf
Перезапускаем службу auditd: service auditd restart

После данных изменений у нас начнут локально записываться логи аудита. Чтобы проверить, что логи аудита начали записываться локально, нужно внести изменения в файл /etc/nginx/nginx.conf или поменять его атрибут, потом посмотреть информацию об изменениях: ausearch -f /etc/nginx/nginx.confq
Также можно воспользоваться поиском по файлу /var/log/audit/audit.log, указав наш тэг: grep nginx_conf /var/log/audit/audit.log

[root@web ~]# ausearch -f /etc/nginx/nginx.conf
----
time->Fri Feb  9 15:44:55 2024
type=CONFIG_CHANGE msg=audit(1707475495.540:155): auid=1000 ses=3 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
----
time->Fri Feb  9 15:44:55 2024
type=PROCTITLE msg=audit(1707475495.540:156): proctitle=7669002F6574632F6E67696E782F6E67696E782E636F6E66
type=PATH msg=audit(1707475495.540:156): item=3 name="/etc/nginx/nginx.conf~" inode=749132 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=CREATE cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PATH msg=audit(1707475495.540:156): item=2 name="/etc/nginx/nginx.conf" inode=749132 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=DELETE cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PATH msg=audit(1707475495.540:156): item=1 name="/etc/nginx/" inode=85 dev=08:01 mode=040755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0


[root@web ~]# grep nginx_conf /var/log/audit/audit.log
type=CONFIG_CHANGE msg=audit(1707475381.099:119): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1707475381.103:120): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1707475388.350:126): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1707475388.350:127): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1


Далее настроим пересылку логов на удаленный сервер. Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins: yum -y install audispd-plugins

Найдем и поменяем следующие строки в файле /etc/audit/auditd.conf: 
log_format = RAW
name format = HOSTNAME

В name_format  указываем HOSTNAME, чтобы в логах на удаленном сервере отображалось имя хоста. 
В файле /etc/audisp/plugins.d/au-remote.conf поменяем параметр active на yes:

active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string

В файле /etc/audisp/audisp-remote.conf требуется указать адрес сервера и порт, на который будут отправляться логи:

remote_server = 192.168.56.15
port = 60

Далее перезапускаем службу auditd: service auditd restart
На этом настройка web-сервера завершена. Далее настроим Log-сервер. 

Отроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf:

tcp_listen_port = 60

Перезапустим службу auditd: service auditd restart
На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла /etc/nginx/nginx.conf и проверить на log-сервере, что пришла информация об изменении атрибута:

[root@web ~]# ls -l /etc/nginx/nginx.conf
-rw-r--r--. 1 root root 2482 Feb  9 15:44 /etc/nginx/nginx.conf
[root@web ~]# chmod +x /etc/nginx/nginx.conf
[root@web ~]# ls -l /etc/nginx/nginx.conf
-rwxr-xr-x. 1 root root 2482 Feb  9 15:44 /etc/nginx/nginx.conf
[root@web ~]# 

[root@log ~]# grep web /var/log/audit/audit.log 
node=web type=CONFIG_CHANGE msg=audit(1707477134.790:72): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1707477134.794:73): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SERVICE_START msg=audit(1707477134.794:74): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=auditd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
node=web type=SYSCALL msg=audit(1707477566.672:75): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=b6d420 a2=1ed a3=7ffe34637ba0 items=1 ppid=977 pid=1109 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=1 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CWD msg=audit(1707477566.672:75):  cwd="/root"
node=web type=PATH msg=audit(1707477566.672:75): item=0 name="/etc/nginx/nginx.conf" inode=465358 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1707477566.672:75): proctitle=63686D6F64002B78002F6574632F6E67696E782F6E67696E782E636F6E66





