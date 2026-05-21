# LAB14-SECURITE
# Lab Android — Bypass de la Detection de Root

**Realise par :** EL OUARZAZI AYA 
**Date :** 21 mai 2026  
**Environnement :** Windows 10, Python 3.10.11, Frida 17.9.1, Android Emulator (AVD x86_64, Android 11)  
**Applications cibles :** com.scottyab.rootbeer.sample, com.android.insecurebankv2

---

## Table des matieres

1. [Contexte et objectifs](#1-contexte-et-objectifs)
2. [Preparation de l environnement](#2-preparation-de-lenvironnement)
3. [Demarrage de frida-server sur l emulateur](#3-demarrage-de-frida-server-sur-lemulateur)
4. [Comprendre Frida](#4-comprendre-frida)
5. [Bypass de la detection de root avec Frida (scripts manuels)](#5-bypass-de-la-detection-de-root-avec-frida-scripts-manuels)
6. [Bypass avec Objection](#6-bypass-avec-objection)
7. [Bypass avec Medusa](#7-bypass-avec-medusa)
8. [Resultats obtenus](#8-resultats-obtenus)
9. [Synthese des methodes](#9-synthese-des-methodes)

---

## 1. Contexte et objectifs

Ce lab a pour but de comprendre et de contourner les mecanismes de detection de root implementes dans des applications Android. Trois outils complementaires ont ete utilises : Frida (injection de scripts JavaScript), Objection (surcouche haut niveau sur Frida) et Medusa (framework de modules Frida pre-construits).

Les techniques de detection de root couramment rencontrees dans les applications sont les suivantes :

- Lecture de `android.os.Build.TAGS` (valeur `test-keys` sur emulateur/appareil roote)
- Verification de l existence de binaires suspects via `java.io.File.exists()` (su, busybox, Superuser.apk)
- Tentative d execution de `su` ou `which su` via `Runtime.getRuntime().exec()`
- Utilisation de bibliotheques tierces telles que RootBeer (`com.scottyab.rootbeer.RootBeer.isRooted()`)
- Verifications natives (C/C++) via les appels libc : `open`, `openat`, `access`, `stat`

---

## 2. Preparation de l environnement

### Outils installes

| Outil | Version | Commande de verification |
|---|---|---|
| Python | 3.10.11 | `python --version` |
| pip | 25.1.1 | `pip --version` |
| Frida (client) | 17.9.1 | `frida --version` |
| ADB | 1.0.41 | `adb version` |
| Objection | 1.12.4 | `objection --version` |
| Medusa | dev | interface CLI |

### Installation des outils Frida et Objection

```powershell
python -m pip install --upgrade frida frida-tools
python -m pip install --upgrade objection
```

### Verification ADB et emulateur

```powershell
adb devices
```

Resultat obtenu :

```
List of devices attached
emulator-5554    offline
emulator-5556    device
```

L emulateur `emulator-5556` (Android 11, x86_64) est l appareil cible actif.

> **Capture d ecran de reference :** <img width="884" height="222" alt="prerequis" src="https://github.com/user-attachments/assets/57fb1fcc-7fec-4139-9e61-d71c568a65bd" />

> Montre la verification de Python 3.10.11, pip, Frida 17.9.1, ADB, et la liste des appareils avec emulator-5556 en etat `device`.

---

## 3. Demarrage de frida-server sur l emulateur

Frida necessite un serveur depose sur l appareil Android pour pouvoir injecter des scripts depuis le PC.

### Architecture CPU de l emulateur

```powershell
adb -s emulator-5556 shell getprop ro.product.cpu.abi
```

Resultat : `x86_64`

### Telechargement de frida-server

La version de `frida-server` doit correspondre exactement a la version du client Frida installe sur le PC (17.9.1).

Fichier telecharge depuis https://github.com/frida/frida/releases :  
`frida-server-17.9.1-android-x86_64.xz`

Decompression sous Windows avec 7-Zip.

### Deploiement sur l emulateur

```powershell
adb -s emulator-5556 push C:\Users\HP\Downloads\frida-server-17.9.1-android-x86_64\frida-server-17.9.1-android-x86_64 /data/local/tmp/frida-server

adb -s emulator-5556 shell chmod 755 /data/local/tmp/frida-server

adb -s emulator-5556 shell /data/local/tmp/frida-server &
```

### Validation

```powershell
frida-ps -Uai
```

La liste des applications installees sur l emulateur doit s afficher.

> **Capture d ecran de reference :** <img width="875" height="50" alt="demarrage-frida" src="https://github.com/user-attachments/assets/bc57ecc2-e0af-447f-848d-f7f5554de8d7" />
> Montre le push du binaire frida-server (64.9 MB/s), le chmod 755 et le lancement en arriere-plan sur emulator-5556.

---

## 4. Comprendre Frida

Frida est un framework d instrumentation dynamique qui permet d injecter du code JavaScript dans un processus Android en cours d execution.

### Parametres essentiels

| Parametre | Role |
|---|---|
| `-U` | Cible un appareil USB (ou emulateur ADB) |
| `-s emulator-5556` | Specifier l emulateur cible |
| `-f <package>` | Spawn : demarrer l app et injecter au tout debut |
| `-n <ProcessName>` | Attach : s attacher a un processus deja lance |
| `--no-pause` | Ne pas suspendre l app apres injection |
| `-l <script.js>` | Charger un script JavaScript |

### Test d injection minimal

Fichier `hello.js` :

```javascript
Java.perform(function () {
  console.log("[+] Script injecte : Java.perform OK");
});
```

Execution :

```powershell
frida -U -s emulator-5556 -f com.scottyab.rootbeer.sample -l hello.js --no-pause
```

---

## 5. Bypass de la detection de root avec Frida (scripts manuels)

### 5.1 Bypass des verifications Java

Le script `bypass_root_basic.js` neutralise les quatre points de detection Java les plus courants.

**Hooks implementes :**

**Hook 1 — Build.TAGS**  
Force la valeur de `android.os.Build.TAGS` a retourner `release-keys` au lieu de `test-keys`.

**Hook 2 — RootBeer**  
Remplace l implementation de `RootBeer.isRooted()` et `isRootedWithBusyBoxCheck()` pour retourner `false`.

**Hook 3 — File.exists()**  
Intercepte les appels a `java.io.File.exists()` et retourne `false` pour une liste de chemins suspects (`/system/bin/su`, `/system/xbin/su`, `/system/app/Superuser.apk`, etc.).

**Hook 4 — Runtime.exec()**  
Bloque les executions de `su`, `which su` et `busybox` en les remplacant par la commande inoffensive `echo`. Couvre les quatre surcharges de `Runtime.exec()`.

Execution :

```powershell
frida -U -s emulator-5556 -f com.scottyab.rootbeer.sample -l bypass_root_basic.js --no-pause
```

### 5.2 Bypass des verifications natives (C/C++)

Certaines applications effectuent des verifications de root directement en code natif via les appels systeme libc. Le script `bypass_native.js` intercepte ces appels.

**Appels syscall hooks :** `open`, `openat`, `access`, `stat`, `lstat`

Pour chacun, si le chemin en argument correspond a un chemin suspect (su, busybox, `/proc/mounts`), `retval` est force a `-1` (echec).

Execution combinee :

```powershell
frida -U -s emulator-5556 -f com.scottyab.rootbeer.sample -l bypass_root_basic.js -l bypass_native.js --no-pause
```

Outil de diagnostic pour identifier les chemins reellement consultes par l application :

```powershell
frida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink com.scottyab.rootbeer.sample
```

---

## 6. Bypass avec Objection

Objection est une interface haut niveau construite sur Frida qui propose des commandes integrees pour les taches courantes de pentest Android, dont le bypass de root.

### Lancement avec bypass automatique au demarrage

```powershell
objection --serial emulator-5556 -g com.scottyab.rootbeer.sample explore --startup-command "android root disable"
```

### Ce que fait la commande `android root disable`

- Force `Build.TAGS` vers une valeur non suspecte
- Fait echouer `File.exists()` sur les chemins su/busybox
- Bloque `Runtime.exec("su")`, `Runtime.exec("which su")`, `Runtime.exec("busybox")`
- Patche les bibliotheques tierces courantes dont RootBeer

### Logs observes dans le terminal

```
(agent) [731961] RootBeer->detectRootCloakingApps() check detected, marking as false.
(agent) [731961] RootBeer->detectTestKeys() check detected, marking as false.
(agent) [731961] RootBeer->checkForBinary() check detected, marking as false.
(agent) [731961] RootBeer->checkSuExists() check detected, marking as false.
(agent) [731961] RootBeer->checkForDangerousProps() check detected, marking as false.
(agent) [731961] RootBeerNative->checkForRoot() check detected, marking as 0.
```

Tous les checks RootBeer sont neutralises avec succes.

> **Capture d ecran de reference :** <img width="769" height="286" alt="script--objection" src="https://github.com/user-attachments/assets/95afaa19-3941-4292-892a-2e1aa21f75e2" />
> Montre Objection v1.12.4 connecte sur emulator-5556 avec Android 11, le job `root-detection-disable` enregistre, et tous les hooks RootBeer marques `false`.

---

## 7. Bypass avec Medusa

Medusa est un framework d analyse dynamique Android base sur Frida proposant des modules pre-construits organises par categorie.

### Demarrage et selection de l appareil

```powershell
python medusa.py
```

A l invite, selectionner l index correspondant a `emulator-5556` :

```
0) Device(id="local", name="Local System", type='local')
1) Device(id="socket", name="Local Socket", type='remote')
2) Device(id="barebone", name="GDB Remote Stub", type='remote')
3) Device(id="emulator-5556", name="Android Emulator 5556", type='usb')

Enter the index of the device to use: 3
```

Medusa affiche les proprietes du systeme. La valeur `ro.build.tags: [test-keys]` confirme que l emulateur est detecte comme roote.

> **Capture d ecran de reference :** <img width="947" height="486" alt="medusa-cli" src="https://github.com/user-attachments/assets/aab69399-65d1-4bf2-aed7-f3b9e071938c" />
> Montre l interface Medusa avec la selection du device index 3 (emulator-5556), les proprietes systeme incluant `ro.build.tags: [test-keys]`, et la liste des applications tierces installes.

### Recherche et chargement du module de bypass root

```
(emulator-5556) medusa> search root
```

Modules disponibles :

```
root_detection/universal_root_detection_bypass
root_detection/rootbeer_detection_bypass
root_detection/rootbeer_detection_bypass_no_obfuscation
root_detection/jailMonkey_react_native
```

Chargement du module universel :

```
(emulator-5556) medusa> use root_detection/universal_root_detection_bypass
```

> **Capture d ecran de reference :** <img width="401" height="127" alt="detection-root-bypass" src="https://github.com/user-attachments/assets/b01d6827-498b-4dc1-9511-237494b1e4d8" />
> Montre la commande `search root`, les quatre modules disponibles, et le chargement de `universal_root_detection_bypass` avec confirmation dans `Current Mods`.

### Lancement sur l application cible

```
(emulator-5556) medusa> run -f com.android.insecurebankv2
```

Reponse : `Module list has been modified, do you want to recompile? (Y/n) Y`

Logs observes :

```
[INFO] Spawned package : com.android.insecurebankv2 on pid 5903
---------- LOADING ANTI-ROOT / ANTI-DEBUG SCRIPT ----------
[!] Build.TAGS -> release-keys
[x] RootBeer hook failed: ClassNotFoundException (normal : InsecureBankV2 n utilise pas RootBeer)
---------- ANTI-ROOT / ANTI-DEBUG SCRIPT LOADED ----------
```

Note : Le message `[x] RootBeer hook failed: ClassNotFoundException` est normal. InsecureBankV2 n integre pas la bibliotheque RootBeer, donc ce hook specifique ne s applique pas. Les autres hooks (Build.TAGS, File.exists, Runtime.exec) sont actifs.

> **Capture d ecran de reference :** <img width="947" height="320" alt="root-succeeded" src="https://github.com/user-attachments/assets/8650fe67-4634-4540-946c-33140b1508e1" />
> Montre Medusa spawnant InsecureBankV2 sur pid 5903, le chargement du script anti-root, Build.TAGS force en release-keys, et les informations de l application.

---

## 8. Resultats obtenus

### Avant bypass — Root detecte

L application RootBeer Sample affiche "ROOTED" avec plusieurs checks en rouge, notamment TestKeys, SU Binary et Dangerous Props.

> **Capture d ecran de reference :** <img width="189" height="384" alt="root-detected" src="https://github.com/user-attachments/assets/7b8e0421-a35d-4aed-8105-7bd6570e5c57" />
> Montre l ecran RootBeer Sample avec le filigrane "ROOTED" en rouge et les checks TestKeys, SU Binary, Dangerous Props echoues (croix rouge).

### Apres bypass — Root non detecte

Apres injection du bypass (Objection ou Medusa), tous les checks passent au vert et l application affiche "NOT ROOTED".

> **Capture d ecran de reference :** <img width="186" height="362" alt="root-not-detected" src="https://github.com/user-attachments/assets/18564aa5-71e6-4efe-aa23-1d70b14f7c0a" />
> Montre l ecran RootBeer Sample avec le filigrane "NOT ROOTED" en vert et la totalite des checks marques comme passes (coches vertes).

---

## 9. Synthese des methodes

| Methode | Outil | Commande principale | Avantages | Limites |
|---|---|---|---|---|
| Scripts manuels Java | Frida | `frida -U -f <pkg> -l bypass_root_basic.js` | Controle total, adaptable | Necessite de connaitre les hooks |
| Scripts manuels natifs | Frida | `frida -U -f <pkg> -l bypass_native.js` | Couvre les checks C/C++ | Listes de chemins a maintenir |
| Commande integree | Objection | `android root disable` | Rapide, pas de script a ecrire | Moins flexible sur l obfuscation |
| Module pre-construit | Medusa | `use root_detection/universal_root_detection_bypass` | Interface claire, modules categories | Dependant des modules disponibles |
| Masquage systeme | Magisk + Zygisk | DenyList + modules | Couvre Play Integrity et systeme | Necessite un appareil physique roote |

### Quand utiliser quelle methode

- **Frida scripts manuels** : quand l application utilise de l obfuscation ou des bibliotheques inconnues, necessitant une enumeration des classes chargees.
- **Objection** : pour un bypass rapide en phase de reconnaissance initiale.
- **Medusa** : pour une approche structuree avec historique des modules utilises.
- **Magisk** : quand l application interroge les services Google (Play Integrity / SafetyNet) ou des proprietes systeme profondes non accessibles depuis le runtime Java.

---
