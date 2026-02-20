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
