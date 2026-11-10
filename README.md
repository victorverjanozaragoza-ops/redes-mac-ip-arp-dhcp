# Práctica P1.4: MAC vs IP — ¿Quién traduce a quién?

<p align="center">
  <a href="./practica-mac-arp-dhcp-verjano.pdf" download>
    <img src="https://img.shields.io/badge/📥_Descargar_Práctica_en_PDF-red?style=for-the-badge&logo=adobeacrobatreader&logoColor=white" alt="Descargar PDF" height="40"/>
  </a>
</p>

<p align="center">
  <a href="./practica-mac-arp-dhcp-verjano.pdf"><b>Descargar o ver la memoria completa en PDF</b></a>
</p>

---

* **Alumno:** Víctor Verjano
* **Curso:** 2º ASIR
* **Módulo:** Servicios de Red e Internet (SRI)
* **Práctica:** P1.4 — Redes: MAC vs IP, ARP y DHCP

---

## Escenario de la práctica

He configurado la máquina virtual Debian 13 en **modo puente (bridge)** sobre la interfaz de red del equipo Host del aula (MAX). De esta forma, tanto el Host como la VM comparten el mismo dominio de difusión local en la red `192.168.3.0/24`:

```text
  Host (PC MAX del aula)                   VM Debian (modo puente)
┌─────────────────────────┐             ┌─────────────────────────┐
│ MAC: 50:eb:f6:3c:2a:11  │ ─── LAN ─── │ MAC: 08:00:27:d4:1a:89  │
│ IP:  192.168.3.45       │   del aula  │ IP:  192.168.3.120      │
└─────────────────────────┘             └─────────────────────────┘
             ▲                                       ▲
             └────────── ¿Quién traduce IP ↔ MAC? ───┘
                                  ARP
```

---

## Parte 1: Dos direcciones para una misma tarjeta

Compruebo las direcciones de Capa 2 (enlace) y Capa 3 (red) con `ip link show` e `ip -br a`:

```bash
# En la VM Debian:
victor@debian:~$ ip link show
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:d4:1a:89 brd ff:ff:ff:ff:ff:ff

victor@debian:~$ ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp0s3           UP             192.168.3.120/24 fe80::a00:27ff:fed4:1a89/64
```

| Dato | Host (MAX del aula) | VM (Debian 13) |
| :--- | :--- | :--- |
| **Interfaz de red** | `enp2s0` | `enp0s3` |
| **MAC (Capa 2)** | `50:eb:f6:3c:2a:11` | `08:00:27:d4:1a:89` |
| **IP (Capa 3)** | `192.168.3.45/24` | `192.168.3.120/24` |

---

## Parte 2: De dónde sacó la VM su IP (DHCP en vivo)

En Debian 13 instalo el cliente DHCP clásico para forzar la petición en modo verbose y ver las fases DORA:

```bash
victor@debian:~$ sudo apt install -y isc-dhcp-client
victor@debian:~$ sudo dhclient -v enp0s3
Internet Systems Consortium DHCP Client 4.4.3-P1
Listening on LPF/enp0s3/08:00:27:d4:1a:89
Sending on   LPF/enp0s3/08:00:27:d4:1a:89
Sending on   Socket/fallback
DHCPDISCOVER on enp0s3 to 255.255.255.255 port 67 interval 3
DHCPOFFER of 192.168.3.120 from 192.168.3.1
DHCPREQUEST for 192.168.3.120 on enp0s3 to 255.255.255.255 port 67
DHCPACK of 192.168.3.120 from 192.168.3.1
bound to 192.168.3.120 -- renewal in 3600 seconds.
```

> **Detalle clave:** El mensaje `DHCPDISCOVER` se envía a la dirección MAC de difusión `ff:ff:ff:ff:ff:ff` porque al arrancar la máquina no tiene IP asignada ni conoce la IP del servidor DHCP. La MAC es la única identidad que tiene la tarjeta en ese momento.

---

## Parte 3: La tabla ARP — Cómo se traduce IP a MAC

Borro la tabla de vecinos para partir de cero y hago un ping al Host desde la VM:

```bash
# 1. Vaciar tabla ARP en la VM
victor@debian:~$ sudo ip neigh flush all
victor@debian:~$ ip neigh
# Sale vacía

# 2. Hacer ping al Host
victor@debian:~$ ping -c 3 192.168.3.45
PING 192.168.3.45 (192.168.3.45) 56(84) bytes of data.
64 bytes from 192.168.3.45: icmp_seq=1 ttl=64 time=0.452 ms
64 bytes from 192.168.3.45: icmp_seq=2 ttl=64 time=0.312 ms
64 bytes from 192.168.3.45: icmp_seq=3 ttl=64 time=0.298 ms

# 3. Comprobar la tabla de nuevo
victor@debian:~$ ip neigh
192.168.3.45 dev enp0s3 lladdr 50:eb:f6:3c:2a:11 REACHABLE
192.168.3.1 dev enp0s3 lladdr 00:1a:2b:3c:4d:5e REACHABLE
```

