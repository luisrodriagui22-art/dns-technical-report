# Informe técnico: Servicio DNS con BIND9 en Debian 13

## Introducción

El sistema DNS (Domain Name System) permite convertir nombres de dominio legibles, por ejemplo `www.LARA.local`, en direcciones IP utilizadas por los equipos de una red. En este proyecto se instaló y configuró un servidor DNS con BIND9 sobre una máquina virtual Debian 13, con el propósito de proporcionar resolución de nombres en una red de laboratorio.

El entorno se implementó con Oracle VirtualBox. La máquina virtual del servidor utiliza dos adaptadores: uno en modo NAT, destinado al acceso a Internet y a la descarga de paquetes; y otro conectado a una red interna, destinado a la comunicación aislada con los clientes de la práctica. Esta separación mejora el control del laboratorio y evita exponer directamente el servidor DNS.

## Características de hardware y sistema operativo

| Elemento | Configuración utilizada |
|---|---|
| Plataforma de virtualización | Oracle VirtualBox |
| Máquina servidor | Máquina virtual Debian 13 (Trixie) |
| Servicio DNS | BIND9 |
| Adaptador 1 (ens0p3) | NAT con DHCP, para acceso a Internet y actualizaciones |
| Adaptador 2 (ens0p8) | Red interna con IP estática 192.168.6.100/24 |
| Nombre del servidor | ns |
| Dominio | LARA.local |
| FQDN del servidor | ns.LARA.local |
| Forwarders DNS | 1.1.1.1 y 8.8.8.8 |
| Privilegios requeridos | Usuario con `sudo` o cuenta `root` |
| Herramientas de diagnóstico | `dig`, `nslookup`, `host`, `named-checkconf` y `named-checkzone` |

La dirección del adaptador NAT (ens0p3) se obtiene automáticamente mediante DHCP y se utiliza únicamente para acceder a repositorios y resolvedores externos. La interfaz de red interna (ens0p8) se configuró con IP estática `192.168.6.100/24`, ya que los clientes DNS deben conocer siempre la dirección del servidor.

## Desarrollo

### Preparación de red

En VirtualBox se configuraron los siguientes adaptadores:

- Adaptador 1 (ens0p3): **NAT**, habilitado para que Debian pueda instalar paquetes y realizar actualizaciones. Se configura con DHCP.
- Adaptador 2 (ens0p8): **Red interna**, con el mismo nombre de red interna en el servidor y las máquinas cliente. Se configura con IP estática.

La configuración de red se realizó editando el archivo `/etc/network/interfaces`:

```bash
sudo nano /etc/network/interfaces
```

Contenido configurado:

```
# Interfaz loopback
auto lo
iface lo inet loopback

# Adaptador 1 - NAT con DHCP
auto ens0p3
iface ens0p3 inet dhcp

# Adaptador 2 - Red interna con IP estática
auto ens0p8
iface ens0p8 inet static
    address 192.168.6.100
    netmask 255.255.255.0
    network 192.168.6.0
    broadcast 192.168.6.255
```

Después de guardar, se reinició el servicio de red o la máquina virtual:

```bash
sudo systemctl restart networking
# o
sudo reboot
```

Para comprobar las interfaces y direcciones disponibles se utiliza:

```bash
ip a
ip route
```

### Instalación de BIND9

Primero se actualizaron los repositorios y se instalaron el servidor DNS y las herramientas de consulta:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils -y
```

### Configuración de red y archivo /etc/network/interfaces

Se editó el archivo de configuración de interfaces para asignar las direcciones IP correspondientes a cada adaptador:

```bash
sudo nano /etc/network/interfaces
```

Contenido aplicado:

```
auto lo
iface lo inet loopback

auto ens0p3
iface ens0p3 inet dhcp

auto ens0p8
iface ens0p8 inet static
    address 192.168.6.100
    netmask 255.255.255.0
```

### Configuración general de BIND9

El archivo principal de opciones se encuentra en:

```text
/etc/bind/named.conf.options
```

Configuración aplicada:

```conf
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 127.0.0.1; 192.168.6.0/24; };
    allow-query { 127.0.0.1; 192.168.6.0/24; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    listen-on { 127.0.0.1; 192.168.6.100; };
    listen-on-v6 { none; };
};
```

Los forwarders `1.1.1.1` y `8.8.8.8` permiten resolver dominios de Internet cuando el servidor no tiene autoridad para el dominio consultado.

### Creación de zonas directas e inversas

**Paso 1: Crear directorio de zonas**

```bash
sudo mkdir -p /etc/bind/zones
```

**Paso 2: Declarar zona directa en named.conf.local**

```bash
sudo nano /etc/bind/named.conf.local
```

Contenido añadido:

```conf
zone "LARA.local" {
    type master;
    file "/etc/bind/zones/db.LARA.local";
};

