# Ejercicio 1 iptables: cortafuegos por nodo

## Preliminares

### Escenario

```ruby
Vagrant.configure("2") do |config|

config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define :ej1iptables1 do |ej1iptables1|
    ej1iptables1.vm.box = "debian/bullseye64"
    ej1iptables1.vm.hostname = "ej1iptables1"
    ej1iptables1.vm.network :private_network,
      :libvirt__network_name => "ej1-iptables-nodo",
      :libvirt__dhcp_enabled => false,
      :ip => "192.168.0.2",
      :libvirt__netmask => '255.255.255.0',
      :libvirt__forward_mode => "veryisolated"
  end

  config.vm.define :ej1iptables2 do |ej1iptables2|
    ej1iptables2.vm.box = "debian/bullseye64"
    ej1iptables2.vm.hostname = "ej1iptables2"
    ej1iptables2.vm.network :private_network,
      :libvirt__network_name => "ej1-iptables-nodo",
      :libvirt__dhcp_enabled => false,
      :ip => "192.168.0.3",
      :libvirt__netmask => '255.255.255.0',
      :libvirt__forward_mode => "veryisolated"
  end

end
```

Instalo un servidor web para dejar abierto el puerto 80:

```console
sudo apt update
sudo apt install apache2
```

### Limpieza de reglas previas

En mi caso no necesito hacer este paso ya que parto de una máquina vagrant nueva.  
Igualmente, dejaré los comandos explicados por aquí.

Borrar todas las reglas de todas las cadenas en la tabla filter:

```console
sudo iptables -F
```

Borrar todas las reglas de todas las cadenas en la tabla nat:

```console
sudo iptables -t nat -F
```

Poner los contadores a cero en todas las cadenas de la tabla filter:

```console
sudo iptables -Z
```

Poner los contadores a cero en todas las cadenas de la tabla nat:

```console
sudo iptables -t nat -Z
```

### Tráfico ssh entrante

Estamos conectados a la máquina por ssh, así que tenemos que permitir el tráfico ssh entrante antes de cambiar las políticas por defecto a DROP *(para no perder la conexión)*:

```console
sudo iptables -A INPUT -s 192.168.121.0/24 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.121.0/24 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

### Cambiar política por defecto

```console
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
```

Debido a estas políticas DROP que hemos aplicado...

No puedo hacer ping a localhost:

```console
vagrant@ej1iptables1:~$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
^C
--- 127.0.0.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2049ms
```

No puedo hacer ping a Internet:

```console
vagrant@ej1iptables1:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
^C
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3062ms
```

Si no se nos corta la conexión ssh a la máquina después de este paso, el par de reglas que aplicamos antes funcionan.

### Tráfico loopback

```console
sudo iptables -A OUTPUT -o lo -p icmp -j ACCEPT
sudo iptables -A INPUT -i lo -p icmp -j ACCEPT
```

Ya funciona el ping a localhost:

```console
vagrant@ej1iptables1:~$ ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.097 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.090 ms
^C
--- 127.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1006ms
rtt min/avg/max/mdev = 0.090/0.093/0.097/0.003 ms
```

### Tráfico ICMP saliente

```console
sudo iptables -A OUTPUT -o eth0 -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i eth0 -p icmp --icmp-type echo-reply -j ACCEPT
```

Ya funciona el ping a Internet:

```console
vagrant@ej1iptables1:~$ ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=11.2 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=17.3 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=46.9 ms
^C
--- 1.1.1.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 11.218/25.145/46.884/15.573 ms
```

### Tráfico DNS saliente

```console
sudo iptables -A OUTPUT -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i eth0 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
```

Pruebo que las consultas DNS funcionan:

```console
vagrant@ej1iptables1:~$ dig @1.1.1.1 www.example.org

; <<>> DiG 9.16.15-Debian <<>> @1.1.1.1 www.example.org
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21960
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;www.example.org.	        IN	A

;; ANSWER SECTION:
www.example.org.	86400	IN	A	93.184.216.34

