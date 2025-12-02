# Guide Pratique : Copie Bit-à-Bit de Disque - Commandes et Tutoriel

## Table des matières

1. [Introduction](#1-introduction)
2. [Préparation du système](#2-préparation-du-système)
3. [Identification de la cible](#3-identification-de-la-cible)
4. [Méthodes d'acquisition](#4-méthodes-dacquisition)
5. [Calcul et vérification des hashes](#5-calcul-et-vérification-des-hashes)
6. [Analyse de l'image](#6-analyse-de-limage)
7. [Outils avancés](#7-outils-avancés)
8. [Troubleshooting](#8-troubleshooting)
9. [**COMMANDES EXÉCUTÉES**](#9-commandes-exécutées) ⭐

---

## 1. Introduction

### 1.1 Qu'est-ce qu'une copie bit-à-bit ?

Une copie bit-à-bit crée une réplique exacte d'un support de stockage, octet par octet. Contrairement à `cp` ou `rsync` qui copient seulement les fichiers, `dd` copie :

- ✅ Tous les secteurs (utilisés et non utilisés)
- ✅ Les fichiers supprimés
- ✅ Le slack space
- ✅ Les métadonnées du système de fichiers
- ✅ Les tables de partitions

### 1.2 Cas d'usage

- **Forensique** : Analyse post-incident
- **Backup** : Sauvegarde complète
- **Migration** : Clonage de disque
- **Récupération** : Données endommagées

---

## 2. Préparation du système

### 2.1 Vérifier les outils disponibles

```bash
# dd (toujours disponible sur Linux)
which dd
dd --version

# dcfldd (version forensique avec hash)
which dcfldd
# Si absent : sudo apt install dcfldd

# Outils de hash
which md5sum sha1sum sha256sum

# Sleuth Kit (optionnel, pour analyse)
which mmls fls icat
# Si absent : sudo apt install sleuthkit
```

### 2.2 Vérifier les disques disponibles

```bash
# Lister tous les disques
lsblk

# Plus de détails
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,LABEL,UUID

# Informations sur un disque spécifique
sudo fdisk -l /dev/sda

# Voir les partitions montées
mount | grep sda
df -h
```

### 2.3 Vérifier l'espace disponible

```bash
# Espace disque disponible
df -h

# Taille de la partition à copier
sudo blockdev --getsize64 /dev/sda1
# Résultat en bytes

# Conversion en Mo/Go
sudo blockdev --getsize64 /dev/sda1 | awk '{print $1/1024/1024 " Mo"}'
sudo blockdev --getsize64 /dev/sda1 | awk '{print $1/1024/1024/1024 " Go"}'
```

### 2.4 Créer la structure de dossiers

```bash
# Créer les répertoires
mkdir -p ~/forensics/copie_disque/{images,hashes,logs}

# Vérifier
tree ~/forensics/copie_disque/
# ou
ls -R ~/forensics/copie_disque/
```

---

## 3. Identification de la cible

### 3.1 Lister les disques et partitions

```bash
# Vue d'ensemble
lsblk

# Détails avec UUID
sudo blkid

# Informations filesystem
sudo file -s /dev/sda1
```

### 3.2 Obtenir les informations de la partition

```bash
# Taille exacte en bytes
sudo blockdev --getsize64 /dev/sda1

# Nombre de secteurs
sudo blockdev --getsz /dev/sda1

# Taille de secteur
sudo blockdev --getss /dev/sda1

# Informations complètes
sudo parted /dev/sda print
```

### 3.3 Vérifier l'état de montage

```bash
# Vérifier si la partition est montée
mount | grep sda1

# Voir qui l'utilise (si montée)
sudo lsof /dev/sda1

# Processus utilisant le point de montage
sudo fuser -v /boot/efi
```

---

## 4. Méthodes d'acquisition

### 4.1 Méthode 1 : dd (standard)

#### Option A : Basique

```bash
# Copie simple
sudo dd if=/dev/sda1 of=~/forensics/copie_disque/images/sda1.img

# Paramètres :
# if = input file (source)
# of = output file (destination)
```

#### Option B : Avec progression

```bash
# dd avec affichage de la progression (Linux récent)
sudo dd if=/dev/sda1 \
        of=~/forensics/copie_disque/images/sda1.img \
        bs=4M \
        status=progress

# bs=4M : block size de 4 Mo (optimal pour SSD)
# status=progress : affiche la progression
```

#### Option C : Avec gestion d'erreurs

```bash
# dd avec options de récupération
sudo dd if=/dev/sda1 \
        of=~/forensics/copie_disque/images/sda1.img \
        bs=4M \
        conv=noerror,sync \
        status=progress

# conv=noerror : continue malgré les erreurs de lecture
# conv=sync : remplit les blocs incomplets avec des zéros
```

#### Option D : Avec pipe pour monitoring

```bash
# dd avec pv (pipe viewer) pour barre de progression
sudo apt install pv

sudo dd if=/dev/sda1 bs=4M | \
  pv -s $(sudo blockdev --getsize64 /dev/sda1) | \
  dd of=~/forensics/copie_disque/images/sda1.img bs=4M

# -s : spécifie la taille totale pour le calcul du pourcentage
```

### 4.2 Méthode 2 : dcfldd (forensique)

```bash
# Installation
sudo apt install dcfldd

# Copie avec hash MD5 intégré
sudo dcfldd if=/dev/sda1 \
             of=~/forensics/copie_disque/images/sda1.img \
             bs=4M \
             hash=md5 \
             hashwindow=1G \
             hashlog=~/forensics/copie_disque/hashes/dcfldd.log \
             status=on

# hash=md5 : calcule le MD5 pendant la copie
# hashwindow=1G : calcule le hash tous les 1 Go
# hashlog : fichier de log des hashes
# status=on : affiche la progression
```

### 4.3 Méthode 3 : ddrescue (pour disques endommagés)

```bash
# Installation
sudo apt install gddrescue

# Copie avec gestion des erreurs avancée
sudo ddrescue -f -n /dev/sda1 \
                   ~/forensics/copie_disque/images/sda1.img \
                   ~/forensics/copie_disque/logs/rescue.log

# -f : force (écrase la destination)
# -n : passe rapide sans retry
# rescue.log : log des secteurs problématiques

# Deuxième passe pour récupérer les secteurs difficiles
sudo ddrescue -d -r3 /dev/sda1 \
                     ~/forensics/copie_disque/images/sda1.img \
                     ~/forensics/copie_disque/logs/rescue.log

# -d : direct disk access
# -r3 : 3 tentatives de retry sur les secteurs difficiles
```

### 4.4 Méthode 4 : dc3dd (forensique avancé)

```bash
# Installation
sudo apt install dc3dd

# Copie avec hash multiple et split
sudo dc3dd if=/dev/sda1 \
           of=~/forensics/copie_disque/images/sda1.img \
           hash=md5 hash=sha256 \
           log=~/forensics/copie_disque/logs/dc3dd.log \
           progress=on

# hash=md5 hash=sha256 : calcule plusieurs hashes
# log : fichier de log détaillé
# progress=on : affiche la progression
```

### 4.5 Comparaison des méthodes

| Outil | Vitesse | Hash intégré | Gestion erreurs | Forensique |
|-------|---------|--------------|-----------------|------------|
| **dd** | ⭐⭐⭐⭐⭐ | ❌ | ⚠️ Basique | ⚠️ |
| **dcfldd** | ⭐⭐⭐⭐ | ✅ MD5 | ✅ | ✅ |
| **ddrescue** | ⭐⭐⭐ | ❌ | ✅✅✅ | ✅ |
| **dc3dd** | ⭐⭐⭐⭐ | ✅ Multi | ✅ | ✅✅ |

---

## 5. Calcul et vérification des hashes

### 5.1 Calculer les hashes de la SOURCE

```bash
# MD5 de la partition source
sudo md5sum /dev/sda1 | tee ~/forensics/copie_disque/hashes/sda1_source.md5

# SHA1
sudo sha1sum /dev/sda1 | tee ~/forensics/copie_disque/hashes/sda1_source.sha1

# SHA256
sudo sha256sum /dev/sda1 | tee ~/forensics/copie_disque/hashes/sda1_source.sha256

# Tous en même temps avec tee
sudo sh -c 'md5sum /dev/sda1 > hashes/sda1_source.md5 & \
            sha1sum /dev/sda1 > hashes/sda1_source.sha1 & \
            sha256sum /dev/sda1 > hashes/sda1_source.sha256'
```

⚠️ **Attention** : Le calcul de hash peut prendre du temps (plusieurs minutes pour un gros disque)

### 5.2 Calculer les hashes de l'IMAGE

```bash
# MD5 de l'image
md5sum ~/forensics/copie_disque/images/sda1.img | \
  tee ~/forensics/copie_disque/hashes/sda1_image.md5

# SHA1
sha1sum ~/forensics/copie_disque/images/sda1.img | \
  tee ~/forensics/copie_disque/hashes/sda1_image.sha1

# SHA256
sha256sum ~/forensics/copie_disque/images/sda1.img | \
  tee ~/forensics/copie_disque/hashes/sda1_image.sha256
```

### 5.3 Vérifier l'intégrité

```bash
# Comparer les hashes visuellement
echo "=== MD5 ==="
cat ~/forensics/copie_disque/hashes/sda1_source.md5
cat ~/forensics/copie_disque/hashes/sda1_image.md5

echo "=== SHA256 ==="
cat ~/forensics/copie_disque/hashes/sda1_source.sha256
cat ~/forensics/copie_disque/hashes/sda1_image.sha256

# Comparer automatiquement
diff ~/forensics/copie_disque/hashes/sda1_source.md5 \
     ~/forensics/copie_disque/hashes/sda1_image.md5

# Si aucune sortie = identique ✅
# Si différence = PROBLÈME ❌
```

### 5.4 Vérification automatique

```bash
# Script de vérification
#!/bin/bash

SOURCE_MD5=$(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_source.md5)
IMAGE_MD5=$(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_image.md5)

if [ "$SOURCE_MD5" == "$IMAGE_MD5" ]; then
    echo "✅ VERIFICATION PASSED : Les hashes MD5 sont identiques"
else
    echo "❌ VERIFICATION FAILED : Les hashes MD5 sont différents !"
    echo "Source : $SOURCE_MD5"
    echo "Image  : $IMAGE_MD5"
fi
```

### 5.5 Créer un fichier de vérification complet

```bash
# Créer un rapport de hashes
cat > ~/forensics/copie_disque/hashes/verification_report.txt <<EOF
=== RAPPORT DE VÉRIFICATION D'INTÉGRITÉ ===
Date: $(date)
Opérateur: $(whoami)

SOURCE: /dev/sda1
MD5:    $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_source.md5)
SHA1:   $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_source.sha1)
SHA256: $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_source.sha256)

IMAGE: sda1.img
MD5:    $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_image.md5)
SHA1:   $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_image.sha1)
SHA256: $(awk '{print $1}' ~/forensics/copie_disque/hashes/sda1_image.sha256)

VERIFICATION:
$( diff -q ~/forensics/copie_disque/hashes/sda1_source.md5 \
           ~/forensics/copie_disque/hashes/sda1_image.md5 \
   && echo "✅ MD5 : PASS" || echo "❌ MD5 : FAIL" )
$( diff -q ~/forensics/copie_disque/hashes/sda1_source.sha256 \
           ~/forensics/copie_disque/hashes/sda1_image.sha256 \
   && echo "✅ SHA256 : PASS" || echo "❌ SHA256 : FAIL" )
EOF

# Afficher le rapport
cat ~/forensics/copie_disque/hashes/verification_report.txt
```

---

## 6. Analyse de l'image

### 6.1 Vérifier le type de fichier

```bash
# Identifier le type d'image
file ~/forensics/copie_disque/images/sda1.img

# Plus de détails
file -s ~/forensics/copie_disque/images/sda1.img
```

### 6.2 Monter l'image

```bash
# Créer un point de montage
sudo mkdir -p /mnt/forensic_image

# Monter en lecture seule
sudo mount -o ro,loop ~/forensics/copie_disque/images/sda1.img /mnt/forensic_image

# Vérifier le montage
mount | grep forensic_image

# Lister le contenu
ls -la /mnt/forensic_image

# Démonter quand terminé
sudo umount /mnt/forensic_image
```

### 6.3 Analyser avec Sleuth Kit

```bash
# Installer Sleuth Kit
sudo apt install sleuthkit

# Afficher les informations du filesystem
fsstat ~/forensics/copie_disque/images/sda1.img

# Lister tous les fichiers (y compris supprimés)
fls -r ~/forensics/copie_disque/images/sda1.img

# Lister avec dates
fls -r -m / ~/forensics/copie_disque/images/sda1.img > timeline.txt

# Extraire un fichier spécifique (par inode)
icat ~/forensics/copie_disque/images/sda1.img [INODE] > extracted_file

# Rechercher une chaîne dans l'image
strings ~/forensics/copie_disque/images/sda1.img | grep "malware"
```

### 6.4 Créer une timeline

```bash
# Générer un body file
fls -r -m / ~/forensics/copie_disque/images/sda1.img > body_file.txt

# Créer une timeline formatée
mactime -b body_file.txt -d > timeline.csv

# Ouvrir avec un tableur
libreoffice timeline.csv
```

### 6.5 Recherche de données

```bash
# Rechercher des patterns (emails, URLs, IPs)
strings ~/forensics/copie_disque/images/sda1.img | grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" > emails.txt

strings ~/forensics/copie_disque/images/sda1.img | grep -E "https?://[^\s]+" > urls.txt

strings ~/forensics/copie_disque/images/sda1.img | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort -u > ips.txt
```

---

## 7. Outils avancés

### 7.1 Guymager (GUI)

```bash
# Installation
sudo apt install guymager

# Lancer (en root)
sudo guymager

# Interface graphique pour :
# - Sélectionner la source
# - Choisir le format (RAW, EWF, AFF)
# - Calculer les hashes automatiquement
# - Voir la progression
```

### 7.2 Autopsy (Analyse forensique)

```bash
# Installation
sudo apt install autopsy

# Lancer
autopsy

# Ouvrir dans le navigateur : http://localhost:9999/autopsy

# Créer un nouveau cas
# Ajouter l'image sda1.img comme data source
# Lancer l'analyse
```

### 7.3 FTK Imager (Alternative)

FTK Imager est disponible pour Windows mais peut être utilisé avec Wine :

```bash
# Installer Wine
sudo apt install wine

# Télécharger FTK Imager Lite
# Exécuter avec Wine
wine FTKImagerLite.exe
```

### 7.4 ewfacquire (Format EnCase)

```bash
# Installation
sudo apt install ewf-tools

# Créer une image au format E01 (EnCase)
sudo ewfacquire /dev/sda1 \
  -t ~/forensics/copie_disque/images/sda1 \
  -C case_number \
  -D description \
  -E examiner \
  -N notes

# Vérifier l'image E01
ewfverify ~/forensics/copie_disque/images/sda1.E01

# Monter l'image E01
sudo ewfmount ~/forensics/copie_disque/images/sda1.E01 /mnt/ewf
sudo mount -o ro,loop /mnt/ewf/ewf1 /mnt/forensic_image
```

---

## 8. Troubleshooting

### 8.1 Erreur : Permission denied

```bash
# Problème
dd: failed to open '/dev/sda1': Permission denied

# Solution
sudo dd if=/dev/sda1 of=image.img
```

### 8.2 Erreur : No space left on device

```bash
# Vérifier l'espace
df -h

# Libérer de l'espace
sudo apt clean
sudo apt autoremove

# Ou changer de destination
dd if=/dev/sda1 of=/mnt/external_drive/image.img
```

### 8.3 Copie très lente

```bash
# Augmenter le block size
dd if=/dev/sda1 of=image.img bs=16M status=progress

# Pour SSD
bs=4M ou bs=16M

# Pour HDD
bs=1M ou bs=4M

# Désactiver le cache (plus rapide sur certains systèmes)
dd if=/dev/sda1 of=image.img bs=4M oflag=direct status=progress
```

### 8.4 Hash différent entre source et image

```bash
# Vérifier la taille
ls -lh image.img
sudo blockdev --getsize64 /dev/sda1

# Refaire la copie
sudo dd if=/dev/sda1 of=image_new.img bs=4M conv=sync,noerror status=progress

# Recalculer les hashes
```

### 8.5 Image corrompue

```bash
# Vérifier l'intégrité
file image.img

# Tenter une récupération
sudo ddrescue /dev/sda1 image_recovered.img rescue.log

# Analyser le log
cat rescue.log
```

---

## 9. COMMANDES EXÉCUTÉES

**⭐ Section mise à jour en temps réel**

Cette section documente les commandes réellement exécutées lors de ce TP.

### 9.1 Préparation

```bash
# Création de la structure
mkdir -p ~/forensics/copie_disque/{images,hashes,logs}

# Vérification des disques
lsblk
# Résultat : /dev/sda1 identifié (976 Mo)

# Informations sur la partition cible
sudo blockdev --getsize64 /dev/sda1
# Résultat : 1023410176 bytes (976 Mo)

sudo file -s /dev/sda1
# Résultat : DOS/MBR boot sector, FAT32

# Vérification de l'espace disponible
df -h /home/nyx/Téléchargements/
# Résultat : 42 Go disponibles
```

### 9.2 Acquisition

**Date de début** : 2 décembre 2025, 13:40:14 CET

```bash
# Copie bit-à-bit avec dd
sudo dd if=/dev/sda1 \
        of=/home/nyx/Téléchargements/Malware/VIRUS/copie_disque/images/sda1.img \
        bs=4M \
        status=progress \
        conv=noerror,sync

# Résultat:
# 234881024 octets (235 MB, 224 MiB) copiés, 1 s, 230 MB/s
# 419430400 octets (419 MB, 400 MiB) copiés, 2 s, 209 MB/s
# 595591168 octets (596 MB, 568 MiB) copiés, 3 s, 198 MB/s
# 784334848 octets (784 MB, 748 MiB) copiés, 4 s, 196 MB/s
# 244+0 enregistrements lus
# 244+0 enregistrements écrits
# 1023410176 octets (1,0 GB, 976 MiB) copiés, 4,59449 s, 223 MB/s

# Statistiques finales:
# - Temps : 4,59 secondes
# - Vitesse moyenne : 223 Mo/s
# - Blocks : 244 enregistrements de 4 Mo
# - Taille : 976 Mo (1 023 410 176 bytes)

# Statut: ✅ COPIE RÉUSSIE
```

### 9.3 Calcul des hashes

```bash
# Hash MD5 de la source (/dev/sda1)
sudo md5sum /dev/sda1 | tee copie_disque/hashes/sda1_source.md5
# ✅ Résultat MD5 : f38ce30b3bb951160988c3c888badf95

# Hash SHA1 de la source
sudo sha1sum /dev/sda1 | tee copie_disque/hashes/sda1_source.sha1
# ✅ Résultat SHA1 : 13e94fc666cea873c72873fed00b1e00c91aa0de

# Hash SHA256 de la source
sudo sha256sum /dev/sda1 | tee copie_disque/hashes/sda1_source.sha256
# ✅ Résultat SHA256 : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402

# Hash MD5 de l'image
md5sum copie_disque/images/sda1.img | tee copie_disque/hashes/sda1_image.md5
# ✅ Résultat MD5 : f38ce30b3bb951160988c3c888badf95

# Hash SHA1 de l'image
sha1sum copie_disque/images/sda1.img | tee copie_disque/hashes/sda1_image.sha1
# ✅ Résultat SHA1 : 13e94fc666cea873c72873fed00b1e00c91aa0de

# Hash SHA256 de l'image
sha256sum copie_disque/images/sda1.img | tee copie_disque/hashes/sda1_image.sha256
# ✅ Résultat SHA256 : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402
```

### 9.4 Vérification

```bash
# Comparaison des hashes MD5
diff copie_disque/hashes/sda1_source.md5 copie_disque/hashes/sda1_image.md5
# ✅ Résultat : Aucune différence (IDENTIQUES)

# Comparaison des hashes SHA256
diff copie_disque/hashes/sda1_source.sha256 copie_disque/hashes/sda1_image.sha256
# ✅ Résultat : Aucune différence (IDENTIQUES)

# Vérification du type de fichier
file copie_disque/images/sda1.img
# ✅ Résultat : DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat",
#              sectors/cluster 8, Media descriptor 0xf8, sectors/track 63,
#              heads 255, hidden sectors 2048, sectors 1998801 (volumes > 32 MB),
#              FAT (32 bit), sectors/FAT 1952, reserved 0x1,
#              serial number 0x842e694c, unlabeled

# Taille finale de l'image
ls -lh copie_disque/images/sda1.img
# ✅ Résultat : -rw-r--r-- 1 root root 976M  2 déc.  13:40 sda1.img

# CONCLUSION: ✅✅✅ Image validée - Copie bit-à-bit authentique
```

### 9.5 Analyse préliminaire

```bash
# Montage de l'image
sudo mkdir -p /mnt/forensic_efi
sudo mount -o ro,loop copie_disque/images/sda1.img /mnt/forensic_efi

# Contenu
ls -la /mnt/forensic_efi

# Démontage
sudo umount /mnt/forensic_efi
```

### 9.6 Métadonnées

```bash
# Création du fichier de métadonnées
cat > copie_disque/logs/acquisition_metadata.txt <<EOF
=== MÉTADONNÉES D'ACQUISITION ===
Date: $(date)
Hostname: $(hostname)
Kernel: $(uname -r)
Opérateur: $(whoami)

Source: /dev/sda1
Type: EFI System Partition (FAT32)
Taille: 1023410176 bytes (976 Mo)

Destination: sda1.img
Méthode: dd (GNU coreutils)
Block size: 4M
Options: noerror,sync,status=progress

MD5 source: [À compléter]
MD5 image: [À compléter]
SHA256 source: [À compléter]
SHA256 image: [À compléter]

Vérification: [PASS/FAIL]
EOF

# Affichage
cat copie_disque/logs/acquisition_metadata.txt
```

### 9.7 Résumé de l'exécution

| Étape | Commande | Temps | Résultat |
|-------|----------|-------|----------|
| Structure | `mkdir -p copie_disque/{images,hashes,logs}` | < 1s | ✅ Créé |
| Identification | `lsblk && blockdev --getsize64 /dev/sda1` | < 1s | ✅ 976 Mo |
| **Copie dd** | `dd if=/dev/sda1 of=sda1.img bs=4M` | **4,59s** | ✅ **976 Mo @ 223 Mo/s** |
| Hash MD5 source | `md5sum /dev/sda1` | ~3 min | ✅ f38ce30b... |
| Hash SHA1 source | `sha1sum /dev/sda1` | ~3 min | ✅ 13e94fc6... |
| Hash SHA256 source | `sha256sum /dev/sda1` | ~3 min | ✅ cc68e7dc... |
| Hash MD5 image | `md5sum sda1.img` | ~3 min | ✅ f38ce30b... |
| Hash SHA1 image | `sha1sum sda1.img` | ~3 min | ✅ 13e94fc6... |
| Hash SHA256 image | `sha256sum sda1.img` | ~3 min | ✅ cc68e7dc... |
| Vérification | `diff hashes/*` | < 1s | ✅ **Identiques** |

**Total effectif** : ~19 minutes (dont ~18 min pour les 6 calculs de hash)

**Performance** :
- Copie SSD : **Excellente** (223 Mo/s sur SSD)
- Intégrité : **100%** (tous les hashes correspondent)
- Qualité : **Forensique** (copie bit-à-bit complète)

---

**Document technique créé pour le TP de forensique numérique**

**Dernière mise à jour** : 2 décembre 2025
