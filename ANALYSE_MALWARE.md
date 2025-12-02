# RAPPORT D'ANALYSE DE MALWARE
**Date:** 2025-12-01
**Environnement:** Kali Linux (isolé)

---

## 1. RÉSUMÉ EXÉCUTIF

**Verdict:** MALWARE CONFIRMÉ - Dropper/Spyware avec capacité d'exfiltration de données

**Niveau de menace:** ÉLEVÉ

**Comportements malveillants identifiés:**
- ✓ Auto-réplication vers répertoire caché
- ✓ Persistance via registre Windows
- ✓ Exfiltration de données via SMTP
- ✓ Credentials hardcodés
- ✓ Logging d'activités

---

## 2. FICHIERS ANALYSÉS

### 2.1 Res.exe (Dropper/Installer)
```
Nom:        Res.exe
Taille:     25,088 octets
Type:       PE32 executable (console)
SHA256:     49f091ade48890bfa22d2b455494be95e52392c478b67e10626222b6aee37e1e
MD5:        d872a3086fbb82ed08a8322c028692dc
Date:       2022-08-22
```

### 2.2 Env.exe (Module d'exfiltration)
```
Nom:        Env.exe
Taille:     53,248 octets
Type:       PE32 executable (GUI)
SHA256:     e09ec2098363a129de143fdaf73ad6e2e61266fba3f638a25214af3a8bc8f2f2
MD5:        abbc02a7e5ff7b884700eac7087cf743
Date:       2022-08-13
```

---

## 3. INDICATEURS DE COMPROMISSION (IOCs)

### 3.1 Fichiers créés
- `c:\WindSyst\` (répertoire caché)
- `c:\WindSyst\log.txt` (fichier de log)
- `c:\WindSyst\Res.exe` (copie du dropper)
- `c:\WindSyst\Env.exe` (copie du module d'exfiltration)
- Toutes les DLLs Qt5 et bibliothèques associées

### 3.2 Clés de registre modifiées
```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
  └─ Valeur pointant vers: C:\WindSyst\Res.exe et C:\WindSyst\Env.exe
```

### 3.3 Credentials hardcodés compromis
```
Email:     aaaaaaaaaaaa@laposte.net
Password:  z98tmFrance
SMTP:      smtp.laposte.net
Alt Email: aaaaaaaaaaaa@gmail.com
SMTP Alt:  smtp.gmail.com
```

### 3.4 Signature de l'auteur
```
Message trouvé: "par le magniquime Hafnium !"
```

---

## 4. ANALYSE COMPORTEMENTALE

### 4.1 Res.exe - Comportement de dropper

**Actions d'installation:**
1. Crée le répertoire `c:\WindSyst`
2. Crée le sous-répertoire `c:\WindSyst\platforms`
3. Copie tous les fichiers via commandes XCOPY:
   - Res.exe → c:\WindSyst\
   - Env.exe → c:\WindSyst\
   - Toutes les DLLs → c:\WindSyst\
   - Qt platform plugins → c:\WindSyst\platforms\

**Commandes d'installation détectées:**
```batch
mkdir c:\WindSyst
XCOPY libgcc_s_dw2-1.dll c:\WindSyst /S
XCOPY libstdc++-6.dll c:\WindSyst /S
XCOPY libwinpthread-1.dll c:\WindSyst /S
XCOPY Qt5Cored.dll c:\WindSyst /S
XCOPY Res.exe c:\WindSyst /S
XCOPY Env.exe c:\WindSyst /S
XCOPY Qt5Widgets.dll c:\WindSyst /S
XCOPY Qt5Network.dll c:\WindSyst /S
XCOPY Qt5Gui.dll c:\WindSyst /S
XCOPY Qt5Core.dll c:\WindSyst /S
mkdir c:\WindSyst\platforms
XCOPY qminimal.dll c:\WindSyst\platforms /S
XCOPY qoffscreen.dll c:\WindSyst\platforms /S
XCOPY qwindows.dll c:\WindSyst\platforms /S
```

**Mécanisme de persistance:**
- Modification de la clé de registre Run pour lancement au démarrage
- Cible: `HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run`

### 4.2 Env.exe - Module d'exfiltration SMTP

**Interface graphique (Qt):**
- Titre: "SMTP Example"
- Fenêtre: "MainWindow"
- Champs du formulaire:
  - Smtp-server
  - Server port
  - Username
  - Password
  - Recipant to
  - Subject
  - Message
  - Boutons: Send, Exit

**Protocole SMTP détecté:**
```
EHLO localhost
AUTH LOGIN
MAIL FROM:<...>
RCPT TO:<...>
DATA
QUIT
```

**Fonctionnalités réseau:**
- QAbstractSocket (Qt networking)
- Gestion des états de connexion
- Gestion d'erreurs réseau
- Lecture/écriture sur socket

**Messages d'état identifiés:**
- "Connected"
- "Server response code:"
- "Message sent"
- "Unexpected reply from SMTP server:"
- "Failed to send message"

---

## 5. PREUVES DE MALVEILLANCE

### Preuve #1: Auto-réplication
Le malware se copie lui-même vers un répertoire caché avec un nom trompeur (`WindSyst` au lieu de `Windows` ou `System`)

### Preuve #2: Persistance non autorisée
Modification du registre Windows pour s'exécuter au démarrage sans consentement utilisateur

### Preuve #3: Credentials hardcodés
Présence de credentials SMTP en clair dans le binaire, indiquant une intention d'exfiltration

### Preuve #4: Communication réseau suspecte
Client SMTP intégré pour envoyer des données vers des serveurs externes

### Preuve #5: Logging caché
Création d'un fichier log dans un répertoire caché pour tracer les activités

### Preuve #6: Signature de l'auteur
Message "par le magniquime Hafnium !" indiquant une attribution potentielle

---

## 6. OUTILS D'ANALYSE RECOMMANDÉS

### 6.1 Analyse statique
- ✓ `strings` - extraction de chaînes (déjà utilisé)
- ✓ `file` - identification de type (déjà utilisé)
- ✓ `objdump` - analyse PE (déjà utilisé)
- `pescan` - analyse approfondie PE
- `peframe` - analyse de sécurité PE
- `capa` - détection de capabilities malveillantes
- `radare2` / `ghidra` - désassemblage complet

### 6.2 Analyse dynamique (ENVIRONNEMENT ISOLÉ REQUIS!)
- `procmon` - monitoring processus Windows
- `regshot` - comparaison de registre avant/après
- `wireshark` - capture trafic réseau
- `fakenet-ng` - simulation réseau
- Sandbox: `Cuckoo`, `ANY.RUN`, `Joe Sandbox`

### 6.3 Vérification de réputation
**À vérifier sur:**
- VirusTotal: https://www.virustotal.com/
- Hybrid Analysis: https://www.hybrid-analysis.com/
- MalwareBazaar: https://bazaar.abuse.ch/
- URLhaus: https://urlhaus.abuse.ch/

**Hashes à soumettre:**
```
Res.exe - SHA256: 49f091ade48890bfa22d2b455494be95e52392c478b67e10626222b6aee37e1e
Env.exe - SHA256: e09ec2098363a129de143fdaf73ad6e2e61266fba3f638a25214af3a8bc8f2f2
```

---

## 7. CHAÎNE D'INFECTION SUPPOSÉE

```
1. [Vecteur initial] → Téléchargement/Email/Drive-by
         ↓
