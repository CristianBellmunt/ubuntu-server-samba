# Ubuntu Server - ADSO Cv1

**Cristian Bellmunt Redon**

**Profesor:** Gregorio Mateu  
**Curso:** 2º ASIX

---

## ÍNDICE

- 🧱 SPRINT 1 – Configuración Base del Servidor
  - 🕒 HORA 1: Preparación del sistema y configuración de red
  - 🕒 HORA 2: Preparación de DNS y desactivación de systemd-resolved
  - 🕒 HORA 3: Instalación y preparación de Samba AD DC
  - 🕒 HORA 4: Promoción a Controlador de Dominio
  - 🕒 HORA 5: Configuración de DNS y verificación del dominio
  - 🕒 HORA 6: Configuración de enrutamiento NAT y políticas de contraseñas
  - 🎯 FIN DEL SPRINT 1
- 🧱 SPRINT 2 – DHCP, Usuarios, Grupos y Carpetas Compartidas
  - 🕒 HORA 1: Instalación y configuración del servidor DHCP
  - 🕒 HORA 2: Creación de Unidades Organizativas (OUs)
  - 🕒 HORA 3: Creación de usuarios en sus respectivas OUs
  - 🕒 HORA 4: Creación de grupos de seguridad y asignación de miembros
  - 🕒 HORA 5: Creación de carpetas compartidas con gestión ACLs (desde Windows)
  - 🎯 FIN DEL SPRINT 2
- 🧱 SPRINT 3 – Integración de Cliente Windows al Dominio
  - 📋 PREPARACIÓN DEL CLIENTE WINDOWS (ANTES DE EMPEZAR)
  - 🕒 HORA 1: Unir el cliente Windows al dominio
  - 🕒 HORA 2: Iniciar sesión con usuarios del dominio
  - 🕒 HORA 3: Acceso a recursos compartidos desde Windows
  - 🕒 HORA 4: Configurar permisos (ACLs) desde Windows
  - 🕒 HORA 5: Verificar restricciones de acceso
  - 🕒 HORA 6: Verificación final y checkpoint del SPRINT 3
  - 🎯 FIN DEL SPRINT 3
- 🧱 SPRINT 4 – RSAT, Cliente Ubuntu y Gestión del Sistema
  - 🕒 HORA 1: Configuración de GPOs con RSAT desde Windows
  - 🕒 HORA 2: Preparación y unión del cliente Ubuntu al dominio
    - 📋 PREPARACIÓN DEL CLIENTE UBUNTU (ANTES DE EMPEZAR)
  - 🕒 HORA 3: Montaje de recursos compartidos desde Linux
  - 🕒 HORA 4: Gestión de procesos (actividad práctica con SSH)
  - 🕒 HORA 5: Tareas programadas con CRON (Backup automático de Samba)
  - 🕒 HORA 6: Seguridad y Auditoría (Samba Audit)
  - 🎯 FIN DEL SPRINT 4

---

## Datos de configuración de la MV

- **Nombre servidor (Hostname):** ls02
- **Dominio:** lab02.lan
- **IP Bridge (WAN):** 172.30.20.26/25 (Gateway 172.30.20.1) -> Interfaz enp0s3
- **IP Interna (LAN):** 192.168.11.2/24 -> Interfaz enp0s8
- **DNS Forwarders:** 10.239.3.7, 10.239.3.8

**Usuario (ubuntu servidor):** cristianbr / admin_21  
**Usuario (windows cliente):** user01 / admin_21  
**Usuario (ubuntu cliente):** user01 / admin_21

**IMPORTANTE:**

Si algo no funciona, revisar:

- Archivo `/etc/netplan/00-installer-config.yaml`, IP y DNS del archivo:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

- Archivo `/etc/resolv.conf`:

```bash
sudo nano /etc/resolv.conf
```

- Archivo `/etc/samba/smb.conf`, DNS forwarders del AD DC:

```bash
sudo nano /etc/samba/smb.conf
sudo systemctl restart samba-ad-dc
```

> Puede que tenga configuraciones diferentes dependiendo del entorno en el que esté (clase o casa)

---

# 🧱 SPRINT 1 – Configuración Base del Servidor

## 🕒 HORA 1: Preparación del sistema y configuración de red

**Objetivo:** Configurar hostname, red estática en ambas interfaces y verificar conectividad.

### 1.1 Cambiar el nombre del servidor

```bash
sudo hostnamectl set-hostname ls02
```

🔍 **Verificación:**

```bash
hostnamectl
```

Debe mostrar: `Static hostname: ls02`

---

### 1.2 Configurar Netplan (red estática)

> ⚠️ **Nota importante sobre cloud-init y Netplan**
>
> En algunos sistemas (VMs o instalaciones automatizadas), Ubuntu puede generar automáticamente el archivo: `/etc/netplan/50-cloud-init.yaml`
>
> Si este archivo existe, cloud-init controla la red y puede sobrescribir la configuración en reinicios.

**Para usar 00-installer-config.yaml de forma persistente:**

1. Crear el archivo 00-installer-config.yaml (a mano):

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Pones toda la configuración ahí.

2. Desactivar cloud-init solo para la red:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Contenido:

```
network: {config: disabled}
```

3. Eliminar el archivo generado por cloud-init:

```bash
sudo rm /etc/netplan/50-cloud-init.yaml
```

4. Comprobar:

```bash
ls /etc/netplan/
```

Debe mostrarse únicamente: `00-installer-config.yaml`

---

Editar el archivo de configuración:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Contenido completo:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 172.30.20.26/25
      routes:
        - to: default
          via: 172.30.20.1
      nameservers:
        addresses:
          - 10.239.3.7
          - 10.239.3.8
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.11.2/24
```

Aplicar cambios:

```bash
sudo netplan apply
```

🔍 **Verificación:**

```bash
ip addr show
```

Debe mostrar:
- `enp0s3: 172.30.20.26/25`
- `enp0s8: 192.168.11.2/24`

Comprobar conectividad externa:

```bash
ping -c 4 www.amazon.es
```

🛠 **Si falla el ping:**

```bash
# Verificar que los DNS sean correctos
resolvectl status

# Comprobar gateway
ip route show

# Si persiste, reintentar aplicar netplan
sudo netplan --debug apply
```

---

### 1.3 Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

> ⚠️ **NOTA:** Este paso puede tardar varios minutos. Es obligatorio antes de instalar Samba.

---

### 1.4 Configurar /etc/hosts

```bash
sudo nano /etc/hosts
```

Añadir estas líneas:

```
127.0.0.1   localhost
127.0.1.1   ls02
192.168.11.2   ls02.lab02.lan ls02
```

🔍 **Verificación:**

```bash
ping -c 2 ls02.lab02.lan
```

Debe responder desde `192.168.11.2`

---

## 🕒 HORA 2: Preparación de DNS y desactivación de systemd-resolved

**Objetivo:** Eliminar conflictos de DNS causados por systemd-resolved antes de instalar Samba.

### 2.1 Detener y deshabilitar systemd-resolved

> ⚠️ **CRÍTICO:** systemd-resolved SIEMPRE causa conflictos con Samba AD DC (puerto 53).

```bash
sudo systemctl disable --now systemd-resolved
```

🔍 **Verificación:**

```bash
sudo systemctl status systemd-resolved
```

Debe mostrar: `inactive (dead)` y `disabled`

---

### 2.2 Eliminar el enlace simbólico de resolv.conf

```bash
sudo unlink /etc/resolv.conf
```

🔍 **Verificación:**

```bash
ls -la /etc/resolv.conf
```

🛠 Si dice `"No such file or directory"`: Perfecto, continúa.  
🛠 Si sigue existiendo como enlace simbólico:

```bash
sudo rm /etc/resolv.conf
```

---

### 2.3 Crear /etc/resolv.conf manualmente (temporal para instalación)

```bash
sudo nano /etc/resolv.conf
```

Contenido:

```
nameserver 10.239.3.7
nameserver 10.239.3.8
search lab02.lan
```

Hacer el archivo inmutable (opcional pero recomendado):

```bash
sudo chattr +i /etc/resolv.conf
```

> Para revertirlo: en vez de escribir el comando con `+i`, poner `-i`
>
> Esto evita que otros servicios lo modifiquen.

🔍 **Verificación DNS:**

```bash
nslookup www.amazon.es
```

Debe resolver correctamente usando 10.239.3.7 o 10.239.3.8

🛠 **Si falla la resolución:**

```bash
# Verificar contenido del archivo
cat /etc/resolv.conf

# Verificar que no haya errores de sintaxis
# Reintentar sin hacer el archivo inmutable
```

---

## 🕒 HORA 3: Instalación y preparación de Samba AD DC

**Objetivo:** Instalar Samba, Kerberos, Winbind y herramientas necesarias.

### 3.1 Instalar paquetes necesarios

```bash
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

**Durante la instalación de Kerberos:**

- Default Kerberos realm: `LAB02.LAN` (en MAYÚSCULAS)
- Kerberos servers: `ls02.lab02.lan`
- Administrative server: `ls02.lab02.lan`

> ⚠️ Si aparece una ventana de configuración automática de Samba: selecciona "No" (configuraremos manualmente).

🛠 **Si la instalación falla por dependencias:**

```bash
sudo apt --fix-broken install
sudo apt install -y samba smbclient winbind krb5-user krb5-config
```

---

### 3.2 Detener y deshabilitar servicios previos

```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
```

🔍 **Verificación:**

```bash
sudo systemctl status smbd
```

Debe mostrar: `inactive (dead)`

---

### 3.3 Respaldar y eliminar configuración por defecto

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

🔍 **Verificación:**

```bash
ls -la /etc/samba/
```

No debe existir `smb.conf` (solo `smb.conf.backup`)

🛠 Si dice `"No such file or directory"`:
- Es normal si es una instalación limpia
- Continúa sin problemas

---

## 🕒 HORA 4: Promoción a Controlador de Dominio

**Objetivo:** Crear un nuevo bosque AD con dominio lab02.lan.

### 4.1 Provisionar el dominio

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Respuestas al asistente:**

```
Realm: LAB02.LAN (mayúsculas)
Domain: LAB02 (NetBIOS, mayúsculas)
Server Role: dc (presiona Enter)
DNS backend: SAMBA_INTERNAL (presiona Enter)
DNS forwarder IP: 10.239.3.7 (DNS de Conselleria u otro DNS dependiendo del entorno)
Administrator password: admin_21 (o tu contraseña segura)
Retype password: admin_21
```

> ⚠️ La contraseña debe cumplir: mínimo 7 caracteres, mayúsculas, minúsculas y números.

---

**Configurar Interfaz de Escucha (Solución "Connection Refused")**

Samba puede intentar escuchar solo en IPv6 por defecto. Forzamos IPv4.

Editar configuración:

```bash
sudo nano /etc/samba/smb.conf
```

Añadir en la sección `[global]`:

```ini
[global]
    ...
    interfaces = lo enp0s8   # (Pon tu interfaz interna real)
    bind interfaces only = yes
    ...
```

Luego reinicia Samba AD DC:

```bash
sudo systemctl restart samba-ad-dc
```

🔍 **Verificación:**

```bash
sudo samba-tool domain level show
```

Debe mostrar los niveles de dominio y bosque.

🛠 **Si falla con "DNS zone already exists":**

```bash
# Eliminar la base de datos anterior
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb

# Volver a ejecutar provision
sudo samba-tool domain provision --use-rfc2307 --interactive
```

---

### 4.2 Copiar configuración de Kerberos

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

🔍 **Verificación:**

```bash
cat /etc/krb5.conf | grep LAB02.LAN
```

Debe aparecer el dominio `LAB02.LAN`

---

### 4.3 Configurar resolv.conf para usar DNS local

Quitar el atributo inmutable (si se puso):

```bash
sudo chattr -i /etc/resolv.conf
```

Editar resolv.conf:

```bash
sudo nano /etc/resolv.conf
```

Contenido NUEVO:

```
nameserver 127.0.0.1
nameserver 10.239.3.7
search lab02.lan
```

Hacer inmutable de nuevo:

```bash
sudo chattr +i /etc/resolv.conf
```

🔍 **Verificación:**

```bash
cat /etc/resolv.conf
```

Debe mostrar `nameserver 127.0.0.1` como primero.

---

### 4.4 Iniciar y habilitar Samba AD DC

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl enable samba-ad-dc
```

🔍 **Verificación:**

```bash
sudo systemctl status samba-ad-dc
```

Debe mostrar: `active (running)`

🛠 **Si falla el start:**

```bash
# Ver logs detallados
sudo journalctl -xeu samba-ad-dc