zone "6.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.6";
};
```

**Paso 3: Crear archivo de zona directa**

```bash
sudo nano /etc/bind/zones/db.LARA.local
```

Contenido:

```dns
$TTL 86400
@   IN  SOA ns.LARA.local. admin.LARA.local. (
        2026092901 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      ns.LARA.local.
ns      IN  A       192.168.6.100
www     IN  A       192.168.6.100
mail    IN  A       192.168.6.20
ftp     IN  CNAME   www.LARA.local.
```

**Paso 4: Crear archivo de zona inversa**

```bash
sudo nano /etc/bind/zones/db.192.168.6
```

Contenido:

```dns
$TTL 86400
@   IN  SOA ns.LARA.local. admin.LARA.local. (
        2026092901 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      ns.LARA.local.
100     IN  PTR     ns.LARA.local.
20      IN  PTR     mail.LARA.local.
```

### Arranque del servicio DNS

Después de configurar las zonas, se habilitó e inició el servicio:

```bash
sudo systemctl enable --now named
sudo systemctl restart named
sudo systemctl status named
```

El estado debe mostrar `active (running)` en verde.

### Validación de configuración

Antes de realizar pruebas, se verificó la sintaxis de los archivos:

```bash
sudo named-checkconf
sudo named-checkzone LARA.local /etc/bind/zones/db.LARA.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.192.168.6
```

Ambas zonas deben mostrar `OK`.

### Pruebas con nslookup

Se realizaron pruebas de resolución DNS utilizando `nslookup`:

**Prueba 1: Consulta de zona directa**

```bash
nslookup www.LARA.local 192.168.6.100
```

Resultado esperado:
```
Server:         192.168.6.100
Address:        192.168.6.100#53

Name:   www.LARA.local
Address: 192.168.6.100
```

**Prueba 2: Consulta de zona inversa**

```bash
nslookup 192.168.6.100 192.168.6.100
```

Resultado esperado:
```
Server:         192.168.6.100
Address:        192.168.6.100#53

100.6.168.192.in-addr.arpa    name = ns.LARA.local.
```

**Prueba 3: Consulta del servidor de nombres**

```bash
nslookup ns.LARA.local 192.168.6.100
```

**Prueba 4: Consulta de dominio externo (vía forwarders)**

```bash
nslookup google.com 192.168.6.100
```

Todas las pruebas se realizaron correctamente, confirmando que el servidor DNS resuelve tanto nombres locales como externos.

## Análisis de incidencias técnicas

| Incidencia o fallo | Causa probable | Comprobación | Solución recomendada |
|---|---|---|---|
| `Unable to locate package bind9` | Índice de paquetes sin actualizar o repositorios mal configurados | `sudo apt update` | Revisar `/etc/apt/sources.list` y ejecutar de nuevo `sudo apt update` antes de instalar. |
| No hay acceso a Internet durante la instalación | Adaptador NAT desactivado o ruta por defecto ausente | `ip route`, `ping -c 3 1.1.1.1` | Activar el adaptador NAT en VirtualBox y comprobar que existe una ruta por defecto. |
| El servicio no inicia después de instalarlo | Error de sintaxis, configuración incompleta o nombre de unidad distinto | `sudo systemctl status named`, `journalctl -u named -n 50` | Validar con `sudo named-checkconf`; comprobar si el servicio instalado se llama `named` o `bind9`. |
| `SERVFAIL` en nslookup | Archivo de zona con error de sintaxis o serial incorrecto | `sudo named-checkzone LARA.local /etc/bind/zones/db.LARA.local` | Corregir el registro indicado, verificar puntos finales en FQDN y aumentar el serial. |
| `NXDOMAIN` en nslookup | El nombre consultado no existe en la zona | `nslookup nombre.LARA.local 192.168.6.100` | Crear o corregir el registro A, CNAME o PTR correspondiente; recargar BIND9. |
| Timeout en nslookup | Firewall bloquea puerto 53 o BIND no escucha en la IP interna | `sudo ss -tulpn | grep ':53'` | Permitir UDP/TCP 53 y ajustar `listen-on` para que incluya 192.168.6.100. |
| Consultas externas fallidas | Forwarders inaccesibles o NAT sin salida | `ping -c 3 1.1.1.1` | Verificar conectividad NAT y usar reenviadores accesibles (1.1.1.1, 8.8.8.8). |

Los registros del servicio son especialmente útiles para localizar errores:

```bash
sudo journalctl -u named -f
```

## Conclusiones

La práctica permitió implementar un servidor DNS BIND9 funcional en Debian 13 dentro de VirtualBox. El proceso completo incluyó:

1. Configuración de red con dos adaptadores (ens0p3 en NAT/DHCP, ens0p8 en red interna con IP 192.168.6.100)
2. Instalación de BIND9 y herramientas de diagnóstico
3. Configuración de opciones generales con forwarders 1.1.1.1 y 8.8.8.8
4. Creación de zona directa (LARA.local) y zona inversa (6.168.192.in-addr.arpa)
5. Arranque del servicio DNS con systemctl
6. Validación con named-checkconf y named-checkzone
7. Pruebas exitosas con nslookup para resolución directa e inversa

La estabilidad del servicio depende de utilizar una IP fija en la interfaz interna, validar todos los archivos antes de reiniciar BIND9, restringir las consultas recursivas a redes de confianza y comprobar el estado del puerto 53. Las pruebas con nslookup confirmaron que el servidor resuelve correctamente tanto nombres del dominio local LARA.local como dominios externos de Internet.
