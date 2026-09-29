# Manual de usuario: Configuración del servicio DNS con BIND9

## Objetivo

Este manual explica los aspectos esenciales para instalar, configurar, comprobar y mantener un servidor DNS BIND9 en Debian 13 dentro de una máquina virtual de VirtualBox. El servidor utiliza un adaptador NAT para acceder a Internet y un adaptador de red interna para atender a los clientes del laboratorio.

> **Nota:** Este manual refleja la configuración real implementada en la práctica: IP `192.168.6.100`, dominio `LARA.local`, servidor `ns.LARA.local`, forwarders `1.1.1.1` y `8.8.8.8`.

## Topología utilizada

| Equipo o elemento | Configuración |
|---|---|
| Servidor DNS | Debian 13 en VirtualBox con BIND9 |
| Adaptador 1 (ens0p3) | NAT con DHCP, para instalar paquetes y actualizar el sistema |
| Adaptador 2 (ens0p8) | Red interna con IP estática 192.168.6.100/24 |
| Nombre del servidor | ns |
| Dominio | LARA.local |
| FQDN completo | ns.LARA.local |
| Forwarders DNS | 1.1.1.1 y 8.8.8.8 |

Todos los clientes que deban usar el DNS deben estar conectados a la misma red interna que el segundo adaptador del servidor (ens0p8).

## Comprobar red

En VirtualBox, apaga la máquina virtual si es necesario y entra en **Configuración > Red**:

1. Configura el **Adaptador 1** como **NAT**.
2. Habilita el **Adaptador 2** y selecciona **Red interna**.
3. Introduce un nombre de red interna, por ejemplo `red-dns`.
4. En cada máquina cliente, habilita también un adaptador de tipo **Red interna** y utiliza exactamente el mismo nombre: `red-dns`.

En Debian, identifica los nombres de las interfaces:

```bash
ip a
```

En este proyecto:
- `ens0p3` = Adaptador 1 (NAT, DHCP)
- `ens0p8` = Adaptador 2 (Red interna, IP estática)

## Configurar interfaces de red

Edita el archivo de configuración de red:

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
```

Guarda con `Ctrl+O`, `Enter` y sal con `Ctrl+X`. Reinicia el servicio de red o la máquina:

```bash
sudo systemctl restart networking
# o
sudo reboot
```

Verifica que las interfaces tienen las direcciones esperadas:

```bash
ip a
ip route
```

Debes ver:
- `ens0p3` con una IP dinámica del rango NAT (ej. `10.0.2.x`)
- `ens0p8` con la IP estática `192.168.6.100/24`
- Una ruta por defecto hacia Internet a través de `ens0p3`

## Instalar BIND9

Actualiza la información de los paquetes e instala BIND9 junto con las herramientas de administración y consulta:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils -y
```

## Configurar opciones DNS

Abre el archivo principal:

```bash
sudo nano /etc/bind/named.conf.options
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

| Opción | Función |
|---|---|
| `recursion yes` | Permite resolver dominios que no pertenecen a las zonas locales. |
| `allow-recursion` | Limita la recursión a localhost y a la red interna 192.168.6.0/24. |
| `forwarders` | Define DNS externos (1.1.1.1 y 8.8.8.8) para resolver dominios de Internet. |
| `listen-on` | Indica las direcciones en las que BIND9 escucha consultas (localhost y 192.168.6.100). |

Guarda en Nano con `Ctrl+O`, pulsa `Enter` y sal con `Ctrl+X`.

## Crear zonas directas e inversas

### Paso 1: Crear directorio de zonas

```bash
sudo mkdir -p /etc/bind/zones
```

### Paso 2: Declarar zonas en named.conf.local

```bash
sudo nano /etc/bind/named.conf.local
```

Añade:

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

### Paso 3: Crear zona directa

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

### Paso 4: Crear zona inversa

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

## Arrancar el servicio DNS

Habilita e inicia el servicio:

```bash
sudo systemctl enable --now named
sudo systemctl restart named
```

Verifica el estado:

```bash
sudo systemctl status named
```

Debe mostrar `active (running)` en verde.

## Validar configuración

Antes de probar, valida los archivos:

```bash
sudo named-checkconf
sudo named-checkzone LARA.local /etc/bind/zones/db.LARA.local
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.192.168.6
```

Ambos deben mostrar `OK`.

## Probar con nslookup

### Prueba 1: Zona directa

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

### Prueba 2: Zona inversa

```bash
nslookup 192.168.6.100 192.168.6.100
```

Resultado esperado:
```
Server:         192.168.6.100
Address:        192.168.6.100#53

