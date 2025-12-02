# Guide Complet : Commandes pour Dump de RAM

## Table des matières

1. [Préparation du système](#1-préparation-du-système)
2. [Installation de LiME](#2-installation-de-lime)
3. [Acquisition de la mémoire](#3-acquisition-de-la-mémoire)
4. [Méthodes alternatives](#4-méthodes-alternatives)
5. [Vérification et intégrité](#5-vérification-et-intégrité)
6. [Installation de Volatility](#6-installation-de-volatility)
7. [Analyse avec Volatility 3](#7-analyse-avec-volatility-3)
8. [Analyse avancée](#8-analyse-avancée)
9. [Extraction de données](#9-extraction-de-données)
10. [Troubleshooting](#10-troubleshooting)
11. [**COMMANDES RÉELLEMENT EXÉCUTÉES**](#11-commandes-réellement-exécutées) ⭐

---

## 1. Préparation du système

### 1.1 Vérifier les informations système

```bash
# Version du kernel
uname -r

# Architecture
uname -m

# Informations complètes
uname -a

# Mémoire RAM totale
free -h
cat /proc/meminfo | grep MemTotal

# Espace disque disponible
df -h /tmp
```

### 1.2 Installer les dépendances

```bash
# Mise à jour du système
sudo apt update

# Installation des outils de compilation
sudo apt install -y build-essential linux-headers-$(uname -r)

# Installation de git
sudo apt install -y git

# Vérifier l'installation des headers
ls /lib/modules/$(uname -r)/build
```

### 1.3 Créer un répertoire de travail

```bash
# Créer un dossier pour le TP
mkdir -p ~/forensics/ram_dump
cd ~/forensics/ram_dump

# Créer un dossier pour les résultats
mkdir -p ~/forensics/ram_dump/results
mkdir -p ~/forensics/ram_dump/analysis
```

---

## 2. Installation de LiME

### 2.1 Télécharger LiME

```bash
# Cloner le dépôt officiel
cd ~/forensics/ram_dump
git clone https://github.com/504ensicsLabs/LiME.git

# Entrer dans le répertoire source
cd LiME/src
```

### 2.2 Compiler LiME

```bash
# Compiler le module kernel
make

# Vérifier la création du module
ls -lh lime-*.ko

# Afficher les informations du module
modinfo lime-*.ko
```

**Sortie attendue :**
```
filename:       lime-6.16.8-kali-amd64.ko
license:        GPL
description:    LiME - Linux Memory Extractor
author:         Joe Sylve
```

### 2.3 Dépannage de compilation

Si la compilation échoue :

```bash
# Vérifier les headers kernel
dpkg -l | grep linux-headers

# Installer les headers correspondants
sudo apt install linux-headers-$(uname -r)

# Nettoyer et recompiler
make clean
make
```

---

## 3. Acquisition de la mémoire

### 3.1 Méthode 1 : Format LiME (Recommandé)

```bash
# Se placer dans le dossier de LiME compilé
cd ~/forensics/ram_dump/LiME/src

# Effectuer le dump au format LiME
sudo insmod lime-*.ko "path=~/forensics/ram_dump/results/ram_dump.lime format=lime"

# Vérifier que le module s'est correctement déchargé
lsmod | grep lime
```

**Options du format :**
- `format=lime` : Format LiME (recommandé pour Volatility)
- `format=raw` : Format brut
- `format=padded` : Format avec padding

### 3.2 Méthode 2 : Format RAW

```bash
# Dump au format RAW (brut)
sudo insmod lime-*.ko "path=~/forensics/ram_dump/results/ram_dump.raw format=raw"
```

### 3.3 Méthode 3 : Dump via réseau (pour systèmes distants)

```bash
# Sur la machine cible
sudo insmod lime-*.ko "path=tcp:4444 format=lime"

# Sur la machine d'analyse
nc -l -p 4444 > ram_dump.lime
```

### 3.4 Vérification de l'acquisition

```bash
# Vérifier la taille du fichier
ls -lh ~/forensics/ram_dump/results/ram_dump.*

# Afficher les détails
stat ~/forensics/ram_dump/results/ram_dump.lime

# Vérifier les permissions
file ~/forensics/ram_dump/results/ram_dump.lime
```

---

## 4. Méthodes alternatives

### 4.1 Utilisation de /proc/kcore (méthode native)

⚠️ **Attention** : Méthode moins fiable que LiME

```bash
# Copier /proc/kcore
sudo dd if=/proc/kcore of=~/forensics/ram_dump/results/kcore_dump.raw bs=1M

# Compresser pour économiser l'espace
sudo dd if=/proc/kcore bs=1M | gzip > ~/forensics/ram_dump/results/kcore_dump.raw.gz
```

### 4.2 Utilisation d'AVML (Microsoft)

```bash
# Télécharger AVML
cd ~/forensics/ram_dump
wget https://github.com/microsoft/avml/releases/download/v0.13.0/avml

# Rendre exécutable
chmod +x avml

# Effectuer le dump
sudo ./avml ~/forensics/ram_dump/results/ram_avml.lime
```

### 4.3 Utilisation de fmem (alternative à LiME)

```bash
# Installer fmem
cd ~/forensics/ram_dump
git clone https://github.com/NateBrune/fmem.git
cd fmem
make

# Charger le module
sudo insmod fmem.ko

# Effectuer le dump
sudo dd if=/dev/fmem of=~/forensics/ram_dump/results/fmem_dump.raw bs=1M
```

---

## 5. Vérification et intégrité

### 5.1 Calculer les hashes

```bash
# MD5
md5sum ~/forensics/ram_dump/results/ram_dump.lime > ~/forensics/ram_dump/results/ram_dump.md5

# SHA1
sha1sum ~/forensics/ram_dump/results/ram_dump.lime > ~/forensics/ram_dump/results/ram_dump.sha1

# SHA256
sha256sum ~/forensics/ram_dump/results/ram_dump.lime > ~/forensics/ram_dump/results/ram_dump.sha256

# Afficher tous les hashes
cat ~/forensics/ram_dump/results/ram_dump.md5
cat ~/forensics/ram_dump/results/ram_dump.sha1
cat ~/forensics/ram_dump/results/ram_dump.sha256
```

### 5.2 Documenter l'acquisition

```bash
# Créer un fichier de métadonnées
cat > ~/forensics/ram_dump/results/acquisition_metadata.txt <<EOF
=== MÉTADONNÉES D'ACQUISITION MÉMOIRE ===
Date et heure: $(date)
Hostname: $(hostname)
Utilisateur: $(whoami)
Kernel: $(uname -r)
Architecture: $(uname -m)
RAM totale: $(free -h | grep Mem | awk '{print $2}')
Méthode: LiME kernel module
Format: LiME format
Fichier: ram_dump.lime
Taille: $(ls -lh ~/forensics/ram_dump/results/ram_dump.lime | awk '{print $5}')
MD5: $(cat ~/forensics/ram_dump/results/ram_dump.md5)
SHA256: $(cat ~/forensics/ram_dump/results/ram_dump.sha256)
EOF

# Afficher les métadonnées
cat ~/forensics/ram_dump/results/acquisition_metadata.txt
```

### 5.3 Protéger le dump

```bash
# Passer en lecture seule
chmod 444 ~/forensics/ram_dump/results/ram_dump.lime

# Créer une copie de travail
cp ~/forensics/ram_dump/results/ram_dump.lime ~/forensics/ram_dump/analysis/ram_dump_work.lime
```

---

## 6. Installation de Volatility

### 6.1 Volatility 3 (Recommandé)

```bash
# Installation via pip
pip3 install volatility3

# Vérifier l'installation
vol3 --help
volatility3 --version

# Alternative : Installation depuis les sources
cd ~/forensics/ram_dump
git clone https://github.com/volatilityfoundation/volatility3.git
cd volatility3
pip3 install -r requirements.txt
python3 vol.py --help
```

### 6.2 Volatility 2 (Legacy)

```bash
# Installation de Volatility 2
cd ~/forensics/ram_dump
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility
pip2 install pycrypto distorm3

# Tester
python2 vol.py --help
```

### 6.3 Vérifier les plugins disponibles

```bash
# Lister tous les plugins Linux de Volatility 3
vol3 --help | grep linux

# Ou de manière plus détaillée
volatility3 --help | grep "linux\." | sort
```

---

## 7. Analyse avec Volatility 3

### 7.1 Informations système

```bash
# Banner du système
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime banners.Banners

# Informations kernel
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.check_syscall.Check_syscall
```

### 7.2 Analyse des processus

```bash
# Liste des processus
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.pslist.PsList

# Arbre des processus (avec relations parent/enfant)
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.pstree.PsTree

# Processus avec toutes les informations
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.psaux.PsAux

# Sauvegarder la liste
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.pslist.PsList > ~/forensics/ram_dump/analysis/processes.txt
```

### 7.3 Analyse réseau

```bash
# Connexions réseau actives
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.netstat.Netstat

# Sauvegarder les connexions
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.netstat.Netstat > ~/forensics/ram_dump/analysis/network_connections.txt

# Interfaces réseau
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.ifconfig.Ifconfig
```

### 7.4 Modules kernel

```bash
# Lister les modules chargés
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.lsmod.Lsmod

# Sauvegarder
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.lsmod.Lsmod > ~/forensics/ram_dump/analysis/kernel_modules.txt
```

### 7.5 Fichiers ouverts

```bash
# Lister tous les fichiers ouverts
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.lsof.Lsof

# Sauvegarder
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.lsof.Lsof > ~/forensics/ram_dump/analysis/open_files.txt
```

---

## 8. Analyse avancée

### 8.1 Recherche de malwares

```bash
# Malfind - Recherche de code injecté
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.malfind.Malfind

# Sauvegarder avec les détails
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.malfind.Malfind > ~/forensics/ram_dump/analysis/malfind_results.txt
```

### 8.2 Analyse de la mémoire d'un processus spécifique

```bash
# Remplacer [PID] par le PID du processus suspect
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime -o ~/forensics/ram_dump/analysis/ linux.proc.Maps --pid [PID]

# Voir les sections mémoire
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.proc.Maps --pid [PID]
```

### 8.3 Lignes de commande des processus

```bash
# Récupérer les lignes de commande
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.pslist.PsList > ~/forensics/ram_dump/analysis/cmdlines.txt
```

### 8.4 Bash history en mémoire

```bash
# Rechercher l'historique bash
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.bash.Bash

# Sauvegarder
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.bash.Bash > ~/forensics/ram_dump/analysis/bash_history.txt
```

### 8.5 Variables d'environnement

```bash
# Extraire les variables d'environnement de tous les processus
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.envars.Envars

# Pour un processus spécifique
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.envars.Envars --pid [PID]
```

---

## 9. Extraction de données

### 9.1 Dumper un processus complet

```bash
# Créer un dossier pour les dumps
mkdir -p ~/forensics/ram_dump/analysis/procdump

# Dumper le processus (remplacer [PID])
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime -o ~/forensics/ram_dump/analysis/procdump/ linux.procdump.ProcDump --pid [PID]
```

### 9.2 Extraire des fichiers de la mémoire

```bash
# Scanner les fichiers en mémoire
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.filescan.FileScan

# Sauvegarder la liste
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.filescan.FileScan > ~/forensics/ram_dump/analysis/files_in_memory.txt
```

### 9.3 Recherche de chaînes suspectes

```bash
# Extraire toutes les chaînes ASCII
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime > ~/forensics/ram_dump/analysis/all_strings.txt

# Rechercher des patterns spécifiques
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -i "password"
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -i "http://"
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}"

# Rechercher des URLs
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -E "https?://" > ~/forensics/ram_dump/analysis/urls.txt
```

### 9.4 Recherche d'adresses IP

```bash
# Extraire toutes les adresses IP
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort -u > ~/forensics/ram_dump/analysis/ip_addresses.txt

# Compter les occurrences
strings ~/forensics/ram_dump/analysis/ram_dump_work.lime | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort | uniq -c | sort -rn
```

---

## 10. Troubleshooting

### 10.1 Erreurs courantes lors de la compilation de LiME

**Erreur : "Kernel headers not found"**
```bash
# Solution
sudo apt install linux-headers-$(uname -r)
```

**Erreur : "Module failed to load"**
```bash
# Vérifier les logs kernel
dmesg | tail -20

# Vérifier les modules chargés
lsmod | grep lime

# Retirer le module si nécessaire
sudo rmmod lime
```

### 10.2 Problèmes d'espace disque

```bash
# Vérifier l'espace disponible
df -h

# Compresser le dump
gzip ~/forensics/ram_dump/results/ram_dump.lime

# Décompresser pour analyse
gunzip ~/forensics/ram_dump/results/ram_dump.lime.gz
```

### 10.3 Volatility ne reconnaît pas le format

```bash
# Vérifier le format du dump
file ~/forensics/ram_dump/analysis/ram_dump_work.lime

# Essayer avec un autre plugin
volatility3 -f ~/forensics/ram_dump/analysis/ram_dump_work.lime banners.Banners

# Vérifier les logs de Volatility
volatility3 -vv -f ~/forensics/ram_dump/analysis/ram_dump_work.lime linux.pslist.PsList
```

### 10.4 Dump trop volumineux

```bash
# Créer un dump partiel (streaming)
sudo insmod lime-*.ko "path=tcp:4444 format=lime" &
nc localhost 4444 | head -c 1G > partial_dump.lime

# Ou compresser à la volée
sudo insmod lime-*.ko "path=/dev/stdout format=lime" | gzip > ram_dump.lime.gz
```

---

## 11. Script d'acquisition automatisé

Créer un script pour automatiser l'acquisition :

```bash
#!/bin/bash
# Script : ram_dump_acquisition.sh

# Variables
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_DIR="$HOME/forensics/ram_dump_${TIMESTAMP}"
LIME_DIR="$HOME/forensics/LiME/src"

# Créer les répertoires
mkdir -p "${OUTPUT_DIR}"

# Fonction de log
log() {
    echo "[$(date +%H:%M:%S)] $1" | tee -a "${OUTPUT_DIR}/acquisition.log"
}

log "=== Début de l'acquisition mémoire ==="

# Informations système
log "Récupération des informations système..."
uname -a > "${OUTPUT_DIR}/system_info.txt"
free -h >> "${OUTPUT_DIR}/system_info.txt"
cat /proc/meminfo >> "${OUTPUT_DIR}/system_info.txt"

# Acquisition avec LiME
log "Insertion du module LiME..."
cd "${LIME_DIR}" || exit 1
sudo insmod lime-*.ko "path=${OUTPUT_DIR}/ram_dump.lime format=lime"

if [ $? -eq 0 ]; then
    log "Acquisition réussie !"
else
    log "ERREUR lors de l'acquisition"
    exit 1
fi

# Calcul des hashes
log "Calcul des hashes d'intégrité..."
md5sum "${OUTPUT_DIR}/ram_dump.lime" > "${OUTPUT_DIR}/ram_dump.md5"
sha256sum "${OUTPUT_DIR}/ram_dump.lime" > "${OUTPUT_DIR}/ram_dump.sha256"

# Vérification
SIZE=$(ls -lh "${OUTPUT_DIR}/ram_dump.lime" | awk '{print $5}')
log "Taille du dump : ${SIZE}"
log "MD5 : $(cat ${OUTPUT_DIR}/ram_dump.md5)"

# Protéger le fichier
chmod 444 "${OUTPUT_DIR}/ram_dump.lime"

log "=== Acquisition terminée ==="
log "Fichiers disponibles dans : ${OUTPUT_DIR}"
```

### Utilisation du script

```bash
# Créer le script
nano ~/forensics/ram_dump_acquisition.sh

# Coller le contenu ci-dessus, puis sauvegarder

# Rendre exécutable
chmod +x ~/forensics/ram_dump_acquisition.sh

# Exécuter
sudo ~/forensics/ram_dump_acquisition.sh
```

---

## 12. Checklist complète

### Avant l'acquisition
- [ ] Vérifier l'espace disque disponible
- [ ] Installer les dépendances (build-essential, kernel headers)
- [ ] Compiler LiME
- [ ] Préparer les répertoires de destination

### Pendant l'acquisition
- [ ] Noter l'heure de début
- [ ] Minimiser l'activité sur le système
- [ ] Surveiller les erreurs dans dmesg
- [ ] Vérifier la progression

### Après l'acquisition
- [ ] Calculer les hashes (MD5, SHA256)
- [ ] Documenter les métadonnées
- [ ] Protéger le fichier en lecture seule
- [ ] Créer une copie de travail
- [ ] Vérifier l'intégrité avec Volatility

### Analyse
- [ ] Lister les processus
- [ ] Examiner les connexions réseau
- [ ] Vérifier les modules kernel
- [ ] Rechercher des malwares
- [ ] Extraire les artefacts importants
- [ ] Documenter les findings

---

## 13. Ressources et références

### Documentation officielle
- **LiME GitHub** : https://github.com/504ensicsLabs/LiME
- **Volatility 3 Docs** : https://volatility3.readthedocs.io/
- **SANS Forensics** : https://www.sans.org/blog/

### Commandes de référence rapide

```bash
# Acquisition
sudo insmod lime-*.ko "path=/tmp/ram.lime format=lime"

# Analyse basique
volatility3 -f ram.lime linux.pslist.PsList
volatility3 -f ram.lime linux.netstat.Netstat
volatility3 -f ram.lime linux.lsmod.Lsmod

# Hash
md5sum ram.lime > ram.md5
sha256sum ram.lime > ram.sha256
```

---

## 11. COMMANDES RÉELLEMENT EXÉCUTÉES

**⭐ Section ajoutée : 2 décembre 2025**

Cette section documente les commandes qui ont été réellement exécutées lors de notre session d'acquisition et d'analyse de la RAM.

### 11.1 Préparation initiale

```bash
# Vérification du système
uname -r
# Résultat: 6.16.8+kali-amd64

free -h
# Résultat: 23 Go de RAM totale, ~12 Go utilisés

df -h /tmp
# Résultat: 12 Go disponibles dans /tmp
```

### 11.2 Compilation de LiME

```bash
# Vérification de LiME (déjà cloné)
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/LiME/src
ls -l lime-6.16.8+kali-amd64.ko
# Module déjà compilé

# Vérification du module
modinfo lime-6.16.8+kali-amd64.ko
# license:        GPL
# description:    LiME - Linux Memory Extractor
# author:         Joe Sylve
```

### 11.3 Acquisition de la RAM

```bash
# Dump de la RAM avec LiME
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/LiME/src
sudo insmod lime-6.16.8+kali-amd64.ko path=/tmp/ram_dump.lime format=lime

# Vérification du dump
ls -lh /tmp/ram_dump.lime
# -r--r--r-- 1 root root 12G  2 déc.  11:10 /tmp/ram_dump.lime

# Déplacement du fichier
sudo mv /tmp/ram_dump.lime /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/
sudo chown nyx:nyx /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.lime
```

**Note** : Le module LiME se décharge automatiquement après l'acquisition.

### 11.4 Calcul des hashes

```bash
# MD5
md5sum /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.lime \
  > /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.md5

# Résultat: ba94611b0be2a89117a3033ea12063f6

# SHA1
sha1sum /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.lime \
  > /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.sha1

# Résultat: 29092d21b39f59e155f44979433683d2a66a0705

# SHA256
sha256sum /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.lime \
  > /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.sha256

# Résultat: 481ad3a47fb48aace25f58618a8d056904620f867ef71a1a3b3bc734cd2b7626
```

### 11.5 Installation de Volatility 3

```bash
# Clone du dépôt Volatility 3
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM
git clone https://github.com/volatilityfoundation/volatility3.git

# Test de Volatility
cd volatility3
python3 vol.py --help
# Volatility 3 Framework 2.27.0
```

### 11.6 Génération des symboles du kernel

```bash
# Vérification de dwarf2json
which dwarf2json
# /usr/bin/dwarf2json

# Installation du package de debug du kernel
sudo apt install -y linux-image-6.16.8+kali-amd64-dbg
# Taille du téléchargement: 1 101 MB
# Espace nécessaire: 7 384 MB

# Vérification du vmlinux
ls -lh /usr/lib/debug/boot/vmlinux-6.16.8+kali-amd64
# -rw-r--r-- 1 root root 366M 24 sept. 18:38 ...

# Création du répertoire des symboles
mkdir -p /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/volatility3/volatility3/symbols/linux

# Génération du fichier ISF
dwarf2json linux \
  --elf /usr/lib/debug/boot/vmlinux-6.16.8+kali-amd64 \
  --system-map /boot/System.map-6.16.8+kali-amd64 \
  | xz -c > /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/volatility3/volatility3/symbols/linux/6.16.8-kali-amd64.json.xz

# Vérification du fichier généré
ls -lh volatility3/volatility3/symbols/linux/
# -rw-r--r-- 1 root root 2,7M  2 déc.  11:20 6.16.8-kali-amd64.json.xz
```

### 11.7 Analyses avec Volatility 3

```bash
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/volatility3

# Extraction des banners système
python3 vol.py -f ../results/ram_dump.lime banners.Banners \
  | tee ../results/analysis_banners.txt

# Résultat:
# Linux version 6.16.8+kali-amd64 (devel@kali.org)
# x86_64-linux-gnu-gcc-14 (Debian 14.3.0-8) 14.3.0
# GNU ld (GNU Binutils for Debian) 2.45
# #1 SMP PREEMPT_DYNAMIC Kali 6.16.8-1kali1 (2025-09-24)

# Tentative d'extraction des modules kernel
python3 vol.py -f ../results/ram_dump.lime linux.lsmod.Lsmod \
  > ../results/analysis_lsmod.txt 2>&1

# Tentative d'extraction des statistiques réseau
python3 vol.py -f ../results/ram_dump.lime linux.sockstat.Sockstat \
  > ../results/analysis_sockstat.txt 2>&1

# Tentative d'extraction de l'historique bash
python3 vol.py -f ../results/ram_dump.lime linux.bash.Bash \
  > ../results/analysis_bash_history.txt 2>&1
```

### 11.8 Vérification des fichiers générés

```bash
# Liste des fichiers résultats
ls -lh /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/

# Sortie:
# -rw-r--r--  481  acquisition_info.txt
# -rw-rw-r--  346  acquisition_metadata.txt
# -rw-r--r--  663  analysis_banners.txt
# -rw-r--r--  71K  analysis_bash_history.txt
# -rw-r--r--  71K  analysis_lsmod.txt
# -rw-r--r--  71K  analysis_sockstat.txt
# -rw-rw-r--  41K  processus_avant_dump.txt
# -r--r--r--  12G  ram_dump.lime
# -rw-r--r--  107  ram_dump.md5
# -rw-r--r--  115  ram_dump.sha1
# -rw-r--r--  139  ram_dump.sha256
```

### 11.9 Résumé de l'exécution

| Étape | Statut | Durée approx. | Notes |
|-------|--------|---------------|-------|
| Compilation LiME | ✅ Déjà fait | - | Module pré-compilé |
| Acquisition RAM | ✅ Réussi | ~2 minutes | 12 Go capturés |
| Calcul MD5 | ✅ Réussi | ~3 minutes | Hash généré |
| Calcul SHA256 | ✅ Réussi | ~3 minutes | Hash généré |
| Install. Volatility | ✅ Réussi | ~1 minute | Clone GitHub |
| Install. debug kernel | ✅ Réussi | ~3 minutes | 1,1 Go téléchargé |
| Génération symboles | ✅ Réussi | ~2 minutes | ISF 2,7 Mo |
| Analyse banners | ✅ Réussi | ~1 minute | Info kernel extraites |
| Analyses avancées | ⚠️ Partiel | ~5 minutes | Problèmes compatibilité |

**Total estimé** : ~20 minutes (hors téléchargements)

### 11.10 Problèmes rencontrés et solutions

#### Problème 1 : Erreur de paramètre LiME

```bash
# ❌ Commande qui a échoué:
sudo insmod lime-6.16.8+kali-amd64.ko \
  "path=/home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.lime format=lime"

# Erreur: Invalid parameters

# ✅ Solution : Utiliser un chemin sans espaces ni caractères spéciaux
sudo insmod lime-6.16.8+kali-amd64.ko path=/tmp/ram_dump.lime format=lime
```

#### Problème 2 : Volatility - Symboles non reconnus

```bash
# ❌ Erreur:
# Unsatisfied requirement plugins.PsList.kernel.symbol_table_name

# ⚠️ Cause probable:
# - Incompatibilité entre kernel 6.16.x et Volatility 3.2.7
# - Format ISF potentiellement incorrect pour ce kernel récent

# 💡 Solutions possibles:
# 1. Mettre à jour Volatility vers version développement
# 2. Utiliser Rekall comme alternative
# 3. Analyser manuellement avec gdb/objdump
# 4. Attendre une mise à jour de Volatility supportant kernel 6.16+
```

#### Problème 3 : Plugin netstat introuvable

```bash
# ❌ Erreur:
# invalid choice linux.netstat.Netstat

# ✅ Solution : Utiliser le bon nom de plugin
python3 vol.py -f ram_dump.lime linux.sockstat.Sockstat
```

### 11.11 Commandes de vérification post-analyse

```bash
# Vérifier l'intégrité du dump original
md5sum -c /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.md5
# ram_dump.lime: OK

sha256sum -c /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/ram_dump.sha256
# ram_dump.lime: OK

# Vérifier que le module LiME n'est plus chargé
lsmod | grep lime
# (aucun résultat - module déchargé automatiquement)

# Vérifier la taille totale des fichiers générés
du -sh /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/
# 12G
```

### 11.12 Commandes pour analyse future

Voici des commandes à essayer pour des analyses plus approfondies une fois les problèmes de symboles résolus :

```bash
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/volatility3

# Liste complète des processus
python3 vol.py -f ../results/ram_dump.lime linux.pslist.PsList

# Arbre des processus
python3 vol.py -f ../results/ram_dump.lime linux.pstree.PsTree

# Recherche de malware
python3 vol.py -f ../results/ram_dump.lime linux.malfind.Malfind

# Fichiers ouverts
python3 vol.py -f ../results/ram_dump.lime linux.lsof.Lsof

# Variables d'environnement
python3 vol.py -f ../results/ram_dump.lime linux.envars.Envars

# Extraction d'un processus spécifique (remplacer PID)
python3 vol.py -f ../results/ram_dump.lime -o ../results/ \
  linux.procdump.ProcDump --pid [PID]

# Recherche de strings (URLs, IPs, etc.)
strings ../results/ram_dump.lime | grep -E "https?://" > ../results/urls.txt
strings ../results/ram_dump.lime | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" \
  | sort -u > ../results/ip_addresses.txt
```

---

**Document technique créé pour le TP de forensique numérique**

**Dernière mise à jour** : 2 décembre 2025, 11:25 CET