2. [Res.exe] Exécution du dropper
         ↓
3. [Installation] Création c:\WindSyst\ + copie fichiers
         ↓
4. [Persistance] Modification registre Run
         ↓
5. [Env.exe] Lancement module exfiltration
         ↓
6. [Exfiltration] Envoi données via SMTP
```

---

## 8. RECOMMANDATIONS

### 8.1 Si système infecté
1. ⚠️ Déconnecter immédiatement du réseau
2. Scanner avec antivirus à jour
3. Vérifier/supprimer clé registre: `HKCU\...\Run`
4. Supprimer répertoire `c:\WindSyst\`
5. Changer tous les mots de passe (credentials potentiellement exfiltrés)
6. Réinstallation système recommandée

### 8.2 Prévention
- Maintenir antivirus à jour
- Ne pas exécuter fichiers d'origine inconnue
- Utiliser sandboxing pour fichiers suspects
- Activer UAC et pare-feu Windows
- Formation utilisateurs sur phishing/malware

---

## 9. ATTRIBUTION

**Signature:** "Hafnium"
**Note:** Cette signature pourrait être:
- Un vrai groupe APT (peu probable - Hafnium est un groupe connu)
- Un false flag
- Un script kiddie utilisant le nom

**Hafnium (groupe réel):** APT chinois connu pour avoir exploité Exchange Server (2021)

---

## 10. CONCLUSION

**Type de malware:** Dropper + Infostealer/Spyware
**Sophistication:** Moyenne (utilise Qt, SMTP, persistance basique)
**Objectif:** Exfiltration de données via email
**Dangerosité:** ÉLEVÉE

**Confirmation de malveillance:** ✓ CONFIRMÉ

Ce logiciel présente tous les indicateurs d'un malware:
- Comportement furtif (répertoire caché avec nom trompeur)
- Persistance non autorisée
- Capacités d'exfiltration
- Absence de fonctionnalités légitimes apparentes

**⚠️ NE PAS EXÉCUTER EN DEHORS D'UN ENVIRONNEMENT ISOLÉ ⚠️**

---

## ANNEXES

### A. Timeline de compilation
```
2022-08-13: Env.exe compilé (module d'exfiltration)
2022-08-22: Res.exe compilé (dropper) - 9 jours après
```

### B. Toolchain de développement
```
Compilateur: GCC MinGW-W64
Versions: 4.9.3 et 5.3.0
Architecture: i686-posix-dwarf
Framework: Qt 5.x
```

### C. DLLs packagées
- Qt5Core.dll (6.09 MB)
- Qt5Gui.dll (6.20 MB)
- Qt5Network.dll (1.79 MB)
- Qt5Widgets.dll (6.35 MB)
- libstdc++-6.dll (1.54 MB)
- libgcc_s_dw2-1.dll (120 KB)
- libwinpthread-1.dll (79 KB)

**Total package:** ~22 MB

---

**FIN DU RAPPORT**
