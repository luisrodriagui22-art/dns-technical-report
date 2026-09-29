# Informe técnico: Servicio DNS con BIND9 en Debian 13

## Introducción

El sistema DNS (Domain Name System) permite convertir nombres de dominio legibles, por ejemplo `www.empresa.local`, en direcciones IP utilizadas por los equipos de una red. En este proyecto se instaló y configuró un servidor DNS con BIND9 sobre una máquina virtual Debian 13, con el propósito de proporcionar resolución de nombres en una red de laboratorio.

El entorno se implementó con Oracle VirtualBox. La máquina virtual del servidor utiliza dos adaptadores: uno en modo NAT, destinado al acceso a Internet y a la descarga de paquetes; y otro conectado a una red interna, destinado a la comunicación aislada con los clientes de la práctica. Esta separación mejora el control del laboratorio y evita exponer directamente el servidor DNS.

## Características de hardware y sistema operativo

| Elemento | Configuración utilizada |
|---|---|
| Plataforma de virtualización | Oracle VirtualBox |
| Máquina servidor | Máquina virtual Debian 13 (Trixie) |
| Servicio DNS | BIND9 |
| Adaptador 1 | NAT, para acceso a Internet y actualizaciones |
| Adaptador 2 | Red interna, para comunicación entre servidor y clientes |
| Privilegios requeridos | Usuario con `sudo` o cuenta `root` |
| Herramientas de diagnóstico | `dig`, `nslookup`, `host`, `named-checkconf` y `named-checkzone` |

La dirección del adaptador NAT suele ser dinámica y se utiliza únicamente para acceder a repositorios y resolvedores externos. La interfaz de red interna debe configurarse con una IP estática, ya que los clientes DNS deben conocer siempre la dirección del servidor.

## Desarrollo

### Preparación de red

En VirtualBox se configuraron los siguientes adaptadores:

- Adaptador 1: **NAT**, habilitado para que Debian pueda instalar paquetes y realizar actualizaciones.
- Adaptador 2: **Red interna**, con el mismo nombre de red interna en el servidor y las máquinas cliente.

En el servidor, se recomienda configurar una dirección IP fija en la interfaz de red interna. En los ejemplos de este informe se utiliza `192.168.56.10/24`. Debe sustituirse por la red real utilizada en la práctica si es distinta.

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

Después se habilitó e inició el servicio:

```bash
sudo systemctl enable --now named
sudo systemctl status named
```

Según la instalación, el servicio puede aparecer como `named.service` o como `bind9.service`. Se puede localizar mediante:

```bash
systemctl list-unit-files | grep -E 'bind9|named'
```

### Configuración general

El archivo principal de opciones se encuentra normalmente en:

```text
/etc/bind/named.conf.options
```

Ejemplo de configuración para una red interna:

```conf
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 127.0.0.1; 192.168.56.0/24; };
    allow-query { 127.0.0.1; 192.168.56.0/24; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    listen-on { 127.0.0.1; 192.168.56.10; };
    listen-on-v6 { none; };
};
```

Los parámetros `allow-recursion` y `allow-query` limitan qué equipos pueden consultar el servidor. En un laboratorio se debe indicar exclusivamente la red interna. La directiva `listen-on` debe contener la IP estática real del adaptador interno; si no se conoce o cambia durante una prueba, se puede usar temporalmente `listen-on { any; };` en un entorno aislado.

### Zona directa

Se creó el directorio donde se guardan los archivos de zonas:

```bash
sudo mkdir -p /etc/bind/zones
```

Después se añadió la zona directa en `/etc/bind/named.conf.local`:

```conf
zone "empresa.local" {
    type master;
    file "/etc/bind/zones/db.empresa.local";
};
```

Archivo de zona directa `/etc/bind/zones/db.empresa.local`:

```dns
$TTL 86400
@   IN  SOA ns1.empresa.local. admin.empresa.local. (
        2026092901 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      ns1.empresa.local.
ns1     IN  A       192.168.56.10
www     IN  A       192.168.56.10
mail    IN  A       192.168.56.20
ftp     IN  CNAME   www.empresa.local.
```

El punto final en los nombres completos, por ejemplo `ns1.empresa.local.`, es importante: indica que se trata de un FQDN. Cada vez que se modifique el archivo de zona se debe aumentar el valor `Serial`.

### Zona inversa

La zona inversa permite relacionar una IP con un nombre. Para la red `192.168.56.0/24` se declaró la siguiente zona en `/etc/bind/named.conf.local`:

```conf
zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.56";
};
```

Archivo `/etc/bind/zones/db.192.168.56`:

```dns
$TTL 86400
@   IN  SOA ns1.empresa.local. admin.empresa.local. (
        2026092901 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS      ns1.empresa.local.
10      IN  PTR     ns1.empresa.local.
20      IN  PTR     mail.empresa.local.
```

