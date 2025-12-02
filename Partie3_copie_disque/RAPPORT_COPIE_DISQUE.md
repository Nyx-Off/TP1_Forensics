# Rapport : Copie Bit-à-Bit de Disque (Disk Imaging)

## Informations générales

- **Date** : 2 décembre 2025
- **Système d'exploitation** : Linux (Kali) 6.16.8+kali-amd64
- **Objectif** : Créer une image forensique bit-à-bit d'une partition disque
- **Opérateur** : nyx
- **Hostname** : Sylren

---

## 1. Introduction

### 1.1 Qu'est-ce qu'une copie bit-à-bit ?

Une copie bit-à-bit (ou **disk imaging**) est une technique forensique qui consiste à créer une copie exacte, octet par octet, d'un support de stockage (disque, partition, clé USB, etc.). Contrairement à une simple copie de fichiers, elle préserve :

- **Tous les octets** : Fichiers, données effacées, slack space, etc.
- **La structure** : Tables de partitions, secteurs de boot, etc.
- **Les métadonnées** : Dates de création, modification, accès
- **Les données cachées** : Fichiers supprimés, données résiduelles

### 1.2 Pourquoi faire une copie bit-à-bit ?

En forensique numérique, créer une image disque est essentiel pour :

1. **Préservation de la preuve** : L'original reste intact
2. **Analyse non-destructive** : On travaille sur la copie
3. **Reproductibilité** : On peut recréer l'analyse
4. **Intégrité** : Hash cryptographique pour vérifier l'authenticité
5. **Aspect légal** : Chaîne de traçabilité respectée

### 1.3 Outils utilisés

Pour Linux, plusieurs outils sont disponibles :

| Outil | Avantages | Inconvénients |
|-------|-----------|---------------|
| **dd** | Standard Unix, partout | Pas de barre de progression, lent |
| **dcfldd** | Version améliorée de dd, hash intégré | Moins répandu |
| **ddrescue** | Excellent pour disques endommagés | Plus complexe |
| **dc3dd** | Hashing, logs, forensique | Nécessite installation |

Pour ce TP, nous utilisons **dd** (standard) et **dcfldd** (forensique).

---

## 2. Méthodologie

### 2.1 Préparation

Avant toute acquisition :

1. **Identifier la cible** : Quelle partition/disque copier ?
2. **Vérifier l'espace** : Assez d'espace pour l'image ?
3. **Protection écriture** : Bloquer l'écriture sur la source (si possible)
4. **Documentation** : Noter toutes les informations système

### 2.2 Choix de la partition

Pour ce TP, nous avons choisi **la partition EFI (/dev/sda1)** :

- **Taille** : 976 Mo (1023410176 bytes)
- **Type** : FAT32 (EFI System Partition)
- **Point de montage** : /boot/efi
- **Justification** : Taille raisonnable pour un exemple pédagogique

### 2.3 Méthode d'acquisition

1. **Démontage** (optionnel mais recommandé)
2. **Copie avec dd** : Outil standard
3. **Calcul des hashes** : MD5, SHA1, SHA256
4. **Vérification** : Comparer les hashes source/destination
5. **Documentation** : Métadonnées et chaîne de traçabilité

---

## 3. Système analysé

### 3.1 Configuration du système

```
Hostname       : Sylren
Kernel         : Linux 6.16.8+kali-amd64
Distribution   : Kali Linux
Architecture   : x86_64
RAM totale     : 23 Go
```

### 3.2 Disques disponibles

```
Disque principal : /dev/sda (238,5 Go SSD)
├── sda1         : 976 Mo   - /boot/efi (FAT32)  ← CIBLE
├── sda2         : 225,2 Go - / (ext4)
└── sda3         : 12,3 Go  - [SWAP]

Disque secondaire : /dev/nvme0n1 (476,9 Go NVMe)
├── nvme0n1p1    : 16 Mo
├── nvme0n1p2    : 476,1 Go (NTFS)
├── nvme0n1p3    : 100 Mo
└── nvme0n1p4    : 657 Mo
```

### 3.3 Partition cible : /dev/sda1

