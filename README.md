# **DNS - настройка и обслуживание. DHCP.**

## **Homework**

Настраиваем split-dns:

- взять стенд https://github.com/erlong15/vagrant-bind
- добавить еще один сервер client2
- завести в зоне dns.lab
- имена
- web1 - смотрит на клиент1
- web2 смотрит на клиент2

- завести еще одну зону newdns.lab
- завести в ней запись
- www - смотрит на обоих клиентов

- настроить split-dns
- клиент1 - видит обе зоны, но в зоне dns.lab только web1

- клиент2 видит только dns.lab

*)
- настроить все без выключения selinux
- ddns тоже должен работать без выключения selinux

**A Bind's DNS lab with Vagrant and Ansible, based on CentOS 7.**

## **Playground**

vagrant ssh client:
- zones: dns.lab, reverse dns.lab and ddns.lab
- ns01 (192.168.50.10)

    master, recursive, allows update to ddns.lab

- ns02 (192.168.50.11)
  
    slave, recursive
 
- client (192.168.50.15)
    
- client2 (192.168.50.16)

- zone transfer: TSIG key
    
Из-за необходимости добавить сервер client2, в named.dns.lab добавим web1 IN A 192.168.50.15 и web2 IN A 192.168.50.16.

Заводим  еще одну зону newdns.lab и запись в ней:

newdns.lab.     IN      A       192.168.50.15
newdns.lab.     IN      A       192.168.50.16
www             IN      CNAME   newdns.lab.

В master-named.conf добавим указание на загрузку зоны.

Чтобы настроить split-dns: клиент1 - видит обе зоны, но в зоне dns.lab только web1, в master-named.conf используем:

- acl - определяет списки доступа клиентов к dns серверу;
- view - ограничивает выдачу ответов от dns сервера в соответсвие с указанными внутри параметрами и перечисленными зонами(match-clients - выбирает вид в зависимости от адреса клиента и allow-query - разрешает клиенту запросы к dns)

С помощью view создадим в зоне dns.lab только web1.dns.lab.

**Проверка прошла успешно:**

[root@admin OtusHW21]# vagrant ssh client

DEPRECATION: The 'sudo' option for the Ansible provisioner is deprecated.

Please use the 'become' option instead.

The 'sudo' option will be removed in a future release of Vagrant.

Last login: Thu Apr 25 17:59:09 2019 from 10.0.2.2

### Welcome to the DNS lab! ###

- Use this client to test the enviroment, with dig or nslookup.

    dig @192.168.50.10 ns01.dns.lab

    dig @192.168.50.11 -x 192.168.50.10

- nsupdate is available in the ddns.lab zone. Ex:

    nsupdate -k /etc/named.zonetransfer.key

    server 192.168.50.10

    zone ddns.lab 

    update add www.ddns.lab. 60 A 192.168.50.15

    send

- rndc is also available to manage the servers

    rndc -c ~/rndc.conf reload

Enjoy!

[vagrant@client ~]$ sudo -i

[root@client ~]#  ping web1.dns.lab

PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.044 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.062 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.064 ms

^C

--- web1.dns.lab ping statistics ---

3 packets transmitted, 3 received, 0% packet loss, time 2001ms

rtt min/avg/max/mdev = 0.044/0.056/0.064/0.012 ms

[root@client ~]# ping web2.dns.lab

ping: web2.dns.lab: Name or service not known

[root@client ~]# ping www.newdns.lab

PING newdns.lab (192.168.50.15) 56(84) bytes of data.

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.037 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.064 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.075 ms

^C

--- newdns.lab ping statistics ---

3 packets transmitted, 3 received, 0% packet loss, time 2002ms

rtt min/avg/max/mdev = 0.037/0.058/0.075/0.018 ms

**Проверим с помощью утилиты dig:**

[root@client ~]# dig any web1.dns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any web1.dns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 10705

;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;web1.dns.lab.			IN	ANY

;; ANSWER SECTION:

web1.dns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:

dns.lab.		3600	IN	NS	ns01.dns.lab.

dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:

ns01.dns.lab.		3600	IN	A	192.168.50.10

ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:08:14 UTC 2019

;; MSG SIZE  rcvd: 127

[root@client ~]# dig any web2.dns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any web2.dns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 40798

;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;web2.dns.lab.			IN	ANY

;; AUTHORITY SECTION:

dns.lab.		600	IN	SOA	ns01.dns.lab. root.dns.lab. 2901201901 3600 600 86400 600

;; Query time: 1 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:09:23 UTC 2019

;; MSG SIZE  rcvd: 87

[root@client ~]#  dig any www.newdns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any www.newdns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55493

;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;www.newdns.lab.			IN	ANY

;; ANSWER SECTION:

www.newdns.lab.		3600	IN	CNAME	newdns.lab.

;; AUTHORITY SECTION:

newdns.lab.		3600	IN	NS	ns02.newdns.lab.

newdns.lab.		3600	IN	NS	ns01.newdns.lab.

;; ADDITIONAL SECTION:

ns01.newdns.lab.	3600	IN	A	192.168.50.10

ns02.newdns.lab.	3600	IN	A	192.168.50.11

;; Query time: 2 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:10:19 UTC 2019

;; MSG SIZE  rcvd: 127

А теперь настоим split-dns, где клиент2 видит только dns.lab:

acl web2 { 192.168.50.16; };

...

