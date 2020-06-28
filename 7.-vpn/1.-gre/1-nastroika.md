# Настройка

Настраивается GRE-туннель следующим образом:

```text
interface Tunnel0
ip address 10.2.2.1 255.255.255.252
```

Поскольку туннель является виртуальным L3 интерфейсом, через который у нас будет происходить маршрутизация, ему должен быть назначен IP-адрес, который выбирается согласно вашему IP-плану, вероятно, из приватной сети.

В качестве адреса источника можно выбрать как IP-адрес выходного интерфейса  
\(белый адрес, предоставленный провайдером\), так и его имя \(FE0/0 в нашем случае\):

```text
tunnel source 100.0.0.1
```

Адрес destination – публичный адрес удалённой стороны:

```text
tunnel destination 200.0.0.1
```

Законченный вид:

```text
interface Tunnel0
ip address 10.2.2.1 255.255.255.252
tunnel source 100.0.0.1
tunnel destination 200.0.0.1
```

Сразу после этого туннель должен подняться:

> R1\#sh int tun 0  
> **Tunnel0 is up, line protocol is up**  
> Hardware is Tunnel  
> Internet address is 10.2.2.1/30  
> **MTU 1514 bytes**, BW 9 Kbit, DLY 500000 usec,  
> reliability 255/255, txload 1/255, rxload 1/255  
> Encapsulation TUNNEL, loopback not set  
> Keepalive not set  
> **Tunnel source 100.0.0.1, destination 200.0.0.1  
> Tunnel protocol/transport GRE/IP**

Вся основная информация здесь отражена. Обратите внимание на размер MTU – он не 1500, как ставится для обычных физических интерфейсов. О параметре MTU мы поговорим в конце статьи

> По умолчанию GRE не проверяет доступность адреса назначения и сразу отправляет туннель в Up. Но стоит только добавить в туннельный интерфейс команду **keepalive X**, как маршрутизатор начинает отсылать кипалайвы и не поднимется, пока не будет ответа.

В нашей тестовой схеме в качестве локальной сети мы просто настроили Loopback-интерфейсы – они всегда в Up'е. Дали им адреса с маской /32. На самом же деле под ними подразумеваются реальные подсети вашего предприятия \(ну, как на картинке\).

```text
interface Loopback0
ip address 10.0.0.0 255.255.255.255
```

На маршрутизаторе у вас должно быть два статических маршрута:

```text
ip route 0.0.0.0 0.0.0.0 100.0.0.2
ip route 10.1.1.0 255.255.255.255 10.2.2.2
```

Первый говорит о том, что шлюзом по умолчанию является адрес провайдера 100.0.0.2:

> R1\#traceroute 200.0.0.1  
> Type escape sequence to abort.  
> Tracing the route to 200.0.0.1  
> 1 **100.0.0.2** 56 msec 48 msec 36 msec  
> 2 200.0.0.1 64 msec \* 60 msec

Второй перенаправляет пакеты, адресованные хосту с адресом 10.1.1.0, на next-hop 10.2.2.2 – это адрес туннеля с обратной стороны.

GRE-туннели являются однонаправленными, и обычно подразумевается наличие обратного туннеля на другой стороне, хотя вообще говоря, это необязательно. Но в нашем случае, когда посередине Интернет, и задача – организовать приватную сеть, с обратной стороны должна быть симметричная настройка:

nsk-obsea-gw1:

```text
interface Tunnel0
ip address 10.2.2.2 255.255.255.252
tunnel source 200.0.0.1
tunnel destination 100.0.0.1
ip route 0.0.0.0 0.0.0.0 200.0.0.2
ip route 10.0.0.0 255.255.255.255 10.2.2.1
```

Пробуем запустить пинг:

> R1\#ping 10.1.1.0  
> Type escape sequence to abort.  
> Sending 5, 100-byte ICMP Echos to 10.1.1.0, timeout is 2 seconds:  
> !!!  
> Success rate is 100 percent \(5/5\), round-trip min/avg/max = 44/71/136 ms  
> R1\#tracer 10.1.1.0  
> Type escape sequence to abort.  
> Tracing the route to 10.1.1.0  
> 1 10.2.2.2 68 msec \* 80 msec
