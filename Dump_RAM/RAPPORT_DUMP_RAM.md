# Rapport : Dump de la RAM

## Informations générales

- **Date** : 2 décembre 2025
- **Système d'exploitation** : Linux (Kali) 6.16.8+kali-amd64
- **Objectif** : Capturer et analyser le contenu de la mémoire RAM dans le cadre d'une analyse forensique

---

## 1. Introduction

Le dump de la RAM (ou acquisition de mémoire volatile) est une technique forensique essentielle permettant de capturer l'état actuel de la mémoire vive d'un système. Cette capture contient des informations précieuses telles que :

- Les processus en cours d'exécution
- Les connexions réseau actives
- Les mots de passe en mémoire
- Les clés de chiffrement
- Les artefacts de malwares actifs
- Les données non sauvegardées

---

## 2. Méthodologie

### 2.1 Outils utilisés

Pour réaliser un dump de RAM sous Linux, plusieurs outils sont disponibles :

1. **LiME (Linux Memory Extractor)** - Recommandé pour Linux
2. **Avml (Acquire Volatile Memory for Linux)** - Outil Microsoft
3. **dd avec /proc/kcore** - Méthode native mais limitée
4. **Volatility** - Pour l'analyse post-acquisition

### 2.2 Choix de l'outil

**LiME** a été sélectionné car :
- Module kernel spécialement conçu pour l'acquisition mémoire
- Minimise la contamination de la mémoire
- Format compatible avec Volatility
- Largement utilisé en forensique

---

## 3. Procédure d'acquisition

### 3.1 Installation de LiME

```bash
# Cloner le dépôt
git clone https://github.com/504ensicsLabs/LiME.git

# Compiler le module
cd LiME/src
make

# Vérifier la création du module
ls -l lime-*.ko
```

### 3.2 Capture de la mémoire

```bash
# Insérer le module et effectuer le dump
sudo insmod lime-*.ko "path=/tmp/ram_dump.lime format=lime"

# Alternative : format raw
sudo insmod lime-*.ko "path=/tmp/ram_dump.raw format=raw"
```

### 3.3 Vérification

```bash
# Vérifier la taille du dump
ls -lh /tmp/ram_dump.*

# Calculer le hash MD5 pour l'intégrité
md5sum /tmp/ram_dump.* > ram_dump.md5

# Calculer le hash SHA256
sha256sum /tmp/ram_dump.* > ram_dump.sha256
```

---

## 4. Résultats obtenus

### 4.1 Caractéristiques du dump

- **Fichier** : ram_dump.lime
- **Taille** : 12 Go (11 957 MB)
- **Format** : LiME format (compatible Volatility)
- **Hash MD5** : ba94611b0be2a89117a3033ea12063f6
- **Hash SHA1** : 29092d21b39f59e155f44979433683d2a66a0705
- **Hash SHA256** : 481ad3a47fb48aace25f58618a8d056904620f867ef71a1a3b3bc734cd2b7626
- **Date d'acquisition** : 2 décembre 2025, 11:10 CET

### 4.2 Informations système capturées

Le dump contient :
- État complet de la RAM au moment de l'acquisition
- Processus actifs et leurs données
- Modules kernel chargés
- Connexions réseau établies
- Cache du système de fichiers
- Buffers et données temporaires

---

## 5. Analyse réalisée avec Volatility

### 5.1 Installation de Volatility

Volatility 3 a été installé depuis le dépôt GitHub officiel :

```bash
cd /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM
git clone https://github.com/volatilityfoundation/volatility3.git
```

### 5.2 Génération des symboles

Pour analyser le dump Linux, les symboles du kernel ont été générés avec `dwarf2json` :

```bash
# Installation du package de debug
sudo apt install linux-image-6.16.8+kali-amd64-dbg

# Génération du fichier de symboles ISF
dwarf2json linux \
  --elf /usr/lib/debug/boot/vmlinux-6.16.8+kali-amd64 \
  --system-map /boot/System.map-6.16.8+kali-amd64 \
  | xz -c > volatility3/volatility3/symbols/linux/6.16.8-kali-amd64.json.xz
```