view "web2" {

        match-clients { web2; };

        allow-query { web2; };

        zone "dns.lab" { ... };

        zone "50.168.192.in-addr.arpa" { ... };

};

**Проверка прошла успешно:**

[root@admin OtusHW21]# vagrant ssh client2

DEPRECATION: The 'sudo' option for the Ansible provisioner is deprecated.

Please use the 'become' option instead.

The 'sudo' option will be removed in a future release of Vagrant.

Last login: Thu Apr 25 18:02:17 2019 from 10.0.2.2

### Welcome to the DNS lab! ###

- Use this client to test the enviroment, with dig or nslookup.

    dig @192.168.50.10 ns01.dns.lab

    dig @192.168.50.11 -x 192.168.50.10

- nsupdate is available in the ddns.lab zone. Ex:

    nsupdate -k /etc/named.zonetransfer.key

    server 192.168.50.10

    zone ddns.lab 

    update add www.ddns.lab. 60 A 192.168.50.15

    send

- rndc is also available to manage the servers

    rndc -c ~/rndc.conf reload

Enjoy!

[vagrant@client2 ~]$ sudo -i

[root@client2 ~]# ping www.dns.lab

ping: www.dns.lab: Name or service not known

[root@client2 ~]# ping web1.dns.lab

PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=1.19 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.571 ms

^C

--- web1.dns.lab ping statistics ---

2 packets transmitted, 2 received, 0% packet loss, time 1001ms

rtt min/avg/max/mdev = 0.571/0.881/1.191/0.310 ms

[root@client2 ~]# ping web2.dns.lab

PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.

64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=1 ttl=64 time=0.078 ms

64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=2 ttl=64 time=0.079 ms

^C

--- web2.dns.lab ping statistics ---

2 packets transmitted, 2 received, 0% packet loss, time 1002ms

rtt min/avg/max/mdev = 0.078/0.078/0.079/0.008 ms

**Проверка с утилитой dig:**

[root@client2 ~]#  dig any www.newdns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any www.newdns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 33859

;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;www.newdns.lab.			IN	ANY

;; AUTHORITY SECTION:

.			10800	IN	SOA	a.root-servers.net. nstld.verisign-grs.com. 2019042500 1800 900 604800 86400


;; Query time: 221 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:16:01 UTC 2019

;; MSG SIZE  rcvd: 118

[root@client2 ~]#  dig any web1.dns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any web1.dns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57290

;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;web1.dns.lab.			IN	ANY

;; ANSWER SECTION:

web1.dns.lab.		3600	IN	A	192.168.50.15

;; AUTHORITY SECTION:

dns.lab.		3600	IN	NS	ns01.dns.lab.

dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:

ns01.dns.lab.		3600	IN	A	192.168.50.10

ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:16:25 UTC 2019

;; MSG SIZE  rcvd: 127

[root@client2 ~]#  dig any web2.dns.lab

; <<>> DiG 9.9.4-RedHat-9.9.4-73.el7_6 <<>> any web2.dns.lab

;; global options: +cmd

;; Got answer:

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48902

;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:

; EDNS: version: 0, flags:; udp: 4096

;; QUESTION SECTION:

;web2.dns.lab.			IN	ANY

;; ANSWER SECTION:

web2.dns.lab.		3600	IN	A	192.168.50.16

;; AUTHORITY SECTION:

dns.lab.		3600	IN	NS	ns01.dns.lab.

dns.lab.		3600	IN	NS	ns02.dns.lab.

;; ADDITIONAL SECTION:

ns01.dns.lab.		3600	IN	A	192.168.50.10

ns02.dns.lab.		3600	IN	A	192.168.50.11

;; Query time: 1 msec

;; SERVER: 192.168.50.10#53(192.168.50.10)

;; WHEN: Thu Apr 25 18:17:42 UTC 2019

;; MSG SIZE  rcvd: 127

** *) настроить все без выключения selinux и ddns тоже должен работать без выключения selinux**

Bind записывает свои конфигурационные файлы по умолчанию - /var/named 

[root@client2 ~]# semanage fcontext -l | grep /var/named/chroot/var/named

**/var/named/chroot/var/named(/.*)?**                  all files          system_u:object_r:named_zone_t:s0 

/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0 

/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0 

/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0 

/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0 

Установим и применим контекст безопасности named_zone_t в директориях через provision ansible для директории и файлов /etc/named и включим SELinux:

- name: SElinux fix for /etc/named

    sefcontext:

      target: "/etc/named(/.*)?"

      setype: named_zone_t

      state: present

  - name: Apply new SELinux file context to filesystem

    command: restorecon -R -v /etc/named

  - name: Enable SELinux

    selinux:

      policy: targeted

      state: enforcing   

Проверка прошла успешно:

[vagrant@client ~]$ sudo -i

[root@client ~]# nsupdate -k /etc/named.zonetransfer.key

> server 192.168.50.10

> update add www.ddns.lab. 60 A 192.168.50.15

> send

> ^Z

[1]+  Stopped                 nsupdate -k /etc/named.zonetransfer.key

[root@client ~]# ping www.ddns.lab

PING www.ddns.lab (192.168.50.15) 56(84) bytes of data.

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.044 ms

64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.057 ms

^C

--- www.ddns.lab ping statistics ---

2 packets transmitted, 2 received, 0% packet loss, time 1001ms

rtt min/avg/max/mdev = 0.044/0.050/0.057/0.009 ms