| Paramètre | Valeur |
|-----------|---------|
| **Device** | /dev/sda1 |
| **Taille** | 976 Mo (1 023 410 176 bytes) |
| **Type** | Partition EFI (FAT32) |
| **Label** | - |
| **Serial Number** | 0x842e694c |
| **Montage** | /boot/efi |
| **Utilisation** | 1% (160 Ko / 974 Mo) |

---

## 4. Acquisition effectuée

### 4.1 Préparation

**Date et heure de début** : 2 décembre 2025, 13:40:14 CET

**Vérifications préalables** :
- ✅ Espace disque disponible : 42 Go disponibles
- ✅ Permissions root obtenues
- ✅ Outils installés (dd, md5sum, sha1sum, sha256sum)

### 4.2 Démontage de la partition (optionnel)

⚠️ **Note** : Pour /boot/efi, il est préférable de NE PAS démonter car c'est nécessaire au démarrage. Nous ferons la copie avec la partition montée en lecture seule.

```bash
# Vérification du montage
mount | grep sda1

# Remontage en lecture seule (optionnel)
sudo mount -o remount,ro /boot/efi
```

### 4.3 Copie avec dd

**Commande exécutée** :

```bash
sudo dd if=/dev/sda1 \
        of=/home/nyx/Téléchargements/Malware/VIRUS/copie_disque/images/sda1.img \
        bs=4M \
        status=progress \
        conv=noerror,sync
```

**Paramètres utilisés** :

- `if=/dev/sda1` : Source (input file)
- `of=sda1.img` : Destination (output file)
- `bs=4M` : Block size de 4 Mo (optimal pour SSD)
- `status=progress` : Affiche la progression
- `conv=noerror,sync` : Continue malgré les erreurs, synchronise

**Résultats** :

```
1023410176 octets (1,0 GB, 976 MiB) copiés, 4,59449 s, 223 MB/s

Statistiques :
- Temps écoulé : 4,59 secondes
- Vitesse moyenne : 223 Mo/s
- Blocks copiés : 244 enregistrements de 4 Mo
- Taille finale : 976 Mo (1 023 410 176 bytes)
```

**Statut** : ✅ **COPIE RÉUSSIE**

### 4.4 Calcul des hashes

#### Hash de la source (/dev/sda1)

```bash
# MD5
sudo md5sum /dev/sda1 > copie_disque/hashes/sda1_source.md5

# SHA1
sudo sha1sum /dev/sda1 > copie_disque/hashes/sda1_source.sha1

# SHA256
sudo sha256sum /dev/sda1 > copie_disque/hashes/sda1_source.sha256
```

**Résultats** :
- **MD5** : f38ce30b3bb951160988c3c888badf95
- **SHA1** : 13e94fc666cea873c72873fed00b1e00c91aa0de
- **SHA256** : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402

#### Hash de l'image (sda1.img)

```bash
# MD5
md5sum copie_disque/images/sda1.img > copie_disque/hashes/sda1_image.md5

# SHA1
sha1sum copie_disque/images/sda1.img > copie_disque/hashes/sda1_image.sha1

# SHA256
sha256sum copie_disque/images/sda1.img > copie_disque/hashes/sda1_image.sha256
```

**Résultats** :
- **MD5** : f38ce30b3bb951160988c3c888badf95
- **SHA1** : 13e94fc666cea873c72873fed00b1e00c91aa0de
- **SHA256** : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402

### 4.5 Vérification de l'intégrité

**Comparaison des hashes** :

```bash
# Comparer MD5
diff copie_disque/hashes/sda1_source.md5 copie_disque/hashes/sda1_image.md5

# Résultat attendu : Aucune différence
```

**Statut** : ✅ **VÉRIFICATION RÉUSSIE**

- ✅ MD5 : **Identique** (f38ce30b3bb951160988c3c888badf95)
- ✅ SHA1 : **Identique** (13e94fc666cea873c72873fed00b1e00c91aa0de)
- ✅ SHA256 : **Identique** (cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402)

**Conclusion** : L'image est une copie bit-à-bit **EXACTE** de la source. L'intégrité est **garantie**.

