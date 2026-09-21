# apache_payara_contingencia_BCE

Automatización Ansible para la gestión de infraestructura del **Banco Central del Ecuador (BCE)** en escenarios de contingencia y **Disaster Recovery (DR)**. Permite operar servidores Apache y Payara, y conmutar la resolución de base de datos entre los datacenters de **Quito** y **Guayaquil** de forma controlada y auditable.

---

## Tabla de contenidos

1. [Descripción general](#1-descripción-general)
2. [Arquitectura y componentes](#2-arquitectura-y-componentes)
3. [Requisitos previos](#3-requisitos-previos)
4. [Estructura del repositorio](#4-estructura-del-repositorio)
5. [Inventario](#5-inventario)
6. [Playbooks disponibles](#6-playbooks-disponibles)
   - [Apache](#apache)
   - [Payara](#payara)
   - [DNS / Base de Datos (Disaster Recovery)](#dns--base-de-datos-disaster-recovery)
7. [Roles](#7-roles)
   - [apache_start](#apache_start)
   - [apache_stop](#apache_stop)
   - [payara_actions](#payara_actions)
   - [edit_bdd_dns](#edit_bdd_dns)
8. [Variables de configuración](#8-variables-de-configuración)
9. [Flujo de Disaster Recovery](#9-flujo-de-disaster-recovery)
10. [Ejemplos de ejecución](#10-ejemplos-de-ejecución)
11. [Correo automático de notificación](#11-correo-automático-de-notificación)
12. [Consideraciones de seguridad](#12-consideraciones-de-seguridad)

---

## 1. Descripción general

Este repositorio automatiza las operaciones críticas necesarias para ejecutar el plan de contingencia del BCE, que implica mover la infraestructura de producción desde el datacenter de **Quito** al de **Guayaquil**. Las tres acciones principales son:

| Área | Operación |
|------|-----------|
| **Apache** | Iniciar o detener el servidor web (soporte para Apache del sistema y XAMPP) |
| **Payara** | Iniciar, detener o consultar estado del servidor de aplicaciones (v5 y v6, con y sin Deployment Groups) |
| **DNS / BDD** | Conmutar la resolución del host de base de datos `bceqasep1` entre Quito (`172.20.40.36`) y Guayaquil (`172.17.10.113`) mediante edición de `/etc/hosts` |

---

## 2. Arquitectura y componentes

```
Quito (producción)                     Guayaquil (contingencia)
┌─────────────────────┐                ┌─────────────────────────┐
│  Apache Web Server  │                │  Apache Web Server      │
│  Payara v5 / v6     │    DR →        │  Payara v5 / v6         │
│  BDD: 172.20.40.36  │ ────────────▶  │  BDD: 172.17.10.113     │
│  (bceqasep1)        │                │  (bceqasep1)            │
└─────────────────────┘                └─────────────────────────┘
```

**Componentes gestionados:**

- **Apache HTTP Server** (apachectl) o **XAMPP** (`/opt/lampp/lampp`)
- **Payara v5** (`/sfw/payara5/glassfish`) y **Payara v6** (`/sfw/payara6/glassfish`)
  - Soporte para topología **DAS + Deployment Groups**
  - Soporte para topología **standalone** (sin Deployment Groups)
- **Base de datos Oracle** (acceso vía hostname `bceqasep1` en `/etc/hosts`)

---

## 3. Requisitos previos

### Control node (máquina que ejecuta Ansible)

- Ansible >= 2.9
- Colección `community.general` (para módulo `mail`)
  ```bash
  ansible-galaxy collection install community.general
  ```
- Python >= 3.8
- Conectividad SSH hacia los servidores del inventario

### Servidores objetivo

- Sistema operativo: RHEL / CentOS (cualquier versión compatible con `apachectl`)
- Acceso SSH con clave privada (usuario `payara` para Payara, usuario con `sudo` para Apache)
- Payara instalado en `/sfw/payara5/` o `/sfw/payara6/`
- Puerto DAS `4848` accesible localmente desde cada servidor Payara

### Credenciales necesarias en tiempo de ejecución

| Credencial | Cómo se provee | Propósito |
|-----------|---------------|-----------|
| `payara_admin_pass` | Prompt interactivo (oculta) | Contraseña del admin de `asadmin` |
| `passphrase` | Variable `-e passphrase=...` o pipeline CI | Passphrase SSL de Apache |
| `ciudad` | Variable `-e ciudad=quito\|guayaquil` | Target del cambio de DNS |
| Clave SSH | Archivo de clave privada referenciado en inventario | Autenticación SSH |

---

## 4. Estructura del repositorio

```
apache_payara_contingencia_BCE/
├── inventories/
│   └── inventory.yml               # Definición de hosts y grupos
│
├── roles/
│   ├── apache_start/               # Rol: iniciar Apache con SSL
│   │   ├── tasks/main.yml
│   │   └── vars/main.yml
│   │
│   ├── apache_stop/                # Rol: detener Apache
│   │   └── tasks/main.yml
│   │
│   ├── edit_bdd_dns/               # Rol: conmutar DNS de base de datos
│   │   ├── tasks/main.yml
│   │   └── vars/main.yml
│   │
│   └── payara_actions/             # Rol: gestión completa de Payara
│       ├── defaults/main.yml       # Valores por defecto
│       ├── vars/main.yml           # Variables de comandos
│       ├── group_vars/
│       │   ├── all.yml             # Variables comunes a todos los hosts
│       │   ├── payara_v5.yml       # AS_HOME para Payara 5
│       │   └── payara_v6.yml       # AS_HOME para Payara 6
│       ├── files/
│       │   └── id_ed25519_gitlab_ci  # Clave SSH para GitLab CI
│       └── tasks/
│           ├── main.yml            # Orquestador principal
│           ├── validate.yml        # Validación de inputs y entorno
│           ├── discover.yml        # Descubrimiento de Deployment Groups
│           ├── start_das_pre.yml   # Levantar DAS antes de instancias
│           ├── start_dg.yml        # Iniciar con Deployment Groups
│           ├── start_standalone.yml # Iniciar modo standalone
│           ├── status.yml          # Consultar estado
│           ├── stop_dg.yml         # Detener Deployment Groups
│           ├── stop_standalone.yml  # Detener modo standalone
│           └── report.yml          # Reporte y envío de correo
│
├── payara_manage.yml               # Playbook interactivo (status/stop/start)
├── payara_actions_todo_en_uno.yml  # Playbook integrado completo
├── payara_start.yml                # Shortcut: iniciar Payara
├── playara_stop.yml                # Shortcut: detener Payara
├── payara_status.yml               # Shortcut: estado de Payara
├── iniciar_servicio_apache.yml     # Playbook: iniciar Apache
├── detener_servicio_apache.yml     # Playbook: detener Apache
└── modificar_dns_contingencia.yml  # Playbook: cambio de DNS (DR)
```

---

## 5. Inventario

Archivo: `inventories/inventory.yml`

```yaml
[payara_v6]
bceqlbtrsitdes01 ansible_host=<IP> ansible_user=payara ansible_ssh_private_key_file=llaves/id_ed25519_gitlab_ci

[payara_v5]
bceqpayaraqa04 ansible_host=<IP> ansible_user=payara ansible_ssh_private_key_file=llaves/id_ed25519_gitlab_ci

[das:children]
payara_v6
payara_v5

[Servidores_Apache]
# Agregar hosts de Apache aquí

[Srv-Contingencia]
# Agregar hosts de contingencia (DNS) aquí
```

**Grupos disponibles:**

| Grupo | Descripción |
|-------|-------------|
| `payara_v5` | Servidores con Payara 5 (`/sfw/payara5/glassfish`) |
| `payara_v6` | Servidores con Payara 6 (`/sfw/payara6/glassfish`) |
| `das` | Agrupa todos los DAS (Payara v5 + v6) |
| `Servidores_Apache` | Servidores web Apache |
| `Srv-Contingencia` | Servidores donde se modifica el DNS de contingencia |

**Importante sobre la clave SSH:**

La ruta `ansible_ssh_private_key_file` es relativa a la raíz del proyecto. Antes de ejecutar:

```bash
# Crear directorio y copiar la clave
mkdir -p llaves/
cp /ruta/a/id_ed25519_gitlab_ci llaves/
chmod 600 llaves/id_ed25519_gitlab_ci

# O usar ruta absoluta en el inventario
ansible_ssh_private_key_file=/home/usuario/.ssh/id_ed25519_gitlab_ci
```

En pipelines GitLab CI, inyectar la clave como variable enmascarada y escribirla al disco antes de ejecutar el playbook.

---

## 6. Playbooks disponibles

### Apache

#### `iniciar_servicio_apache.yml`

Inicia el servidor web Apache en los nodos del grupo `Servidores_Apache`. Detecta automáticamente si el servidor es Apache del sistema (`apachectl`) o XAMPP.

**Flujo de ejecución:**

1. Detecta el tipo de servidor web instalado
2. Crea directorio seguro `/etc/httpd/passphrase/` (permisos `0700`)
3. Escribe passphrase SSL en archivo temporal (permisos `0600`)
4. Crea script auxiliar `httpd-read-pass.sh` para suministrar el passphrase
5. Aplica contexto SELinux al script
6. Configura `SSLPassPhraseDialog` temporalmente en `ssl.conf`
7. Inicia Apache (`apachectl start`) o XAMPP (`/opt/lampp/lampp start`)
8. Restaura la configuración original de `ssl.conf`
9. Elimina todos los archivos temporales

```bash
ansible-playbook -i inventories/inventory.yml iniciar_servicio_apache.yml \
  -e "passphrase=MiPassphraseSSL"
```

#### `detener_servicio_apache.yml`

Detiene el servidor web Apache en los nodos del grupo `Servidores_Apache`.

**Flujo de ejecución:**

1. Muestra el estado actual del servicio antes de detener
2. Detiene Apache (`apachectl stop`) o XAMPP (`/opt/lampp/lampp stop`)
   - Para XAMPP: mata procesos MySQL huérfanos antes de detener
3. Verifica que el servicio quedó efectivamente detenido

```bash
ansible-playbook -i inventories/inventory.yml detener_servicio_apache.yml
```

---

### Payara

Todos los playbooks de Payara operan sobre los nodos del grupo `das`. Requieren la contraseña del administrador de `asadmin`.

#### `payara_manage.yml` — Gestión interactiva

Solicita acción y contraseña de forma interactiva. Útil para ejecución manual.

```bash
ansible-playbook -i inventories/inventory.yml payara_manage.yml
# Solicita:
#   payara_action: [status/stop/start] (default: status)
#   payara_admin_pass: (oculta)
```

#### `payara_status.yml` — Consultar estado

```bash
ansible-playbook -i inventories/inventory.yml payara_status.yml
# Solicita: payara_admin_pass
```

#### `payara_start.yml` — Iniciar Payara

```bash
ansible-playbook -i inventories/inventory.yml payara_start.yml
# Solicita: payara_admin_pass
```

#### `payara_stop.yml` — Detener Payara

```bash
ansible-playbook -i inventories/inventory.yml payara_stop.yml
# Solicita: payara_admin_pass
```

#### `payara_actions_todo_en_uno.yml` — Playbook integrado completo

Versión todo-en-uno con lógica extendida: pre-tasks integradas, descubrimiento de topología, reintentos, y reporte por correo. Recomendado para pipelines CI/CD.

```bash
ansible-playbook -i inventories/inventory.yml payara_actions_todo_en_uno.yml \
  -e "payara_action=start" \
  --extra-vars '{"payara_admin_pass": "mipassword"}'
```

**Opciones adicionales para Payara:**

```bash
# Operar solo sobre Payara v5
ansible-playbook -i inventories/inventory.yml payara_start.yml --limit payara_v5

# Gestionar también el DAS (start-domain / stop-domain)
ansible-playbook -i inventories/inventory.yml payara_start.yml \
  -e "manage_das=true"

# No descubrir DG automáticamente (modo manual)
ansible-playbook -i inventories/inventory.yml payara_start.yml \
  -e "dg_autodiscover=false"
```

---

### DNS / Base de Datos (Disaster Recovery)

#### `modificar_dns_contingencia.yml`

Modifica `/etc/hosts` en los nodos del grupo `Srv-Contingencia` para apuntar el hostname `bceqasep1` al datacenter indicado.

| Ciudad | IP activada | IP comentada |
|--------|-------------|-------------|
| `quito` | `172.20.40.36` (Quito) | `172.17.10.113` (Guayaquil) |
| `guayaquil` | `172.17.10.113` (Guayaquil) | `172.20.40.36` (Quito) |

```bash
# Conmutar a Guayaquil (activar contingencia)
ansible-playbook -i inventories/inventory.yml modificar_dns_contingencia.yml \
  -e "ciudad=guayaquil"

# Retornar a Quito (restaurar producción)
ansible-playbook -i inventories/inventory.yml modificar_dns_contingencia.yml \
  -e "ciudad=quito"
```

---

## 7. Roles

### `apache_start`

Gestiona el inicio de Apache con soporte de passphrase SSL. Compatible con:
- **Apache del sistema** (`apachectl` disponible en PATH)
- **XAMPP** (`/opt/lampp/lampp` existente)

Maneja de forma segura el passphrase SSL mediante archivos temporales que se eliminan al finalizar, con y sin errores.

### `apache_stop`

Detiene Apache (sistema o XAMPP) de forma limpia. Para XAMPP, mata procesos MySQL huérfanos previo a la parada. Verifica el estado final del servicio.

### `payara_actions`

Rol principal de gestión de Payara. Soporta las tres acciones (`status`, `stop`, `start`) con lógica adaptativa según la topología detectada:

**Topologías soportadas:**

| Topología | Descripción |
|-----------|-------------|
| `DEPLOYMENT_GROUPS` | DAS con uno o más Deployment Groups |
| `STANDALONE` | DAS sin Deployment Groups |
| `SIN_DG` | Instancias standalone sin DAS gestionado |

**Características del rol:**

- **Descubrimiento automático** de Deployment Groups vía `list-deployment-groups`
- **Reintentos inteligentes**: si una instancia no levanta/baja, reintenta individualmente
- **Kill forzado**: para instancias rebeldes usa `stop-instance --kill=true`
- **Password seguro**: escribe contraseña en archivo temporal (`/tmp`, permisos `0600`) y lo elimina siempre (bloque `always`)
- **Validación de entorno**: compara `$AS_HOME` real del servidor con el definido en `group_vars`
- **Reporte por correo**: envía resultado detallado al equipo `uioinf-ibd-web@bce.ec`

**Flujo de tasks:**

```
validate.yml        → Valida inputs, verifica asadmin, crea password file
start_das_pre.yml   → (solo si action=start y manage_das=true) Levanta DAS
discover.yml        → Descubre Deployment Groups y topología
  ├── status.yml          → (action=status)
  ├── stop_dg.yml         → (action=stop, tiene DG)
  ├── stop_standalone.yml → (action=stop, sin DG)
  ├── start_dg.yml        → (action=start, tiene DG)
  └── start_standalone.yml → (action=start, sin DG)
[always] Borra password file
report.yml          → Calcula resultado global, envía correo, falla si hay errores
```

### `edit_bdd_dns`

Edita `/etc/hosts` para conmutar la resolución del hostname de base de datos `bceqasep1` entre los datacenter de Quito y Guayaquil. Usa expresiones regulares para comentar/descomentar líneas de forma idempotente.

---

## 8. Variables de configuración

### Variables globales (`roles/payara_actions/group_vars/all.yml`)

| Variable | Valor por defecto | Descripción |
|----------|-----------------|-------------|
| `payara_action` | `status` | Acción a ejecutar: `status`, `stop`, `start` |
| `manage_das` | `false` | Si `true`, administra el DAS (start-domain/stop-domain) |
| `dg_autodiscover` | `true` | Descubrimiento automático de Deployment Groups |
| `das_host` | `localhost` | Host para comandos `asadmin` |
| `das_port` | `4848` | Puerto del DAS |
| `das_secure` | `true` | Usar `--secure=true` en asadmin |
| `admin_user` | `admin` | Usuario administrador de asadmin |
| `payara_domain` | `production` | Nombre del dominio Payara |
| `check_retries` | `10` | Reintentos de verificación de estado |
| `check_delay` | `15` | Segundos entre reintentos |
| `das_check_retries` | `10` | Reintentos para verificar DAS |
| `das_check_delay` | `10` | Segundos entre reintentos del DAS |
| `mail_to` | `uioinf-ibd-web@bce.ec` | Destinatario del correo de reporte |
| `mail_from` | `dcordova@bce.ec` | Remitente del correo |
| `mail_host` | `correo.bce.fin.ec` | Servidor SMTP |
| `mail_port` | `25` | Puerto SMTP |

### Variables por versión de Payara

| Variable | Payara v5 | Payara v6 |
|----------|-----------|-----------|
| `as_home` | `/sfw/payara5/glassfish` | `/sfw/payara6/glassfish` |

### Variables de runtime (se pasan en ejecución)

| Variable | Playbook | Descripción |
|----------|----------|-------------|
| `payara_admin_pass` | Todos los de Payara | Contraseña del admin asadmin |
| `passphrase` | Apache | Passphrase SSL del servidor web |
| `ciudad` | `modificar_dns_contingencia.yml` | `quito` o `guayaquil` |

---

## 9. Flujo de Disaster Recovery

Este es el procedimiento completo para mover la infraestructura de Quito a Guayaquil:

### Fase 1 — Verificar estado actual (Quito)

```bash
# Verificar que Payara está corriendo en Quito
ansible-playbook -i inventories/inventory.yml payara_status.yml
```

### Fase 2 — Detener servicios en Quito

```bash
# Detener Apache en Quito
ansible-playbook -i inventories/inventory.yml detener_servicio_apache.yml \
  --limit Servidores_Apache_Quito

# Detener Payara en Quito
ansible-playbook -i inventories/inventory.yml payara_stop.yml \
  --limit payara_quito \
  -e "manage_das=true"
```

### Fase 3 — Conmutar base de datos a Guayaquil

```bash
# Modificar /etc/hosts para apuntar a BDD de Guayaquil
ansible-playbook -i inventories/inventory.yml modificar_dns_contingencia.yml \
  -e "ciudad=guayaquil"
```

### Fase 4 — Iniciar servicios en Guayaquil

```bash
# Iniciar Payara en Guayaquil
ansible-playbook -i inventories/inventory.yml payara_start.yml \
  --limit payara_guayaquil \
  -e "manage_das=true"

# Iniciar Apache en Guayaquil
ansible-playbook -i inventories/inventory.yml iniciar_servicio_apache.yml \
  --limit Servidores_Apache_Guayaquil \
  -e "passphrase=MiPassphraseSSL"
```

### Fase 5 — Verificar estado en Guayaquil

```bash
ansible-playbook -i inventories/inventory.yml payara_status.yml \
  --limit payara_guayaquil
```

### Retorno a producción (Quito)

Para revertir, ejecutar en orden inverso cambiando `--limit` y `ciudad=quito`.

---

## 10. Ejemplos de ejecución

```bash
# ─── PAYARA ────────────────────────────────────────────────────────────────

# Estado de todos los servidores Payara
ansible-playbook -i inventories/inventory.yml payara_status.yml

# Estado solo de Payara v6
ansible-playbook -i inventories/inventory.yml payara_status.yml --limit payara_v6

# Iniciar Payara gestionando también el DAS
ansible-playbook -i inventories/inventory.yml payara_start.yml -e "manage_das=true"

# Detener Payara v5 únicamente
ansible-playbook -i inventories/inventory.yml payara_stop.yml --limit payara_v5

# Modo interactivo (pide acción y contraseña)
ansible-playbook -i inventories/inventory.yml payara_manage.yml

# ─── APACHE ────────────────────────────────────────────────────────────────

# Iniciar Apache con passphrase SSL
ansible-playbook -i inventories/inventory.yml iniciar_servicio_apache.yml \
  -e "passphrase=MiPassphraseSSL"

# Detener Apache
ansible-playbook -i inventories/inventory.yml detener_servicio_apache.yml

# ─── DNS / BDD ─────────────────────────────────────────────────────────────

# Activar contingencia Guayaquil
ansible-playbook -i inventories/inventory.yml modificar_dns_contingencia.yml \
  -e "ciudad=guayaquil"

# Restaurar producción Quito
ansible-playbook -i inventories/inventory.yml modificar_dns_contingencia.yml \
  -e "ciudad=quito"

# ─── OPCIONES ÚTILES ───────────────────────────────────────────────────────

# Dry-run (simular sin aplicar cambios)
ansible-playbook -i inventories/inventory.yml payara_start.yml --check

# Ver tareas que se ejecutarían
ansible-playbook -i inventories/inventory.yml payara_start.yml --list-tasks

# Modo verbose para debug
ansible-playbook -i inventories/inventory.yml payara_status.yml -vvv

# Usar inventario alternativo
ansible-playbook -i inventories/inventory_prod.yml payara_start.yml
```

---

## 11. Correo automático de notificación

Tras cada operación de Payara, el sistema envía automáticamente un correo a `uioinf-ibd-web@bce.ec` con el siguiente contenido:

**Asunto según resultado:**

| Acción | Resultado | Asunto |
|--------|-----------|--------|
| `status` | — | `[Payara] Estado actual reportado (consulta, sin cambios)` |
| `start`/`stop` | Exitoso | `[Payara] OK - Acción start/stop ejecutada correctamente` |
| `start`/`stop` | Con errores | `[Payara] ALERTA - Problemas ejecutando acción start/stop` |

**Contenido del correo (por host):**

- Topología detectada (Deployment Groups o Standalone)
- Lista de Deployment Groups procesados
- Estado del DAS (OK / FALLIDO)
- Estado general de la operación
- Instancias que quedaron en estado incorrecto (si aplica)
- Salida cruda del estado final (`list-instances`)

---

## 12. Consideraciones de seguridad

### Buenas prácticas aplicadas

- Contraseña de asadmin nunca se guarda en disco de forma permanente; se escribe en archivo temporal con permisos `0600` y se elimina en bloque `always` (con o sin error)
- Outputs con contraseñas usan `no_log: true`
- Passphrase de Apache se maneja en archivos temporales con permisos restrictivos
- `.gitignore` excluye claves SSH (`*id_ed25519*`, `*id_rsa*`, `llaves/`), archivos de vault y passwords

### Recomendaciones adicionales

- **Claves SSH**: No commitear claves privadas en el repositorio. Inyectarlas como variable de entorno enmascarada en el pipeline de GitLab CI y escribirlas al disco antes de ejecutar:
  ```bash
  mkdir -p llaves && echo "$SSH_PRIVATE_KEY" > llaves/id_ed25519_gitlab_ci && chmod 600 llaves/id_ed25519_gitlab_ci
  ```
- **Ansible Vault**: Para entornos productivos, considerar cifrar variables sensibles con `ansible-vault encrypt_string`
- **Rotación de contraseñas**: La contraseña de `asadmin` debería rotarse periódicamente y jamás guardarse en el repositorio

---

## Licencia

MIT License — Copyright 2026 daytwo-internal