;; Query time: 176 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue Jan 25 21:14:54 UTC 2022
;; MSG SIZE  rcvd: 60
```

### Tráfico http saliente

```console
sudo iptables -A OUTPUT -o eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
```

Compruebo que funciona:

```console
vagrant@ej1iptables1:~$ curl http://portquiz.net:80
Port test successful!
Your IP: 90.168.73.6
```

### Tráfico https saliente

```console
sudo iptables -A OUTPUT -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -i eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
```

Compruebo que funciona:

```console
vagrant@ej1iptables1:~$ curl portquiz.net:443
Port test successful!
Your IP: 90.168.73.6
```

### Tráfico http entrante

```console
sudo iptables -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -o eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
```

Compruebo que funciona desde mi host:

```console
atlas@olympus:~/vagrant/ej1-iptables-nodo$ telnet 192.168.121.148 80
Trying 192.168.121.148...
Connected to 192.168.121.148.
Escape character is '^]'.

```

## Ejercicios

### Ejercicio 1

> Permitir conexiones ssh al exterior

```console
sudo iptables -A OUTPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

Pruebo que funciona conectando a mi host:

```console
vagrant@ej1iptables1:~$ ssh atlas@192.168.121.1
The authenticity of host '192.168.121.1 (192.168.121.1)' can't be established.
ECDSA key fingerprint is SHA256:xbn++TBj5GyQdixTeO+cP7UGJzvMspD3AVK0JD1RbLw.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.121.1' (ECDSA) to the list of known hosts.
atlas@192.168.121.1's password:
Linux olympus 5.10.0-8-amd64 #1 SMP Debian 5.10.46-4 (2021-08-03) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Fri Sep 17 11:55:03 2021
atlas@olympus:~$
```

### Ejercicio 2

> Denegar tráfico http entrante desde nuestro host

Antes de añadir la regla de bloqueo, debemos tener en cuenta nuestra situación actual de reglas en INPUT:

