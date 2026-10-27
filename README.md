# Proyecto Ansible — Despliegue automatizado en openSUSE

Despliegue automático de **Apache, PHP, phpMyAdmin, MariaDB, Samba, PostgreSQL y FTP** en máquinas virtuales **openSUSE Leap 16**, usando **roles de Ansible**.

🔗 **Repositorio:** https://github.com/nachoMejia/proyecto_ansible.git

---

## Guía de despliegue del proyecto

En esta guía se explica paso a paso cómo instalar todo el proyecto clonándolo desde **GitHub**.

### Recursos necesarios

| Máquina | Descripción |
|---|---|
| **VM MASTER** | Es la máquina virtual en la que correrá todo el proyecto. Aquí se clona el repositorio y se ejecuta Ansible. |
| **VM DESTINO** | Una o más máquinas virtuales que recibirán toda la instalación que se ejecute desde la VM MASTER. Aquí se instalan todos los servicios automáticamente. |

```
┌──────────────────┐          SSH          ┌──────────────────┐
│    VM MASTER     │  ──────────────────▶  │    VM DESTINO    │
│ Ansible + repo   │                       │ Aquí se instala  │
└──────────────────┘                       └──────────────────┘
```

### Índice de la guía

0. [Configurar la red de las VM's](#0-configurar-la-red-de-las-vms)
1. [Preparar la VM DESTINO](#1-preparar-la-vm-destino)
2. [Preparar la VM MASTER](#2-preparar-la-vm-master)
3. [Clonar el proyecto](#3-clonar-el-proyecto)
4. [Crear y copiar la llave SSH](#4-llave-ssh-crearla-y-copiarla-al-servidor)
5. [Poner la IP de la VM DESTINO](#5-poner-la-ip-de-la-vm-destino-en-el-archivo-hosts)
6. [Configurar la contraseña del vault](#6-la-contraseña-del-vault)
7. [Probar conexión (opcional)](#7-probar-la-conexión-opcional)
8. [Ejecutar el despliegue](#8-desplegar)
9. [Comprobación](#9-comprobar-que-todo-funciona)

---

## 0. Configurar la red de las VM's

Las máquinas virtuales tienen que estar conectadas a la **misma red**. Si no lo están, sigue estos pasos **en todas las VM's**:

1. Ve a **VM → Settings → Network Adapter**.
2. Selecciona **Bridged**.
3. Enciende las VM's.
4. En la terminal, verifica la IP con:

   ```bash
   ip a
   ```

5. Ve a la línea `inet` de la tarjeta de red.

> [!IMPORTANT]
> Las IP de todas las VM's tienen que **empezar igual** (por ejemplo `192.168.0.10` y `192.168.0.12`). Si no son iguales, verifica la conexión, cambia la IP o reinicia la red a sus valores preestablecidos.

6. *(Opcional)* Para verificar la conexión, ejecuta:

   ```bash
   ping IP_OTRA_VM
   ```

   Presiona `Ctrl + C` para salir.

> [!TIP]
> Anota la IP de las VM's DESTINO, la usarás varias veces.

---

## 1. Preparar la VM DESTINO

1. Entra a la terminal.
2. Entra en modo superusuario y coloca tu contraseña:

   ```bash
   su -
   ```

### Ejecutar SSH

SSH es lo que permite que la **VM MASTER** pueda instalar los servicios en la **VM DESTINO**.

#### a) Revisar si SSH está instalado

```bash
rpm -q openssh-server
```

- Si responde algo como `openssh-server-9.x...` → ya está instalado.
- Si no, instálalo con:

  ```bash
  sudo zypper install -y openssh-server
  ```

#### b) Habilitar SSH y permitirlo en el firewall

```bash
sudo systemctl enable --now sshd
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

#### c) Comprobar que está activo

```bash
systemctl status sshd
```

Debe decir **`active (running)`**.

#### d) Permitir que root entre por SSH

1. Abre el archivo de configuración:

   ```bash
   vim /etc/ssh/sshd_config
   ```

2. Busca la línea `PermitRootLogin`.
3. Pulsa `i` para editar.
4. Tiene que quedar así:

   ```
   PermitRootLogin yes
   ```

5. Pulsa `ESC`, escribe `:wq` y presiona `Enter` para guardar y salir.
6. Reinicia el servicio:

   ```bash
   systemctl restart sshd
   ```

> [!NOTE]
> En openSUSE Leap 16 el archivo `/etc/ssh/sshd_config` puede aparecer **vacío**, porque la configuración original está en `/usr/etc/ssh/`. Si te pasa, puedes hacerlo con un solo comando en lugar de usar vim:
> ```bash
> echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/10-root-login.conf
> systemctl restart sshd
> ```

---

## 2. Preparar la VM MASTER

Ve a la **VM MASTER** e instala los programas necesarios:

```bash
sudo zypper refresh
sudo zypper install -y ansible python313-passlib git openssh-clients
```

| Programa | Para qué sirve |
|---|---|
| `ansible` | Es el encargado del despliegue. |
| `python313-passlib` | Ansible lo necesita para crear las contraseñas de los usuarios de Samba y FTP. |
| `git` | Para clonar el repositorio. |
| `openssh-clients` | Trae `ssh`, `ssh-keygen` y `ssh-copy-id`. |

Comprueba que Ansible está instalado:

```bash
ansible --version
```

---

## 3. Clonar el proyecto

1. Ve a tu carpeta personal:

   ```bash
   cd ~
   ```

2. Clona el repositorio:

   ```bash
   git clone https://github.com/nachoMejia/proyecto_ansible.git
   ```

3. Entra a la carpeta del proyecto y mira los directorios:

   ```bash
   cd proyecto_ansible
   ls
   ```

Deberías ver todo el contenido del proyecto (`ansible.cfg`, `hosts`, `playbook.yml`, `group_vars`, etc.).

> [!WARNING]
> **TODOS** los comandos de Ansible se ejecutan **dentro de esta carpeta**. Si cierras la terminal, vuelve a entrar con `cd ~/proyecto_ansible`.

---

## 4. Llave SSH: crearla y copiarla al servidor

Para que Ansible entre a la VM DESTINO **sin pedir contraseña cada vez**, usamos una **llave SSH**.

#### a) Crear la llave *(solo 1 vez)*

```bash
ssh-keygen -t ed25519
```

Te hará preguntas: presiona **`ENTER` en todo** (ubicación por defecto y sin frase de contraseña).

#### b) Copiar la llave a la VM DESTINO

```bash
ssh-copy-id root@IP_VM_DESTINO
```

- Si pregunta `Are you sure you want to continue connecting (yes/no)?` → escribe **`yes`**.
- Luego pide la **contraseña de root de la VM DESTINO** (la que pusiste al instalar openSUSE).
- Al escribirla **no se ve nada en pantalla**; es normal, escríbela y presiona `Enter`.

> [!TIP]
> Si tienes más VM's DESTINO, **repite este paso con cada una**.

#### c) Comprobar

```bash
ssh root@IP_VM_DESTINO
```

- Si entra **sin pedir contraseña**, está listo.
- Sal escribiendo `exit`.

---

## 5. Poner la IP de la VM DESTINO en el archivo `hosts`

El archivo `hosts` le dice a Ansible **a qué máquina conectarse**.

1. Abre el archivo:

   ```bash
   vim hosts
   ```

2. Presiona `i` para editar.
3. Añade la IP de la VM DESTINO:

   ```ini
   [servidor]
   NOMBRE_VM ansible_host=IP_VM_DESTINO

   [servidor:vars]
   ansible_user=root
   ansible_ssh_common_args='-o StrictHostKeyChecking=no'
   ```

   - **`NOMBRE_VM`** → el nombre que quieras darle a tu VM. Tiene que haber un nombre, no lo dejes vacío.
   - **`IP_VM_DESTINO`** → la IP de la VM DESTINO.
   - Si tienes más VM's, continúa la lista con el mismo formato:

     ```ini
     [servidor]
     destino1 ansible_host=192.168.0.12
     destino2 ansible_host=192.168.0.13
     ```

4. Para salir: `ESC`, escribe `:wq` y presiona `Enter`.

---

## 6. La contraseña del vault

Las contraseñas de los servicios están **cifradas** en el archivo `group_vars/servidor/vault.yml`.

> [!IMPORTANT]
> **Contraseña del vault:** `aso12345`

#### a) Crear el archivo `.vault_pass`

Ansible lee la contraseña del vault desde un archivo llamado `.vault_pass`. Este archivo **no viene en el repositorio** (está en el `.gitignore` por seguridad), así que hay que crearlo dentro de la carpeta del proyecto:

```bash
echo 'aso12345' > .vault_pass
chmod 600 .vault_pass
```

> [!NOTE]
> El `chmod 600` hace que solo tu usuario pueda leer el archivo.

#### b) Revisar que exista el archivo `.vault_pass`

Como empieza con punto, es un archivo oculto. Para verlo usa `ls -a`:

```bash
ls -a
```

#### c) Editar las contraseñas *(opcional)*

```bash
ansible-vault edit group_vars/servidor/vault.yml
```

- Ahí podrás editar todas las contraseñas de los servicios.
- Pulsa `i` para editar; `ESC` y escribe `:wq` para salir.
- Al guardar, el archivo **se vuelve a cifrar solo**.

---

## 7. Probar la conexión *(opcional)*

```bash
ansible servidor -m ping
```

Tiene que salir:

```
NOMBRE_VM | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 8. Desplegar

```bash
ansible-playbook playbook.yml
```

Verás todas las tareas que se están ejecutando y algunos mensajes:

| Mensaje | Color | Significado |
|---|---|---|
| `ok` | Verde | Ya estaba hecho, no cambió nada. |
| `changed` | Amarillo | Se hizo un cambio (instaló, copió, configuró). |
| `skipping` | Celeste | No hacía falta hacer esa tarea. |
| `failed` | Rojo | Error (lee el mensaje rojo). |

Al final aparecerá el resumen **PLAY RECAP**.

> [!IMPORTANT]
> Lo importante es que diga **`unreachable=0`** y **`failed=0`**.

---

## 9. Comprobar que todo funciona

> Cambia `IP_VM_DESTINO` por la IP de tu VM DESTINO.

| Servicio | Cómo probar | Resultado esperado |
|---|---|---|
| **Apache** | Abrir en el navegador `http://IP_VM_DESTINO` | Responde la página de Apache |
| **phpMyAdmin** | Abrir `http://IP_VM_DESTINO/phpMyAdmin` | Pantalla de login. Usuario: `root` · Contraseña: la de `vault_mysql_root_password` |
| **Samba** | `smbclient //IP_VM_DESTINO/aso -U asosamba` | Entra a la carpeta compartida `/opt/aso` |
| **PostgreSQL** | En la VM DESTINO: `psql -h localhost -U aso_pg -d aso_pg` | Pide contraseña y entra a la base `aso_pg` |
| **FTP** | `ftp IP_VM_DESTINO` · Usuario: `asoftp` | Entra a la carpeta del usuario |

También puedes revisar que todos estén corriendo con:

```bash
systemctl status apache2 mariadb smb postgresql vsftpd
```

Todos deben decir **`active (running)`**.