# 🚀 Guía de Instalación de Red Hat Ansible Automation Platform (AAP) - Containerizada

## 📖 Introducción

Este documento describe el proceso paso a paso para la instalación de **Red Hat Ansible Automation Platform (AAP) 2.6** en su modalidad **containerizada** sobre una máquina RHEL 9.7. La instalación utiliza el empaquetado (bundle) proporcionado por Red Hat.

Ansible Automation Platform es una solución empresarial que proporciona herramientas para automatizar TI a gran escala, incluyendo:

| Componente | Descripción |
| :--- | :--- |
| **Automation Controller** | Motor de automatización (antes AWX/Tower) para ejecutar playbooks. |
| **Automation Hub** | Repositorio privado de colecciones y roles certificados. |
| **EDA (Event-Driven Ansible)** | Motor para automatización basada en eventos. |
| **Automation Gateway** | Punto de entrada unificado para la plataforma. |

---

## 📋 Prerrequisitos

### 🖥️ Requisitos del Sistema

| Recurso | Requisito Mínimo | Recomendado |
| :--- | :--- | :--- |
| **Sistema Operativo** | RHEL 8.x / 9.x | RHEL 9.x |
| **CPU** | 4 vCPUs | 8+ vCPUs |
| **RAM** | 16 GB | 32+ GB |
| **Almacenamiento** | 50 GB | 200+ GB |
| **Acceso a Internet** | Necesario para Red Hat Subscription Manager | ✅ |

### 🔑 Credenciales Necesarias

| Credencial | Descripción |
| :--- | :--- |
| **Red Hat Subscription** | Usuario y contraseña para registrar el sistema. |
| **Usuarios locales** | `aapuser`, `sacaci`, `racmacta` (según el entorno). |

---

## ⚙️ Configuración Inicial del Sistema

### 1. 🏷️ Configurar Nombre de Host y Archivo Hosts

```bash
# Establecer el nombre de host del sistema
sudo hostnamectl set-hostname linux.local

# Editar el archivo /etc/hosts para incluir la resolución local
sudo vi /etc/hosts
```
> **Ejemplo de entrada en `/etc/hosts`:**
> ```
> 127.0.0.1   localhost localhost.localdomain
> 192.168.100.47 linux.local linux
> ```
***Nota***: Por algun motivo, despues de muchas pruebas de instalación, opte por colocar el host con el nombre **linux.local**. Fue lo que me permitio seguir con la instalación.
<br>

---

### 2. 🔐 Registrar el Sistema en Red Hat Subscription Manager

```bash
subscription-manager register --username <USUARIO> --password <CONTRASEÑA>
```

> **⚠️ Advertencia:** Reemplace `<USUARIO>` y `<CONTRASEÑA>` con las credenciales de Red Hat. **Nunca** almacene contraseñas en texto plano en scripts. Utilice variables de entorno o un gestor de secretos.

---

### 3. 👥 Configuración de Usuarios y Sudoers

#### 3.1 Configurar Sudoers para el Grupo Wheel

Edite el archivo `/etc/sudoers`:

```bash
sudo vi /etc/sudoers
```

Modifique la línea correspondiente al grupo `wheel` para permitir comandos sin contraseña:

```bash
## Allows people in group wheel to run all commands
#%wheel ALL=(ALL)       ALL

## Same thing without a password
%wheel  ALL=(ALL)       NOPASSWD: ALL
```

#### 3.2 Crear Usuarios y Asignar Grupos

```bash
# Crear usuario aapuser
sudo useradd aapuser

# Establecer shell bash
sudo usermod -m -s /bin/bash aapuser

# Establecer contraseña (redhat en el ejemplo)
sudo passwd aapuser

# Agregar al grupo wheel (sudo sin contraseña)
sudo usermod -aG wheel aapuser
sudo usermod -aG users aapuser

# NOTA: El siguiente comando deshabilita el login del usuario (usado en algunos casos)
# sudo usermod -s /sbin/nologin aapuser
```

#### 3.3 Crear Archivos de Sudoers Específicos por Usuario

```bash
# Para el usuario aapuser
sudo echo "aapuser ALL=(ALL)       NOPASSWD: ALL" > /etc/sudoers.d/aapuser

# Asegurar permisos correctos
sudo chmod 400 /etc/sudoers.d/*
```
---

### 4. 📦 Instalar Paquetes y Herramientas Base

```bash
sudo dnf install -y ansible-core wget git rsync
```

---

## 🏗️ Instalación de Ansible Automation Platform

### 5. 📥 Descargar y Descomprimir el Instalador

```bash
# Descomprimir el bundle descargado
tar zxvfp ansible-automation-platform-containerized-setup-bundle-2.6-5-x86_64.tar.gz

# Renombrar el directorio
mv ansible-automation-platform-containerized-setup-bundle-2.6-5-x86_64/ aap/
```
---

### 6. 🔧 Configurar Permisos del Directorio

```bash
# Asignar propietario aap:users
chown -R aapuser:users aap/
```
---

### 7. 🔑 Configurar Clave SSH para Acceso Sin Contraseña

```bash
# Generar par de claves SSH
ssh-keygen

# Copiar clave pública al servidor local
ssh-copy-id aapuser@linux.local
```
---

### 8. ⚙️ Configurar Variables de Entorno para Colecciones

```bash
cd aap/
export ANSIBLE_COLLECTIONS_PATH=/home/sacaci/aap/collections
echo $ANSIBLE_COLLECTIONS_PATH
```
---

### 9. 📝 Crear el Archivo de Inventario

Cree el archivo `inventory` en el directorio `aap/` con el siguiente contenido:

