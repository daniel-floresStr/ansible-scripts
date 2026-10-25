# Despliegue automatizado con Ansible en openSUSE Leap 16.0

Proyecto de la materia Aplicaciones de Sistemas Operativos (ASO). Automatiza con Ansible el despliegue de servicios Linux sobre máquinas virtuales openSUSE Leap 16.0: FTP, LAMP, Samba y PostgreSQL, organizados en roles.

## Agregar una máquina nueva al despliegue
 
Pasos para incorporar un equipo **vacío** (openSUSE Leap 16.0 recién instalado) y desplegar
en él los roles `common`, `lamp` y `samba`.
 
### Valores a reemplazar
 
| Placeholder | Significado | Ejemplo |
|---|---|---|
| `<alias>` | Nombre de la máquina en el inventario | `pc03` |
| `<IP>` | Dirección IP de la máquina nueva | `192.168.100.80` |
| `<usuario>` | Usuario con permisos `sudo` en esa máquina | `daniel` |
 
### Requisitos previos
 
- openSUSE Leap 16.0 instalado.
- Un usuario con permisos `sudo`.
- Conectividad de red con el nodo master y salida a internet (para descargar paquetes).
---
 
### 1. En la máquina nueva
 
```bash
sudo systemctl enable sshd
python3 --version
ip a
```
 
- `sshd` debe quedar activo. Anotar la IP que muestra `ip a`.
- Si no hay Python 3, instalarlo antes de continuar, porque Ansible lo necesita desde la primera tarea:
```bash
  sudo zypper install -y python3
```
 
---
 
### 2. En el nodo master
 
**2.1 Verificar red y puerto SSH**
```bash
ping <IP>
```
 
**2.2 Copiar la llave SSH** (la contraseña se pide solo esta vez)
```bash
ssh-copy-id <usuario>@<IP>
ssh <usuario>@<IP> hostname
```
El segundo comando debe entrar **sin pedir contraseña**.
 
**2.3 Agregar la máquina al inventario**
 
Editar `hosts` y añadir la línea **debajo de `[servidor]`**, sin usuario ni contraseña:
```ini
[servidor]
<alias> ansible_host=<IP>
```
Comprobar el resultado:
```bash
cat hosts
```
 
**2.4 Definir los datos de conexión de esa máquina**
 
Solo es necesario si su usuario o la contraseña de `sudo` **son distintos** a los de
`group_vars/all`. Si son iguales, saltar al paso 2.5.
 
```bash
mkdir -p host_vars/<alias>
 
cat > host_vars/<alias>/vars.yml << 'EOF'
---
ansible_user: "{{ vault_ansible_user }}"
ansible_become_password: "{{ vault_become_password }}"
EOF
 
cat > host_vars/<alias>/vault.yml << 'EOF'
---
vault_ansible_user: "<usuario>"
vault_become_password: "<contraseña_sudo>"
EOF
 
ansible-vault encrypt host_vars/<alias>/vault.yml
```
 
> Reemplazar `<contraseña_sudo>` antes de ejecutar el bloque. Si `sudo` en esa máquina pide la
> contraseña de **root**, usar esa. El archivo `vault.yml` queda cifrado y puede subirse a GitHub.
 
**2.5 Probar la conexión**
```bash
ansible <alias> -m ping
ansible <alias> -b -m command -a "whoami"
```
Resultado esperado: `SUCCESS` y `root`. No continuar si alguno falla.
 
**2.6 Desplegar (dos veces)**
```bash
time ansible-playbook site.yml --limit <alias>
time ansible-playbook site.yml --limit <alias>
```
 
| Ejecución | Resultado esperado |
|---|---|
| Primera | Muchas tareas en `changed`; instala todo y tarda más |
| Segunda | `changed=0 failed=0` (el despliegue es idempotente) |
 
En una máquina vacía **no** se necesita `--skip-tags mariadb` ni `samba_replace_config: false`:
los valores por defecto aplican el modo completo.
 
---
 
### 3. Verificación
 
```bash
ansible <alias> -b -a "systemctl is-active apache2 mariadb smb nmb firewalld"
curl -s -o /dev/null -w "%{http_code}\n" http://<IP>/info.php
curl -s -o /dev/null -w "%{http_code}\n" http://<IP>/phpMyAdmin/
```
 
Se esperan cinco respuestas `active` y dos códigos `200`.
 
Desde un equipo Windows, probar el recurso compartido en `\\<IP>\compartida` con el usuario
`sambauser` y crear un archivo. Debe quedar con permisos `0664` en el destino:
```bash
ansible <alias> -b -a "ls -l /srv/compartida"
```
 
---
 
### Solución de problemas
 
| Síntoma | Causa probable | Solución |
|---|---|---|
| `UNREACHABLE` | Red distinta, `sshd` apagado o llave no copiada | Revisar `ping`, `nc -zv <IP> 22` y repetir `ssh-copy-id` |
| `Incorrect sudo password` | Contraseña equivocada en el vault, o `sudo` pide la de root | Corregir con `ansible-vault edit host_vars/<alias>/vault.yml` |
| `No such file or directory: python` | Falta Python 3 en la máquina | `sudo zypper install -y python3` |
| Falla `Instalar paquetes` | Sin internet o sin acceso a los repositorios | En la máquina: `sudo zypper ref` |
| `Host key verification failed` | Huella SSH antigua de otra máquina con la misma IP | `ssh-keygen -R <IP>` |


---