---

## 5. Informations forensiques

### 5.1 Chaîne de traçabilité

| Information | Valeur |
|-------------|---------|
| **Date d'acquisition** | 2 décembre 2025 |
| **Heure de début** | 13:40:14 CET |
| **Heure de fin** | 13:40:18 CET (durée : 4,59s) |
| **Opérateur** | nyx |
| **Hostname** | Sylren |
| **Système** | Kali Linux 6.16.8+kali-amd64 |
| **Méthode** | dd (GNU coreutils) |
| **Source** | /dev/sda1 (EFI partition, FAT32) |
| **Destination** | sda1.img |
| **Taille exacte** | 1 023 410 176 bytes (976 Mo) |
| **Vitesse** | 223 Mo/s |
| **Hash MD5 source** | f38ce30b3bb951160988c3c888badf95 |
| **Hash SHA1 source** | 13e94fc666cea873c72873fed00b1e00c91aa0de |
| **Hash SHA256 source** | cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402 |
| **Hash MD5 image** | f38ce30b3bb951160988c3c888badf95 |
| **Hash SHA1 image** | 13e94fc666cea873c72873fed00b1e00c91aa0de |
| **Hash SHA256 image** | cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402 |
| **Vérification** | ✅ **PASS** - Hashes identiques |

### 5.2 Métadonnées de l'acquisition

**Fichier de métadonnées** : `copie_disque/logs/acquisition_metadata.txt`

```
=== MÉTADONNÉES D'ACQUISITION DISQUE ===
Date système     : 2 décembre 2025, 13:40:14 CET
Hostname         : Sylren
Opérateur        : nyx
Kernel           : 6.16.8+kali-amd64
Distribution     : Kali Linux

Source           : /dev/sda1
Type             : Partition EFI (FAT32)
Taille           : 976 Mo (1 023 410 176 bytes)
Point de montage : /boot/efi
Utilisation      : 1% (160 Ko)

Destination      : sda1.img
Chemin complet   : /home/nyx/Téléchargements/Malware/VIRUS/copie_disque/images/
Méthode          : dd (GNU coreutils)
Block size       : 4 Mo
Options          : noerror,sync,status=progress
Durée            : 4,59 secondes
Vitesse          : 223 Mo/s

Hashes source    :
  MD5            : f38ce30b3bb951160988c3c888badf95
  SHA1           : 13e94fc666cea873c72873fed00b1e00c91aa0de
  SHA256         : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402

Hashes image     :
  MD5            : f38ce30b3bb951160988c3c888badf95
  SHA1           : 13e94fc666cea873c72873fed00b1e00c91aa0de
  SHA256         : cc68e7dcb1a0918e3d9aa47119f4226b7a4a1dc1c2ec7fa1fcfb46ce3f8c1402

Vérification     : ✅ PASS - Copie bit-à-bit authentique
```

---

## 6. Analyse de l'image

### 6.1 Vérification du type de fichier

```bash
file copie_disque/images/sda1.img
```

**Résultat attendu** : DOS/MBR boot sector, FAT32

### 6.2 Montage de l'image (lecture seule)

```bash
# Créer un point de montage
mkdir -p /tmp/forensic_mount

# Monter l'image en lecture seule
sudo mount -o ro,loop copie_disque/images/sda1.img /tmp/forensic_mount

# Lister le contenu
ls -la /tmp/forensic_mount

# Démonter
sudo umount /tmp/forensic_mount
```

### 6.3 Extraction d'informations

**Avec mmls (Sleuth Kit)** :

```bash
mmls copie_disque/images/sda1.img
```

**Avec fsstat (Sleuth Kit)** :

```bash
fsstat copie_disque/images/sda1.img
```

---

## 7. Bonnes pratiques forensiques

### 7.1 Principes respectés

1. ✅ **Non-altération** : La source n'a pas été modifiée
2. ✅ **Documentation** : Toutes les étapes sont documentées
3. ✅ **Hashing** : Intégrité vérifiée par hash cryptographique
4. ✅ **Traçabilité** : Métadonnées complètes enregistrées
5. ✅ **Reproductibilité** : Commandes documentées et reproductibles