# Errores comunes:
# - Puerto 53 ocupado → verificar que systemd-resolved esté detenido
# - Archivo de configuración → verificar /etc/samba/smb.conf existe
```

**Si el puerto 53 está ocupado:**

```bash
# Verificar qué proceso usa el puerto 53
sudo ss -tulpn | grep :53

# Si aparece systemd-resolved
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl start samba-ad-dc
```

---

## 🕒 HORA 5: Configuración de DNS y verificación del dominio

**Objetivo:** Configurar DNS, reenviadores y verificar funcionamiento del AD.

> En caso de NO haber editado/usado de forma directa el archivo `/etc/resolv.conf` para los DNS, editar el archivo.

### 5.1 Verificar que Samba DNS está funcionando

```bash
host -t A lab02.lan
host -t A ls02.lab02.lan
host -t SRV _ldap._tcp.lab02.lan
```

**Resultado esperado:**

- `lab02.lan` → 192.168.11.2
- `ls02.lab02.lan` → 192.168.11.2
- `_ldap._tcp.lab02.lan` → registro SRV apuntando a ls02.lab02.lan

🛠 **Si no resuelve:**

```bash
# Reinicia (sí, hazme caso)
sudo reboot now

# Verificar que nameserver sea 127.0.0.1
cat /etc/resolv.conf

# Verificar logs de Samba
sudo journalctl -xeu samba-ad-dc | tail -50

# Ver todos los registros DNS
sudo samba-tool dns query 127.0.0.1 lab02.lan @ ALL -U Administrator%admin_21
```

---

### 5.2 Configurar DNS forwarders en Samba

> ⚠️ **IMPORTANTE:** Samba NO crea forwarders automáticamente aunque se especifique en provision.

**Verificar forwarders actuales:**

```bash
sudo samba-tool dns serverinfo 127.0.0.1 -U Administrator%admin_21
```

**Si no aparece ningún forwarder, añadirlos manualmente:**

```bash
# Editar smb.conf
sudo nano /etc/samba/smb.conf
```

Buscar la sección `[global]` y AÑADIR (o modificar si existe):

```ini
[global]
    # ... (resto de configuración)
    dns forwarder = 10.239.3.7
```

**Reiniciar Samba:**

```bash
sudo systemctl restart samba-ad-dc
```

🔍 **Verificación de resolución externa:**

```bash
nslookup www.amazon.es 127.0.0.1
```

Debe resolver correctamente usando el forwarder de Conselleria.

🛠 **Si persiste el fallo de resolución externa:**

```bash
# Verificar que el forwarder esté en smb.conf
grep "dns forwarder" /etc/samba/smb.conf

# Probar añadir forwarder alternativo
sudo nano /etc/samba/smb.conf
# Cambiar:
# dns forwarder = 10.239.3.8
```

---

### 5.3 Probar autenticación Kerberos

```bash
kinit Administrator@LAB02.LAN
```

Introduce la contraseña: `admin_21`

🔍 **Verificación:**

```bash
klist
```

Debe mostrar un ticket válido para `Administrator@LAB02.LAN`

🛠 **Si falla "Clock skew too great":**

```bash
# Sincronizar hora del sistema
sudo timedatectl set-ntp true

# Verificar zona horaria
timedatectl

# Reintentar kinit
kinit Administrator@LAB02.LAN
```

🛠 **Si falla "Cannot find KDC for realm":**

```bash
# Verificar krb5.conf
cat /etc/krb5.conf | grep -A 5 LAB02.LAN

# Si no existe, copiar de nuevo
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

---

## 🕒 HORA 6: Configuración de enrutamiento NAT y políticas de contraseñas

**Objetivo:** Configurar NAT para clientes internos y establecer políticas de contraseñas.

### 6.1 Activar IP forwarding

```bash
sudo nano /etc/sysctl.conf
```

Descomentar o añadir:

```
net.ipv4.ip_forward=1
```

Aplicar:

```bash
sudo sysctl -p
```

🔍 **Verificación:**

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Debe devolver: `1`

---

### 6.2 Configurar NAT con iptables

**enp0s3** → adaptador puente / salida a Internet  
**enp0s8** → red interna

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

**Guardar reglas de forma permanente:**

```bash
sudo apt install -y iptables-persistent
```

Durante la instalación:
- Save current IPv4 rules? → **Yes**
- Save current IPv6 rules? → **Yes**

**Guardar manualmente (si ya estaba instalado):**

```bash
sudo netfilter-persistent save
```

🔍 **Verificación:**

```bash
sudo iptables -t nat -L -v
```

Debe aparecer la regla `MASQUERADE` en la cadena `POSTROUTING`.

🛠 **Si las reglas no persisten tras reinicio:**

```bash
# Guardar manualmente
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Verificar que el servicio esté habilitado
sudo systemctl enable netfilter-persistent
```

---

### 6.3 Ver políticas de contraseñas actuales

```bash
sudo samba-tool domain passwordsettings show
```

---

### 6.4 Modificar políticas de contraseñas (TODAS las opciones disponibles)

**Longitud mínima de contraseña:**

```bash
sudo samba-tool domain passwordsettings set --min-pwd-length=8
```

**Complejidad de contraseña (activar/desactivar):**

```bash
# Activar complejidad (requiere mayúsculas, minúsculas, números)
sudo samba-tool domain passwordsettings set --complexity=on

# Desactivar complejidad (útil para entorno de pruebas)
sudo samba-tool domain passwordsettings set --complexity=off
```

**Historial de contraseñas (cuántas contraseñas anteriores se recuerdan):**

```bash
sudo samba-tool domain passwordsettings set --history-length=12
```

**Edad máxima de contraseña (días antes de expirar):**

```bash
sudo samba-tool domain passwordsettings set --max-pwd-age=60
```

**Edad mínima de contraseña (días antes de poder cambiar):**

```bash
sudo samba-tool domain passwordsettings set --min-pwd-age=0
```

**Duración del bloqueo de cuenta (minutos):**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-duration=30
```

**Número de intentos incorrectos antes de bloqueo:**

```bash
sudo samba-tool domain passwordsettings set --account-lockout-threshold=5
```

**Ventana de observación para intentos fallidos (minutos):**

```bash
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=15
```

🔍 **Verificación después de cada cambio:**

```bash
sudo samba-tool domain passwordsettings show
```

---

### 6.5 Verificación integral del dominio

**Comprobar nivel funcional:**

```bash
sudo samba-tool domain level show
```

**Comprobar usuarios:**

```bash
sudo samba-tool user list
```

Debe aparecer: `Administrator`

**Comprobar grupos:**

```bash
sudo samba-tool group list
```

Deben aparecer grupos por defecto: `Domain Admins`, `Domain Users`, etc.

**Verificar todos los registros DNS:**

```bash
sudo samba-tool dns query ls02.lab02.lan lab02.lan @ ALL -U Administrator%admin_21
```

Debe listar registros A, SRV, NS del dominio.

---

### 6.6 Checkpoint final del SPRINT 1

✅ **Lista de comprobaciones obligatorias:**

1. **Hostname:**
```bash
hostnamectl
```
→ debe mostrar `ls02`

2. **Red configurada:**
```bash
ip addr show
```
→ enp0s3 (172.30.20.26) y enp0s8 (192.168.11.2)

3. **Conectividad externa:**
```bash
ping -c 2 www.amazon.es
```
→ debe funcionar

4. **systemd-resolved deshabilitado:**
```bash
sudo systemctl status systemd-resolved
```
→ `inactive (dead)` y `disabled`

5. **DNS local funcionando:**
```bash
host lab02.lan
```
→ resuelve a 192.168.11.2

6. **Samba AD DC activo:**
```bash
sudo systemctl status samba-ad-dc
```
→ `active (running)`

7. **Kerberos funcionando:**
```bash
klist
```
→ muestra ticket válido de `Administrator@LAB02.LAN`

8. **NAT configurado:**
```bash
sudo iptables -t nat -L
```
→ regla `MASQUERADE` presente

9. **DNS forwarder funcionando:**
```bash
nslookup www.amazon.es 127.0.0.1
```
→ resuelve correctamente

10. **Registros SRV del dominio:**
```bash
host -t SRV _ldap._tcp.lab02.lan
```
→ devuelve registro SRV

---

## 🛠 PLAN DE RESCATE SI TODO FALLA

**Si el dominio está completamente roto:**

```bash
# 1. Detener Samba
sudo systemctl stop samba-ad-dc