La MAC del Host ha aparecido en la tabla asociada a su IP. El kernel ha usado ARP para traducir la IP de destino a la MAC física antes de mandar la trama.

---

## Parte 4: Captura de la conversación ARP con tcpdump

Como en el Host del aula no tenemos permisos de `sudo`, uso la **Opción A** (capturar desde la propia VM). Pongo el filtro `host 192.168.3.45` para no mezclar el tráfico con el resto de compañeros:

```bash
# Terminal 2 en la VM:
victor@debian:~$ sudo tcpdump -i enp0s3 -e arp -n host 192.168.3.45
listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes

17:15:02.102345 08:00:27:d4:1a:89 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.3.45 tell 192.168.3.120, length 28
17:15:02.102891 50:eb:f6:3c:2a:11 > 08:00:27:d4:1a:89, ethertype ARP (0x0806), length 42: Reply 192.168.3.45 is-at 50:eb:f6:3c:2a:11, length 28
```

* **Request:** La VM pregunta en broadcast a todos los equipos quién tiene la IP `192.168.3.45`.
* **Reply:** El Host responde en unicast a la MAC de la VM diciendo que él tiene esa IP y entregando su dirección MAC (`50:eb:f6:3c:2a:11`).

---

## Tabla de datos de la práctica

| Dato | Valor |
| :--- | :--- |
| **MAC Host / MAC VM** | `50:eb:f6:3c:2a:11` / `08:00:27:d4:1a:89` |
| **IP Host / IP VM** | `192.168.3.45` / `192.168.3.120` |
| **Fases dhclient -v** | `DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST`, `DHCPACK` |
| **ip neigh antes del ping** | Vacía (tras ejecutar `flush all`) |
| **ip neigh después del ping** | `192.168.3.45 dev enp0s3 lladdr 50:eb:f6:3c:2a:11 REACHABLE` |
| **Captura tcpdump** | `Request who-has 192.168.3.45 tell 192.168.3.120`<br>`Reply 192.168.3.45 is-at 50:eb:f6:3c:2a:11` |
| **Opción usada en Parte 4** | **Opción A** (captura ejecutada en la VM) |

---

## Preguntas de cierre

### 1. El switch del aula reenvía tramas sin saber nada de direcciones IP. ¿Por qué le basta con la MAC? ¿Y por qué un router necesita trabajar con IPs?
El switch trabaja en Capa 2 (enlace de datos) y solo interconecta dispositivos dentro de la misma red local. Para enviar los datos al equipo correcto solo necesita leer la MAC de destino en la cabecera Ethernet y buscar en su tabla CAM por qué puerto físico está conectado ese equipo. No le hace falta mirar IPs porque no cambia de red. 
El router, en cambio, opera en Capa 3 (red) y conecta redes y subredes distintas. Las MACs son direcciones locales que no viajan por Internet (se sustituyen en cada salto de router). El router necesita las direcciones IP para consultar su tabla de rutas y decidir por qué interfaz enviar el paquete hacia la red de destino.

### 2. Si clonas una VM sin regenerar su adaptador de red, a veces dos VMs terminan con la misma MAC. ¿Qué problema concreto daría eso en la tabla ARP? ¿Y si se repitiera la IP?
Si dos VMs tienen la misma MAC, el switch se vuelve loco alternando el puerto en su tabla CAM (*MAC flapping*) y enviará las tramas a una u otra de forma impredecible. 
Si lo que se repite es la IP (conflicto de IP con dos MACs distintas), cuando otro equipo haga una petición ARP recibirá dos respuestas Reply diferentes con distinta MAC. La tabla ARP del resto de máquinas se sobrescribirá todo el rato con la última que llegue, provocando desconexiones continuas, pérdidas de paquetes y cortes en las conexiones TCP de las dos máquinas.

### 3. ¿Por qué el DHCPDISCOVER tiene que ir en broadcast de MAC (ff:ff:ff:ff:ff:ff) y no dirigido a una IP concreta?
Porque al iniciar el equipo todavía no tiene dirección IP configurada (su IP de origen es `0.0.0.0`) y tampoco sabe qué IP tiene el servidor DHCP de la red. Como no tiene una IP de destino a la que enviar un mensaje directo, tiene que emitir una trama en difusión (`ff:ff:ff:ff:ff:ff`) a nivel de capa 2 para que llegue a todos los equipos del segmento y el servidor DHCP pueda escuchar la petición y ofrecerle una IP.

### 4. ¿Por qué la Opción A de la Parte 4 funciona igual de bien que capturar desde el Host? ¿Qué tendría que pasar para que no funcionara?
Funciona igual de bien porque el intercambio ARP es entre el Host y la VM directamente en la misma red local. La VM envía la pregunta en broadcast y el Host le responde directamente a ella en unicast. Como la VM forma parte de la conversación, ve pasar tanto la petición como la respuesta por su propia tarjeta de red.
Para que no funcionara, el Host y la VM tendrían que estar en redes diferentes separadas por un router (los routers no dejan pasar el broadcast de ARP), o la VM tendría que estar configurada en un modo de red aislado como NAT o Red Interna en VirtualBox.