![reglas input](https://i.imgur.com/0V6MUei.png)

Como se puede ver, si añadiéramos la regla de bloqueo al final no serviría para nada porque siempre haría match con la que permite todo el tráfico http entrante *(la marcada en rojo)*.

¿Solución? Añadirla por encima de la regla marcada en rojo.

Necesitamos saber el número de línea específico, así que ejecutamos:

```console
sudo iptables -L -v -n --line-numbers
```

![número de línea](https://i.imgur.com/dtRzY05.png)

Vamos a añadir la regla de bloqueo en la posición marcada para que funcione:

```console
sudo iptables -I INPUT 7 -s 192.168.121.1 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j DROP
```

![regla drop en posición correcta](https://i.imgur.com/uao9InU.png)

Ya tenemos la regla en la posición que queríamos. Ahora comprobamos que bloquea el tráfico:

```console
atlas@olympus:~/vagrant/ej1-iptables-nodo$ telnet 192.168.121.148 80
Trying 192.168.121.148...

```

### Ejercicio 3

> Permitir tráfico DNS saliente sólo a 192.168.202.2. Comprobar que no se puede hacer un dig @1.1.1.1

Nos encontramos en una situación parecida a la del ejercicio anterior, pero en este caso tenemos que borrar 2 reglas antiguas que permitían el tráfico DNS saliente a todo:  

![reglas a borrar 53](https://i.imgur.com/DF7WB6e.png)

Incluso si añadiéramos las nuevas reglas por encima de estas, cuando hiciésemos dig a 1.1.1.1 llegaría a hacer match con estas reglas y permitiría el tráfico, y eso no es lo que queremos.

Primero, borramos las reglas antiguas:

```console
sudo iptables -D OUTPUT -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -D INPUT -i eth0 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
```

Segundo, añadimos las nuevas reglas:

```console
sudo iptables -A OUTPUT -d 192.168.202.2 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -s 192.168.202.2 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
```

Pruebo que las consultas DNS a `192.168.202.2` funcionan:

```console
vagrant@ej1iptables1:~$ dig @192.168.202.2 www.example.org

; <<>> DiG 9.16.22-Debian <<>> @192.168.202.2 www.example.org
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44911
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a5bc7b7e522180d07f1d986561f12cdcd431eb1130baa84c (good)
;; QUESTION SECTION:
;www.example.org.	        IN	A

;; ANSWER SECTION:
www.example.org.	86010	IN	A	93.184.216.34

;; AUTHORITY SECTION:
example.org.	        72269	IN	NS	a.iana-servers.net.
example.org.	        72269	IN	NS	b.iana-servers.net.

;; ADDITIONAL SECTION:
a.iana-servers.net.	158669	IN	A	199.43.135.53
b.iana-servers.net.	158669	IN	A	199.43.133.53
a.iana-servers.net.	158669	IN	AAAA	2001:500:8f::53
b.iana-servers.net.	158669	IN	AAAA	2001:500:8d::53

;; Query time: 0 msec
;; SERVER: 192.168.202.2#53(192.168.202.2)
;; WHEN: Wed Jan 26 11:13:30 UTC 2022
;; MSG SIZE  rcvd: 224
```

Pruebo que las consultas DNS a `1.1.1.1` no funcionan:

```console
vagrant@ej1iptables1:~$ dig @1.1.1.1 www.example.org

; <<>> DiG 9.16.22-Debian <<>> @1.1.1.1 www.example.org
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

### Ejercicio 4

> Bloquear tráfico http saliente a www.josedomingo.org (utilizar la ip). ¿Se puede acceder a fp.josedomingo.org?

De nuevo, tendríamos una situación de conflicto de reglas si la siguiente la añadiéramos al final, porque antes tenemos una que permite todo el tráfico http saliente y los matchs se quedarían siempre ahí:

![regla 80 todo](https://i.imgur.com/BBqCosU.png)

Tendremos que añadir la regla de bloqueo por encima de la anterior mostrada:

```console
sudo iptables -I OUTPUT 4 -d 37.187.119.60 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j DROP
```

Se nos quedarían las reglas así:

![reglas http después de bloqueo](https://i.imgur.com/Fs6Zm31.png)

Pruebo que **no funciona** el tráfico http saliente a `www.josedomingo.org`:

```console
vagrant@ej1iptables1:~$ curl www.josedomingo.org


```

Se quedará esperando infinitamente porque no hay conexión.

Pruebo a ver si funciona el tráfico http saliente a `fp.josedomingo.org`:

```console
vagrant@ej1iptables1:~$ curl fp.josedomingo.org


```

Tampoco hay conexión. No funciona ya que `fp.josedomingo.org` se acaba traduciendo a la misma IP que hemos bloqueado antes.

Pruebo que cualquier otro tráfico http saliente funciona:

```console
vagrant@ej1iptables1:~$ telnet www.example.org 80
Trying 93.184.216.34...
Connected to www.example.org.
Escape character is '^]'.

```

### Ejercicio 5

> Permitir tráfico saliente a babuino-smtp.gonzalonazareno.org por el puerto 25

```console
sudo iptables -A OUTPUT -d 192.168.203.3 -p tcp --dport 25 -j ACCEPT
sudo iptables -A INPUT -s 192.168.203.3 -p tcp --sport 25 -j ACCEPT
```

Pruebo que funciona con telnet:

```console
vagrant@ej1iptables1:~$ telnet babuino-smtp.gonzalonazareno.org 25
Trying 192.168.203.3...
Connected to babuino-smtp.gonzalonazareno.org.
Escape character is '^]'.
220 babuino-smtp.gonzalonazareno.org ESMTP Postfix (Debian/GNU)

```

### Ejercicio 6

#### Parte 1

> Instalar mariadb-server y habilitar conexiones remotas

```console
sudo apt install mariadb-server
```

`/etc/mysql/mariadb.conf.d/50-server.cnf`:

```console
bind-address            = 0.0.0.0
```

Reinicio:

```console
sudo systemctl restart mariadb
```

#### Parte 2

> Permitir tráfico entrante desde mi host

```console
sudo iptables -A INPUT -s 192.168.121.1 -p tcp --dport 3306 -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.121.1 -p tcp --sport 3306 -j ACCEPT
```

Pruebo que funciona:

```console
atlas@olympus:~/vagrant/ej1-iptables-nodo$ nc -zvw10 192.168.121.148 3306
Connection to 192.168.121.148 3306 port [tcp/mysql] succeeded!
```

#### Parte 3

> Comprobar que desde otro cliente bloquea el tráfico

```console
vagrant@ej1iptables2:~$ nc -zvw10 192.168.0.2 3306
192.168.0.2: inverse host lookup failed: Unknown host
(UNKNOWN) [192.168.0.2] 3306 (mysql) : Connection timed out
```