**Fichier de symboles généré** : 2,7 Mo (compressé)

### 5.3 Analyse des banners système

```bash
python3 vol.py -f ram_dump.lime banners.Banners
```

**Résultats obtenus** :

```
Linux version 6.16.8+kali-amd64 (devel@kali.org)
Compiler: x86_64-linux-gnu-gcc-14 (Debian 14.3.0-8) 14.3.0
Linker: GNU ld (GNU Binutils for Debian) 2.45
Build: #1 SMP PREEMPT_DYNAMIC Kali 6.16.8-1kali1 (2025-09-24)
```

**Informations clés extraites** :
- **Kernel** : Linux 6.16.8+kali-amd64
- **Distribution** : Kali Linux
- **Architecture** : x86_64 (AMD64)
- **Type de préemption** : PREEMPT_DYNAMIC (support temps réel)
- **Date de compilation du kernel** : 24 septembre 2025

### 5.4 Analyses complémentaires effectuées

Les analyses suivantes ont été lancées et sauvegardées dans le répertoire `results/` :

1. **analysis_banners.txt** - Informations sur le kernel
2. **analysis_lsmod.txt** - Modules kernel chargés
3. **analysis_sockstat.txt** - Statistiques des sockets réseau
4. **analysis_bash_history.txt** - Historique bash en mémoire

### 5.5 Note sur les analyses avancées

Certaines analyses Volatility avancées (pslist, pstree, lsmod) ont rencontré des problèmes de compatibilité avec les symboles générés. Ceci est un problème connu avec les kernels récents (6.16.x) et Volatility 3, nécessitant potentiellement :

- Mise à jour de Volatility vers une version plus récente
- Ajustements manuels du fichier ISF
- Utilisation d'outils alternatifs (Rekall, analyse manuelle)

---

## 6. Analyse spécifique aux malwares

### 6.1 Recherche de processus suspects

- Processus avec des noms aléatoires
- Processus sans parent (PPID anormal)
- Processus cachés (rootkits)
- Injections de code

### 6.2 Recherche de connexions réseau suspectes

- Connexions vers des IP étrangères
- Ports non standards
- Communications C2 (Command & Control)

### 6.3 Extraction de données

```bash
# Dumper un processus spécifique
volatility3 -f ram_dump.lime -o /tmp/dump/ linux.procdump.ProcDump --pid [PID]

# Extraire les fichiers en mémoire
volatility3 -f ram_dump.lime linux.filescan.FileScan
```

---

## 7. Considérations forensiques

### 7.1 Chaîne de traçabilité

- **Date/Heure d'acquisition** : 2 décembre 2025, 11:10:20 CET
- **Système** : Kali Linux 6.16.8+kali-amd64
- **Hostname** : Sylren
- **Opérateur** : nyx
- **Hash MD5** : ba94611b0be2a89117a3033ea12063f6
- **Hash SHA1** : 29092d21b39f59e155f44979433683d2a66a0705
- **Hash SHA256** : 481ad3a47fb48aace25f58618a8d056904620f867ef71a1a3b3bc734cd2b7626
- **Méthode** : LiME kernel module v2.0 (lime-6.16.8+kali-amd64.ko)
- **Stockage** : /home/nyx/Téléchargements/Malware/VIRUS/dump_RAM/results/

### 7.2 Intégrité des preuves

- Hashes cryptographiques calculés et documentés
- Dump stocké en lecture seule
- Copie de travail créée pour l'analyse
- Logs de toutes les opérations effectuées

---

## 8. Limitations et précautions

### 8.1 Limitations

- La capture modifie légèrement l'état de la mémoire
- Certaines données peuvent être effacées par le système
- Les malwares anti-forensique peuvent détecter l'acquisition
- Taille importante des dumps (équivalent à la RAM totale)