# 2. Eliminar base de datos
sudo rm -rf /var/lib/samba/private/*
sudo rm -rf /var/lib/samba/*.tdb

# 3. Eliminar configuración
sudo rm /etc/samba/smb.conf

# 4. Volver a provision (HORA 4.1)
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Si DNS no resuelve absolutamente nada:**

```bash
# 1. Verificar resolv.conf
cat /etc/resolv.conf
# Debe tener: nameserver 127.0.0.1

# 2. Verificar que Samba esté corriendo
sudo systemctl status samba-ad-dc

# 3. Verificar puerto 53
sudo ss -tulpn | grep :53
# Solo debe aparecer samba

# 4. Probar consulta directa
dig @127.0.0.1 lab02.lan
```

---

## 🎯 FIN DEL SPRINT 1

Has completado:

- ✅ Configuración de red dual (puente + interna)
- ✅ Eliminación de systemd-resolved (conflicto crítico resuelto)
- ✅ Instalación de Samba AD DC
- ✅ Creación del dominio lab02.lan
- ✅ Configuración de DNS interno y forwarders
- ✅ Configuración de Kerberos
- ✅ Enrutamiento/NAT para clientes futuros
- ✅ Políticas de contraseñas configurables

**Estado del servidor:**

- Dominio: lab02.lan (LAB02)
- DNS: Funcionando en 127.0.0.1
- Kerberos: Autenticación activa
- NAT: Red interna puede salir a Internet
- Políticas: Configuradas según necesidad

---

# 🧱 SPRINT 2 – DHCP, Usuarios, Grupos y Carpetas Compartidas

> ⚠️ **RECORDATORIO - Administrator vs administrador:**
> - `Administrator` → Usuario por defecto de Samba (inglés, mayúscula inicial)
> - `administrador` → Si creaste este usuario manualmente (español, minúscula)
> - En comandos Kerberos: `kinit Administrator@LAB02.LAN`
> - En comandos samba-tool: `-U Administrator%admin_21`
>
> ⚠️ **IMPORTANTE:** En este sprint usaremos `Administrator` (el usuario por defecto).

---

## 🕒 HORA 1: Instalación y configuración del servidor DHCP

**Objetivo:** Instalar isc-dhcp-server y configurarlo para la red interna (192.168.11.0/24).

### 1.1 Instalar el servidor DHCP

```bash
sudo apt install -y isc-dhcp-server
```

> ⚠️ Es normal que el servicio falle al iniciar porque aún no está configurado.

🔍 **Verificación:**

```bash
sudo systemctl status isc-dhcp-server
```

Debe mostrar: `failed` o `inactive` (es lo esperado)

---

### 1.2 Configurar la interfaz del DHCP

```bash
sudo nano /etc/default/isc-dhcp-server
```

Buscar la línea `INTERFACESv4=""` y modificar:

```
INTERFACESv4="enp0s8"
```

Dejar `INTERFACESv6=""` vacío (no usaremos IPv6).

🔍 **Verificación:**

```bash
cat /etc/default/isc-dhcp-server | grep INTERFACESv4
```

Debe mostrar: `INTERFACESv4="enp0s8"`

---

### 1.3 Configurar el rango DHCP

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

Buscar las líneas comentadas de ejemplo y AÑADIR AL FINAL del archivo:

```
# Configuración para lab02.lan
subnet 192.168.11.0 netmask 255.255.255.0 {
    range 192.168.11.100 192.168.11.150;
    option domain-name "lab02.lan";
    option subnet-mask 255.255.255.0;
    option domain-name-servers 192.168.11.2;
    option routers 192.168.11.2;
    option broadcast-address 192.168.11.255;
    default-lease-time 600;
    max-lease-time 7200;
}
```

**Explicación rápida:**

- Rango → .100 a .150 (51 IPs disponibles)
- DNS → el propio servidor (192.168.11.2)
- Gateway → el propio servidor (192.168.11.2)
- Lease → 10 minutos por defecto, máximo 2 horas

---

### 1.4 Verificar sintaxis de la configuración

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

🔍 **Debe devolver:**

```
Internet Systems Consortium DHCP Server 4.4.x
Copyright 2004-2022 Internet Systems Consortium.
All rights reserved.
Sin errores de sintaxis.
```

🛠 **Si aparecen errores:**

```bash
# Errores comunes:
# - Falta punto y coma al final de línea
# - Comillas mal cerradas
# - Palabra clave mal escrita
# Revisar línea indicada en el error
sudo nano +NÚMERO_LÍNEA /etc/dhcp/dhcpd.conf
```

---

### 1.5 Reiniciar y habilitar el servicio

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

🔍 **Verificación:**

```bash
sudo systemctl status isc-dhcp-server
```

Debe mostrar: `active (running)`

🛠 **Si falla al iniciar:**

```bash
# Ver logs detallados
sudo journalctl -xeu isc-dhcp-server

# Errores comunes:
# 1. "Not configured to listen on any interfaces"
#    → revisar /etc/default/isc-dhcp-server
# 2. "Configuration file contains unknown option"
#    → revisar sintaxis en /etc/dhcp/dhcpd.conf
# 3. "Can't open /var/lib/dhcp/dhcpd.leases"
sudo touch /var/lib/dhcp/dhcpd.leases
sudo systemctl restart isc-dhcp-server
```

---

### 1.6 Verificar archivo de concesiones

```bash
cat /var/lib/dhcp/dhcpd.leases
```

Por ahora estará vacío o con solo la fecha de inicio (sin clientes conectados).

---

## 🕒 HORA 2: Creación de Unidades Organizativas (OUs)

**Objetivo:** Crear la estructura organizativa del dominio.

### 2.1 Verificar conexión al dominio

```bash
sudo samba-tool domain level show
```

Debe mostrar información del dominio `LAB02.LAN`

🛠 **Si falla la conexión:**

```bash
# Verificar que Samba esté corriendo
sudo systemctl status samba-ad-dc

# Verificar DNS
host lab02.lan
```

---

### 2.2 Crear OUs principales

```bash
sudo samba-tool ou create "OU=IT_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=HR_Department,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Students,DC=lab02,DC=lan"
sudo samba-tool ou create "OU=Groups,DC=lab02,DC=lan"
```

🔍 **Verificación:**

```bash
sudo samba-tool ou list
```

Debe mostrar:

```
OU=IT_Department,DC=lab02,DC=lan
OU=HR_Department,DC=lab02,DC=lan
OU=Students,DC=lab02,DC=lan
OU=Groups,DC=lab02,DC=lan
```

🛠 **Si da error "Already exists":**

```bash
# Es normal si ejecutas el comando dos veces
# Para eliminar una OU:
sudo samba-tool ou delete "OU=nombre,DC=lab02,DC=lan"

# Para eliminar una OU con objetos dentro (CUIDADO):
sudo samba-tool ou delete "OU=nombre,DC=lab02,DC=lan" --force-subtree-delete
```

---

## 🕒 HORA 3: Creación de usuarios en sus respectivas OUs

**Objetivo:** Crear usuarios organizados en la estructura de OUs.

### 3.1 Crear usuario Bob en IT_Department

```bash
sudo samba-tool user create bob admin_21 \
    --userou="OU=IT_Department" \
    --given-name="Bob" \
    --surname="Smith" \
    --must-change-at-next-login
```

> **IMPORTANTE:** lee lo que hace `--must-change-at-next-login` si realmente lo quieres usar.

**Parámetros explicados:**

- `bob` → nombre de usuario
- `admin_21` → contraseña inicial
- `--userou` → especifica la OU donde se creará
- `--must-change-at-next-login` → obliga a cambiar contraseña en primer inicio

🔍 **Verificación:**

```bash
sudo samba-tool user show bob
```

Debe mostrar información del usuario incluyendo la OU.

🛠 **Si falla "Constraint violation":**

```bash
# La contraseña no cumple la política
# Cambiar temporalmente la complejidad:
sudo samba-tool domain passwordsettings set --complexity=off

# Crear el usuario
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"

# Reactivar complejidad
sudo samba-tool domain passwordsettings set --complexity=on
```

---

### 3.2 Crear usuario Alice en HR_Department

```bash
sudo samba-tool user create alice admin_21 \
    --userou="OU=HR_Department" \
    --given-name="Alice" \
    --surname="Johnson"
```

---

### 3.3 Crear usuarios en Students (user01, user02, user03)

```bash
sudo samba-tool user create user01 admin_21 \
    --userou="OU=Students" \
    --given-name="User" \
    --surname="One"

sudo samba-tool user create user02 admin_21 \
    --userou="OU=Students" \
    --given-name="User" \
    --surname="Two"

sudo samba-tool user create user03 admin_21 \
    --userou="OU=Students" \
    --given-name="User" \
    --surname="Three"
```

> ⚠️ **Nota:** A los estudiantes NO les ponemos `--must-change-at-next-login` para facilitar pruebas.

---

### 3.4 Crear usuario techsupport en contenedor Users

```bash
sudo samba-tool user create techsupport admin_21 \
    --given-name="Tech" \
    --surname="Support"
```

> ⚠️ Sin `--userou` → se crea automáticamente en `CN=Users,DC=lab02,DC=lan`

---

### 3.5 Verificar todos los usuarios creados

```bash
sudo samba-tool user list
```

Debe mostrar:

```
Administrator
bob
alice
user01
user02
user03
techsupport
```

**Ver información detallada de un usuario:**

```bash
sudo samba-tool user show bob
```

**Verificar en qué OU está un usuario:**

```bash
# Instalar el paquete
sudo apt install -y ldb-tools

# Verificar en qué OU está el usuario
sudo ldbsearch -H /var/lib/samba/private/sam.ldb "(sAMAccountName=bob)" dn
```

Debe mostrar algo como:

```
dn: CN=Bob Smith,OU=IT_Department,DC=lab02,DC=lan
```

🛠 **Si un usuario se creó en la OU incorrecta:**

```bash
# No se puede mover directamente, hay que:
# 1. Eliminar el usuario
sudo samba-tool user delete bob

# 2. Volver a crearlo en la OU correcta
sudo samba-tool user create bob admin_21 --userou="OU=IT_Department"
```

---

## 🕒 HORA 4: Creación de grupos de seguridad y asignación de miembros

**Objetivo:** Crear grupos en la OU Groups y asignar usuarios.

### 4.1 Crear grupos de seguridad

```bash
sudo samba-tool group add Finance \
    --groupou="OU=Groups"

sudo samba-tool group add HR \
    --groupou="OU=Groups"

sudo samba-tool group add "IT Support" \
    --groupou="OU=Groups"
```

> ⚠️ **Nota:** Los nombres con espacios van entre comillas.

🔍 **Verificación:**

```bash
sudo samba-tool group list | grep -E "Finance|HR|IT Support"
```

Debe mostrar:

```
Finance
HR
IT Support
```

🛠 **Si el grupo ya existe:**

```bash
# Error: "group already exists"
# Para eliminar un grupo:
sudo samba-tool group delete Finance

# Verificar que no tenga miembros antes
sudo samba-tool group listmembers Finance
```

---

### 4.2 Añadir usuarios a grupos

**user01 → Finance:**

```bash
sudo samba-tool group addmembers Finance user01
```

**alice → HR:**

```bash
sudo samba-tool group addmembers HR alice
```

**bob → IT Support:**

```bash
sudo samba-tool group addmembers "IT Support" bob
```

**techsupport → IT Support:**

```bash
sudo samba-tool group addmembers "IT Support" techsupport
```

🔍 **Verificación por grupo:**

```bash
sudo samba-tool group listmembers Finance
sudo samba-tool group listmembers HR
sudo samba-tool group listmembers "IT Support"
```

**Verificar a qué grupos pertenece un usuario:**

```bash
sudo samba-tool user show bob | grep memberOf
```

🛠 **Si necesitas eliminar un usuario de un grupo:**

```bash
sudo samba-tool group removemembers Finance user01
```

🛠 **Si falla "No such user":**

```bash
# Verificar que el usuario existe
sudo samba-tool user list | grep user01

# Verificar mayúsculas/minúsculas (son sensibles)
sudo samba-tool group addmembers Finance user01
```

---

## 🕒 HORA 5: Creación de carpetas compartidas con gestión ACLs (desde Windows)

**Objetivo:** Crear estructura de carpetas con permisos amplios en Linux y delegar la gestión de ACLs a Windows.

**Filosofía de este enfoque:**

- Servidor (Linux/Samba): Configura el servicio y el almacenamiento con permisos base amplios
- Gestión (Windows): Configura los permisos (ACLs) visualmente desde el explorador
- Cliente (Linux/Windows): Consume los recursos y verifica las restricciones

> ⚠️ **IMPORTANTE:** NO configuraremos ACLs complejas desde Linux. Todo se gestiona desde Windows.

### 5.1 Crear estructura de directorios

```bash
sudo mkdir -p /srv/samba/FinanceDocs
sudo mkdir -p /srv/samba/HRDocs
sudo mkdir -p /srv/samba/Public
```

🔍 **Verificación:**

```bash
ls -la /srv/samba/
```

Debe mostrar las tres carpetas creadas.

---

### 5.2 Instalar librerías necesarias para usar "Domain Users"

> ⚠️ **CRÍTICO:** Para poder usar "Domain Users" como grupo propietario en Linux, necesitas estas librerías:

```bash
sudo apt-get install -y libnss-winbind libpam-winbind
```

Actualizar caché de librerías:

```bash
sudo ldconfig
```

🔍 **Verificación:**

```bash
dpkg -l | grep winbind
```

Debe mostrar:

```
ii  libnss-winbind
ii  libpam-winbind
ii  winbind
```

---

### 5.3 Configurar winbind en smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

Dentro de la sección `[global]`, AÑADIR (o verificar que existen):

```ini
[global]
    # ... (resto de configuración existente)
    # Configuración de winbind
    winbind use default domain = yes
    template shell = /bin/bash
    template homedir = /home/%U
```

Guardar y cerrar (Ctrl+O, Enter, Ctrl+X).

🔍 **Verificación:**

```bash
sudo testparm -s | grep winbind
```

Debe mostrar:

```
winbind use default domain = yes
```

Reiniciar Samba para aplicar cambios:

```bash
sudo systemctl restart samba-ad-dc
```

---

### 5.4 Configurar permisos base amplios en Linux

> ⚠️ **IMPORTANTE:** Damos permisos "abiertos" aquí. Samba/Windows restringirá el acceso mediante ACLs.

```bash
sudo chown root:"Domain Users" /srv/samba/FinanceDocs
sudo chown root:"Domain Users" /srv/samba/HRDocs
sudo chown root:"Domain Users" /srv/samba/Public
sudo chmod 770 /srv/samba/FinanceDocs
sudo chmod 770 /srv/samba/HRDocs
sudo chmod 770 /srv/samba/Public
```

🔍 **Verificación:**

```bash
ls -la /srv/samba/
```

Debe mostrar:

```
drwxrwx--- root Domain Users FinanceDocs
drwxrwx--- root Domain Users HRDocs
drwxrwx--- root Domain Users Public
```

🛠 **Si falla "Domain Users: invalid group":**

Esto significa que winbind aún no está resolviendo los grupos. Prueba:

```bash
# Opción 1: Verificar que samba-ad-dc incluye winbind integrado
ps aux | grep winbindd

# Opción 2: Usar permisos más amplios temporalmente
sudo chown root:root /srv/samba/FinanceDocs
sudo chown root:root /srv/samba/HRDocs
sudo chown root:root /srv/samba/Public
sudo chmod 777 /srv/samba/FinanceDocs
sudo chmod 777 /srv/samba/HRDocs
sudo chmod 777 /srv/samba/Public
```

> ⚠️ **Nota:** Usar 777 es seguro aquí porque Samba controlará el acceso real mediante `valid users` y ACLs de Windows. Los permisos Linux son solo la "capa base".

---

### 5.5 Configurar recursos compartidos en smb.conf

```bash
sudo nano /etc/samba/smb.conf
```

Añadir AL FINAL del archivo (después de `[netlogon]` y `[sysvol]`):

```ini
[FinanceDocs]
    path = /srv/samba/FinanceDocs
    read only = no
    # Habilitar ACLs estilo Windows
    vfs objects = acl_xattr
    map acl inherit = yes

[HRDocs]
    path = /srv/samba/HRDocs
    read only = no
    # Habilitar ACLs estilo Windows
    vfs objects = acl_xattr
    map acl inherit = yes

[Public]
    path = /srv/samba/Public
    read only = no
    guest ok = yes
    # Habilitar ACLs estilo Windows
    vfs objects = acl_xattr
    map acl inherit = yes
```

**Explicación de parámetros:**

- `vfs objects = acl_xattr` → Habilita soporte completo de ACLs NTFS
- `map acl inherit = yes` → Permite heredar permisos estilo Windows
- `guest ok = yes` → Solo en Public, permite acceso sin autenticación (opcional)

Guardar y cerrar.

---

### 5.6 Verificar sintaxis y reiniciar Samba

**Verificar sintaxis:**

```bash
sudo testparm
```

Debe decir: `Loaded services file OK.`

🛠 **Si hay errores:**

```bash
# Ver la línea exacta del error
sudo testparm -s

# Revisar sección específica
sudo testparm -s | grep -A 5 "\[FinanceDocs\]"
```

**Reiniciar Samba:**

```bash
sudo smbcontrol all reload-config
```

O reiniciar el servicio completo:

```bash
sudo systemctl restart samba-ad-dc
```

🔍 **Verificación de recursos compartidos:**

```bash
sudo smbclient -L localhost -U Administrator%admin_21
```

Debe listar:

```
Sharename       Type      Comment
---------       ----      -------
FinanceDocs     Disk
HRDocs          Disk
Public          Disk
netlogon        Disk
sysvol          Disk
IPC$            IPC       IPC Service (Samba 4.x.x)
```

🛠 **Si no aparecen los recursos:**

```bash
# Ver logs
sudo tail -50 /var/log/samba/log.smbd

# Verificar estado del servicio
sudo systemctl status samba-ad-dc

# Reiniciar completamente
sudo systemctl restart samba-ad-dc

# Intentar de nuevo
sudo smbclient -L localhost -U Administrator%admin_21
```

---

### 5.7 PRUEBAS BÁSICAS DE ACCESO DESDE LINUX (antes de configurar desde Windows)

> ⚠️ **IMPORTANTE:** En este punto, TODOS los usuarios autenticados pueden acceder a todo porque no hemos configurado restricciones todavía. Eso se hará desde Windows.

**Prueba 1: Verificar que user01 puede acceder a FinanceDocs:**

```bash
sudo smbclient //localhost/FinanceDocs -U user01%admin_21
```

Dentro del prompt `smb: \>`:

```
mkdir test_inicial
ls
exit
```

Debe funcionar (aún no hay restricciones).

**Prueba 2: Verificar que alice puede acceder a HRDocs:**

```bash
sudo smbclient //localhost/HRDocs -U alice%admin_21
```

Dentro del prompt:

```
mkdir test_hr_inicial
ls
exit
```

Debe funcionar (aún no hay restricciones).

**Prueba 3: Verificar acceso a Public:**

```bash
sudo smbclient //localhost/Public -U user02%admin_21
```

Dentro del prompt:

```
mkdir test_public
ls
exit
```

Debe funcionar.

---

### 5.8 Checkpoint de la HORA 5

✅ **Lista de comprobaciones obligatorias:**

1. **Carpetas creadas:**
```bash
ls -la /srv/samba/
```
→ FinanceDocs, HRDocs, Public

2. **Permisos amplios configurados:**
```bash
ls -la /srv/samba/ | grep -E "rwx|777"
```
→ Permisos 770 o 777 visibles

3. **winbind configurado en smb.conf:**
```bash
sudo testparm -s | grep "winbind use default domain"
```
→ `winbind use default domain = yes`

4. **VFS ACL habilitado en recursos:**
```bash
sudo testparm -s | grep "vfs objects"
```
→ `vfs objects = acl_xattr` en cada recurso

5. **Recursos compartidos visibles:**
```bash
sudo smbclient -L localhost -U Administrator%admin_21
```
→ FinanceDocs, HRDocs, Public listados

6. **Acceso funcional (sin restricciones aún):**
```bash
sudo smbclient //localhost/FinanceDocs -U user01%admin_21 -c "ls"
```
→ Lista contenido sin error

7. **ACLs habilitadas en sistema de archivos:**
```bash
mount | grep "/ " | grep acl
```
→ Aparece "acl"

---

### 5.9 PLAN DE RESCATE SI HAY PROBLEMAS

**Si "Domain Users" no funciona como grupo:**

```bash
# Usar permisos 777 con root:root
sudo chown root:root /srv/samba/*
sudo chmod 777 /srv/samba/FinanceDocs
sudo chmod 777 /srv/samba/HRDocs
sudo chmod 777 /srv/samba/Public
# Esto es seguro porque Samba controla el acceso real
```

**Si los recursos no aparecen:**

```bash
# Verificar sintaxis
sudo testparm -s

# Ver logs específicos
sudo tail -100 /var/log/samba/log.smbd

# Reiniciar Samba
sudo systemctl restart samba-ad-dc

# Verificar firewall (si existe)
sudo ufw status
```

**Si winbind use default domain no aparece:**

```bash
# Añadir manualmente en [global]
sudo nano /etc/samba/smb.conf
# Buscar [global] y añadir:
# winbind use default domain = yes

# Guardar y reiniciar
sudo systemctl restart samba-ad-dc
```

---

## NOTA IMPORTANTE: Gestión de Permisos desde Windows

Esta configuración está lista para que en el SIGUIENTE sprint o cuando tengas un cliente Windows:

1. **Desde el Cliente Windows (unido al dominio):**
   - Abre el Explorador de archivos
   - Navega a `\\ls02\FinanceDocs` (o `\\192.168.11.2\FinanceDocs`)
   - Clic derecho → Propiedades → Seguridad
   - Editar → Añadir grupo Finance → Dar permisos Modificar
   - Editar → Añadir grupo HR → DENEGAR acceso
   - Aplicar

2. Estas ACLs se almacenan en Linux gracias a `vfs objects = acl_xattr`

3. Puedes verificar las ACLs desde Linux:

```bash
sudo getfacl /srv/samba/FinanceDocs/
```

Por ahora, el servidor está listo. La restricción de permisos se hará cuando integres clientes Windows.

---

## 🎯 FIN DEL SPRINT 2

Has completado:

- ✅ Servidor DHCP funcional (rango 192.168.11.100-150)
- ✅ Estructura de 4 OUs organizativas
- ✅ 7 usuarios creados y distribuidos correctamente
- ✅ 3 grupos de seguridad (Finance, HR, IT Support)
- ✅ Membresías de grupos configuradas
- ✅ 3 carpetas compartidas con permisos POSIX
- ✅ Recursos compartidos en Samba funcionando
- ✅ Control de acceso por grupos verificado

**Estado actual del dominio:**

- Dominio: lab02.lan funcionando completamente
- DHCP: Listo para asignar IPs a clientes
- Estructura AD: Organizada por departamentos
- Permisos: Basados en grupos de seguridad
- Recursos: Compartidos y protegidos

---

# 🧱 SPRINT 3 – Integración de Cliente Windows al Dominio

**Objetivo:** Configurar un cliente Windows 10 Pro, unirlo al dominio lab02.lan y verificar acceso a recursos compartidos.

## 📋 PREPARACIÓN DEL CLIENTE WINDOWS (ANTES DE EMPEZAR)

**Requisitos previos:**

- Máquina virtual con Windows 10 Pro (o superior)
- Nombre de equipo de la VM: `lc02` (Lab Client 02)
- Al menos 4GB RAM, 2 CPUs, 50GB disco

### Configuración de red en VirtualBox

Abrir configuración de la VM `lc02` en VirtualBox:

1. Clic derecho en la VM → Configuración → Red

**Adaptador 1 (Red Interna):**

- Habilitar adaptador de red: ✓
- Conectado a: Red interna
- Nombre: intnet
- Modo promiscuo: Permitir todo

> ⚠️ **IMPORTANTE:** El cliente SOLO tiene un adaptador de red (Red Interna). NO configurar adaptador puente.

### Configuración de red en Windows

Después de iniciar Windows:

1. Abrir Configuración de red:
   - Panel de control → Redes e Internet → Centro de redes y recursos compartidos
   - Clic en "Ethernet" o "Conexión de área local"
   - Propiedades

2. Configurar TCP/IPv4:
   - Seleccionar "Protocolo de Internet versión 4 (TCP/IPv4)"
   - Propiedades
   - Seleccionar "Usar la siguiente dirección IP"

**Configuración:**

```
Dirección IP:                    192.168.11.100
Máscara de subred:               255.255.255.0
Puerta de enlace predeterminada: 192.168.11.2
Servidor DNS preferido:          192.168.11.2
Servidor DNS alternativo:        (dejar en blanco o 10.239.3.7)
```

3. Aceptar y cerrar todas las ventanas

🔍 **Verificación de red:**

Abrir CMD o PowerShell:

```cmd
ipconfig /all
```

Debe mostrar:

```
Dirección IPv4:  192.168.11.100
Máscara de subred: 255.255.255.0
Puerta de enlace: 192.168.11.2
Servidores DNS:   192.168.11.2
```

Probar conectividad:

```cmd
ping 192.168.11.2
ping ls02.lab02.lan
ping www.amazon.es
```

Todos deben responder correctamente.

🛠 **Si falla el ping a ls02.lab02.lan:**

```cmd
nslookup ls02.lab02.lan
```

Debe devolver: `192.168.11.2`

Si no resuelve:

- Verificar que el DNS sea 192.168.11.2
- Ejecutar: `ipconfig /flushdns`
- Reintentar

🛠 **Si falla el ping a www.amazon.es:**

- Verificar que la puerta de enlace sea 192.168.11.2
- Verificar que el servidor Ubuntu tenga NAT configurado (SPRINT 1, HORA 5)
- Ejecutar en el servidor: `sudo iptables -t nat -L -v`

---

### Configurar nombre del equipo

1. Sistema → Acerca de → Cambiar nombre de este equipo
2. Nombre del equipo: `LC02`
3. Reiniciar cuando se solicite

🔍 **Verificación:**

Abrir CMD:

```cmd
hostname
```

Debe devolver: `LC02`

---

### Verificar edición de Windows

> ⚠️ **CRÍTICO:** Necesitas Windows 10 Pro, Enterprise o Education. Windows 10 Home NO puede unirse a un dominio.

**Verificar edición:**

1. Sistema → Acerca de
2. Buscar "Edición de Windows"

Debe decir:

- Windows 10 Pro
- Windows 10 Enterprise
- Windows 10 Education

🛠 **Si tienes Windows 10 Home:**

- Necesitas actualizar a Pro (licencia de pago)
- O usar otra VM con Windows 10 Pro

---

### Estado inicial del cliente

✅ **Antes de comenzar el SPRINT 3, verifica:**

1. **Red configurada:**
```cmd
ipconfig
```
→ IP: 192.168.11.100, DNS: 192.168.11.2

2. **DNS funcional:**
```cmd
nslookup lab02.lan
```
→ Devuelve: 192.168.11.2

3. **Conectividad al servidor:**
```cmd
ping ls02.lab02.lan
```
→ Responde correctamente

4. **Conectividad externa:**
```cmd
ping www.amazon.es
```
→ Responde correctamente

5. **Nombre del equipo:**
```cmd
hostname
```
→ `LC02`

6. **Edición correcta:**
   - Sistema → Acerca de → Windows 10 Pro (o superior)

---

## 🕒 HORA 1: Unir el cliente Windows al dominio

**Objetivo:** Integrar el cliente Windows en el dominio lab02.lan.

### 1.1 Preparar credenciales de administrador del dominio

Antes de unir al dominio, ten a mano:

- Nombre de dominio: `lab02.lan`
- Usuario administrador: `Administrator`
- Contraseña: `admin_21`

---

### 1.2 Unir el equipo al dominio

1. Abrir Configuración del sistema:
   - Clic derecho en "Este equipo" → Propiedades
   - O: Sistema → Acerca de → Configuración avanzada del sistema

2. Cambiar configuración:
   - Pestaña "Nombre de equipo"
   - Botón "Cambiar..."

3. Configurar dominio:
   - Miembro de: Seleccionar Dominio
   - Escribir: `lab02.lan`
   - Clic en "Aceptar"

4. Credenciales de administrador:
   - Se abrirá ventana pidiendo credenciales
   - Usuario: `Administrator`
   - Contraseña: `admin_21`
   - Clic en "Aceptar"

5. Mensaje de bienvenida:
   - Debe aparecer: "Bienvenido al dominio lab02.lan"
   - Clic en "Aceptar"

6. Reiniciar el equipo:
   - Clic en "Aceptar" → "Cerrar"
   - Reiniciar ahora

🔍 **Verificación después del reinicio:**

1. Pantalla de inicio de sesión:
   - Debe aparecer "Iniciar sesión en: LAB02"
   - O ver opción para cambiar dominio

2. Iniciar sesión con usuario del dominio:
   - Usuario: `Administrator` (o `LAB02\Administrator`)
   - Contraseña: `admin_21`

🛠 **Si falla la unión al dominio:**

**Error: "No se puede encontrar el dominio"**

```cmd
nslookup lab02.lan
nslookup ls02.lab02.lan
nslookup _ldap._tcp.lab02.lan
```

```cmd
ipconfig /all
```

Verificar que apunta a 192.168.11.2. Si no, cambiar DNS y reintentar.

**Error: "Credenciales incorrectas"**

- Verificar que el usuario es `Administrator` (con A mayúscula)
- Verificar la contraseña: `admin_21`
- Intentar con: `LAB02\Administrator`

**Error: "No se puede conectar al dominio"**

```cmd
ping ls02.lab02.lan
```

```powershell
Test-NetConnection -ComputerName ls02.lab02.lan -Port 389
```

Si falla, verificar firewall del servidor.

Desde el servidor Ubuntu, verificar que Samba esté corriendo:

```bash
sudo systemctl status samba-ad-dc
```

---

### 1.3 Verificar cuenta de equipo en el dominio

Desde el servidor Ubuntu:

```bash
# Listar equipos del dominio
sudo samba-tool computer list
```

Debe aparecer: `LC02$` (el símbolo `$` indica cuenta de equipo)

Ver información del equipo:

```bash
sudo samba-tool computer show LC02$
```

---

## 🕒 HORA 2: Iniciar sesión con usuarios del dominio

**Objetivo:** Verificar que los usuarios del dominio pueden iniciar sesión en el cliente Windows.

### 2.1 Cerrar sesión de Administrator

1. Cerrar sesión:
   - Inicio → Usuario → Cerrar sesión

2. Pantalla de inicio de sesión:
   - Debe mostrar "Iniciar sesión en: LAB02"

---

### 2.2 Iniciar sesión con user01

1. En la pantalla de inicio:
   - Usuario: `user01`
   - Contraseña: `admin_21`
   - Iniciar sesión

2. Primera vez:
   - Windows creará el perfil del usuario
   - Puede tardar 1-2 minutos
   - Se abrirá el escritorio de Windows

🔍 **Verificación:**

Abrir CMD o PowerShell:

```cmd
whoami
```

Debe devolver: `lab02\user01`

```cmd
echo %USERDOMAIN%
echo %USERNAME%
```

Debe devolver:

```
LAB02
user01
```

---

### 2.3 Probar con otros usuarios

Cerrar sesión y probar con:

**bob:**
- Usuario: `bob`
- Contraseña: `admin_21`

**alice:**
- Usuario: `alice`
- Contraseña: `admin_21`

🛠 **Si falla el inicio de sesión de un usuario:**

**Error: "El nombre de usuario o contraseña es incorrecta"**

```bash
# Verificar desde el servidor que el usuario existe
sudo samba-tool user list | grep user01

# Ver información del usuario
sudo samba-tool user show user01

# Verificar estado de la cuenta
sudo samba-tool user show user01 | grep -i "disabled\|locked"
```

**Si el usuario está deshabilitado:**

```bash
sudo samba-tool user enable user01
```

**Si la contraseña está mal o expiró:**

```bash
sudo samba-tool user setpassword user01
# Introducir nueva contraseña: admin_21
```

---

## 🕒 HORA 3: Acceso a recursos compartidos desde Windows

**Objetivo:** Verificar que los usuarios pueden acceder a las carpetas compartidas.

### 3.1 Mapear recurso FinanceDocs como user01

Iniciar sesión como `user01` en el cliente Windows.

1. Abrir Explorador de archivos

2. Acceder a recursos del servidor:
   - En la barra de direcciones, escribir: `\\ls02.lab02.lan`
   - O: `\\192.168.11.2`
   - Presionar Enter

3. Debe mostrar los recursos compartidos:
   - FinanceDocs
   - HRDocs
   - Public
   - NETLOGON
   - SYSVOL

4. Acceder a FinanceDocs:
   - Doble clic en FinanceDocs
   - Debe abrirse la carpeta

5. Crear archivo de prueba:
   - Clic derecho → Nuevo → Documento de texto
   - Nombre: `test_user01.txt`
   - Escribir algo dentro y guardar

🔍 **Verificación desde el servidor:**

```bash
sudo ls -la /srv/samba/FinanceDocs/
```

Debe aparecer el archivo `test_user01.txt`.

---

### 3.2 Intentar acceder a HRDocs como user01 (debe fallar)

> ⚠️ **IMPORTANTE:** En este momento, como NO hemos configurado ACLs desde Windows, user01 TODAVÍA puede acceder a HRDocs. Esto es correcto.

Desde el Explorador de archivos:

1. Navegar a `\\ls02.lab02.lan`
2. Intentar abrir HRDocs
3. Debe abrirse (aún sin restricciones)

Esto se corregirá en la HORA 4 cuando configuremos ACLs desde Windows.

---

### 3.3 Mapear unidad de red

**Mapear FinanceDocs como unidad Z:**

1. En el Explorador de archivos:
   - Clic en "Este equipo"
   - Cinta superior → "Conectar a unidad de red"

2. Configurar:
   - Unidad: `Z:`
   - Carpeta: `\\ls02.lab02.lan\FinanceDocs`
   - ✓ Reconectar al iniciar sesión
   - ✓ Conectar con credenciales diferentes (si es necesario)
   - Finalizar

3. Debe aparecer unidad `Z:` en "Este equipo"

🔍 **Verificación:**

Abrir CMD:

```cmd
net use
```

Debe mostrar:

```
Z:  \\ls02.lab02.lan\FinanceDocs
```

---

## 🕒 HORA 4: Configurar permisos (ACLs) desde Windows

**Objetivo:** Configurar ACLs avanzadas para restringir acceso a carpetas por grupo.

### 4.1 Iniciar sesión como Administrator

Cerrar sesión de user01 e iniciar sesión como:

- Usuario: `Administrator`
- Contraseña: `admin_21`

---

### 4.2 Configurar permisos en FinanceDocs

1. Navegar a:
   - `\\ls02.lab02.lan\FinanceDocs`

2. Clic derecho en la carpeta FinanceDocs (en la barra de arriba, no dentro)
   - Propiedades

3. Pestaña "Seguridad":
   - Clic en "Opciones avanzadas"

4. Deshabilitar herencia:
   - Clic en "Deshabilitar herencia"
   - Seleccionar: "Reemplazar todas las entradas de permisos de objetos secundarios por entradas de permisos heredables de este objeto"

5. Eliminar permisos innecesarios:
   - Seleccionar y eliminar grupos/usuarios que NO sean:
     - Administradores del dominio
     - SYSTEM
     - CREATOR OWNER (permite que quien cree un archivo dentro tenga control sobre ese archivo)
     - Finance (si existe)

6. Añadir grupo Finance:
   - Clic en "Agregar"
   - "Seleccionar una entidad de seguridad"
   - Escribir: `Finance`
   - "Comprobar nombres" (debe subrayarse)
   - Aceptar
   - Permisos:
     - Control total: ✓
     - O: Modificar: ✓
   - Aplicar a: Esta carpeta, subcarpetas y archivos
   - Aceptar

7. Denegar acceso al grupo HR:
   - Clic en "Agregar"
   - Seleccionar entidad: `HR`
   - Tipo: Denegar
   - Permisos básicos: Control total (marcar en la columna Denegar)
   - Aceptar

8. Aplicar y Aceptar en todas las ventanas

> **NOTA:** Si sale un aviso de Windows mediante una ventana que dice algo como:
>
> *"Seguridad de Windows. La directiva de supervisión actual de este equipo no tiene activada la auditoría. Si este equipo obtiene la directiva de auditoría del dominio, pida al administrador de dominio que active la auditoría con el Editor de directivas de grupo. De lo contrario, utilice el Editor de directivas de equipo local para configurar la directiva de supervisión localmente en este equipo."*
>
> No hagas caso, básicamente es como si Windows te estuviera diciendo: "Oye, si quisieras registrar estos accesos en los logs, ahora mismo no lo estoy haciendo."

🔍 **Verificación:**

Desde el cliente Windows (como Administrator):

- Propiedades de FinanceDocs → Seguridad
- Debe mostrar:
  - Finance: Permitir - Control total
  - HR: Denegar - Control total
  - Administradores del dominio: Permitir - Control total
  - SYSTEM: Permitir - Control total

---

### 4.3 Configurar permisos en HRDocs

Repetir el mismo proceso para HRDocs:

1. `\\ls02.lab02.lan\HRDocs` → Propiedades → Seguridad → Opciones avanzadas
2. Deshabilitar herencia
3. Añadir grupo HR con Control total (Permitir)
4. Añadir grupo Finance con Control total (Denegar)
5. Aplicar

---

### 4.4 Configurar permisos en Public

1. `\\ls02.lab02.lan\Public` → Propiedades → Seguridad → Opciones avanzadas
2. Deshabilitar herencia
3. NO marcar la casilla de "Reemplazar todas las entradas de permisos de objetos secundarios por entradas de permisos heredables de este objeto"
4. Añadir grupo Domain Users con Lectura y Ejecución (Permitir)
5. Debe de quedarse con las siguientes entidades:
   - Administrator → Control total
   - Domain Users → Lectura y ejecución
   - CREATOR OWNER → Control total (subcarpetas/archivos)
6. Aplicar

---

## 🕒 HORA 5: Verificar restricciones de acceso

**Objetivo:** Comprobar que las ACLs funcionan correctamente.

### 5.1 Probar acceso como user01 (grupo Finance)

Cerrar sesión de Administrator e iniciar como `user01`.

Navegar a `\\ls02.lab02.lan`:

**Prueba 1: Acceder a FinanceDocs (debe funcionar)**

- Doble clic en FinanceDocs
- Debe abrir sin problemas
- Crear archivo: `test_acl_user01.txt`
- Debe crearse correctamente

**Prueba 2: Acceder a HRDocs (debe FALLAR)**

- Doble clic en HRDocs
- Debe mostrar: "No tiene permiso para acceder a \ls02.lab02.lan\HRDocs"

✅ Correcto - El grupo Finance está denegado en HRDocs

**Prueba 3: Acceder a Public (solo lectura)**

- Doble clic en Public
- Debe abrir
- Intentar crear archivo
- Debe mostrar: "No tiene permisos para guardar en esta ubicación"

✅ Correcto - Public es solo lectura

---

### 5.2 Probar acceso como alice (grupo HR)

Cerrar sesión de user01 e iniciar como `alice`.

Navegar a `\\ls02.lab02.lan`:

**Prueba 1: Acceder a HRDocs (debe funcionar)**

- Doble clic en HRDocs
- Crear archivo: `test_acl_alice.txt`
- Debe funcionar

**Prueba 2: Acceder a FinanceDocs (debe FALLAR)**

- Doble clic en FinanceDocs
- Debe mostrar error de permisos

✅ Correcto

---

### 5.3 Verificar ACLs desde el servidor Linux

Desde el servidor Ubuntu:

```bash
# Ver ACLs de FinanceDocs
sudo getfacl /srv/samba/FinanceDocs/

# Ver ACLs de HRDocs
sudo getfacl /srv/samba/HRDocs/
```

Debe mostrar las ACLs configuradas desde Windows almacenadas en formato extendido.

---

## 🕒 HORA 6: Verificación final y checkpoint del SPRINT 3

**Objetivo:** Verificar que todo el SPRINT 3 está completado correctamente.

### 6.1 Verificación de unión al dominio

Desde el servidor Ubuntu:

```bash
# Listar equipos del dominio
sudo samba-tool computer list
```

Debe mostrar: `LC02$`

---

### 6.2 Verificación de inicio de sesión

Desde el cliente Windows:

1. Cerrar sesión
2. Iniciar sesión con diferentes usuarios: `user01`, `alice`, `bob`
3. Todos deben poder iniciar sesión correctamente

---

### 6.3 Verificación de recursos compartidos

Desde el cliente Windows (como Administrator):

```cmd
net view \\ls02.lab02.lan
```

Debe listar:

```
FinanceDocs
HRDocs
Public
NETLOGON
SYSVOL
```

---

### 6.4 Verificación de ACLs

**Como user01:**

- ✅ Puede acceder a FinanceDocs
- ❌ NO puede acceder a HRDocs
- ✅ Puede leer (no escribir) en Public

**Como alice:**

- ❌ NO puede acceder a FinanceDocs
- ✅ Puede acceder a HRDocs
- ✅ Puede leer (no escribir) en Public

---

### 6.5 Checkpoint final del SPRINT 3

✅ **Lista de comprobaciones obligatorias:**

1. **Cliente configurado correctamente:**
```cmd
ipconfig
```
→ IP: 192.168.11.100, DNS: 192.168.11.2

2. **Unido al dominio:**
```cmd
systeminfo | findstr /B /C:"Dominio"
```
→ Dominio: lab02.lan

3. **Usuarios pueden iniciar sesión:**
   - Probar con `user01`, `alice`, `bob`

4. **Recursos compartidos accesibles:**
```cmd
net view \\ls02.lab02.lan
```

5. **ACLs funcionando:**
   - user01 accede a FinanceDocs, NO a HRDocs
   - alice accede a HRDocs, NO a FinanceDocs

6. **Equipo visible desde el servidor:**
```bash
sudo samba-tool computer list
```
→ `LC02$`

---

## 🛠 PLAN DE RESCATE GENERAL

**Si el cliente no se puede unir al dominio:**

1. Verificar DNS: `nslookup lab02.lan`
2. Verificar conectividad: `ping ls02.lab02.lan`
3. Verificar Samba en servidor: `sudo systemctl status samba-ad-dc`
4. Ver logs del servidor: `sudo tail -100 /var/log/samba/log.samba`

**Si los usuarios no pueden iniciar sesión:**

1. Verificar que el usuario existe: `sudo samba-tool user list`
2. Verificar que no está bloqueado: `sudo samba-tool user show user01`
3. Resetear contraseña: `sudo samba-tool user setpassword user01`

**Si no aparecen recursos compartidos:**

1. Verificar smb.conf: `sudo testparm -s`
2. Verificar que Samba esté corriendo: `sudo systemctl status samba-ad-dc`
3. Reiniciar Samba: `sudo systemctl restart samba-ad-dc`

**Si las ACLs no funcionan:**

1. Verificar vfs objects en smb.conf: `sudo testparm -s | grep vfs`
2. Verificar que se configuraron desde Windows (Propiedades → Seguridad)
3. Eliminar y reconfigurar ACLs desde Windows

---

## 🎯 FIN DEL SPRINT 3

Has completado:

- ✅ Configuración completa de cliente Windows (red, nombre, DNS)
- ✅ Unión del cliente al dominio lab02.lan
- ✅ Inicio de sesión con usuarios del dominio
- ✅ Acceso a recursos compartidos desde Windows
- ✅ Mapeo de unidades de red
- ✅ Configuración de ACLs desde Windows
- ✅ Verificación de restricciones de acceso por grupo

**Estado actual del entorno:**

- Servidor Ubuntu: Controlador de dominio Samba AD DC funcionando
- Cliente Windows: Unido al dominio, usuarios pueden iniciar sesión
- Recursos compartidos: Accesibles con control de acceso por grupo
- ACLs: Configuradas desde Windows, funcionando correctamente

**Siguiente:** SPRINT 4 → RSAT, Cliente Ubuntu y Gestión del Sistema

---

# 🧱 SPRINT 4 – GPOs, Cliente Ubuntu y Gestión del Sistema

**Objetivo:** Crear y aplicar GPOs directamente desde el servidor Ubuntu usando samba-tool (alternativamente usando RSAT), integrar cliente Ubuntu al dominio, montar recursos compartidos, gestionar procesos, automatizar backups y configurar auditoría.

---

## 🕒 HORA 1: Configuración de GPOs (Comandos Ubuntu + RSAT Windows)

**Objetivo:** Crear GPOs usando comandos en Ubuntu Server y configurarlas con RSAT desde Windows.

> ⚠️ **ESTRUCTURA DE ESTA HORA:**
> - **PARTE A:** Crear GPOs vacías desde Ubuntu Server (solo estructura, sin configuración compleja, ya que unicamente se pueden configurar en las contraseñas)
> - **PARTE B:** Configurar GPOs avanzadas desde Windows con RSAT (restricciones, wallpapers, etc.)

---

## 📋 PARTE A: CREACIÓN DE GPOs DESDE UBUNTU SERVER (COMANDOS)

**Objetivo:** Crear la estructura de GPOs usando comandos samba-tool desde el servidor.

> **IMPORTANTE:** se puede hacer directamente desde Windows también (es más fácil)
>
> ⚠️ **LIMITACIÓN:** Los comandos de Samba solo permiten crear GPOs vacías (sin políticas configuradas). La configuración real se hace con RSAT.

### A.1 Autenticación Kerberos previa

Desde el servidor Ubuntu:

```bash
# Limpiar tickets anteriores
kdestroy

# Obtener nuevo ticket de Administrator
kinit Administrator
```

Introducir contraseña: `admin_21`

🔍 **Verificación:**

```bash
klist
```

Debe mostrar ticket válido para `Administrator@LAB02.LAN`

Alternativamente, trabajar como root:

```bash
sudo -i
```

---

### A.2 Ver GPOs existentes por defecto

```bash
sudo samba-tool gpo listall
```

🔍 **Debe mostrar al menos:**

```
GPO          : {31B2F340-016D-11D2-945F-00C04FB984F9}
display name : Default Domain Policy
path         : \\lab02.lan\sysvol\lab02.lan\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}
dn           : CN={31B2F340-016D-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=lab02,DC=lan
version      : 0
flags        : NONE

GPO          : {6AC1786C-016F-11D2-945F-00C04FB984F9}
display name : Default Domain Controllers Policy
path         : \\lab02.lan\sysvol\lab02.lan\Policies\{6AC1786C-016F-11D2-945F-00C04FB984F9}
dn           : CN={6AC1786C-016F-11D2-945F-00C04FB984F9},CN=Policies,CN=System,DC=lab02,DC=lan
version      : 0
flags        : NONE
```

---

### A.3 Crear GPO vacía: Restricciones de Usuario

```bash
sudo samba-tool gpo create "Restricciones_Usuarios" -U Administrator
```

Introducir contraseña: `admin_21`

🔍 **Debe mostrar:**

```
GPO 'Restricciones_Usuarios' created as {XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}
```

> ⚠️ **IMPORTANTE:** Anotar el GUID generado (lo necesitarás para vincular).

---

### A.4 Crear GPO vacía: Configuración de Escritorio

```bash
sudo samba-tool gpo create "Configuracion_Escritorio" -U Administrator
```

Anotar el GUID generado.

---

### A.5 Verificar GPOs creadas

```bash
sudo samba-tool gpo listall
```

Deben aparecer las dos GPOs nuevas: `Restricciones_Usuarios` y `Configuracion_Escritorio`.

Ver detalles de una GPO específica:

```bash
sudo samba-tool gpo show "{GUID_DE_LA_GPO}"
```

---

### A.6 Vincular GPO a una OU

**Vincular "Restricciones_Usuarios" a OU=Students:**

```bash
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Restricciones_Usuarios}" -U Administrator
```

**Vincular "Configuracion_Escritorio" a OU=Students:**

```bash
sudo samba-tool gpo setlink "OU=Students,DC=lab02,DC=lan" "{GUID_DE_Configuracion_Escritorio}" -U Administrator
```

🔍 **Verificación del enlace:**

```bash
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

Debe mostrar los GUIDs de ambas GPOs vinculadas.

---

### A.7 Políticas de Contraseñas y Bloqueo (ya configurado en SPRINT 1 - HORA 6)

> ⚠️ **NOTA:** Las políticas de contraseñas y bloqueo de cuenta YA FUERON CONFIGURADAS en SPRINT 1, HORA 6 usando `samba-tool domain passwordsettings`.

**Verificar configuración actual:**

```bash
sudo samba-tool domain passwordsettings show
```

🔍 **Debe mostrar:**

```
Password information for domain 'DC=lab02,DC=lan'

Password complexity: on
Store plaintext passwords: off
Password history length: 24
Minimum password length: 7
Minimum password age (days): 1
Maximum password age (days): 42
Account lockout duration (mins): 30
Account lockout threshold (attempts): 0
Reset account lockout after (mins): 30
```

**Si necesitas modificar alguna política:**

```bash
# Longitud mínima
sudo samba-tool domain passwordsettings set --min-pwd-length=8

# Complejidad
sudo samba-tool domain passwordsettings set --complexity=on

# Edad máxima (días)
sudo samba-tool domain passwordsettings set --max-pwd-age=90

# Historial
sudo samba-tool domain passwordsettings set --history-length=12

# Umbral de bloqueo (intentos)
sudo samba-tool domain passwordsettings set --account-lockout-threshold=5

# Duración del bloqueo (minutos)
sudo samba-tool domain passwordsettings set --account-lockout-duration=30

# Ventana de observación (minutos)
sudo samba-tool domain passwordsettings set --reset-account-lockout-after=15
```

**Verificar cambios:**

```bash
sudo samba-tool domain passwordsettings show
```

---

### A.8 Verificar estructura física de las GPOs

```bash
# Navegar al directorio de políticas
cd /var/lib/samba/sysvol/lab02.lan/Policies/

# Listar GPOs
ls -la
```

Debe mostrar directorios con los GUIDs de las GPOs creadas.

**Ver contenido de una GPO:**

```bash
ls -la /var/lib/samba/sysvol/lab02.lan/Policies/{GUID}/
```

Debe mostrar:

```
GPT.INI
Machine/
User/
```

---

### A.9 Checkpoint de la Parte A

✅ **Verificaciones obligatorias:**

1. **Ticket Kerberos válido:**
```bash
klist
```

2. **GPOs vacías creadas:**
```bash
sudo samba-tool gpo listall | grep -E "Restricciones|Configuracion"
```

3. **GPOs vinculadas a OU=Students:**
```bash
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

4. **Políticas de contraseñas configuradas:**
```bash
sudo samba-tool domain passwordsettings show
```

5. **Estructura física existe:**
```bash
ls -la /var/lib/samba/sysvol/lab02.lan/Policies/
```

---

## 🖥 PARTE B: CONFIGURACIÓN DE GPOs DESDE WINDOWS (RSAT)

**Objetivo:** Configurar las GPOs creadas en la Parte A usando RSAT desde el cliente Windows.

> ⚠️ **IMPORTANTE:** Las GPOs están vacías (creadas en Parte A). Ahora las configuraremos con políticas reales.

### B.1 Instalar RSAT en Windows 10

Desde el cliente Windows (como Administrator del dominio):

1. Abrir Configuración:
   - Inicio → Configuración → Aplicaciones

2. Características opcionales:
   - Aplicaciones y características → Características opcionales → Agregar una característica

3. Buscar e instalar RSAT:
   - Buscar: `RSAT`
   - Instalar:
     - RSAT: Herramientas de administración de directivas de grupo
     - RSAT: Herramientas de AD DS y AD LDS (llamado: RSAT: Herramientas de servicios de dominio de Active Directory y servicios de directorio ligero)

4. Esperar a que se instalen (puede tardar 2-5 minutos)

🔍 **Verificación:**

- Inicio → Buscar: `Administración de directivas de grupo`
- Debe aparecer la aplicación

🛠 **Si no aparece RSAT en Características opcionales:**

```powershell
# Ejecutar PowerShell como Administrador
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

---

### B.2 Abrir Consola de Administración de Directivas de Grupo

1. Inicio → Administración de directivas de grupo

2. Conectar al dominio:
   - Se debe conectar automáticamente a `lab02.lan`
   - Si no, clic derecho en "Administración de directivas de grupo" → Cambiar controlador de dominio → `ls02.lab02.lan`

3. Expandir:
   - Bosque: lab02.lan
   - Dominios
   - lab02.lan
   - Objetos de directiva de grupo

🔍 **Verificación:**

- Deben aparecer las GPOs creadas en la Parte A:
  - `Restricciones_Usuarios`
  - `Configuracion_Escritorio`

🛠 **Si falla la conexión:**

- Verificar que estás logueado como Administrator del dominio
- Verificar conectividad: `ping ls02.lab02.lan`
- Verificar DNS: `nslookup lab02.lan`

---

### B.3 Configurar GPO: Prohibir acceso al Panel de Control

**Editar la GPO "Restricciones_Usuarios":**

1. Clic derecho en `Restricciones_Usuarios` → Editar

Navegar a: Configuración de usuario > Directivas > Plantillas administrativas > Panel de Control > Personalización > Prohibit access to Control Panel and PC settings

2. Doble clic en "Prohibit access to Control Panel and PC settings"
   - Seleccionar: Habilitado
   - Aplicar → Aceptar

3. Cerrar el Editor de directivas de grupo

---

### B.4 Configurar GPO: Fondo de Escritorio

**Editar la GPO "Configuracion_Escritorio":**

1. Clic derecho en `Configuracion_Escritorio` → Editar

Navegar a: Configuración de usuario > Directivas > Plantillas administrativas > Panel de Control > Personalización > Impedir cambiar el fondo de pantalla

Navegar a: Configuración de usuario > Directivas > Plantillas administrativas > Active Desktop > Active Desktop > Tapiz del escritorio

2. Cerrar el Editor

---

### B.5 Verificar vínculos de GPOs en RSAT

En la Consola de Administración de directivas de grupo:

1. Expandir:
   - lab02.lan → Students

2. Verificar que aparezcan vinculadas:
   - `Restricciones_Usuarios`
   - `Configuracion_Escritorio`

🔍 **Verificación desde el servidor Ubuntu:**

```bash
sudo samba-tool gpo getlink "OU=Students,DC=lab02,DC=lan"
```

---

### B.6 Forzar aplicación de GPOs en el cliente

Desde el cliente Windows (como user01):

```cmd
gpupdate /force
```

Esperar a que termine (puede tardar 1-2 minutos).

**Reiniciar sesión:**

- Cerrar sesión
- Iniciar como `user01`

🔍 **Verificación:**

- Intentar abrir Panel de Control → Debe estar bloqueado
- El fondo de escritorio debe cambiar automáticamente

---

### B.7 Verificar GPOs aplicadas desde el cliente

```cmd
REM Ver GPOs aplicadas al usuario actual
gpresult /r

REM Generar reporte HTML detallado
gpresult /h C:\gpo_report.html
```

Abrir `gpo_report.html` en navegador:

- Debe mostrar `Restricciones_Usuarios` y `Configuracion_Escritorio` aplicadas

---

### B.8 GPOs adicionales comunes (ejemplos rápidos)

Todas estas se configuran desde RSAT, editando la GPO correspondiente:

**Bloquear Command Prompt (CMD)**

```
User Configuration → Administrative Templates → System
→ Prevent access to the command prompt
→ Habilitado
```

**Deshabilitar Task Manager**

```
User Configuration → Administrative Templates → System → Ctrl+Alt+Del Options
→ Remove Task Manager
→ Habilitado
```

**Bloquear acceso al Registro**

```
User Configuration → Administrative Templates → System
→ Prevent access to registry editing tools
→ Habilitado
```

**Ocultar unidades específicas**

```
User Configuration → Administrative Templates → Windows Components → File Explorer
→ Hide these specified drives in My Computer
→ Habilitado → Seleccionar unidades
```

---

## 🎯 CHECKPOINT FINAL DE LA HORA 1

✅ **Resumen completo:**

**PARTE A (Comandos Ubuntu):**

- ✅ GPOs vacías creadas con `samba-tool gpo create`
- ✅ GPOs vinculadas a OUs con `samba-tool gpo setlink`
- ✅ Políticas de contraseñas configuradas (SPRINT 1 - HORA 6)

**PARTE B (RSAT Windows):**

- ✅ RSAT instalado en cliente Windows
- ✅ GPOs configuradas con políticas reales
- ✅ GPOs aplicadas y verificadas en clientes

**Estado final:**

- 2 GPOs creadas: `Restricciones_Usuarios`, `Configuracion_Escritorio`
- Vinculadas a: `OU=Students`
- Políticas aplicadas: Prohibir Panel de Control, Fondo de Escritorio
- Políticas de dominio: Contraseñas y bloqueo (ya configuradas en SPRINT 1)

---

## 🛠 SOLUCIÓN DE PROBLEMAS - GENERAL

**GPO no se aplica en clientes:**

```cmd
REM Desde cliente Windows
gpupdate /force
gpresult /r

REM Verificar errores
eventvwr.msc
REM Ir a: Windows Logs → System → Filtrar por "Group Policy"
```

**Error de permisos al editar GPO desde RSAT:**

```bash
# Desde servidor Ubuntu
sudo samba-tool ntacl sysvolreset
sudo systemctl restart samba-ad-dc
```

**RSAT no muestra las GPOs creadas:**

```bash
# Verificar que existen en el servidor
sudo samba-tool gpo listall

# Verificar conectividad desde Windows
ping ls02.lab02.lan
nslookup lab02.lan

# Reconectar RSAT
# En Consola de Administración → Cambiar controlador → ls02.lab02.lan
```

---

## 📝 CONCLUSIÓN

**Diferencias clave:**

- Comandos Ubuntu (`samba-tool`): Solo crea estructura (GPOs vacías, vínculos)
- RSAT Windows: Configura el contenido real de las GPOs (políticas, restricciones)

**Flujo de trabajo recomendado:**

1. Crear GPOs vacías desde Ubuntu (Parte A)
2. Vincularlas a OUs desde Ubuntu (Parte A)
3. Configurar políticas desde Windows con RSAT (Parte B)
4. Aplicar y verificar desde clientes (Parte B)

---

## 🕒 HORA 2: Preparación y unión del cliente Ubuntu al dominio

**Objetivo:** Configurar cliente Ubuntu Desktop/Server y unirlo a lab02.lan.

## 📋 PREPARACIÓN DEL CLIENTE UBUNTU (ANTES DE EMPEZAR)

**Requisitos:**

- Ubuntu 22.04/24.04 Desktop o Server
- Nombre de VM sugerido: `lc02-ubu` (Lab Client Ubuntu 02)
- 2GB RAM mínimo, 20GB disco

**Configuración de red en VirtualBox:**

- Adaptador 1: Red Interna (intnet)

---

### 2.1 Configurar red estática en Ubuntu

Editar Netplan:

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Contenido:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.11.101/24
      routes:
        - to: default
          via: 192.168.11.2
      nameservers:
        addresses:
          - 192.168.11.2
          - 10.239.3.7
```

Aplicar:

```bash
sudo netplan apply
```

🔍 **Verificación:**

```bash
ip addr show enp0s3
ping 192.168.11.2
ping ls02.lab02.lan
```

---

### 2.2 Configurar nombre del equipo

```bash
sudo hostnamectl set-hostname lc02-ubu
```

🔍 **Verificación:**

```bash
hostnamectl
```

Debe mostrar: `Static hostname: lc02-ubu`

---

### 2.3 Editar /etc/hosts

```bash
sudo nano /etc/hosts
```

Añadir:

```
127.0.0.1    localhost
127.0.1.1    lc02-ubu
192.168.11.2 ls02.lab02.lan ls02
192.168.11.2 lab02.lan
```

🔍 **Verificación:**

```bash
ping lab02.lan
ping ls02.lab02.lan
```

Debe responder desde `192.168.11.2`

---

### 2.4 Instalar paquetes necesarios

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools adcli krb5-user samba-common-bin packagekit
```

**Durante la instalación de Kerberos:**

- Default realm: `LAB02.LAN` (mayúsculas)
- Kerberos servers: `ls02.lab02.lan`
- Administrative server: `ls02.lab02.lan`

---

### 2.5 Descubrir el dominio

```bash
sudo realm discover lab02.lan
```

🔍 **Debe mostrar:**

```
lab02.lan
  type: kerberos
  realm-name: LAB02.LAN
  domain-name: lab02.lan
  configured: no
  server-software: active-directory
  client-software: sssd
  required-package: sssd-tools
  required-package: sssd
  required-package: adcli
  required-package: samba-common-bin
```

🛠 **Si no encuentra el dominio:**

```bash
# Verificar DNS
nslookup lab02.lan
nslookup _ldap._tcp.lab02.lan

# Verificar /etc/resolv.conf
cat /etc/resolv.conf
# Debe tener: nameserver 192.168.11.2
```

---

### 2.6 Unir el cliente al dominio

```bash
sudo realm join -U Administrator lab02.lan --verbose
```

Introducir contraseña: `admin_21`

🔍 **Verificación:**

```bash
sudo realm list
```

Debe mostrar:

```
lab02.lan
  type: kerberos
  realm-name: LAB02.LAN
  domain-name: lab02.lan
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
```

**Verificar desde el servidor Ubuntu:**

```bash
sudo samba-tool computer list
```

Debe aparecer: `lc02-ubu$`

---

### 2.7 Configurar SSSD para home directories

```bash
sudo nano /etc/sssd/sssd.conf
```

Añadir o modificar en la sección `[domain/lab02.lan]`:

```ini
[domain/lab02.lan]
# ... (configuración existente)
fallback_homedir = /home/%u@%d
default_shell = /bin/bash
```

Reiniciar SSSD:

```bash
sudo systemctl restart sssd
```

---

### 2.8 Permitir inicio de sesión de usuarios del dominio

```bash
sudo pam-auth-update --enable mkhomedir
```

Seleccionar con ESPACIO:

- `[*]` Create home directory on login

Tab → Ok

---

### 2.9 Iniciar sesión con usuario del dominio

**Opción 1: Desde terminal (SSH o TTY):**

```bash
su - bob@lab02.lan
```

Introducir contraseña: `admin_21`

🔍 **Verificación:**

```bash
whoami
pwd
```

Debe mostrar:

```
bob@lab02.lan
/home/bob@lab02.lan
```

**Opción 2: Desde interfaz gráfica (si es Ubuntu Desktop):**

- Cerrar sesión
- Iniciar sesión con: `bob@lab02.lan`
- Contraseña: `admin_21`

---

## 🕒 HORA 3: Montaje de recursos compartidos desde Linux

**Objetivo:** Acceder a las carpetas compartidas de Samba desde el cliente Ubuntu.

### 3.1 Instalar herramientas CIFS

```bash
sudo apt update
sudo apt install -y cifs-utils smbclient
```

---

### 3.2 Verificar recursos compartidos disponibles

```bash
smbclient -L //ls02.lab02.lan -U bob
```

Introducir contraseña: `admin_21`

🔍 **Debe listar:**

```
Sharename       Type      Comment
---------       ----      -------
FinanceDocs     Disk
HRDocs          Disk
Public          Disk
NETLOGON        Disk
SYSVOL          Disk
```

Con dirección IP:

```bash
smbclient -L //192.168.11.2 -U bob
```

---

### 3.3 Probar acceso a un recurso compartido

```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
```

Dentro del prompt `smb: \>`:

```
ls
mkdir test_from_ubuntu
ls
exit
```

---

### 3.4 Montaje manual de recursos

**Crear punto de montaje:**

```bash
sudo mkdir -p /mnt/financedocs
```

**Montar FinanceDocs:**

```bash
sudo mount -t cifs //ls02.lab02.lan/FinanceDocs /mnt/financedocs -o username=user01,password=admin_21,uid=1000,gid=1000
```

🔍 **Verificación:**

```bash
ls -la /mnt/financedocs
```

**Crear archivo de prueba:**

```bash
echo "Test from Ubuntu client" | sudo tee /mnt/financedocs/test_ubuntu.txt
```

**Desmontar:**

```bash
sudo umount /mnt/financedocs
```

---

### 3.5 Montaje automático en /etc/fstab

**Crear archivo de credenciales (más seguro):**

```bash
sudo nano /root/.smbcredentials
```

Contenido:

```
username=user01
password=admin_21
domain=LAB02
```

**Proteger el archivo:**

```bash
sudo chmod 600 /root/.smbcredentials
```

**Editar fstab:**

```bash
sudo nano /etc/fstab
```

Añadir al final:

```
# Recursos compartidos de Samba
//ls02.lab02.lan/FinanceDocs /mnt/financedocs cifs credentials=/root/.smbcredentials,uid=1000,gid=1000,iocharset=utf8 0 0
```

**Crear punto de montaje:**

```bash
sudo mkdir -p /mnt/financedocs
```

**Probar montaje:**

```bash
sudo mount -a
```

🔍 **Verificación:**

```bash
df -h | grep financedocs
ls -la /mnt/financedocs
```

**Reiniciar y verificar que monta automáticamente:**

```bash
sudo reboot
```

Después del reinicio:

```bash
df -h | grep financedocs
```

---

## 🕒 HORA 4: Gestión de procesos (actividad práctica con SSH)

**Objetivo:** Controlar procesos remotamente desde el servidor vía SSH.

### 4.1 Instalar paquete sl en el cliente Ubuntu

Desde el cliente Ubuntu:

```bash
sudo apt install -y sl
```

---

### 4.2 Conectar desde el servidor al cliente vía SSH

Desde el servidor Ubuntu (ls02):

```bash
ssh bob@192.168.11.101
```

Introducir contraseña: `admin_21`

🛠 **Si falla la conexión SSH:**

```bash
# Desde el cliente, verificar que SSH esté instalado y corriendo
sudo apt install -y openssh-server
sudo systemctl status ssh
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

### 4.3 Ejecutar proceso sl desde el cliente

Desde el cliente Ubuntu (en otra terminal o TTY):

Iniciar sesión como `bob@lab02.lan` y ejecutar:

```bash
sl
```

El tren debe pasar por la pantalla.

---

### 4.4 Identificar el proceso desde el servidor (vía SSH)

Desde el servidor conectado vía SSH al cliente:

```bash
# Identificarlo más fácil
ps aux | awk '$11=="sl"' | grep sl

# Forma más simple
ps -aux | grep -w "sl"
```

Anotar el PID (ejemplo: `12345`)

---

### 4.5 Pausar el proceso

```bash
kill -19 12345
```

**Explicación:**

- Signal 19 = SIGSTOP (pausar proceso)

🔍 **Verificación:**

- El tren en la pantalla del cliente se debe congelar

---

### 4.6 Reanudar el proceso

```bash
kill -18 12345
```

**Explicación:**

- Signal 18 = SIGCONT (continuar proceso)

🔍 **Verificación:**

- El tren debe continuar moviéndose

---

### 4.7 Terminar el proceso

```bash
kill -9 12345
```

**Explicación:**

- Signal 9 = SIGKILL (terminar inmediatamente, no se puede ignorar)

🔍 **Verificación:**

- El proceso sl desaparece
- Verificar:

```bash
ps aux | grep sl
```

---

### 4.8 Monitoreo de procesos (breve)

**Ver procesos en tiempo real:**

```bash
top
```

**Filtrar por usuario:**

Presionar `u` → escribir `bob` → Enter

Salir: Presionar `q`

---

## 🕒 HORA 5: Tareas programadas con CRON (Backup automático de Samba)

**Objetivo:** Crear script de backup y programarlo con CRON.

### 5.1 Crear script de backup

Desde el servidor Ubuntu:

```bash
sudo nano /root/backup_samba.sh
```

Contenido completo del script:

```bash
#!/bin/bash

# --- CONFIGURACIÓN ---
DIR_DESTINO="/root/backups"
LOG_FILE="/var/log/samba_backup.log"
DIAS_A_GUARDAR=30

# --- COMANDOS (Rutas absolutas para CRON) ---
TAR=/bin/tar
DATE=/bin/date
ECHO=/bin/echo
FIND=/usr/bin/find
MKDIR=/bin/mkdir

# --- VARIABLES ---
FECHA=$($DATE +%F_%H-%M)
NOMBRE_ARCHIVO="backup_ad_$FECHA.tar.gz"
RUTA_COMPLETA="$DIR_DESTINO/$NOMBRE_ARCHIVO"

# --- CREAR DIRECTORIO SI NO EXISTE ---
if [ ! -d "$DIR_DESTINO" ]; then
    $MKDIR -p "$DIR_DESTINO"
fi

# --- 1. EJECUTAR BACKUP ---
$TAR -czf "$RUTA_COMPLETA" /var/lib/samba /etc/samba 2>/dev/null

# --- 2. VERIFICACIÓN Y LOG ---
if [ $? -eq 0 ]; then
    $ECHO "[$FECHA] OK: Backup creado exitosamente: $NOMBRE_ARCHIVO" >> $LOG_FILE
    # --- 3. LIMPIEZA (Solo si el backup salió bien) ---
    $FIND $DIR_DESTINO -name "backup_ad_*.tar.gz" -mtime +$DIAS_A_GUARDAR -delete
else
    $ECHO "[$FECHA] ERROR: Falló la creación del backup. Revisa espacio en disco." >> $LOG_FILE
fi
```

Guardar y cerrar.

---

### 5.2 Dar permisos de ejecución

```bash
sudo chmod +x /root/backup_samba.sh
```

---

### 5.3 Probar el script manualmente

```bash
sudo /root/backup_samba.sh
```

🔍 **Verificación:**

```bash
# Ver archivos de backup
ls -lh /root/backups/

# Ver log
cat /var/log/samba_backup.log
```

Debe aparecer:

```
[2026-02-XX_XX-XX] OK: Backup creado exitosamente: backup_ad_2026-02-XX_XX-XX.tar.gz
```

---

### 5.4 Programar con CRON

Editar crontab de root:

```bash
sudo crontab -e
```

Si es la primera vez, seleccionar editor (nano recomendado).

Añadir al final:

```
# Backup automático de Samba cada día a las 2:00 AM
0 2 * * * /root/backup_samba.sh
```

Guardar y cerrar.

🔍 **Verificación:**

```bash
sudo crontab -l
```

Debe mostrar la línea añadida.

---

### 5.5 Ejemplos de configuraciones CRON

```
# Cada 6 horas
0 */6 * * * /root/backup_samba.sh

