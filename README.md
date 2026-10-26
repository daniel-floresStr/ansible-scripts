# Despliegue automatizado con Ansible en openSUSE Leap 16.0

Proyecto de la materia Aplicaciones de Sistemas Operativos (ASO). Automatiza con Ansible el despliegue de servicios Linux sobre máquinas virtuales openSUSE Leap 16.0, organizado en roles: `common`, `lamp` y `samba`.

**Repositorio:** https://github.com/daniel-floresStr/Ansible-scripts  
**Compartido con el docente:** gcussi@yahoo.com  
**Integrantes:** ver `team_list.txt`

Los valores escritos entre `< >` (`<ALIAS>`, `<IP_DESTINO>`, `<USUARIO>`, `<CONTRASEÑA_SUDO>`) se reemplazan por los valores reales, **sin** los símbolos `< >`.

## Requisitos previos

- openSUSE Leap 16.0 instalado en la máquina destino.
- Un usuario con permisos sudo en la máquina destino.
- Conectividad de red entre la máquina master y la máquina destino, y salida a internet en ambas (para descargar paquetes).

## Paso 1. Preparar la máquina master (solo la primera vez)

**1.1** Instalar Ansible y git:

```
zypper install ansible
zypper install git
```

**1.2** Clonar el proyecto y entrar a la carpeta:

```
git clone https://github.com/daniel-floresStr/Ansible-scripts.git
cd ansible-scripts
```

**1.3** Crear la llave SSH de la máquina master (si no existe `~/.ssh/id_ed25519.pub`).

Primero comprobar si ya existe:

```
ls ~/.ssh/id_ed25519.pub
```

- Si muestra el archivo, la llave ya existe: saltar este paso y seguir con el Paso 2.
- Si dice `No such file or directory`, crearla con el comando siguiente (pulsar Enter en todas las preguntas):

```
ssh-keygen -t ed25519
```

## Paso 2. En la máquina destino

**2.1** Verificar que está vacía:

```
rpm -qa | grep -Ei 'apache2|mariadb|php|phpmyadmin|samba|vsftpd|postgresql'
systemctl is-active apache2 mariadb smb nmb vsftpd
id sambauser
ls -ld /srv/compartida /srv/www/htdocs/info.php
```

Resultado esperado: `rpm` solo muestra paquetes cliente o librerías (mariadb-client, samba-client, samba-libs, etc.), los servicios salen `inactive` o `unknown`, `no such user` para `sambauser` y `No such file or directory` para las carpetas.

**2.2** Activar SSH y comprobar los requisitos:

```
sudo systemctl enable --now sshd
systemctl is-active sshd
python3 --version
ip a
sudo zypper ref
```

`sshd` debe quedar `active`. Anotar la IP que muestra `ip a` (`<IP_DESTINO>`).

Si no hay Python 3, instalarlo antes de continuar, porque Ansible lo necesita desde la primera tarea:

```
sudo zypper install -y python3
```

## Paso 3. En la máquina master

**3.1** Verificar red:

```
ping -c 3 <IP_DESTINO>
```

**3.2** Copiar la llave SSH (la contraseña se pide solo esta vez):

```
ssh-copy-id <USUARIO>@<IP_DESTINO>
ssh <USUARIO>@<IP_DESTINO> hostname
```

El segundo comando debe entrar sin pedir contraseña.

**3.3** Agregar la máquina al inventario:

```
vim hosts
```

Añadir la línea debajo de `[servidor]`, sin usuario ni contraseña:

```
[servidor]
<ALIAS> ansible_host=<IP_DESTINO>
```

Comprobar el resultado:

```
cat hosts
```

**3.4** Definir los datos de conexión de esa máquina.

Solo es necesario si su usuario o la contraseña de sudo son distintos a los de `group_vars/all`. Si son iguales, saltar al paso 3.5.

Ansible usa por defecto, con todas las máquinas, el usuario de `remote_user` (en `ansible.cfg`) y la contraseña de sudo `vault_become_password` (en `group_vars/all/vault.yml`). Este paso solo hace falta si la máquina destino tiene otros datos.

**Cómo comprobar si coinciden:** ejecutar primero el paso 3.5 y decidir según el resultado.

| Resultado del paso 3.5 | Qué significa | Qué hacer |
|---|---|---|
| `SUCCESS` y `root` | Los datos por defecto sirven | Saltar el 3.4 y seguir al 3.6 |
| `Permission denied` | El usuario no coincide | Hacer el 3.4 |
| `Incorrect sudo password` | La contraseña de sudo no coincide | Hacer el 3.4 |

