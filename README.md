# VMware ESXi & Ubuntu Server Deployment

[![FR](https://img.shields.io/badge/Language-Français-blue.svg)](README.md)

## Vue d'ensemble

Ce projet démontre la mise en place d'un environnement de virtualisation complet basé sur VMware ESXi (déployé lui-même comme machine virtuelle dans VMware Workstation), avec une machine virtuelle Ubuntu Server. Il illustre des concepts avancés de virtualisation imbriquée (nested virtualization) et propose une solution complète de bout en bout.

## Objectifs du projet

- Déployer VMware ESXi comme machine virtuelle dans VMware Workstation
- Créer et configurer une machine virtuelle Ubuntu Server sur l'hyperviseur ESXi
- Tester et valider l'accès distant à l'environnement via VMware vSphere Client

## Vidéo de démonstration

https://drive.google.com/file/d/1h5Ol_RxB9HMJbpEOhpNxL3YKRLYQ6EVt/view?usp=drive_link

## Documentation

- [Rapport complet du projet (PDF)](Virtual.pdf) - Documentation détaillée avec captures d'écran et explications étape par étape
- [Guide de dépannage](troubleshooting_guide.md) - Solutions aux problèmes courants rencontrés

# Guide d'installation ESXi avec VM Ubuntu

Ce guide détaillé vous accompagne à travers l'ensemble du processus d'installation et de configuration d'un environnement de virtualisation avec VMware ESXi et Ubuntu Server. Il combine les instructions pas à pas avec une documentation visuelle de chaque étape clé du processus.

## Table des matières

1. [Prérequis matériels et logiciels](#prérequis-matériels-et-logiciels)
2. [Préparation de l'environnement hôte](#préparation-de-lenvironnement-hôte)
3. [Création de la VM ESXi dans Workstation](#étape-1-création-de-la-vm-esxi-dans-workstation)
4. [Installation d'ESXi](#étape-2-installation-desxi)
5. [Configuration réseau d'ESXi](#étape-3-configuration-réseau-desxi)
6. [Accès à ESXi via vSphere Client](#étape-4-accès-à-esxi-via-vsphere-client)
7. [Vue d'ensemble de l'interface vSphere](#étape-5-vue-densemble-de-linterface-vsphere)
8. [Création d'un datastore](#étape-6-création-dun-datastore)
9. [Configuration du commutateur virtuel](#étape-7-configuration-du-commutateur-virtuel)
10. [Téléversement de l'ISO Ubuntu](#étape-8-téléversement-de-liso-ubuntu)
11. [Création d'une VM Ubuntu](#étape-9-création-dune-vm-ubuntu)
12. [Installation d'Ubuntu Server](#étape-10-installation-dubuntu-server)
13. [Configuration réseau de la VM Ubuntu](#étape-11-configuration-réseau-de-la-vm-ubuntu)
14. [Test d'accès SSH à la VM Ubuntu](#étape-12-test-daccès-ssh-à-la-vm-ubuntu)
15. [Test du serveur web Apache](#étape-13-test-du-serveur-web-apache)
16. [Configuration détaillée de l'ESXi](#étape-14-configuration-détaillée-de-lesxi)
17. [Monitoring des performances](#étape-15-graphiques-de-performances-de-la-vm-ubuntu)
18. [Calcul de plage d'adressage](#notes-pour-le-calcul-de-plage-dadressage)
19. [Dépannage courant](#dépannage-courant)

## Prérequis matériels et logiciels

Avant de commencer, assurez-vous de disposer des éléments suivants :

- VMware Workstation Pro (version 16 ou supérieure)
- Image ISO VMware ESXi (VMware-ESXi.iso)
- Image ISO Ubuntu Server 22.04 LTS (à télécharger depuis [ubuntu.com](https://ubuntu.com/download/server))
- Ordinateur avec ressources suffisantes :
  - Processeur compatible avec la virtualisation (Intel VT-x ou AMD-V)
  - Minimum 16 Go de RAM recommandé
  - Au moins 200 Go d'espace disque disponible

## Préparation de l'environnement hôte

### 1. Activation de la virtualisation dans le BIOS

1. Redémarrez votre ordinateur et accédez au BIOS (généralement en appuyant sur F2, DEL ou F10 au démarrage)
2. Localisez les paramètres de virtualisation :
   - Pour les processeurs Intel : Intel Virtualization Technology (VT-x)
   - Pour les processeurs AMD : AMD-V ou SVM Mode
3. Activez cette option :
   ```
   BIOS > OC > SVM Mode > Enable
   ```
![Texte alternatif](/Screen-Shots/screenchot-bios.png)

4. Sauvegardez les modifications et redémarrez

### 2. Désactivation de Hyper-V dans Windows

Si vous utilisez Windows, vous devez désactiver Hyper-V pour éviter les conflits avec VMware :

Ouvrez PowerShell en tant qu'administrateur et Exécutez les commandes suivantes :

Disable Hyper-V
```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

Disable Device Guard and Credential Guard
```powershell
REG ADD "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v "EnableVirtualizationBasedSecurity" /t REG_DWORD /d 0 /f
REG ADD "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\LSA" /v "LsaCfgFlags" /t REG_DWORD /d 0 /f
```

Disable Core Isolation Memory Integrity
```powershell
REG ADD "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v "Enabled" /t REG_DWORD /d 0 /f
```

Configure Windows to not use the hypervisor
```powershell
bcdedit /set hypervisorlaunchtype off
```

> **⚠️ Note :** Redemarrer le pc pour effectuer le changement.


### 3. Vérification des adaptateurs réseau

1. Ouvrez une invite de commande
2. Exécutez la commande :
   
   ```
   ipconfig /all
   ```
   
4. Notez les informations suivantes :
   - Adresse IPv4 de votre interface principale
   - Masque de sous-réseau
   - Passerelle par défaut
   - Serveurs DNS

![Texte alternatif](/Screen-Shots/o1.png)

On difini plage d'adressage, pour moi c'est :
```
192.168.1.2 ---------------  192.168.1.254
```
configuration reseau ESXi :
```
192.168.1.64
```
UBuntu Server :
```
192.168.1.65
```
Adresse pasrelle ne change pas pour les deux 5 (ESXi, Ubuntu Server) :
```
192.168.1.1
```
DNS Google :
```
8.8.8.8
8.8.4.4
```

## Étape 1: Création de la VM ESXi dans Workstation

1. Lancez VMware Workstation
2. Allez dans **File** > **New Virtual Machine**
3. Sélectionnez **Custom (advanced)** et cliquez sur **Next**
4. Choisissez la version de compatibilité matérielle la plus récente et cliquez sur **Next**
5. Sélectionnez **I will install the operating system later** et cliquez sur **Next**
6. Pour le système d'exploitation :
   - Guest OS: **VMware ESXi**
   - Version: **ESXi 7.x** (ou la version correspondant à votre ISO)
7. Nommez la VM **ESXi-Lab** et définissez son emplacement, puis cliquez sur **Next**

![Texte alternatif](/Screen-Shots/o.png)

8. Configuration du processeur :
   - Number of processors: **1**
   - Number of cores per processor: **4** (ou plus si disponible)
   - Cliquez sur **Next**

![Texte alternatif](/Screen-Shots/o2.png)

9. Allouez **8 Go** de RAM (8192 MB), puis cliquez sur **Next**

![Texte alternatif](/Screen-Shots/o3.png)

10. Pour le réseau, configurez les adaptateurs :
    - Premier adaptateur : **Use Bridged Network (Bridged)**
    - Cliquez sur **Next**
    - Plus tard, nous ajouterons un second adaptateur en mode Host-only

![Texte alternatif](/Screen-Shots/o4.png)

11. Sélectionnez **Paravirtualized** comme contrôleur SCSI, puis cliquez sur **Next**
12. Sélectionnez **Create a new virtual disk**, puis cliquez sur **Next**
13. Configurez le disque dur :
    - Capacité : **148 GB**
    - Sélectionnez **Store virtual disk as a single file**
    - Cliquez sur **Next**

![Texte alternatif](/Screen-Shots/o5.png)

14. Vérifiez le nom du fichier de disque et cliquez sur **Next**
15. Cliquez sur **Customize Hardware** pour les configurations supplémentaires :
    - Ajoutez une seconde carte réseau en mode **Host-only**
    - Configurez le lecteur CD/DVD pour utiliser l'image ISO d'ESXi
    - Assurez-vous que **Connect at power on** est coché

16. **IMPORTANT** : Ajoutez les options de virtualisation imbriquée :
    - Sélectionnez **Processors**
    - Cochez **Virtualize Intel VT-x/EPT or AMD-V/RVI**
    - Cochez **Virtualize CPU performance counters** (optionnel, peut être désactivé en cas d'erreur)

![Texte alternatif](/Screen-Shots/o6.png)

17. Cliquez sur **Close** pour fermer la fenêtre de personnalisation
18. Vérifiez les paramètres finaux et cliquez sur **Finish**

## Étape 2: Installation d'ESXi

1. Démarrez la VM ESXi-Lab

![Texte alternatif](/Screen-Shots/o7.png)

2. Le programme d'installation démarre automatiquement
3. Appuyez sur **Enter** pour démarrer l'installation
4. Appuyez sur **F11** pour accepter le contrat de licence
5. Sélectionnez le disque pour l'installation (il n'y en a qu'un) et appuyez sur **Enter**

![Texte alternatif](/Screen-Shots/o8.png)

6. Sélectionnez la disposition du clavier et appuyez sur **Enter**
7. Entrez un mot de passe root fort et appuyez sur **Enter**
8. Confirmez le mot de passe et appuyez sur **Enter**

![Texte alternatif](/Screen-Shots/o9.png)

9. Appuyez sur **F11** pour confirmer l'installation

![Texte alternatif](/Screen-Shots/o10.png)

10. Attendez que l'installation se termine (quelques minutes)

![Texte alternatif](/Screen-Shots/o11.png)

## Étape 3: Configuration réseau d'ESXi

1. Une fois l'installation terminée, la VM redémarre automatiquement
2. L'écran principal d'ESXi s'affiche avec l'adresse IP attribuée

![Texte alternatif](/Screen-Shots/o12.png)

> **Commentaire :** les adresse dans le screenshot sont deja configuree, vous aver rencontrer des adresse differentes avec (STATIC) au lieux que (DHCP).

3. Appuyez sur **F2** pour accéder aux options de configuration
4. Connectez-vous avec :
   - User: **root**
   - Password: *le mot de passe que vous avez défini*
5. Sélectionnez **Configure Management Network**

![Texte alternatif](/Screen-Shots/o13.png)

6. Sélectionnez **IPv4 Configuration**
7. Configurez une adresse IP statique :
   - Sélectionnez **Set static IPv4 address and network configuration**
   - IPv4 Address: **192.168.x.x** (adresse compatible avec votre réseau, voir [Préparation de l'environnement hôte](#préparation-de-lenvironnement-hôte) )
   - Subnet Mask: **255.255.255.0**
   - Default Gateway: **192.168.x.x** (passerelle de votre réseau)
   - Sélectionnez **Use dynamic IPv4 address and network configuration** pour fixer l'address configuree

![Texte alternatif](/Screen-Shots/o14.png)

8. Appuyez sur **Enter** pour confirmer
9. Sélectionnez **DNS Configuration**
10. Configurez les DNS :
    - Primary DNS Server: **8.8.8.8**
    - Secondary DNS Server: **8.8.4.4**
    - Hostname: **esxi-lab**

![Texte alternatif](/Screen-Shots/o15.png)

11. Appuyez sur **Enter** pour confirmer
12. Appuyez sur **Esc** pour revenir au menu principal
13. Sélectionnez **Y** lorsqu'on vous demande si vous voulez appliquer les changements et redémarrer le réseau

## Étape 4: Accès à ESXi via vSphere Client

1. Sur votre ordinateur hôte, ouvrez un navigateur web
2. Entrez l'adresse IP de votre ESXi: **https://192.168.x.x** (l'adresse que vous avez configurée)
3. Vous verrez un avertissement concernant le certificat, cliquez sur **Avancé** puis **Continuer vers le site**

4. Vous arriverez à l'écran de connexion du client vSphere HTML5

![Texte alternatif](/Screen-Shots/o16.png)

## Étape 5: Vue d'ensemble de l'interface vSphere

1. Connectez-vous à l'interface vSphere avec :
   - User name: **root**
   - Password: *le mot de passe que vous avez défini*
2. Cliquez sur **Se connecter**
3. Vous accédez à l'interface principale de vSphere

![Texte alternatif](/Screen-Shots/o17.png)

4. Explorez les différents onglets disponibles :
   - **Hôte** : Informations sur l'hôte ESXi
   - **VMs** : Gestion des machines virtuelles
   - **Stockage** : Gestion des datastores
   - **Réseau** : Configuration réseau

![Texte alternatif](/Screen-Shots/o18.png)
![Texte alternatif](/Screen-Shots/o19.png)
![Texte alternatif](/Screen-Shots/o20.png)

## Étape 6: Création d'un datastore

1. Dans l'interface vSphere, cliquez sur l'onglet **Stockage**
2. Cliquez sur **Datastores**
3. Cliquez sur **Nouveau datastore**

![Texte alternatif](/Screen-Shots/o21.png)

4. Sélectionnez **Créer un nouveau datastore VMFS**
5. Cliquez sur **Suivant**
6. Nommez le datastore **datastore1**
7. Sélectionnez le disque disponible
8. Cliquez sur **Suivant**

![Texte alternatif](/Screen-Shots/o22.png)


9. Sélectionnez **VMFS 6** comme version
10. Cliquez sur **Suivant**
11. Utilisez toute la capacité disponible du disque
12. Cliquez sur **Suivant**
13. Vérifiez les paramètres et cliquez sur **Terminer**

## Étape 7: Configuration du commutateur virtuel
> **Commantaire** Si vSwitch n'existe pas si existe skipper cette etape.

1. Dans l'interface vSphere, cliquez sur l'onglet **Réseau**
2. Cliquez sur **Commutateurs virtuels**

![Texte alternatif](/Screen-Shots/o23.png)

3. Cliquez sur **Ajouter un commutateur réseau standard**
4. Configurez le vSwitch :
   - Nom : **vSwitch1**
   - Nombre de ports : **24**

5. Cliquez sur **Ajouter**
6. Sélectionnez le vSwitch créé et cliquez sur **Ajouter un port group**
7. Configurez le port group :
   - Nom : **VM Network**
   - ID VLAN : **0** (ou selon votre configuration réseau)

8. Cliquez sur **OK**

## Étape 8: Téléversement de l'ISO Ubuntu

1. Téléchargez l'image ISO d'Ubuntu Server 22.04 LTS depuis le site officiel [ubuntu.com](https://ubuntu.com/download/server)
2. Dans l'interface vSphere, cliquez sur l'onglet **Stockage**
3. Sélectionnez le datastore créé précédemment
4. Cliquez sur **Parcourir**
5. Cliquez sur **Créer un dossier** et nommez-le **iso**

![Texte alternatif](/Screen-Shots/o24.png)
![Texte alternatif](/Screen-Shots/o25.png)

6. Ouvrez le dossier **iso**
7. Cliquez sur **Upload**
8. Sélectionnez le fichier ISO d'Ubuntu que vous avez téléchargé
9. Attendez que le téléversement soit terminé

![Texte alternatif](/Screen-Shots/o26.png)

## Étape 9: Création d'une VM Ubuntu

1. Dans l'interface vSphere, cliquez sur **Virtual Machines**
2. Cliquez sur **Create / Register VM**

![Texte alternatif](/Screen-Shots/o27.png)

3. Sélectionnez **Create a new virtual machine**
4. Cliquez sur **Next**
5. Configurez la VM :
   - Nom : **Ubuntu-Server**
   - Compatibilité : **ESXi 7.0 virtual machine** (ou votre version d'ESXi)
6. Cliquez sur **Next**
7. Sélectionnez le datastore **datastore1**
8. Cliquez sur **Next**
9. Configurez le système d'exploitation :
   - Guest OS family: **Linux**
   - Guest OS version: **Ubuntu Linux (64-bit)**

![Texte alternatif](/Screen-Shots/o28.png)
![Texte alternatif](/Screen-Shots/o29.png)

10. Cliquez sur **Next**
11. Configurez le matériel virtuel :
    - CPU : **2** vCPU
    - Memory : **4 GB**
    - Hard disk 1: **20 GB**

![Texte alternatif](/Screen-Shots/o30.png)

    - CD/DVD Drive 1: Sélectionnez **Datastore ISO file** et naviguez vers votre ISO Ubuntu

![Texte alternatif](/Screen-Shots/o31.png)

    - Network Adapter 1: Connectez-la au port groupe **VM-Network** deja exister ou que tu as cree

12. Cliquez sur **Next**
13. Vérifiez les paramètres et cliquez sur **Finish**

![Texte alternatif](/Screen-Shots/o32.png)

## Étape 10: Installation d'Ubuntu Server

1. Sélectionnez la VM Ubuntu-Server dans la liste
2. Cliquez sur **Power On**
3. Cliquez sur **Launch Remote Console** ou **Open browser console**
4. L'installation d'Ubuntu démarre

![Texte alternatif](/Screen-Shots/o33.png)

5. Sélectionnez la langue (Français)

![Texte alternatif](/Screen-Shots/o34.png)

6. Configurez la langue et la disposition du clavier
7. Sélectionnez **Installer Ubuntu Server**

![Texte alternatif](/Screen-Shots/o35.png)

8. Configurez la connexion réseau (DHCP par défaut)
   > **Commentaire**: Continuer Sans reseau pour configurer plus tard

![Texte alternatif](/Screen-Shots/o36.png)

9. Configurez le stockage avec les options par défaut

![Texte alternatif](/Screen-Shots/o37.png)

10. Configurez le profil :
    - Nom : **Admin**
    - Nom du serveur : **ubuntu-esxi**
    - Nom d'utilisateur : **admin**
    - Mot de passe : *votre mot de passe*

![Texte alternatif](/Screen-Shots/o38.png)

11. Sélectionnez l'installation du serveur SSH

![Texte alternatif](/Screen-Shots/o39.png)

12. Continuez l'installation jusqu'à la fin

![Texte alternatif](/Screen-Shots/o40.png)

13. Lorsque l'installation est terminée, cliquez sur **Redémarrer maintenant**

## Étape 11: Configuration réseau de la VM Ubuntu

1. Après le redémarrage, connectez-vous à Ubuntu avec l'utilisateur **admin**

2. Mettez à jour le système :
   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

![Texte alternatif](/Screen-Shots/o41.png)

3. Installez les VMware Tools :
   ```bash
   sudo apt install open-vm-tools -y
   ```

![Texte alternatif](/Screen-Shots/o42.png)

4. Configurez une adresse IP statique :
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

5. Modifiez le fichier comme suit :
   ```yaml
   network:
     version: 2
     ethernets:
       ens160:  # Le nom de l'interface peut varier
         dhcp4: no
         addresses: [192.168.x.x/24]
         gateway4: 192.168.x.x
         nameservers:
           addresses: [8.8.8.8, 8.8.4.4]
   ```

![Texte alternatif](/Screen-Shots/o43.png)

6. Sauvegardez avec Ctrl+O, puis Enter, et quittez avec Ctrl+X
7. Appliquez la configuration :
   ```bash
   sudo netplan apply
   ```
![Texte alternatif](/Screen-Shots/o44.png)

8. Vérifiez la configuration :
   ```bash
   ip addr show
   ```

![Texte alternatif](/Screen-Shots/o45.png)

## Étape 12: Test d'accès SSH à la VM Ubuntu

1. Depuis votre ordinateur hôte, ouvrez un terminal
2. Connectez-vous en SSH à la VM Ubuntu :
   ```bash
   ssh admin@192.168.x.x
   ```
3. Entrez le mot de passe lorsqu'il est demandé
4. Une fois connecté, vous devriez voir l'invite de commande Ubuntu

![Texte alternatif](/Screen-Shots/o46.png)

5. Exécutez quelques commandes basiques pour vérifier le système :
   ```bash
   hostname
   uname -a
   ip addr show
   df -h
   ```

![Texte alternatif](/Screen-Shots/o47.png)

## Étape 13: Test du serveur web Apache

1. Dans la session SSH, installez Apache :
   ```bash
   sudo apt install apache2 -y
   ```

2. Vérifiez l'état du service :
   ```bash
   sudo systemctl status apache2
   ```

![Texte alternatif](/Screen-Shots/o48.png)

3. Créez une page web de test :
   ```bash
   echo '<html><body><h1>Test de VM Ubuntu sur ESXi réussi!</h1></body></html>' | sudo tee /var/www/html/index.html
   ```

![Texte alternatif](/Screen-Shots/o49.png)

4. Sur votre ordinateur hôte, ouvrez un navigateur web
5. Accédez à l'adresse IP de la VM : **http://192.168.x.x**
6. Vous devriez voir la page web de test

![Texte alternatif](/Screen-Shots/o50.png)
## AND
<p align="center">
    <img src="/Screen-Shots/done.png" alt="icon" width="250" style="vertical-align:middle; border:0;">
</p>


## Notes pour le calcul de plage d'adressage

Pour configurer correctement les adresses IP statiques, suivez cette méthode à partir de la sortie `ipconfig /all` :

1. Identifiez l'interface réseau bridged :
   - Notez l'adresse IP (ex : 192.168.1.100)
   - Notez le masque de sous-réseau (ex : 255.255.255.0)
   - Notez la passerelle par défaut (ex : 192.168.1.1)

2. Pour l'ESXi, choisissez une adresse IP dans le même sous-réseau que votre interface bridged :
   - Si votre PC est sur 192.168.1.100, vous pourriez utiliser 192.168.1.200 pour l'ESXi
   - Utilisez le même masque (255.255.255.0)
   - Utilisez la même passerelle (192.168.1.1)

3. Pour la VM Ubuntu, choisissez une autre adresse IP disponible :
   - Par exemple : 192.168.1.201
   - Utilisez le même masque et la même passerelle

4. Pour l'interface host-only :
   - Identifiez le sous-réseau de l'adaptateur VMware Host-Only (typiquement 192.168.*.*)
   - Configurez une adresse IP dans ce sous-réseau si nécessaire

### Exemple de calcul

Si votre PC a l'adresse 192.168.1.50/24 :
- Masque = 255.255.255.0
- Réseau = 192.168.1.0
- Broadcast = 192.168.1.255
- Plage utilisable = 192.168.1.1 à 192.168.1.254

Attribution recommandée :
- ESXi = 192.168.1.200
- VM Ubuntu = 192.168.1.201
- Passerelle = 192.168.1.1 (routeur/box internet)

## Dépannage courant

### Problèmes de virtualisation imbriquée

Si vous rencontrez des erreurs au démarrage d'ESXi :
1. Vérifiez que la virtualisation est activée dans le BIOS
2. Assurez-vous que l'option "Virtualize Intel VT-x/EPT or AMD-V/RVI" est cochée dans les paramètres de la VM
3. Si l'erreur persiste avec "Virtualize CPU performance counters", décochez cette option
4. Désactivez tous les hyperviseurs concurrents (Hyper-V, WSL2, etc.)

### Problèmes de connectivité réseau

Si l'ESXi ou la VM Ubuntu n'a pas de connectivité réseau :
1. Vérifiez que le mode réseau de l'adaptateur est correctement configuré (bridged pour l'accès externe)
2. Contrôlez que les adresses IP statiques sont dans le bon sous-réseau
3. Testez avec `ping` pour vérifier la connectivité de base
4. Vérifiez les paramètres du pare-feu de l'ESXi
5. Commandes de diagnostic :
   ```bash
   # Sur l'hôte Windows
   ipconfig /all
   ping 192.168.x.x  # adresse de l'ESXi ou de la VM

   # Sur la VM Ubuntu
   ip addr show
   ip route show
   ping 192.168.x.x  # adresse de l'hôte ou de la passerelle
   ping 8.8.8.8      # test de connectivité Internet
