# Manual de usuario: Configuración del servicio DNS con BIND9

## Objetivo

Este manual explica los aspectos esenciales para instalar, configurar, comprobar y mantener un servidor DNS BIND9 en Debian 13 dentro de una máquina virtual de VirtualBox. El servidor utiliza un adaptador NAT para acceder a Internet y un adaptador de red interna para atender a los clientes del laboratorio.

> **Importante:** Las direcciones IP y el nombre de dominio utilizados son ejemplos. Sustituye `192.168.56.10`, `192.168.56.0/24` y `empresa.local` por los valores de tu práctica si son diferentes.

## Topología utilizada

| Equipo o elemento | Configuración |
|---|---|
| Servidor DNS | Debian 13 en VirtualBox con BIND9 |
| Adaptador 1 | NAT, para instalar paquetes y actualizar el sistema |
| Adaptador 2 | Red interna, para los equipos cliente |
| IP DNS interna de ejemplo | `192.168.56.10/24` |
| Dominio de ejemplo | `empresa.local` |

Todos los clientes que deban usar el DNS deben estar conectados a la misma red interna que el segundo adaptador del servidor.

## Comprobar red

En VirtualBox, apaga la máquina virtual si es necesario y entra en **Configuración > Red**:

1. Configura el **Adaptador 1** como **NAT**.
2. Habilita el **Adaptador 2** y selecciona **Red interna**.
3. Introduce un nombre de red interna, por ejemplo `red-dns`.
4. En cada máquina cliente, habilita también un adaptador de tipo **Red interna** y utiliza exactamente el mismo nombre: `red-dns`.

En Debian, comprueba los nombres de las interfaces y la dirección IP:

```bash
ip a
ip route
```

Configura una IP estática en la interfaz de red interna. El método depende de la herramienta de red instalada en tu Debian. Tras configurarla, comprueba que la interfaz dispone de una dirección como `192.168.56.10/24`:

```bash
ip a
```

La interfaz NAT debe conservar una ruta por defecto para acceder a Internet. Verifícala con:

```bash
ip route
ping -c 3 1.1.1.1
```

## Instalar BIND9

Actualiza la información de los paquetes e instala BIND9 junto con las herramientas de administración y consulta:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils -y
```

Activa el servicio para que se inicie automáticamente y arráncalo ahora:

```bash
sudo systemctl enable --now named
sudo systemctl status named
```

Si el nombre `named` no existe, comprueba el nombre de la unidad instalada:

```bash
systemctl list-unit-files | grep -E 'bind9|named'
```

Utiliza el nombre detectado en los comandos posteriores, por ejemplo `bind9` en lugar de `named`.

## Configurar opciones DNS

Abre el archivo principal:

```bash
sudo nano /etc/bind/named.conf.options
```

Utiliza una configuración equivalente a esta, adaptando la IP y red internas:

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

| Opción | Función |
|---|---|
| `recursion yes` | Permite resolver dominios que no pertenecen a las zonas locales. |
| `allow-recursion` | Limita la recursión a localhost y a la red interna. |
| `allow-query` | Restringe qué clientes pueden consultar el servidor. |
| `forwarders` | Define DNS externos para resolver dominios de Internet. |
| `listen-on` | Indica las direcciones en las que BIND9 escucha consultas. |

Guarda en Nano con `Ctrl+O`, pulsa `Enter` y sal con `Ctrl+X`.

## Crear zona directa

Crea un directorio para los archivos de zona:

```bash
sudo mkdir -p /etc/bind/zones
```

Edita la configuración de zonas locales:

```bash
sudo nano /etc/bind/named.conf.local
```

Añade la declaración de zona:

```conf
zone "empresa.local" {
    type master;
    file "/etc/bind/zones/db.empresa.local";
};
```

Crea el archivo de zona directa:

```bash
sudo nano /etc/bind/zones/db.empresa.local
```

Pega el siguiente contenido:

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

### Añadir equipos nuevos

Para añadir un equipo, crea un registro A en el archivo de zona. Por ejemplo, para el host `pc1` con IP `192.168.56.30`:

```dns
pc1     IN  A       192.168.56.30
```

Después de cada cambio, incrementa el número de serie. Por ejemplo:

```text
2026092901 -> 2026092902
```

## Crear zona inversa

La zona inversa permite obtener un nombre a partir de una dirección IP. En `/etc/bind/named.conf.local`, añade:

```conf
zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.56";
};
```

Crea el archivo de zona inversa:

```bash
sudo nano /etc/bind/zones/db.192.168.56
```

Contenido de ejemplo:

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
30      IN  PTR     pc1.empresa.local.
```

