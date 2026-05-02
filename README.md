#  OWASP UnCrackable Level 3 – Reverse Engineering Writeup

##  Présentation

Ce projet présente l'analyse et le reverse engineering de l'application Android **OWASP UnCrackable Level 3**. L'objectif principal est de contourner les mécanismes de protection, analyser le code natif et extraire la clé secrète cachée.

---

##  Objectifs

- Contourner la détection de root et de modification
- Désactiver les protections anti-debug natives
- Analyser le code natif
- Extraire la clé secrète

---

##  Outils utilisés

| Outil | Usage |
|---|---|
| Apktool | Décompilation / reconstruction de l'APK |
| Ghidra | Analyse du code natif |
| ADB | Communication avec l'émulateur Android |
| Python | Décodage XOR de la clé |
| Émulateur Android (x86_64) | Environnement d'exécution |

---

##  Vérification de l'environnement

```bash
adb devices
adb shell getprop ro.product.cpu.abi
```
<img width="475" height="59" alt="image" src="https://github.com/user-attachments/assets/79ab695b-8b77-4891-8645-91ffd59b8497" />
<img width="910" height="127" alt="image" src="https://github.com/user-attachments/assets/95c30060-67db-4a20-8a89-70743b3e7d55" />


---

## 🧩 Étape 1 – Décompilation de l'APK

La première étape consiste à décompiler l'application afin d'accéder au code Smali.

```bash
apktool d UnCrackable-Level3.apk -o uncrackable3
```
<img width="490" height="475" alt="image" src="https://github.com/user-attachments/assets/72007f45-aca1-4de1-aa51-9ba8cec8ba93" />
<img width="928" height="361" alt="image" src="https://github.com/user-attachments/assets/a2f017f3-2e24-4374-ba4b-b67538568dbd" />



---

##  Étape 2 – Contournement des protections Root & Tamper

Le fichier suivant est modifié :

```
smali/sg/vantagepoint/uncrackable3/MainActivity.smali
```
<img width="785" height="463" alt="image" src="https://github.com/user-attachments/assets/f4e66bc3-dc08-47ed-b15a-13dbf3268f5a" />

Le code responsable de l'affichage du message d'erreur est remplacé par :


```smali
:cond_0
goto :cond_1
```
<img width="838" height="471" alt="image" src="https://github.com/user-attachments/assets/32ebe01d-1b69-40ac-bf70-330a45c165e2" />
<img width="854" height="250" alt="image" src="https://github.com/user-attachments/assets/bf32bfbb-b56c-4bb5-ba37-673165effa92" />


```
Ce patch permet de désactiver les vérifications de root et de modification.
```
##   Étape 3 – Reconstruction et signature de l'APK
Après modification, l'application est reconstruite et signée :

bashapktool b uncrackable3 -o UnCrackable-Level3-patched.apk
apksigner sign --ks "%USERPROFILE%\.android\debug.keystore" UnCrackable-Level3-patched.apk
adb uninstall owasp.mstg.uncrackable3
adb install -r UnCrackable-Level3-patched.apk


<img width="821" height="394" alt="image" src="https://github.com/user-attachments/assets/00b4a534-43d4-4f9f-96ee-06b61c9a6d8b" />
<img width="502" height="305" alt="image" src="https://github.com/user-attachments/assets/6c77b4de-978c-41e6-bde1-2890f30fcd4a" />
Vérifie le keystore
<img width="502" height="157" alt="image" src="https://github.com/user-attachments/assets/e8060564-f37a-4231-94e1-9fe0d9f8d03d" />


```bash

& "$env:LOCALAPPDATA\Android\Sdk\build-tools\36.1.0\apksigner.bat" sign --ks "$HOME\.android\debug.keystore" UnCrackable-Level3-patched.apk
```
Quand il demande le mot de passe :
android


<img width="511" height="215" alt="image" src="https://github.com/user-attachments/assets/9072503f-ce77-4bcf-a99f-550e2d1cb63d" />

```bash
adb uninstall owasp.mstg.uncrackable3
adb install -r UnCrackable-Level3-patched.apk
```
<img width="537" height="169" alt="image" src="https://github.com/user-attachments/assets/3f4da20f-801d-4b1f-be16-e4fe7f823fde" />

---

---


##  Étape 4 – Analyse de la librairie native

La logique principale de l’application est implémentée dans une librairie native, ce qui rend l’analyse plus complexe que du simple code Java.

<img width="1608" height="798" alt="image" src="https://github.com/user-attachments/assets/2812bfb7-3059-4c70-b9a9-a9a8102ea51c" />


---

###  Import dans Ghidra

Nous avons importé la librairie dans Ghidra pour analyser son contenu.

<img width="1608" height="798" alt="image" src="https://github.com/user-attachments/assets/dfa53935-56a9-43d1-bb7a-578b2edae693" />



---

###   Exploration des fonctions

Dans le **Symbol Tree**, nous avons identifié les fonctions importantes :

- `Java_sg_vantagepoint_uncrackable3_CodeCheck_bar` → fonction JNI principale  
- `goodbye` → fonction appelée en cas de détection (crash)  
- Plusieurs fonctions internes `FUN_...`

