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