```ini
# ============================================================
# Inventario para Red Hat Ansible Automation Platform 2.6
# Instalación Containerizada
# ============================================================

# === AAP Gateway ===
[automationgateway]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === AAP Controller ===
[automationcontroller]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === AAP Execution Nodes ===
[execution_nodes]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === AAP Automation Hub ===
[automationhub]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === AAP EDA Controller ===
[automationeda]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === Base de Datos ===
[database]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === Redis ===
[redis]
linux.local ansible_host=192.168.100.47 ansible_user=aapuser ansible_become=true

# === Variables Globales ===
[all:vars]
# Credenciales PostgreSQL
postgresql_admin_username=postgres
postgresql_admin_password=redhat

# Instalación desde bundle
bundle_install=true
bundle_dir='{{ lookup("ansible.builtin.env", "PWD") }}/bundle'

# Configuración Redis
redis_mode=standalone
ansible_connection=local

# === Gateway ===
gateway_admin_password=redhat
gateway_pg_host=linux.local
gateway_pg_password=redhat

# === Controller ===
controller_admin_password=redhat
controller_pg_host=linux.local
controller_pg_password=redhat

# === Automation Hub ===
hub_admin_password=redhat
hub_pg_host=linux.local
hub_pg_password=redhat

# === EDA Controller ===
eda_admin_password=redhat
eda_pg_host=linux.local
eda_pg_password=redhat
controller_main_url=https://linux.local
```

> **⚠️ Advertencia de Seguridad:** Las contraseñas en este ejemplo (`redhat`) son ilustrativas. **Nunca** utilice contraseñas débiles en entornos de producción. Utilice un gestor de secretos como **Ansible Vault**, **Hashicorp Vault** o **Azure Key Vault**.

---

### 10. 🚀 Ejecutar el Playbook de Instalación

```bash
ansible-playbook -i inventory ansible.containerized_installer.install
```

---

## 📋 Explicación del Archivo de Inventario

### 🏷️ Secciones de Hosts

| Sección | Componente | Función |
| :--- | :--- | :--- |
| `[automationgateway]` | Automation Gateway | Punto de entrada unificado para la plataforma. |
| `[automationcontroller]` | Automation Controller | Ejecuta playbooks y gestiona automatizaciones. |
| `[execution_nodes]` | Execution Nodes | Nodos que ejecutan tareas de automatización. |
| `[automationhub]` | Automation Hub | Repositorio privado de colecciones. |
| `[automationeda]` | EDA Controller | Motor de automatización basada en eventos. |
| `[database]` | Base de Datos | PostgreSQL para almacenamiento de la plataforma. |
| `[redis]` | Redis | Cache para la plataforma. |

### 🔧 Variables Clave

| Variable | Descripción |
| :--- | :--- |
| `postgresql_admin_password` | Contraseña del administrador de PostgreSQL. |
| `controller_admin_password` | Contraseña del administrador de Automation Controller. |
| `gateway_admin_password` | Contraseña del administrador de Automation Gateway. |
| `hub_admin_password` | Contraseña del administrador de Automation Hub. |
| `eda_admin_password` | Contraseña del administrador de EDA Controller. |
| `bundle_install` | Indica que se usa el instalador empaquetado (bundle). |
| `bundle_dir` | Ruta al directorio del bundle. |

---

### 🔄 Versión Alternativa del Inventario (con ansible_connection=local)

La segunda versión del inventario utiliza `ansible_connection=local` en lugar de `ansible_user` y `ansible_become`:

```ini
[automationcontroller]
linux.local ansible_host=192.168.100.47 ansible_connection=local

[all:vars]
ansible_connection=local
```

Esta configuración es útil cuando se ejecuta el playbook directamente en la máquina local (sin necesidad de SSH).

---

## ✅ Verificación de la Instalación

Una vez completada la instalación, puede verificar que la plataforma está funcionando:

1.  **Acceda al Automation Controller:**
    - URL: `https://linux.local`
    - Usuario: `admin`
    - Contraseña: La configurada en `controller_admin_password` (ej. `redhat`).

2.  **Verifique el estado de los servicios:**

    ```bash
    # Listar pods de AAP (si se usa OpenShift/K8s)
    oc get pods -n aap

    # O verificar servicios en el sistema
    systemctl status aap-*
    ```

3.  **Pruebe la conectividad:**

    ```bash
    ansible linux.local -m ping
    ```

---

## 📚 Glosario de Términos

| Término | Definición |
| :--- | :--- |
| **Red Hat Ansible Automation Platform (AAP)** | Plataforma empresarial de Red Hat para automatización a gran escala. |
| **Automation Controller** | Componente que ejecuta playbooks y gestiona inventarios y proyectos. |
| **Automation Hub** | Repositorio de colecciones y roles de Ansible certificados por Red Hat. |
| **EDA (Event-Driven Ansible)** | Motor que permite ejecutar automatizaciones basadas en eventos. |
| **Ansible Core** | Motor de automatización de código abierto que ejecuta playbooks. |
| **Bundle Install** | Método de instalación que no requiere descargar dependencias de internet. |
| **Ansible Vault** | Herramienta para cifrar datos sensibles en Ansible. |
| **OpenShift (OCP)** | Plataforma de contenedores de Red Hat basada en Kubernetes. |
| **Podman / Docker** | Herramientas para gestionar contenedores. |
| **Redis** | Base de datos en memoria utilizada como cache y broker de mensajes. |
| **PostgreSQL** | Base de datos relacional utilizada por AAP para almacenar configuración y logs. |

---
Mario Fribla
***Ingeniero Cloud***