# Cada domingo a las 3:00 AM
0 3 * * 0 /root/backup_samba.sh

# Cada día a las 23:30
30 23 * * * /root/backup_samba.sh

# Cada hora
0 * * * * /root/backup_samba.sh
```

**Formato de CRON:**

```
* * * * * comando
│ │ │ │ │
│ │ │ │ └─── Día de la semana (0-7, 0 y 7 = Domingo)
│ │ │ └───── Mes (1-12)
│ │ └─────── Día del mes (1-31)
│ └───────── Hora (0-23)
└─────────── Minuto (0-59)
```

---

### 5.6 Verificar ejecución del CRON

**Ver logs del sistema:**

```bash
sudo grep CRON /var/log/syslog | tail -20
```

**Ver log específico del backup:**

```bash
tail -f /var/log/samba_backup.log
```

---

### 5.7 Restaurar un backup (si es necesario)

```bash
# Ver backups disponibles
ls -lh /root/backups/

# Restaurar un backup específico
sudo tar -xzf /root/backups/backup_ad_2026-02-XX_XX-XX.tar.gz -C /
```

> ⚠️ **PRECAUCIÓN:** Esto sobrescribirá los archivos actuales. Solo usar en emergencias.

---

## 🕒 HORA 6: Seguridad y Auditoría (Samba Audit)

**Objetivo:** Configurar auditoría de accesos a recursos compartidos de Samba.

### 6.1 Configurar módulo de auditoría en smb.conf

Editar smb.conf:

```bash
sudo nano /etc/samba/smb.conf
```

Buscar la sección `[FinanceDocs]` y AÑADIR:

```ini
[FinanceDocs]
    path = /srv/samba/FinanceDocs
    read only = no
    vfs objects = acl_xattr full_audit
    map acl inherit = yes
    # AUDITORÍA AVANZADA
    full_audit:prefix = %u|%I|%m|%S
    full_audit:success = mkdirat renameat unlinkat pwrite
    full_audit:failure = connect
    full_audit:facility = local7
    full_audit:priority = NOTICE