### Validación y puesta en marcha

Antes de reiniciar se comprobaron los archivos:

```bash
sudo named-checkconf
sudo named-checkzone empresa.local /etc/bind/zones/db.empresa.local
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/zones/db.192.168.56
```

Si los resultados son correctos, se recarga el servicio:

```bash
sudo systemctl restart named
```

Las pruebas locales se realizan con:

```bash
dig @127.0.0.1 empresa.local
dig @192.168.56.10 www.empresa.local
dig -x 192.168.56.10 @192.168.56.10
```

Desde un cliente conectado a la misma red interna se configura como DNS la dirección IP interna del servidor y se ejecuta:

```bash
dig @192.168.56.10 www.empresa.local
nslookup www.empresa.local 192.168.56.10
```

## Análisis de incidencias técnicas

| Incidencia o fallo | Causa probable | Comprobación | Solución recomendada |
|---|---|---|---|
| `Unable to locate package bind9` | Índice de paquetes sin actualizar o repositorios mal configurados | `sudo apt update` | Revisar `/etc/apt/sources.list` y ejecutar de nuevo `sudo apt update` antes de instalar. |
| No hay acceso a Internet durante la instalación | Adaptador NAT desactivado o ruta por defecto ausente | `ip route`, `ping -c 3 1.1.1.1` | Activar el adaptador NAT en VirtualBox y comprobar que existe una ruta por defecto. |
| Error de resolución de nombres al usar `apt` | DNS del sistema no funciona todavía | `cat /etc/resolv.conf`, `ping -c 3 deb.debian.org` | Comprobar el DNS temporal del sistema o probar conectividad por IP antes de configurar BIND9. |
| El servicio no inicia después de instalarlo | Error de sintaxis, configuración incompleta o nombre de unidad distinto | `sudo systemctl status named`, `journalctl -u named -n 50` | Validar con `sudo named-checkconf`; comprobar si el servicio instalado se llama `named` o `bind9`. |
| Puerto 53 ocupado | Otro servicio DNS como `dnsmasq` o `systemd-resolved` usa el puerto | `sudo ss -tulpn | grep ':53'` | Detener o reconfigurar el servicio en conflicto; no detener servicios sin confirmar cuál ocupa el puerto. |
| `permission denied` al cargar una zona | Permisos o propietario incorrectos en el archivo de zona | `ls -l /etc/bind/zones/` | Asegurar que los archivos sean legibles por el proceso BIND: `sudo chown root:bind /etc/bind/zones/*` y `sudo chmod 644 /etc/bind/zones/*`. |
| `SERVFAIL` | Archivo de zona con un error de sintaxis o referencias incorrectas | `sudo named-checkzone empresa.local /etc/bind/zones/db.empresa.local` | Corregir el registro indicado por el validador, verificar puntos finales en FQDN y aumentar el serial. |
| `NXDOMAIN` | El nombre consultado no existe en la zona | `dig @192.168.56.10 nombre.empresa.local` | Crear o corregir el registro A, CNAME o PTR correspondiente; recargar BIND9. |
| Tiempo de espera en el cliente | Cliente y servidor no están en la misma red interna, firewall bloquea DNS o BIND no escucha en la IP interna | `ping 192.168.56.10`, `sudo ss -tulpn | grep ':53'` | Verificar el nombre de la red interna en VirtualBox, permitir UDP/TCP 53 y ajustar `listen-on`. |
| Recursión rechazada | Red del cliente no incluida en `allow-recursion` | Revisar `named.conf.options` | Añadir únicamente la subred interna correcta a `allow-recursion` y recargar el servicio. |
| Consultas externas lentas o fallidas | NAT sin salida o servidores `forwarders` inaccesibles | `ping -c 3 1.1.1.1`, `dig @1.1.1.1 example.com` | Corregir NAT, revisar la ruta por defecto y usar reenviadores accesibles. |

Los registros del servicio son especialmente útiles para localizar errores. Los comandos principales son:

```bash
sudo journalctl -u named -f
sudo journalctl -u bind9 -f
```

## Conclusiones

La práctica permitió implementar un servidor DNS BIND9 funcional en Debian 13 dentro de VirtualBox. El uso de dos adaptadores separa la conectividad de Internet necesaria para instalar y actualizar el software de la red interna usada para las consultas DNS de los clientes.

La estabilidad del servicio depende de utilizar una IP fija en la interfaz interna, validar todos los archivos antes de reiniciar BIND9, restringir las consultas recursivas a redes de confianza y comprobar el estado del puerto 53. Las incidencias observadas muestran que la mayoría de los fallos se detectan mediante la validación de configuración, los registros de `systemd` y las pruebas con `dig`.