<img width="1588" height="760" alt="image" src="https://github.com/user-attachments/assets/a6b7778c-0bce-4919-9bd5-62606ee57719" />

---

###  Identification de la protection anti-debug

Une fonction suspecte est identifiée (`FUN_00103910`) :

- Appel à `fork`
- Vérification via `ptrace`
- Logique conditionnelle pouvant mener à un crash

<img width="1722" height="838" alt="image" src="https://github.com/user-attachments/assets/f763a566-b94f-4474-9e20-2943a88ad673" />

---

###  Patch de la protection

Objectif : désactiver complètement la fonction de protection.

#### Étapes :

1. Sélection de la première instruction (`PUSH`)
2. Clic droit → **Patch Instruction**
3. Remplacement par :


<img width="1137" height="621" alt="image" src="https://github.com/user-attachments/assets/2436f946-3231-4ea8-ae5b-cb65b6bfa70a" />
<img width="1132" height="720" alt="image" src="https://github.com/user-attachments/assets/57027c2b-f7b0-40db-942b-09bca1d549ad" />


---



##  Étape 5 – Identification de l'obfuscation

L'analyse révèle plusieurs techniques d'obfuscation :

- Appels `malloc` répétés
- Calculs pseudo-aléatoires (LCG)
- Boucles inutiles
- Code volontairement complexe

Ces éléments servent uniquement à compliquer l'analyse statique.
<img width="554" height="226" alt="image" src="https://github.com/user-attachments/assets/239bbb1a-2224-45f2-9418-c3434a33daab" />

---

##  Étape 6 – Localisation de la génération de la clé

Une fonction importante est identifiée :

```
FUN_001012c0(local_48);
```

Cette fonction initialise le buffer contenant la clé encodée.
<img width="548" height="315" alt="image" src="https://github.com/user-attachments/assets/0af333d8-5b84-464f-bf0f-fab353f76851" />
<img width="731" height="358" alt="image" src="https://github.com/user-attachments/assets/75a3c29f-ab4f-4ecc-87fa-bd32256bad32" />

---

##  Étape 7 – Extraction de la clé encodée

À la fin de la fonction, les valeurs suivantes sont trouvées :

```c
*param_1       = 0x1549170f1311081d;
param_1[1]     = 0x15131d5a1903000d;
param_1[2]     = 0x14130817005a0e08;

```
<img width="494" height="281" alt="image" src="https://github.com/user-attachments/assets/42560051-35a7-4a62-84f2-949ee3659d84" />



---

##  Étape 8 – Conversion Little Endian

Les valeurs sont converties en format little-endian :

```
1d 08 11 13 0f 17 49 15
0d 00 03 19 5a 1d 13 15
08 0e 5a 00 17 08 13 14
```

Ce qui donne la séquence hexadécimale :

```
1d0811130f1749150d0003195a1d1315080e5a0017081314
```

---

##  Étape 9 – Décodage XOR

Un script Python permet de décoder la clé :

```python
encoded = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")
xor_key = b"pizzapizzapizzapizzapizzapizza"

secret = bytes(a ^ b for a, b in zip(encoded, xor_key))
print(secret.decode())
```
<img width="692" height="295" alt="image" src="https://github.com/user-attachments/assets/6046a064-cf5c-4625-b2a0-6e8e1d76ac67" />

---

##  Clé finale

```
making owasp great again
```
<img width="703" height="109" alt="image" src="https://github.com/user-attachments/assets/6a2f454a-ad01-4ff5-b14c-3e90d653c64e" />

---

##  Étape 10 – Vérification

La clé est saisie dans l'application patchée, ce qui affiche le message de succès.
<img width="209" height="436" alt="image" src="https://github.com/user-attachments/assets/c658237c-732a-4ac3-842c-38c16c883a4f" />

---
##  Résultat avant / après patch

Avant toute modification, l’application détecte les protections (root / tampering) et empêche son utilisation.

 Application normale (avant patch) :
<img width="322" height="610" alt="image" src="https://github.com/user-attachments/assets/352eb451-8102-43ab-9e34-4a75f59bf764" />

 Message affiché :
- "Rooting or tampering detected"
- L’application se ferme immédiatement

---

Après application des patches (Smali + librairie native), l’application fonctionne correctement.

 Application après patch :
<img width="400" height="715" alt="image" src="https://github.com/user-attachments/assets/465a879b-21cf-4517-94b2-6d20284a440c" />

 Résultat :
- Plus de détection de root  
- Plus de crash  
- Champ de saisie accessible  
- Bouton **VERIFY** fonctionnel  

---

###  Conclusion de cette étape

- Les protections Java et natives ont été contournées  
- L’application peut maintenant être utilisée normalement  
- Nous pouvons passer à l’analyse de la logique interne pour extraire la clé  


##  Leçons apprises

- Le code natif permet de masquer la logique critique
- L'obfuscation complexifie l'analyse mais ne protège pas réellement
- Les données importantes se trouvent souvent à la fin des fonctions
- Le XOR est une méthode classique de dissimulation de secrets

---

##  Conclusion

Ce challenge démontre que :

- L'obfuscation seule ne suffit pas pour sécuriser une application
- Le reverse engineering permet d'extraire des informations sensibles
- Les protections natives peuvent être contournées avec les bons outils