El número inicial de cada registro PTR corresponde al último octeto de la IP. Por ejemplo, `30` representa `192.168.56.30`.

## Validar y activar

Antes de aplicar cambios, valida siempre los archivos:

```bash
sudo named-checkconf
sudo named-checkzone empresa.local /etc/bind/zones/db.empresa.local
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/zones/db.192.168.56
```

Si no aparecen errores, recarga el servicio:

```bash
sudo systemctl restart named
```

Para cambios rutinarios también se puede utilizar:

```bash
sudo systemctl reload named
```

## Realizar pruebas

### Pruebas en el servidor

```bash
dig @127.0.0.1 empresa.local
dig @192.168.56.10 www.empresa.local
dig -x 192.168.56.10 @192.168.56.10
```

La respuesta debe incluir los registros configurados. Para consultar un dominio de Internet mediante los reenviadores:

```bash
dig @192.168.56.10 debian.org
```

### Pruebas desde un cliente

1. Conecta el cliente a la misma red interna de VirtualBox.
2. Asigna una IP de la misma subred, por ejemplo `192.168.56.30/24`.
3. Define como servidor DNS del cliente `192.168.56.10`.
4. Ejecuta una consulta:

```bash
dig @192.168.56.10 www.empresa.local
nslookup www.empresa.local 192.168.56.10
```

## Consultar estado y registros

| Tarea | Comando |
|---|---|
| Ver estado del servicio | `sudo systemctl status named` |
| Reiniciar el servicio | `sudo systemctl restart named` |
| Recargar configuración | `sudo systemctl reload named` |
| Ver los últimos eventos | `sudo journalctl -u named -n 50` |
| Ver eventos en tiempo real | `sudo journalctl -u named -f` |
| Verificar sintaxis global | `sudo named-checkconf` |
| Verificar una zona | `sudo named-checkzone empresa.local /etc/bind/zones/db.empresa.local` |
| Comprobar escucha en puerto 53 | `sudo ss -tulpn | grep ':53'` |

## Solución de problemas

### BIND9 no inicia

Ejecuta:

```bash
sudo systemctl status named
sudo journalctl -u named -n 50 --no-pager
sudo named-checkconf
```

Corrige cualquier error indicado antes de reiniciar el servicio.

### Error al instalar BIND9

Si aparece que no se encuentra el paquete, actualiza los repositorios:

```bash
sudo apt update
sudo apt install bind9 bind9-utils dnsutils -y
```

Si no hay conexión, revisa que el adaptador NAT esté habilitado y comprueba la ruta por defecto:

```bash
ip route
ping -c 3 1.1.1.1
```

### Los clientes no reciben respuesta

Comprueba que el cliente y el servidor usan la misma red interna de VirtualBox. Después, desde el cliente, verifica primero la conectividad IP:

```bash
ping -c 3 192.168.56.10
```

En el servidor, verifica que el puerto DNS está abierto y BIND9 escucha en la IP interna:

```bash
sudo ss -tulpn | grep ':53'
```

Revisa que `listen-on`, `allow-query` y `allow-recursion` contienen la IP o red correcta.

### Aparece SERVFAIL

Un `SERVFAIL` suele indicar un error en el archivo de zona. Valida la zona:

```bash
sudo named-checkzone empresa.local /etc/bind/zones/db.empresa.local
```

Corrige el registro que señale el comando. Comprueba los puntos finales de los nombres completos y el formato del registro SOA.

### Aparece NXDOMAIN

`NXDOMAIN` indica que el registro solicitado no existe. Comprueba el archivo de zona, añade el registro A, CNAME o PTR necesario, incrementa el serial y recarga BIND9.

### El puerto 53 ya está ocupado

Identifica qué proceso utiliza el puerto:

```bash
sudo ss -tulpn | grep ':53'
```

No detengas servicios sin identificar primero el proceso. Si existe un conflicto con otro servicio DNS, decide cuál debe atender las consultas o modifica su configuración.

## Recomendaciones de seguridad

- Permite consultas recursivas únicamente desde la red interna de confianza.
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
5. Prueba el registro con `dig`.

Este procedimiento evita aplicar archivos de zona con errores y facilita la detección de incidencias.