### 7.2 Recommandations

Pour une investigation forensique réelle :

1. **Bloqueur d'écriture** : Utiliser un write blocker matériel
2. **Disque vierge** : Destination sur média neuf et stérilisé
3. **Double copie** : Créer deux images pour redondance
4. **Protection physique** : Sceller l'original après acquisition
5. **Documentation photo** : Photographier le matériel avant/après
6. **Format forensique** : Utiliser E01 (EnCase) ou AFF pour compression

---

## 8. Utilisation de l'image

### 8.1 Analyse avec Autopsy

```bash
# Lancer Autopsy
autopsy

# Créer un nouveau cas
# Ajouter l'image sda1.img comme source de données
# Analyser avec les modules disponibles
```

### 8.2 Analyse avec Sleuth Kit

```bash
# Lister les fichiers
fls -r copie_disque/images/sda1.img

# Extraire un fichier spécifique (inode)
icat copie_disque/images/sda1.img [inode] > extracted_file

# Timeline
mactime -b body_file.txt > timeline.csv
```

### 8.3 Montage pour analyse manuelle

```bash
# Montage en lecture seule
sudo mount -o ro,loop,noexec copie_disque/images/sda1.img /mnt/forensic

# Analyse
find /mnt/forensic -type f -name "*.log"
grep -r "malware" /mnt/forensic

# Démontage
sudo umount /mnt/forensic
```

---

## 9. Conclusions

### 9.1 Résumé de l'acquisition

| Critère | Résultat |
|---------|----------|
| **Acquisition** | ✅ Réussie |
| **Intégrité** | ✅ Vérifiée (hash identiques) |
| **Documentation** | ✅ Complète |
| **Utilisabilité** | ✅ Image montable |
| **Conformité forensique** | ✅ Oui |

### 9.2 Points clés

1. ✅ **Image créée** : 976 Mo copiés avec succès
2. ✅ **Intégrité vérifiée** : Hashes source = hashes image
3. ✅ **Format standard** : Image RAW (.img) utilisable par tous les outils
4. ✅ **Documentation forensique** : Chaîne de traçabilité respectée
5. ✅ **Analyse possible** : Image montable et analysable

### 9.3 Fichiers générés

```
copie_disque/
├── images/
│   └── sda1.img (976 Mo) - Image bit-à-bit
├── hashes/
│   ├── sda1_source.md5
│   ├── sda1_source.sha1
│   ├── sda1_source.sha256
│   ├── sda1_image.md5
│   ├── sda1_image.sha1
│   └── sda1_image.sha256
└── logs/
    └── acquisition_metadata.txt
```

### 9.4 Prochaines étapes

- ⏳ Analyse du contenu de la partition EFI
- ⏳ Extraction des fichiers de boot
- ⏳ Corrélation avec le dump RAM
- ⏳ Recherche d'artefacts malveillants
- ⏳ Documentation complète des findings

---

## 10. Références

### 10.1 Documentation

- **dd man page** : `man dd`
- **Sleuth Kit** : https://www.sleuthkit.org/
- **NIST Guidelines** : Special Publication 800-86
- **SANS Digital Forensics** : https://www.sans.org/

### 10.2 Outils recommandés

- **dd / dcfldd** : Acquisition bit-à-bit
- **Guymager** : Interface graphique pour imaging
- **FTK Imager** : Outil Windows/Linux
- **Autopsy** : Analyse forensique
- **The Sleuth Kit** : Suite d'outils CLI

### 10.3 Formats d'image

| Format | Extension | Compression | Métadonnées | Outil |
|--------|-----------|-------------|-------------|-------|
| RAW | .img, .dd | Non | Non | dd |
| E01 | .E01 | Oui | Oui | EnCase, FTK |
| AFF | .aff | Oui | Oui | Afflib |
| EWF | .Ex01 | Oui | Oui | ewfacquire |

---

**Document réalisé dans le cadre du TP d'analyse forensique**

**Date** : 2 décembre 2025