**Si hay que hacerlo, crear los datos propios de esa máquina:**

```
mkdir -p host_vars/<ALIAS>
vim host_vars/<ALIAS>/vars.yml
```

Contenido de `vars.yml` (con un espacio después de cada dos puntos):

```
---
ansible_user: "{{ vault_ansible_user }}"
ansible_become_password: "{{ vault_become_password }}"
```

Crear el vault cifrado:

```
ansible-vault create host_vars/<ALIAS>/vault.yml
```

Contenido (si sudo en esa máquina pide la contraseña de root, usar esa):

```
---
vault_ansible_user: "<USUARIO>"
vault_become_password: "<CONTRASEÑA_SUDO>"
```

Comprobar que quedó cifrado (debe empezar con `$ANSIBLE_VAULT`):

```
head -1 host_vars/<ALIAS>/vault.yml
```

El archivo `vault.yml` queda cifrado y puede subirse a GitHub.

**3.5** Probar la conexión:

```
ansible-inventory --host <ALIAS>
ansible <ALIAS> -m ping
ansible <ALIAS> -b -m command -a "whoami"
```

Resultado esperado: `SUCCESS` y `root`. No continuar si alguno falla.

**3.6** Desplegar (dos veces):

```
time ansible-playbook site.yml --limit <ALIAS>
time ansible-playbook site.yml --limit <ALIAS>
```

| Ejecución | Resultado esperado |
|---|---|
| Primera | Muchas tareas en `changed`; instala todo y tarda más |
| Segunda | `changed=0 failed=0` (el despliegue es idempotente) |

En una máquina vacía no se necesita `--skip-tags mariadb` ni `samba_replace_config: false`: los valores por defecto aplican el modo completo.

## Paso 4. Verificación

```
ansible <ALIAS> -b -a "systemctl is-active apache2 mariadb smb nmb firewalld"
curl -s -o /dev/null -w "%{http_code}\n" http://<IP_DESTINO>/info.php
curl -s -o /dev/null -w "%{http_code}\n" http://<IP_DESTINO>/phpMyAdmin/
```

Se esperan cinco respuestas `active` y dos códigos `200`.

Desde un equipo Windows, probar el recurso compartido: abrir el Explorador de archivos, escribir en la barra de direcciones

```
\\<IP_DESTINO>\compartida
```

entrar con el usuario `sambauser` y su contraseña, y crear un archivo. Debe quedar con permisos `0664` en el destino:

```
ansible <ALIAS> -b -a "ls -l /srv/compartida"
```

Comprobar el firewall y el cifrado de las contraseñas:

```
ansible <ALIAS> -b -a "firewall-cmd --list-services"
head -1 group_vars/all/vault.yml
```

`firewall-cmd` debe incluir `http` y `samba`. La primera línea del vault debe ser `$ANSIBLE_VAULT;1.1;AES256`.

## Solución de problemas

| Problema | Causa probable | Solución |
|---|---|---|
| `UNREACHABLE` | Red distinta, sshd apagado o llave no copiada | Revisar `ping`, `nc -zv <IP_DESTINO> 22` y repetir `ssh-copy-id` |
| `Permission denied (publickey)` | Ansible usa otro usuario | `ansible-inventory --host <ALIAS>` y corregir `ansible_user` |
| `Incorrect sudo password` | Contraseña equivocada en el vault, o sudo pide la de root | `ansible-vault edit host_vars/<ALIAS>/vault.yml` |
| `mapping values are not allowed` | Falta un espacio después de `:` | Corregir `vars.yml` |
| `No such file or directory: python` | Falta Python 3 en la máquina | `sudo zypper install -y python3` |
| Falla "Instalar paquetes" | Sin internet o sin acceso a los repositorios | En la máquina: `sudo zypper ref` |
| `Host key verification failed` | Huella SSH antigua de otra máquina con la misma IP | `ssh-keygen -R <IP_DESTINO>` |
| `git clone` no conecta | La máquina master no tiene salida a internet | Revisar `ip route` (gateway fijo viejo) |
| `ping` responde pero SSH no conecta | El firewall de la máquina destino bloquea el puerto 22 | En la máquina destino: `sudo firewall-cmd --list-services` (debe incluir `ssh`). Si no está: `sudo firewall-cmd --permanent --add-service=ssh` y luego `sudo firewall-cmd --reload` |
