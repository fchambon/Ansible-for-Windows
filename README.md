# Quick Start Ansible pour Windows Server !

**Prerequis pour la machine de controle :**<br/>
Pour une distro Ubuntu 16.04 LTS : <br/>
1/ Installation Ansible <br/>
- Packages requis pour les modules Azure Python SDK: <br/>
```
sudo apt-get update && sudo apt-get install -y libssl-dev libffi-dev python-dev python-pip
```
- Packages Ansible pour azure:
```
sudo pip install ansible[azure]
```
- Azure Cli <br/>
```
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```
- S'authentifier <br/>
```
az login
```
- Module Python pour Windows <br/>
```
sudo pip install pywinrm
```
