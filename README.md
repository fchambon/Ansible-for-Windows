# Quick Start Ansible pour Windows Server dans azure !

Voici un "quickstart" pour gérer les configurations des serveurs "Windows Server 2016/2019" dans Azure.<br/>
Dans ce quickstart, la communication entre la machine de controle Ansible et les serveurs cibles Windows se fera en SSH avec une authentification par clé publique<br/>

**Prérequis pour les machines cibles Windows Server:**<br/>
Pour les machines cibles "Windows Server", il faudra installer un serveur OpenSSH pour Windows avec authentification par clé publique.<br/>
Pour Windows Server 2019, c'est une nouvelle "feature". https://docs.microsoft.com/fr-fr/windows-server/administration/openssh/openssh_install_firstuse<br/>
Pour Windows Server 2016, il faut recuperer le Server OpenSHH ici :https://github.com/PowerShell/Win32-OpenSSH/releases (prendre OpenSSH-Win64.zip)<br/>


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
 -> redémarrer les services "OpenSSH SSH Server" & "OpenSSH Authentication Agent" <br/>
 -> test "ssh pierrc@ipvmazure" <br/>
 
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
 -> redémarrer les services "OpenSSH SSH Server" & "OpenSSH Authentication Agent" <br/>
 -> test "ssh pierrc@ipvmazure"<br/>

Pour gagner du temps, voici un template ARM qui déploie automatiquement un Windows Server 2019 avec un serveur OpenSSH https://github.com/Pierre-Chesne/Windows-Server-2019-OpenSSH plus le paramétrage d'authentification par clé publique<br/>

**Prérequis pour la machine de controle Ansible:**<br/>
Pour une distribution Ubuntu 16.04 LTS: <br/>
- Installation des packages requis pour les modules Azure Python SDK: <br/>
```
sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
```
- Installation packages Ansible pour azure:
```
sudo pip install ansible[azure]
```
- Installation Azure Cli (option pour installer des ressources Azure ou pour inventaire dynamique) <br/>
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
- Inventaire statique pour la gestion des machines cibles. Exemple : <br>
```
vim winhosts
```
Copier:
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
Une fois le test de connexion reussi reste plus qu'à écrire les playbooks<br/>
**Les modules Windows Ansible pour Windows**<br/>
Les modules Ansible pour Windows sont ici : https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html<br/>
Par exemple, vous avez une VM Windows Server 2019 avec un disque data , voici un exemple de "playbook" qui va initialiser le disque (sysvol), installater l' Active Directory et rebooter le serveur:<br/>
```
---
- hosts: win
  remote_user: pierrc

  tasks:
  - name: Recuperation des infos disque
    win_disk_facts:

  - name: Initialisation du disque data 
    win_shell:
        "Initialize-Disk -Number 2"
    when: (ansible_disks[2].partition_style == "RAW")
    notify: 
        - Creation de la partition
        - Formatage de la partition

  handlers:
  - name: Creation de la partition  
    win_partition:
      drive_letter: S
      partition_size: -1
      disk_number: 2

  - name: Formatage de la partition
    win_shell:
     "Format-Volume -DriveLetter S"

- hosts: win
  remote_user: pierrc

  tasks:
  - name: Installation du role "Active Directory"  
    win_feature:
      name: AD-Domain-Services
      include_management_tools: yes
      include_sub_features: yes
      state: present

  - name: Setup de l "Active Directory"
    win_domain:
      create_dns_delegation: no
      database_path: S:\Windows\NTDS
      dns_domain_name: ma-pme.local
      domain_mode: Win2012R2
      domain_netbios_name: MA-PME
      forest_mode: Win2012R2
      safe_mode_password: Password123$
      sysvol_path: S:\Windows\SYSVOL
    register: domain_install
  
  - name: reboot server
    win_reboot:
      pre_reboot_delay: 15
    when: domain_install.reboot_required
...
```
Reste plus qu'a exécuter le playbook !
```
ansible-playbook -i winhosts win.yml
```



