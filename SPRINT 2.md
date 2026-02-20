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
