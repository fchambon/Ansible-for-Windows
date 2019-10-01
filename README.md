# Quick Start Ansible pour Windows Server dans azure !

Voici un "quickstart" pour gérer les configurations des serveurs "Windows Server 2016/2019" dans Azure.<br/>
Dans ce quickstart, la communication entre la machine de controle Ansible et les serveurs cibles Windows se fera en SSH avec une authentification par cle publique<br/>

**Prerequis pour les machines cibles Windows Server:**<br/>
Pour les machines cibles "Windows Server", il faudra installer un serveur OpenSSH pour Windows avec authentification par cle publique.<br/>
Pour Windows Server 2019 c'est une nouvelle "feature". https://docs.microsoft.com/fr-fr/windows-server/administration/openssh/openssh_install_firstuse Exemple de template : https://github.com/Pierre-Chesne/Windows-Server-2019-OpenSSH <br/>
Pour Windows Server 2016 https://github.com/PowerShell/Win32-OpenSSH/releases (prendre OpenSSH-Win64.zip)<br/>

Pour le paramétrage du serveur OpenSSH (Windows 2016/2019) il faut :<br/>
- Pour Windows Server 2016 <br/>
 -> copier "OpenSSH-Win64" dans "c:\Program Files" <br/>
 -> Executer "install-sshd.ps1" <br/>
 -> demarrer les services "OpenSSH SSH Server" & "OpenSSH Authentication Agent" <br/>
 -> creer un repertoire .ssh dans votre profil (ex: c:\users\pierrc\.ssh) <br/>
 -> creer un fichier texte "authorized_keys" (ex: c:\users\pierrc\.ssh\authorized_keys )<br/>
 -> editez le fichier "authorized_keys" et copier votre cle publique <br/>
 -> Ouvrir le Pare-feu Windows en entre pour le protocol SSH (TCP 22) <br/>
 -> Dans la base de registre HKLM/SOFTWARE/OpenSSH creer une "String Value" "DefaultShell" en valeur mettre le chemin de PowerShell (C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe)<br/>
 -> modifier le fichier C:\ProgramData\ssh\sshd_config:<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> decommenter 'PubkeyAuthentication yes'<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> commenter '#Match Group administrators'<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> commenter '#AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys'<br/>
 -> redemarrer les services "OpenSSH SSH Server" & "OpenSSH Authentication Agent" <br/>
 -> test "ssh pierrc@ipvmazure"

- Pour Windows Server 2019 <br/>
 -> Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0<br/>
 -> créer un repertoire .ssh dans votre profil (ex: c:\users\pierrc\.ssh) <br/>
 -> créer un fichier texte "authorized_keys" (ex: c:\users\pierrc\.ssh\authorized_keys )<br/>
 -> editez le fichier "authorized_keys" et copier votre cle publique <br/>
 -> demarrer les services "OpenSSH SSH Server"
 -> Dans la base de registre HKLM/SOFTWARE/OpenSSH créer une "String Value" "DefaultShell" en valeur mettre le chemin de PowerShell (C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe)<br/>
 -> modifier le fichier C:\ProgramData\ssh\sshd_config:<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> decommenter 'PubkeyAuthentication yes'<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> commenter '#Match Group administrators'<br/>
     &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;-> commenter '#AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys'<br/>
 -> redemarrer les services "OpenSSH SSH Server" & "OpenSSH Authentication Agent" <br/>
 -> test "ssh pierrc@ipvmazure"

**Prerequis pour la machine de controle :**<br/>
Pour une distro Ubuntu 16.04 LTS : <br/>
Installation Ansible <br/>
- Installation des packages requis pour les modules Azure Python SDK: <br/>
```
sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
```
- Installation packages Ansible pour azure:
```
sudo pip install ansible[azure]
```
- Installation Azure Cli (option pour installer des ressources Azure) <br/>
```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
- S'authentifier <br/>
```
az login
```
- Installation du package Python pour Windows <br/>
```
sudo pip install pywinrm
```
- Inventaire pour la gestion des machines cibles. Exemple : <br>
```
code winhosts
```
```
[win]
ip ou url
ip ou url

[win:vars]
ansible_connection=ssh
ansible_shell_type=powershell
```
Test de la connexion:<br/>
```
ansible -i winhosts win -m win_ping
```
Retour:<br/>
```
104.40.217.162 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
Une fois le test de connexion reussi reste plus qu'a ecrire les playbooks<br/>
**Les modules Windows Ansible pour Windows**<br/>
Les modules Ansible pour Windows sont ici : https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html<br/>
Par exemple, vous avez une VM Windows Server 2019 avec un disque data, voici un exemple de "plabook" avec trois roles pour initialiser le disque et l'installation d'un Active Directory:<br/>

```
ansible-playbook -i winhosts win.yml
```
```
---
- hosts: win #bloc win dans l'inventaire (winhost)
  remote_user: pierrc
  
  roles:
    - set_disk # intialisation du disque / Creation de la partition / Formatage du disque
    - add_ad   # Ajout du role Active Directory
    - set_ad   # parametrage de l Active Directory
...
```
Role "set_disk":<br/>
```
---
- name: Initialisation du disque data "s"
  # module win_shell -> PS Initilize-Disk
  win_shell:
   "Initialize-Disk -Number 2"

- name: creation de la partition
  # module win_partition
  win_partition:
    drive_letter: S
    partition_size: -1
    disk_number: 2

- name: formatage
  # module win_shell -> PS Format-Volume
  win_shell:
    "Format-Volume -DriveLetter S"
...
```
Role "add_ad":<br/>
```
---
- name: Installation du role "Active Directory"
  # module win_feature
  win_feature:
    name: AD-Domain-Services
    include_management_tools: yes
    include_sub_features: yes
    state: present
...
```
Role "set_ad":<br/>
```
---
- name: Setup de l AD
  # module win_domain
  win_domain:
    dns_domain_name: 'ma-pme.local'
    safe_mode_password: 'Password123$'
    sysvol_path: S:\Windows\SYSVOL
  register: win_domain

- name: Reboot le serveur
  # module win_reboot
  win_reboot:
    pre_reboot_delay: 15
  when: win_domain.reboot_required
...
```



