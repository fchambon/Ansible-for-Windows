# Quick Start Ansible pour Windows Server dans azure !

Voici un "quickstart" pour gérer les configurations des serveurs "Windows Server 2016/2019" dans Azure<br/>
Dans ce quickstart la communication entre la machine de controle Ansible et les serveurs se fera en SSH avec une authentification par cle publique<br/>
Pour les machines cibles "Windows Server" il faudra installer un serveur OpenSSH pour Windows.<br/>
Pour Windows Server 2019 c'est une nouvelle "feature". Ex: https://github.com/Pierre-Chesne/Windows-Server-2019-OpenSSH <br/>
Pour Windows Server 2016 

**Prerequis pour la machine de controle :**<br/>
Pour une distro Ubuntu 16.04 LTS : <br/>
1/ Installation Ansible <br/>
- Installation des packages requis pour les modules Azure Python SDK: <br/>
```
sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
```
- Installation packages Ansible pour azure:
```
sudo pip install ansible[azure]
```
- Installation Azure Cli <br/>
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
ip pou url

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