100.6.168.192.in-addr.arpa    name = ns.LARA.local.
```

### Prueba 3: Servidor de nombres

```bash
nslookup ns.LARA.local 192.168.6.100
```

### Prueba 4: Dominio externo

```bash
nslookup google.com 192.168.6.100
```

Si todas las pruebas devuelven respuestas correctas, el servidor DNS está funcionando correctamente.

## Consultar estado y registros

| Tarea | Comando |
|---|---|
| Ver estado del servicio | `sudo systemctl status named` |
| Reiniciar el servicio | `sudo systemctl restart named` |
| Recargar configuración | `sudo systemctl reload named` |
| Ver los últimos eventos | `sudo journalctl -u named -n 50` |
| Ver eventos en tiempo real | `sudo journalctl -u named -f` |
| Verificar sintaxis global | `sudo named-checkconf` |
| Verificar una zona | `sudo named-checkzone LARA.local /etc/bind/zones/db.LARA.local` |
| Comprobar escucha en puerto 53 | `sudo ss -tulpn | grep ':53'` |

## Solución de problemas

### BIND9 no inicia

```bash
sudo systemctl status named
sudo journalctl -u named -n 50 --no-pager
sudo named-checkconf
```

### nslookup devuelve SERVFAIL

```bash
sudo named-checkzone LARA.local /etc/bind/zones/db.LARA.local
```

Corrige errores de sintaxis, verifica puntos finales en FQDN y aumenta el serial.

### nslookup devuelve NXDOMAIN

El nombre no existe en la zona. Añade el registro A, CNAME o PTR necesario en el archivo de zona correspondiente.

### nslookup timeout

Verifica conectividad y puerto 53:

```bash
ping -c 3 192.168.6.100
sudo ss -tulpn | grep ':53'
```

## Recomendaciones de seguridad

- Permite consultas recursivas únicamente desde la red interna de confianza (192.168.6.0/24).
- No expongas este servidor de laboratorio a Internet mediante reglas de redirección de puertos.
- Mantén Debian y BIND9 actualizados mediante `sudo apt update` y `sudo apt upgrade`.
- Valida la configuración antes de cada reinicio o recarga.
- Revisa periódicamente los registros del servicio para detectar errores y consultas anómalas.
- Realiza copias de seguridad de `/etc/bind/` antes de modificaciones importantes.

## Mantenimiento básico

Cuando edites una zona:

1. Actualiza o añade el registro necesario.
2. Incrementa el valor `Serial` del SOA.
3. Valida el archivo con `named-checkzone`.
4. Recarga el servicio con `sudo systemctl reload named`.
5. Prueba el registro con `nslookup` o `dig`.

## Resumen de configuración aplicada

| Parámetro | Valor |
|---|---|
| Interfaz NAT | ens0p3 (DHCP) |
| Interfaz red interna | ens0p8 (192.168.6.100/24) |
| Dominio | LARA.local |
| Servidor DNS | ns.LARA.local |
| Forwarders | 1.1.1.1, 8.8.8.8 |
| Zona directa | LARA.local |
| Zona inversa | 6.168.192.in-addr.arpa |
| Herramienta de prueba | nslookup |
