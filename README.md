# devops_101

> ***DevOps for the Desperate**. A Hands-On Survival Guide*. [Libro](https://nostarch.com/devops-desperate), [repo](https://github.com/bradleyd/devops_for_the_desperate).

- [devops\_101](#devops_101)
  - [Entornos de desarrollo y operaciones](#entornos-de-desarrollo-y-operaciones)
  - [Objetivos](#objetivos)
  - [Proyectos](#proyectos)
    - [Proyecto 1. Vagrant + Ansible](#proyecto-1-vagrant--ansible)
      - [1.1 (Ch. 1) Instalación de Vagrant y Ansible](#11-ch-1-instalación-de-vagrant-y-ansible)
      - [1.2 Configuraciones: hardware, VirtualBox, Vagrant](#12-configuraciones-hardware-virtualbox-vagrant)
      - [1.3 Implementación del `Vagrantfile`](#13-implementación-del-vagrantfile)
      - [1.4 Ansible: `site.yml`](#14-ansible-siteyml)
      - [1.5 (Ch. 2) Ansible: usuarios, grupos y contraseñas](#15-ch-2-ansible-usuarios-grupos-y-contraseñas)
        - [`pam_pwquality.yml`](#pam_pwqualityyml)
        - [`user_and_group.yml`](#user_and_groupyml)
        - [Demo: usuarios y permisos](#demo-usuarios-y-permisos)
      - [1.6 (Ch. 3) Ansible: ssh and 2FA](#16-ch-3-ansible-ssh-and-2fa)
        - [Generar claves ssh y `authorized_keys.yml`](#generar-claves-ssh-y-authorized_keysyml)
        - [`two_factor.yml` y `google_authenticator`](#two_factoryml-y-google_authenticator)


## Entornos de desarrollo y operaciones

Nuestro hardware:

| Máquina       | Procesador                    | RAM   | Almacenamiento                            | OS            | ¿Multiboot?
| ---           | ---                           | ---   | ---                                       | ---           | ---
| Acer EX2511   | i5-4210U (4)<br> @ 2.70 GHz   | 16GB | 1x240GB SSD<br> 1x480 SSD                  | Pop!_OS 22.04 | Arch Linux
| **MSI GL76**  | i7-11800H (16)<br> @ 4.60 GHz | 32GB | 1x2TB NVMe<br> 1x1TB NVMe<br> 1x1TB HDD    | Pop!_OS 22.04 | No
| **Pi 5**      | ...                           | ...   | ...                                       | ...           | No
 

<!--
Cloud IaaS:

<table>
<thead>
<tr>
  <th>Provider
  <th>Cuenta
  <th>Servicios
  <th>Integración
</tr>
</thead>
<tbody>
<tr>
    <td rowspan=3>AWS
    <td>pq2
    <td>VM + IP fija
    <td>DonDominio: pabloqpacin.com
</tr>
<tr>
    <td>pqp
    <td colspan=2>...
</tr>
<tr>
    <td>p.q
    <td colspan=2>...
</tr>
<tr>
    <td rowspan=2>Trevenque
    <td colspan=3>... vSphere, Plesk...
</tr>
<tr>
    <td colspan=3>...
</tr>
<tr>
    <td>GCP
    <td colspan=3>...
</tr>
</tbody>
</table>
 -->


## Objetivos

Tecnologías que queremos aprender:

<table>
<thead>
<tr>
    <th>Proyecto
    <th colspan=2>Tecnologías
    <th>Entorno/
    <th>Plataforma
</thead>
<tbody>
<tr>
    <td>1
    <td><b>Vagrant
    <td><b>Ansible
    <td>Local (Acer EX2511)
    <td>VirtualBox
<tr>
    <td>2
    <td colspan=2>Terraform
    <td>Remoto
    <td>AWS
<tr>
    <td>3
    <td>Kubernetes
    <td>CI/CD
    <td>...
    <td>...
</tbody>
</table>


## Proyectos

**IMPORTANTE**: <u>clonar el repo</u> para manejar los archivos de los proyectos.

```bash
git clone https://github.com/pabloqpacin/devops_101.git $HOME/devops_101
```

### Proyecto 1. Vagrant + Ansible

<!-- - [ ] [/vagrant](/vagrant/)
- [ ] [/ansible](/ansible/) -->

Nos conectamos con `ssh` desde nuestra máquina de desarrollo *MSI GL76*  a la de operaciones *Acer EX2511*. Ambas están en nuestra red local y pilotan el sistema operativo *Pop!_OS* (derivado de Ubuntu).

La máquina *EX2511* tiene el OS instalado en `/dev/sdb1` (esta sería la partición *root* o `/`). Previamente hemos creado la partición `/dev/sdb2` con idea de almacenar VMs. Aunque no es necesario, decidimos dar persistencia al montaje de particiones con los siguientes comandos.

```bash
sudo mkdir -p /media/$USER/LAB
UUID=$(blkid /dev/sdb2 | awk '{print $3}' | awk -F '=' '{print $2}' | tr -d '"')
echo "UUID=$UUID /media/$USER/LAB ext4 defaults 0  2" | \
    sudo tee -a /etc/fstab
sudo mount -a
# df -h | grep /media/$USER/LAB
```


#### 1.1 (Ch. 1) Instalación de Vagrant y Ansible

Instalamos **Vagrant** (Ubuntu/Debian).

<!--
```bash
if command -v vagrant &>/dev/null; then
    echo "Vagrant is already installed."
else
    DISTRO=$(grep 'ID_LIKE' /etc/os-release | awk -F '=' '{print $2}' | tr -d '"')
    case $DISTRO in
        'ubuntu debian' | 'ubuntu' | 'debian')
            wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
            echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
            sudo apt update && sudo apt install vagrant
            ;;
        *)
            echo "Distro not supported. Terminating script."
            exit 1
            ;;
    esac
fi
```
-->

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

Instalar **Ansible** no es necesario para operar con Vagrant, pero dado que nuestro `Vagrantfile` hace uso de Ansible, mejor instalarlo ya (Ubuntu/Debian). Si no lo hacemos, habría que comentar las líneas relevantes del `Vagrantfile`.

```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```


#### 1.2 Configuraciones: hardware, VirtualBox, Vagrant

Hemos preparado [el script `vagrant_vbox_env.sh`](/scripts/vagrant_vbox_env.sh) para realizar varias tareas importantes.

1. Asignar a la variable de entorno `$VAGRANT_HOME` el valor `/var/vagrant.d` (por defecto sería `~/.vagrant.d`). Aquí se almacenarán varios archivos de configuración de **Vagrant**. Cada imagen o *box* que descarguemos pesará medio GB así que puede llegar a pesar mucho y preferimos dejar este tipo de *bloat* fuera de `/home`.

<!-- Primero descargamos el script y lo guardamos como `~/vagrant_vbox_env.sh`. Si hemos clonado el repositorio también podríamos copiarlo o hacer un symlink en local.

```bash
curl -so ~/vagrant_vbox_env.sh \
    https://raw.githubusercontent.com/pabloqpacin/devops-101/main/vagrant/vagrant_vbox_env_sh
``` -->

2. Queremos almacenar las VMs en la **partición** `/dev/sdb2` montada como `/media/$USER/LAB` de forma persistente. Igualmente verificamos que la partición está montada y si no es así se intenta mediante con el comando `gio mount -d /dev/sdb2`. Desafortunadamente este comando requiere iniciar la sesión gráfica de escritorio tipo Gnome, Cosmic..., por eso la persistencia.

3. Finalmente, revisamos y definimos el directorio donde **VirtualBox** almacenará por defecto las VMs. 

```bash
# VBoxManage list systemproperties | grep "Default machine folder" 
VBoxManage setproperty machinefolder /media/$USER/LAB/VBox
```

Hacemos que la shell (*zsh* o *bash*) ejecute nuestro script verificador al iniciarse.

```bash
echo -e "\nsource ~/devops_101/scripts/vagrant_vbox_env.sh" >> ~/.zshrc || \
echo -e "\nsource ~/devops_101/scripts/vagrant_vbox_env.sh" >> ~/.bashrc
```

Con todo preparado, podemos iniciar una nueva shell e instalar los plugins necesarios para este proyecto.

```bash
# watch tree $VAGRANT_HOME

vagrant plugin update
vagrant plugin install vagrant-vbguest
# vagrant plugin install vagrant-share vagrant-disksize
vagrant plugin list
```


#### 1.3 Implementación del `Vagrantfile`

Nos vamos al directorio `vagrant` de nuestro repositorio.

```bash
cd ~/devops_101/vagrant
```

Repasamos [nuestro `Vagrantfile`](/vagrant/Vagrantfile).

```Vagrantfile
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"
    config.vm.hostname = "vagrant-ubuntu-2204"

    config.vm.network "public_network", bridge: "enp2s0"

    config.vbguest.auto_update = false

    config.vm.provider "virtualbox" do |vb|
        vb.name = "vagrant-ubuntu-2204"
        vb.memory = "2048"
        vb.cpus = 2
    end

    config.vm.synced_folder ".", "vagrant", disabled: true

    config.vm.provision "ansible" do |ansible|
        ansible.playbook = "../ansible/site.yml"
        ansible.compatibility_mode = "2.0"
    end

end
```

<!-- - [ ] ¿Guardar VM en grupo de VBox? -->
<!-- - [ ] `config.disksize.size = '50GB'` -->
<!-- - [ ] ¿Desactivar primera interfaz NAT? -->
<!-- - [ ] Asignar interfaz a red NAT -->

Verificamos que el `Vagrantfile` está correcto, iniciamos y verificamos la implantación.

> **NOTA**: el comando `vagrant` solo tiene en cuenta las VMs asociadas al `Vagrantfile` del directorio actual en la shell (`pwd`).


```bash
# vagrant list-commands
vagrant validate
vagrant up
# vagrant up --debug
vagrant status
```

Podemos ejecutar comandos en la VM y conectarnos a la nueva VM. Podemos detener/apagar la VM, y eliminarla. También podemos verificar las imágenes/*boxes*.

```bash
vagrant ssh -c "sudo apt update && sudo apt install neofetch --no-install-recommends && neofetch"
# vagrant ssh

vagrant halt
# vagrant destroy

vagrant box list
```

> **NOTA**: si abrimos la GUI de VirtualBox podremos ver las nuevas VMs y es posible conectarse a ellas. Para el login, el usuario y la contraseña son `vagrant`.


#### 1.4 Ansible: `site.yml`

Con **Ansible** ya instalado, nos aseguramos de que nuestro `Vagrantfile` contiene estas líneas:

```Vagrantfile
config.vm.provision "ansible" do |ansible|
    ansible.playbook = "../ansible/site.yml"
    ansible.compatibility_mode = "2.0"
end
```

Este código cargará nuestro *playbook* (archivo de configuración) de Ansible principal para este proyecto. Será necesario ir modificando este archivo `site.yml` para implementar cosas. El resto de esta documentación/proyecto tratará en detalle el resto de archivos `.yml` y las operaciones con Ansible.

Este sería nuestro `site.yml` actualmente.

```yaml
---
- name: Provision VM
  hosts: all
  become: true
  become_method: sudo
  remote_user: ubuntu
  tasks:
    #  - import_tasks: chapter2/pam_pwquality.yml
    #  - import_tasks: chapter2/user_and_group.yml
    #  - import_tasks: chapter3/authorized_keys.yml
    #  - import_tasks: chapter3/two_factor.yml
    #  - import_tasks: chapter4/web_application.yml
    #  - import_tasks: chapter4/sudoers.yml
    #  - import_tasks: chapter5/firewall.yml
  handlers:
    #  - import_tasks: handlers/restart_ssh.yml
```

Al levantar la VM con **Vagrant**, se ejecutará este *playbook* con éxito, si bien al no tener tareas específicas (los *playbooks* que las llevarán a cabo están comentados) no se hará ningún *provisioning* en la VM.

> Decidimos que los archivos sean `.yml` y no `.yaml` por seguir el estilo de la documentación oficial (eg. [Ansible YAML file syntax and structure](https://developers.redhat.com/learning/learn:ansible:yaml-essentials-ansible/resource/resources:ansible-yaml-file-syntax-and-structure)), además de que es el estilo propuesto en el libro. Igualmente cambiamos la línea `become: yes` por `become: true`.


#### 1.5 (Ch. 2) Ansible: usuarios, grupos y contraseñas

Iremos creando los archivos `.yml` con las tareas en el directorio `ansible/chapter2/`. Para operar con ellos habrá que descomentar las líneas relevantes en `ansible/site.yml`.

Si la VM ya existe (ya hicimos `vagrant up`) usaremos este comando para aplicar **Ansible** <!--según se define en nuestro `Vagrantfile`-->.

```bash
# vagrant up
vagrant provision
# vagrant provision --debug
```


##### `pam_pwquality.yml`

En esta primera tarea se instala el paquete `libpam-pwquality` y se edita el archivo de configuración `/etc/pam.d/common-password` para imponer las siguientes restricciones en la creación de contraseñas:

- Un mínimo de 12 caracteres
- Una letra minúscula
- Una letra mayúscula
- Un caracter numérico
- Un caracter no alfanumérico
- Tres intentos
- Desactivar invalidación de root

```yaml
---
- name: Install libpam-pwquality
  package:
    name: "libpam-pwquality"
    state: present

- name: Configure pam_pwquality
  lineinfile:
    path: "/etc/pam.d/common-password"
    regexp: "pam_pwquality.so"
    line: "password required pam_pwquality.so minlen=12 lcredit=-1 ucredit=-1 dcredit=-1 ocredit=-1 retry=3 enforce_for_root"
    state: present

    #- name: Limit Password Reuse
    #  lineinfile:
    #    dest: "/etc/pam.d/common-password"
    #    regexp: "remember=5"
    #    line: "password sufficient pam_unix.so use_authtok remember=5"
    #    state: present
```

<!-- - [ ] ¿Cómo se instala? Supongo que `apt install foo` pero bueno, en otros casos podría ser `snap install bar`... -->

##### `user_and_group.yml`

> **IMPORTANTE**: es insecuro tener contraseñas o llaves en un repo público. Implementar [**Ansible Vault**](https://docs.ansible.com/ansible/latest/vault_guide/index.html)... <!--https://docs.ansible.com/ansible/2.9/user_guide/vault.html-->


En nuestra máquina (no la VM) vamos a necesitar los programas `pwgen` para generar contraseñas seguras y `mkpasswd` para generar los *hashes* de estas contraseñas. Escribiremos el *hash* en el siguiente archivo `.yml`. <!--Aunque la contraseña no nos hace falta, podemos guardarla en nuestro **gestor de contraseñas** favorito, KeePassXC.-->

```bash
sudo apt update
sudo apt install pwgen whois

pass=$(pwgen --secure --capitalize --numerals --symbols 12 1)

echo $pass | mkpasswd --stdin --method=sha-512; echo $pass
    # $6$QJmzvbMhlt7C.qOO$uSkIZs/nINf2HFR/.nerO3qfRzIOR53BwZVwJspkkKdrO1KLOzIcW7hG7UWAhGTh/VJVvxhbZO7qloGqGs30E/
    # ]aR8WG{/yqG}
```


Este es el archivo, y con estas tareas conseguimos lo siguiente:

- crear el grupo *developers*
- crear el usuario *bender*
- añadir a *bender* al grupo *developers*
- crear el directorio `/opt/engineering`
- crear un archivo en el nuevo directorio

```yaml
- name: Ensure group 'developers' exists
  group:
    name: developers
    state: present

- name: Create the user 'bender'
  user:
    name: bender
    shell: /bin/bash
    password: $6$QJmzvbMhlt7C.qOO$uSkIZs/nINf2HFR/.nerO3qfRzIOR53BwZVwJspkkKdrO1KLOzIcW7hG7UWAhGTh/VJVvxhbZO7qloGqGs30E/

- name: Assign 'bender' to the 'developers' group
  user:
    name: bender
    groups: developers
    append: yes

- name: Create a directory named 'engineering'
  file:
    path: /opt/engineering
    state: directory
    mode: 0750
    group: developers

- name: Create a file in the engineering directory
  file:
    path: "/opt/engineering/private.txt"
    state: touch
    mode: 0770
    group: developers
```

<!-- - [ ] en principio solo local, ¿y en carpeta compartida (tema Vagrant)? -->

Este es un buen momento para editar el `site.yml` y ejecutar `vagrant provision`.

##### Demo: usuarios y permisos

Nos logueamos en la VM. Nuestro usuario debería ser `vagrant`.

```bash
cd ~/devops_101/vagrant
# vagrant ssh -c "whoami"
vagrant ssh
```

Verificamos que existen el usuario *bender* y el grupo *developers*.

```bash
getent passwd bender
    # bender:x:1002:1003::/home/bender:/bin/bash

getent group developers bender
    # developers:x:1002:bender
    # bender:x:1003:
```

Para el archivo, primero comprobamos que *vagrant* no tiene acceso, luego nos logueamos como *bender* y comprobamos que tenemos acceso.

```bash
ls -la /opt/engineering/
    # ls: cannot open directory '/opt/engineering/': Permission denied

# su bender
    # ]aR8WG{/yqG}

sudo su - bender

# groups
    # bender developers

ls -la /opt/engineering/
    # drwxr-x--- 2 root developers 4096 Jul  6 14:54 .
    # drwxr-xr-x 3 root root       4096 Jul  6 14:54 ..
    # -rwxrwx--- 1 root developers    4 Jul  6 15:07 private.txt
```

#### 1.6 (Ch. 3) Ansible: ssh and 2FA

Para el usuario *bender* de nuestra VM. Desactivaremos el acceso por ssh con contraseña y habilitaremos 2FA con **llaves ssh** y *google authenticator*.

##### Generar claves ssh y `authorized_keys.yml`

En nuestra máquina de operaciones generamos nuevas claves **ssh** (privada y pública). El siguiente comando nos pedirá una *passphrase*, que debemos guardar en un gestor de contraseñas o similar.

```bash
ssh-keygen -t rsa -f ~/.ssh/dftd -C dftd

# read -p "Passphrase para las llaves 'dftd': " passphrase
# ssh-keygen -t rsa -f ~/.ssh/dftd -C dftd -N "$passphrase"
```

Ahora podemos editar nuestro `site.yml` para incluir el archivo `ansible/chapter3/authorized_keys.yml`.

```yaml
---
- name: Set authorized key file from local user
  authorized_key:
    user: bender 
    state: present
    key: "{{ lookup('file', lookup('env','HOME') + '/.ssh/dftd.pub') }}"
```


##### `two_factor.yml` y `google_authenticator`

<!--
For this example, you’ll use a *time-based one-time password (TOTP)* to
satisfy the “something you have” portion, along with your public key for
access. You’ll use the `Google Authenticator` package to configure your VM to
use TOTP tokens for logging in. These TOTP tokens are usually generated
from an application like `oathtool` (*https://www.nongnu.org/oath-toolkit/*) and
are valid for only a short period of time. I have taken the liberty of creating
10 TOTP tokens that Ansible will use for you, but I will also show you how
to use oathtool (more on this later).
-->

El según factor de autenticación va a ser **TOTP** (*Time-based one-time password*) mediante `GoogleAuthenticator`. Lo ideal sería, teniendo preparada la app de Android **Google Authenticator**, acceder como *bender* a la VM tras instalar el paquete `libpam-google-authenticate` (1ª tarea a continuación), ejecutarlo de la siguiente forma, escanear el QR con el móvil e introducir en la terminal el código que salga en la app.

```bash
google-authenticator -f -t -d -r 3 -R 30 -w 17 -e 10
    # Warning: pasting the following URL into your browser exposes the OTP secret to Google:
    #   https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/vagrant@vagrant-ubuntu-2204%3Fsecret%3DX5AH<...>JT7Q%26issuer%3Dvagrant-ubuntu-2204
    # <QR>
    # Your new secret key is: X5AH<...>JT7Q
    # Enter code from app (-1 to skip): 972935

    # Code confirmed
    # Your emergency scratch codes are:
    #   11880927
    #   35111193
    #   32810950
    #   87502136
    #   81456931
    #   79721071
    #   31977925
    #   28440037
    #   12366122
    #   65260038
```

Al completarse el proceso, se crea el archivo `~/.google_authenticator`. En este caso en lugar de generar todo esto, vamos a copiar un archivo proporcionado por el autor del **libro** (2ª tarea).

> **IMPORTANTE**: de nuevo, no hay seguridad si publicamos en internet los tokens y las llaves secretas; lo suyo sería usar ***Ansible Vault***, [*HashiCorp's Vault*](https://www.vaultproject.io/) o algo similar.




En definitiva, el archivo `.yml` de este apartado cumplirá los siguientes objetivos:
1. Instalar `libpam-google-authenticate`
2. Copiar un archivo de configuración de `GoogleAuthenticator` <!--OJO-->
3. Desactivar el login por contraseña para **ssh** (mediante *PAM*)
4. Configurar *PAM* para usar `GoogleAuthenticator` para el login por **ssh**
5. Activar `ChallengeResponseAuthentication`
6. Configurar Método de Autenticación para *bender*, *vagrant* y *ubuntu*
7. Insertar línea adicional "Restart SSH Server"


```yaml
- name: Install the libpam-google-authenticator package
  apt:
    name: "libpam-google-authenticator"
    update_cache: yes
    state: present

- name: Copy over Preconfigured GoogleAuthenticator config
  copy:
    src: ../ansible/chapter3/google_authenticator
    dest: /home/bender/.google_authenticator
    owner: bender
    group: bender
    mode: '0600'
  no_log: true

- name: Disable password authentication for SSH
  lineinfile:
    dest: "/etc/pam.d/sshd"
    regex: "@include common-auth"
    line: "#@include common-auth"

- name: Configure PAM to use GoogleAuthenticator for SSH logins
  ansible.builtin.blockinfile:
    path: "/etc/pam.d/sshd"
    prepend_newline: true
    insertafter: EOF
    block: |
        auth required pam_google_authenticator.so nullok"

# - name: Set ChallengeResponseAuthentication to Yes
#   lineinfile:
#     dest: "/etc/ssh/sshd_config"
#     regexp: "^ChallengeResponseAuthentication (yes|no)"
#     line: "ChallengeResponseAuthentication yes"
#     state: present

# - name: Set Authentication Methods for bender, vagrant, and ubuntu
#   blockinfile:
#     path: "/etc/ssh/sshd_config"
#     block: |
#       Match User "ubuntu,vagrant"
#           AuthenticationMethods publickey
#       Match User "bender,!vagrant,!ubuntu"
#           AuthenticationMethods publickey,keyboard-interactive
#     state: present
#   notify: "Restart SSH Server"
```

<!--
4ª (Configure PAM to use GoogleAuthenticator for SSH logins)

This task tells PAM about the Google Authenticator module. It uses the
Ansible lineinfile module again to edit the PAM sshd file. This time, you
just want to add the auth line to the bottom of the PAM file, which lets PAM
know it should use Google Authenticator as an authentication mechanism.
The nullok option at the end of the line tells PAM that this authentication
method is optional, which allows you to avoid locking out users until they
have successfully configured 2FA. In a production environment, you should
remove the nullok option once all users have enabled 2FA.
-->

<!-- ### Proyecto 2. Terraform -->
<!-- ### Proyecto 3. Kubernetes + CI/CD -->
