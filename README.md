# Практика с SELinux

## Задание № 1. Запустить nginx на нестандартном порту 3-мя разными способами
### Начало
Смотрим состояние SELinux и firewalld. 
```
[root@nginx01 ~]# sestatus 
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31

[root@nginx01 ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Устанавили nginx и проверили его работоспособность на порту 80.  
Меняем порт для nginx на 8888.  
Меняем порт и перезапускаем  
!["Меняем порт для Nginx"](https://github.com/mus-cat/otus-study-m3l17/blob/main/I/ChangeNginxPort.png)

```
[root@nginx01 ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

root@nginx01 ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Wed 2023-03-08 09:30:17 UTC; 7min ago
  Process: 2265 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2332 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2331 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2267 (code=exited, status=0/SUCCESS)

Mar 08 09:30:17 nginx01 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Mar 08 09:30:17 nginx01 nginx[2332]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Mar 08 09:30:17 nginx01 nginx[2332]: nginx: [emerg] bind() to 0.0.0.0:8888 failed (13: Permission denied)
Mar 08 09:30:17 nginx01 nginx[2332]: nginx: configuration file /etc/nginx/nginx.conf test failed
Mar 08 09:30:17 nginx01 systemd[1]: nginx.service: control process exited, code=exited status=1
Mar 08 09:30:17 nginx01 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Mar 08 09:30:17 nginx01 systemd[1]: Unit nginx.service entered failed state.
Mar 08 09:30:17 nginx01 systemd[1]: nginx.service failed.

[root@nginx01 ~]# tail /var/log/audit/audit.log
type=AVC msg=audit(1678267817.638:562): avc:  denied  { name_bind } for  pid=2332 comm="nginx" src=8888 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

### Подход с использованием setsebool и параметром nis_enabled 
```
[root@nginx01 ~]# setsebool nis_enabled on
```
Пытаемся запустить nginx. И о чудо, все работает!
!["SELinux & setsebool"](https://github.com/mus-cat/otus-study-m3l17/blob/main/I/StartNginx_Use_setsebool.png)
Возвращаем переменную в исходное состояние и пытаемся перезапустить сервис... Не работает...

### Подход с добавлением используемого нами порта в список разрещенных для сервиса.
```
[root@nginx01 ~]# semanage port -a -t http_port_t -p tcp 8888
[root@nginx01 ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      8888, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Пробуем запустить nginx. И все работает!
!["Add port"](https://github.com/mus-cat/otus-study-m3l17/blob/main/I/StartNginx_Use_openport.png)

Возвращаем контекст http_port_t в исходное состояние
```
[root@nginx01 ~]# semanage port -d -t http_port_t -p tcp 8888
[root@nginx01 ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

### Подход с созданием модуля selinux для запуска nginx
Создаем модуль
```
[root@nginx01 ~]# grep nginx /var/log/audit/audit.log.old | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Загружаем полученный модуль nginx.
```
[root@nginx01 ~]# semodule -i nginx.pp
```
пытаемся перезапустить сервис nginx и проверяем, что он работает.
!["Make module"](https://github.com/mus-cat/otus-study-m3l17/blob/main/I/StartNginx_Use_audit2allow.png)
 
#### Замечание. Выполняем команду
```
[root@nginx01 ~]# grep nginx /var/log/audit/audit.log.old | audit2allow -r -m nginx

module nginx 1.0;

require {
	type httpd_t;
	type unreserved_port_t;
	class tcp_socket name_bind;
}

#============= httpd_t ==============

#!!!! This avc can be allowed using the boolean 'nis_enabled'
allow httpd_t unreserved_port_t:tcp_socket name_bind;
```

Затем команду:
```
[root@nginx01 ~]# semanage port -l | grep unreserved
unreserved_port_t              sctp     1024-65535
unreserved_port_t              tcp      61001-65535, 1024-32767
unreserved_port_t              udp      61001-65535, 1024-32767
```

Видно, что результат "шире" чем нам необходимо, т.е. при использовании созданного программой модля, **nginx** может выполнять bind на большую группу портов, чем нам было необходимо.

## Задание № 2. Обеспечить работоспособность приложения при включенном selinux

### Диагностика проблемы

1. Проверяем логи командой `journalctl --unit named`
и находим запись : `/etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denie`

2. Командой `ps aux | grep named` смотрим под какой учётной записью работает демон named
Видим, что под записью - `named`

3. Смотрим права на папку и файлы в **/etc/named**
```
root@ns01 ~]# ls -l /etc/named
total 16
drw-rwx---. 2 root named  56 Mar  5 12:12 dynamic
-rw-rw----. 1 root named 784 Mar  5 12:12 named.50.168.192.rev
-rw-rw----. 1 root named 610 Mar  5 12:11 named.dns.lab
-rw-rw----. 1 root named 609 Mar  5 12:11 named.dns.lab.view1
-rw-rw----. 1 root named 657 Mar  5 12:12 named.newdns.lab
[root@ns01 ~]# ls -l /etc/named/dynamic/
total 8
-rw-rw----. 1 named named 509 Mar  5 12:12 named.ddns.lab
-rw-rw----. 1 named named 509 Mar  5 12:12 named.ddns.lab.view1
```
Прав хватает...., но не хватает...

4. Чтобы долго не искать записи в **/var/log/audit/audit.log** на сервере выполняем
```
tail -f /var/log/audit/audit.log
```

на клиенте попробывал выполнить динамическое обновление зоны
```
nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```

На сервере увидели несколько сообщений, одно из которых следующего содержания:
```
type=AVC msg=audit(1678033045.637:1928): avc:  denied  { create } for  pid=5045 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
```
Делаем вывод, что в SELinux отсутствует правило разрешающее процессу с контекстом `named_t` (соответствующий процессу dns-сервера) работать с файловыми объекстами с контекстом `etc_t`.  
Смотрим контекст директории **/etc/named/dynamic**
```
ls -ldZ /etc/named/dynamic
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   /etc/named/dynamic/
```

### Приступаем к устранению проблемы
Внимательно читаем **man named** в части SELinux...

#### Что можно сделать:
1. Перевести SELinux в  permessive режим или совсем отключить.
2. Назначить "руками" соответствующий контекст на **/etc/named/dynamic/**
3. Перенести файлы в "правильное" место тсправив конфигурацию
4. Использовать утилиту **audit2allow**, для разрешения доступа к файлам

Правильными варинтами, на мой взгляд, являются 2 и 3. Вариант с отключением SELinux с точки зрения безопасности, наименее верный. Про вариант с использованием модуля созданного с применением программы **audit2allow** написано в конце. Поэтому был выбран вариант с назначением контекста "руками" на папку **/etc/named/dynamic/**. К тому же такой подход позволяет выполнить меньше действий для обеспечения работоспособности dns-сервера, даже по сравнению с переносом места файлов динамического обнавления в предусмотренное для них место.

```
[root@ns01 ~]# chcon -t named_cache_t /etc/named/dynamic
```

Пробуем..  
На клиенте получаем:  
!["Success dns update"](https://github.com/mus-cat/otus-study-m3l17/blob/main/II/SuccessUpdate_1.png)
!["Correct DNS answer"](https://github.com/mus-cat/otus-study-m3l17/blob/main/II/SuccessDNSServerAnswer_1.png)
На сервере:  
!["Видим на сервере"](https://github.com/mus-cat/otus-study-m3l17/blob/main/II/OnServerSide_1.png)

Внесены изменениея в исходный файлы **playbook.yml** используемый для создания стенда, добавлено назначение контекство SELinux на нужные объекты файловой системы.

Почему решил не использовать **audit2allow**? Программа создает следующиее правило:
```
[root@ns01 ~]# grep named /var/log/audit/audit.log | audit2allow -m named

module named 1.0;

require {
	type etc_t;
	type named_t;
	class file create;
}

#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
```

Т.е. в этом случае, весь смысл ограничений SELinux на файлы внутри дериктории **/etc** для различных процессов теряется.
Вместе с тем, продолжает действовать стандартная система прав Linux. Что в данном случае не приводит к проблемам с безопасностью при использовании модуля созданного с помощью **audit2allow**, несмотря на то, что сервис named с точки зрения  SELinux имеет права на запись в **/etc**. Данная ситуация может привести к проблемам, если в следствии ошибки на какую-то важную директорию в **/etc** дадут права достаточные чтобы сервис named мог получить к ней доступ.
