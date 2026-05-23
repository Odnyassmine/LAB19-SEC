# Snake APK Writeup – PWNSEC Challenge

<p align="center">
  <img src="images/banner.png" width="900"/>
</p>

---

## Overview

Ce challenge Android repose sur une vulnérabilité de désérialisation SnakeYAML (`CVE-2022-1471`) combinée à plusieurs protections anti-analysis :

- Anti-root
- Anti-emulator
- Anti-Frida
- Vérifications d’Intent
- Chargement JNI natif

L’objectif est d’exploiter la désérialisation YAML afin d’instancier une classe cachée (`BigBoss`) qui déclenche une fonction native générant le flag.

---

# Étape 1 — Préparation de l’environnement

## Outils nécessaires

Installer :

- Jadx-GUI
- apktool
- apksigner ou uber-apk-signer
- adb
- Un émulateur Android API 28 ou inférieur recommandé

---

## Installation de l’APK original

```bash
adb install snake.apk
```

---

# Étape 2 — Analyse statique avec Jadx

Ouvrir `snake.apk` dans Jadx-GUI.

<p align="center">
  <img src="images/jadx_overview.png" width="900"/>
</p>

---

## Package principal

```text
com.pwnsec.snake
```

## Classe d’entrée

```text
MainActivity
```

---

## Logique importante observée

Dans `MainActivity` :

L’application vérifie un extra Intent :

```java
SNAKE == "BigBoss"
```

Si la condition est valide :

1. L’application accède au stockage externe
2. Cherche :

```text
/sdcard/Snake/Skull_Face.yml
```

3. Lit le fichier YAML
4. Parse le contenu avec SnakeYAML

---

<p align="center">
  <img src="images/mainactivity_check.png" width="850"/>
</p>

---

# Classe importante : BigBoss

```text
com.pwnsec.snake.BigBoss
```

## Comportement

La classe :

- Charge une librairie native avec `System.loadLibrary(...)`
- Contient une méthode prenant une chaîne en paramètre
- Vérifie :

```text
Snaaaaaaaaaaaaaake
```

Si la valeur est correcte :

- Appelle une fonction JNI
- Génère dynamiquement le flag
- Affiche le flag dans `logcat`

---

<p align="center">
  <img src="images/bigboss_class.png" width="850"/>
</p>

---

# Protections observées

## Anti-root

Détection via :

- `Build.TAGS`
- `test-keys`
- `/system/app/Superuser.apk`
- Exécution de `su`

## Anti-emulator

Vérifications :

- `ro.hardware`
- `ro.product.model`
- propriétés QEMU

## Anti-Frida

Présent dans la librairie native :

- détection de ports
- détection de processus
- détection hooks

---

<p align="center">
  <img src="images/protections.png" width="850"/>
</p>

---

# Étape 3 — Patch Smali

Les protections empêchent l’exécution normale.

Il faut patcher l’APK.

---

## Décompilation

```bash
apktool d snake.apk -o snake_smali
```

---

## Navigation

```bash
cd snake_smali/smali/com/pwnsec/snake/
```

---

# Recherche des protections

Dans Jadx ou grep :

```text
root
su
frida
emulator
test-keys
```

---

# Bypass des checks

## Exemple classique

Avant :

```smali
if-nez v0, :cond_bad
```

Après :

```smali
goto :cond_safe
```

---

## Forcer un retour false

Avant :

```smali
return v0
```

Après :

```smali
const/4 v0, 0x0
return v0
```

---

<p align="center">
  <img src="images/smali_patch.png" width="850"/>
</p>

---

# Recompilation

```bash
apktool b snake_smali -o snake_patched.apk
```

---

# Signature

```bash
apksigner sign --ks ton_keystore.jks snake_patched.apk
```

---

# Installation

```bash
adb install -r snake_patched.apk
```

---

# Permissions

Autoriser :

- READ_EXTERNAL_STORAGE
- WRITE_EXTERNAL_STORAGE

---

# Étape 4 — Exploitation SnakeYAML

Créer le dossier :

```bash
adb shell mkdir -p /sdcard/Snake
```

---

# Payload YAML

Créer :

```text
Skull_Face.yml
```

Contenu :

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```

---

<p align="center">
  <img src="images/yaml_payload.png" width="850"/>
</p>

---

# Explication du payload

## Global Tag SnakeYAML

```yaml
!!com.pwnsec.snake.BigBoss
```

Force SnakeYAML à instancier directement la classe Java.

---

## Paramètre du constructeur

```yaml
["Snaaaaaaaaaaaaaake"]
```

Passe la chaîne exacte attendue par `BigBoss`.

---

# Pourquoi ça fonctionne

SnakeYAML `< 2.0` permet la désérialisation arbitraire de classes Java :

```text
CVE-2022-1471
```

La désérialisation déclenche :

1. Instanciation de `BigBoss`
2. Chargement JNI
3. Exécution native
4. Génération du flag

---

# Push du fichier YAML

```bash
adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml
```

---

# Étape 5 — Lancement avec Intent

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```

---

<p align="center">
  <img src="images/intent_launch.png" width="850"/>
</p>

---

# Explication

L’Intent :

```text
-e SNAKE BigBoss
```

satisfait la condition dans `MainActivity`.

Cela déclenche :

```text
Lecture YAML
→ Désérialisation SnakeYAML
→ Instanciation BigBoss
→ Appel JNI
→ Flag dans logcat
```

---

# Étape 6 — Récupération du flag

## Filtrage ciblé

```bash
adb logcat | grep -i "PWNSEC"
```

## Ou logs plus larges

```bash
adb logcat | grep -i "snake"
```

---

<p align="center">
  <img src="images/final_flag.png" width="900"/>
</p>

---

# Flag

```text
PWNSEC{W3'r3_N0t_T00l5_0f_The_g0v3rnm3n7_0R_4ny0n3_3ls3}
```

---

# Résumé du flow d’exploitation

```text
Intent Extra
    ↓
Lecture fichier YAML
    ↓
SnakeYAML Deserialization
    ↓
Instanciation BigBoss
    ↓
JNI Native Call
    ↓
Flag dans logcat
```

---

# Notes importantes

## Difficulté principale

Le vrai challenge réside dans :

- le bypass anti-root
- le bypass anti-emulator
- le bypass anti-frida

---

# Conseils

## Recherche rapide des protections

Dans Jadx :

```text
Build.TAGS
test-keys
su
frida
isEmulator
```

---

# Important

Le flag n’est PAS stocké en clair dans l’APK.

Il est généré dynamiquement dans la librairie native.

---

# Recommandation environnement

Utiliser :

- Android 9 ou inférieur
- émulateur propre
- sans Magisk
- sans Frida actif

---

# Conclusion

Ce challenge démontre :

- l’exploitation de SnakeYAML (`CVE-2022-1471`)
- les risques de désérialisation unsafe
- l’utilisation de JNI pour cacher une logique sensible
- les techniques de bypass Android protections via patch Smali

---

<p align="center">
  <img src="images/end.png" width="900"/>
</p>