```

Repetir para `[HRDocs]` y `[Public]`.

---

### 6.2 Configurar rsyslog para capturar logs

Crear archivo de configuración:

```bash
sudo nano /etc/rsyslog.d/samba-audit.conf
```

Contenido:

```
# Desviar mensajes del canal local7 (Samba Audit) a un log dedicado
local7.notice /var/log/samba_audit.log

# Evitar que se dupliquen en syslog
& stop
```

---

### 6.3 Crear archivo de log y configurar permisos

```bash
# Crear el archivo vacío
sudo touch /var/log/samba_audit.log

# Cambiar propietario y permisos
sudo chown syslog:adm /var/log/samba_audit.log
sudo chmod 640 /var/log/samba_audit.log
```

🔍 **Verificación:**

```bash
ls -la /var/log/samba_audit.log
```

Debe mostrar:

```
-rw-r----- 1 syslog adm 0 ... /var/log/samba_audit.log
```

---

### 6.4 Reiniciar servicios

```bash
# Reiniciar rsyslog
sudo systemctl restart rsyslog

# Recargar configuración de Samba
sudo smbcontrol all reload-config
```

🔍 **Verificación:**

```bash
sudo systemctl status rsyslog
sudo systemctl status samba-ad-dc
```

Ambos deben estar `active (running)`.

---

### 6.5 Generar eventos de auditoría

Desde el cliente Windows (como user01):

1. Navegar a `\\ls02.lab02.lan\FinanceDocs`
2. Crear un archivo: `audit_test.txt`
3. Modificar el archivo
4. Eliminar el archivo

Desde el cliente Ubuntu:

```bash
smbclient //ls02.lab02.lan/FinanceDocs -U user01
```

Dentro del prompt:

```
mkdir audit_folder
rmdir audit_folder
exit
```

---

### 6.6 Verificar logs de auditoría

Desde el servidor:

```bash
tail -f /var/log/samba_audit.log
```

🔍 **Debe mostrar algo como:**

```
user01|192.168.11.100|LC02|FinanceDocs|mkdirat|ok|audit_folder
user01|192.168.11.100|LC02|FinanceDocs|unlinkat|ok|audit_folder
user01|192.168.11.100|LC02|FinanceDocs|pwrite|ok|audit_test.txt
```

**Formato:**

```
Usuario | IP Cliente | Nombre Cliente | Recurso | Acción | Resultado | Archivo/Carpeta
```

---

### 6.7 Filtrar logs por usuario o acción

**Ver solo acciones de user01:**

```bash
grep "user01" /var/log/samba_audit.log
```

**Ver solo creaciones de carpetas:**

```bash
grep "mkdirat" /var/log/samba_audit.log
```

**Ver solo fallos:**

```bash
grep "fail" /var/log/samba_audit.log
```

**Ver actividad de las últimas 24 horas:**

```bash
grep "$(date +%Y-%m-%d)" /var/log/samba_audit.log
```

---

### 6.8 Rotación de logs (opcional pero recomendado)

Crear configuración de logrotate:

```bash
sudo nano /etc/logrotate.d/samba-audit
```

Contenido:

```
/var/log/samba_audit.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    missingok
    create 0640 syslog adm
}
```

**Explicación:**

- `daily`: Rotar cada día
- `rotate 30`: Mantener 30 archivos
- `compress`: Comprimir archivos antiguos
- `create 0640 syslog adm`: Permisos del nuevo archivo

---

## 🎯 CHECKPOINT FINAL DEL SPRINT 4

✅ **Verificaciones obligatorias:**

1. **GPOs con RSAT:**
```cmd
REM Desde cliente Windows (como user01)
gpresult /r
```
→ Debe listar GPOs aplicadas: `Prohibir_Panel_Control`, `Fondo_Escritorio`

2. **Cliente Ubuntu unido al dominio:**
```bash
# Desde cliente Ubuntu
sudo realm list
```
→ `configured: kerberos-member`

3. **Montaje de recursos:**
```bash
# Desde cliente Ubuntu
df -h | grep financedocs
ls /mnt/financedocs
```
→ Debe mostrar el recurso montado

4. **Gestión de procesos:**
```bash
# Desde servidor vía SSH al cliente
ps aux | grep bob
```
→ Debe listar procesos de bob

5. **CRON funcionando:**
```bash
# Desde servidor
sudo crontab -l
ls -lh /root/backups/
cat /var/log/samba_backup.log
```
→ Backup programado, archivos creados, logs presentes

6. **Auditoría activa:**
```bash
# Desde servidor
tail -20 /var/log/samba_audit.log
```
→ Debe mostrar eventos de acceso a recursos

---

## 🛠 PLAN DE RESCATE GENERAL

**Si RSAT no se instala:**

- Verificar versión de Windows (debe ser Pro/Enterprise/Education)
- Usar PowerShell:

```powershell
Get-WindowsCapability -Name RSAT* -Online | Add-WindowsCapability -Online
```

**Si cliente Ubuntu no se une al dominio:**

- Verificar DNS: `nslookup lab02.lan`
- Verificar conectividad: `ping ls02.lab02.lan`
- Ver logs:

```bash
sudo journalctl -xe
```

**Si recursos no montan:**

- Verificar credenciales en `/root/.smbcredentials`
- Probar montaje manual primero
- Ver logs de Samba:

```bash
sudo tail -50 /var/log/samba/log.smbd
```

**Si CRON no ejecuta:**

- Verificar sintaxis: `sudo crontab -l`
- Ver logs: `sudo grep CRON /var/log/syslog`
- Probar script manualmente: `sudo /root/backup_samba.sh`

**Si auditoría no registra eventos:**

- Verificar configuración en smb.conf:

```bash
sudo testparm -s | grep full_audit
```

- Verificar rsyslog: `sudo systemctl status rsyslog`
- Reiniciar Samba: `sudo systemctl restart samba-ad-dc`

---

## 🎯 FIN DEL SPRINT 4

Has completado:

- ✅ RSAT instalado y GPOs configuradas desde Windows
- ✅ Cliente Ubuntu unido al dominio lab02.lan
- ✅ Recursos compartidos montados automáticamente
- ✅ Gestión remota de procesos vía SSH
- ✅ Backup automático con CRON
- ✅ Auditoría completa de accesos a Samba

**Estado actual del entorno:**

- Servidor Ubuntu: DC completo con auditoría
- Cliente Windows: Unido, GPOs aplicadas, ACLs configuradas
- Cliente Ubuntu: Unido, recursos montados, gestión remota
- Automatización: Backups diarios, limpieza automática
- Seguridad: Auditoría de todos los accesos

**Siguiente:** SPRINT 5 → Forest Trust entre dominios