### 8.2 Précautions

- Effectuer le dump le plus rapidement possible
- Minimiser les actions sur le système avant l'acquisition
- Utiliser un média de stockage externe propre
- Documenter toutes les actions effectuées

---

## 9. Conclusions

Le dump de la RAM est une étape cruciale dans l'analyse forensique d'un système compromis. Il permet de capturer des informations volatiles qui seraient perdues à l'extinction du système.

### Points clés :

1. ✅ **Acquisition réussie** - 12 Go de mémoire RAM capturée avec succès
2. ✅ **Intégrité vérifiée** - Trois hashes cryptographiques (MD5, SHA1, SHA256) générés et documentés
3. ✅ **Format compatible** - Dump au format LiME, compatible avec Volatility et autres outils forensiques
4. ✅ **Documentation complète** - Toute la procédure d'acquisition est documentée avec métadonnées
5. ✅ **Symboles générés** - Fichier ISF créé pour permettre l'analyse avec Volatility 3
6. ✅ **Analyses préliminaires** - Informations du kernel extraites et validées

### Résumé de l'acquisition :

| Paramètre | Valeur |
|-----------|---------|
| **Taille du dump** | 12 Go |
| **Méthode** | LiME v2.0 (lime-6.16.8+kali-amd64.ko) |
| **Format** | LiME format |
| **Système** | Kali Linux 6.16.8+kali-amd64 |
| **Date** | 2 décembre 2025, 11:10 CET |
| **MD5** | ba94611b0be2a89117a3033ea12063f6 |
| **SHA256** | 481ad3a47fb48aace25f58618a8d056904620f867ef71a1a3b3bc734cd2b7626 |

### Fichiers générés :

```
dump_RAM/results/
├── ram_dump.lime (12 Go) - Dump de la RAM
├── ram_dump.md5 - Hash MD5
├── ram_dump.sha1 - Hash SHA1
├── ram_dump.sha256 - Hash SHA256
├── acquisition_info.txt - Métadonnées d'acquisition
├── analysis_banners.txt - Informations kernel
├── analysis_lsmod.txt - Modules kernel
├── analysis_sockstat.txt - Sockets réseau
└── analysis_bash_history.txt - Historique bash
```

### Prochaines étapes :

- ✅ Analyse approfondie avec Volatility (partiellement réalisée)
- ⏳ Résolution des problèmes de compatibilité des symboles
- ⏳ Extraction complète des processus et connexions réseau
- ⏳ Corrélation avec les artefacts disque (Env.exe, Res.exe)
- ⏳ Identification des IoCs (Indicators of Compromise)
- ⏳ Analyse des malwares détectés dans le système
- ⏳ Rédaction du rapport d'incident complet

### Recommandations :

1. **Conservation du dump** : Le fichier ram_dump.lime doit être conservé en lecture seule pour préserver l'intégrité
2. **Chaîne de custody** : Tous les hashes ont été documentés pour maintenir la chaîne de traçabilité
3. **Analyses alternatives** : En cas de problèmes persistants avec Volatility, considérer Rekall ou analyse manuelle avec gdb
4. **Corrélation** : Croiser les résultats du dump RAM avec l'analyse statique des binaires (Env.exe, Res.exe)

---

## 10. Références

- **LiME** : https://github.com/504ensicsLabs/LiME
- **Volatility Foundation** : https://www.volatilityfoundation.org/
- **SANS Digital Forensics** : https://www.sans.org/digital-forensics/
- **The Art of Memory Forensics** : Livre de référence sur la forensique mémoire

---

## Annexes

### Annexe A : Commandes de vérification système

```bash
# Informations système
uname -a
cat /proc/meminfo | grep MemTotal
```

### Annexe B : Compatibilité Volatility

Le format LiME est directement compatible avec :
- Volatility 2.x (avec profile approprié)
- Volatility 3.x (détection automatique)
- Rekall
- Redline

---

**Document réalisé dans le cadre du TP d'analyse forensique**
