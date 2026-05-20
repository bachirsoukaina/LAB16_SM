# LAB16_SM

**LAB 16 : Inspection HTTPS Android : Désactivation du SSL Pinning avec Objection + Proxy (Burp/mitmproxy)**

**Auteur :** Soukaina Bachir  
**Cours :** Sécurité des applications mobiles – MLIAEdu  
**Date :** 20/05/2026

---

## ⚠️ Avertissement éthique

N'appliquez ce lab que dans un cadre légal (appareils/apps vous appartenant ou audit autorisé). Le contournement de l'SSL pinning sert à analyser la sécurité réseau d'une application, pas à compromettre des systèmes en production.

---

## 🎯 Objectifs du lab

- Installer et utiliser Objection (surcouche Frida) pour désactiver l'SSL pinning d'une app Android
- Mettre en place un proxy (Burp/mitmproxy) et installer le certificat CA sur l'appareil
- Lancer l'app avec Objection et exécuter `android sslpinning disable` efficacement (spawn ou attach)
- Valider la capture du trafic HTTPS en clair et dépanner les cas courants

---

## 📋 Prérequis

- PC Windows (PowerShell) ou macOS/Linux avec Python 3.8+ et pip
- **ADB** (Android Platform Tools)
- **Frida** côté PC et **frida-server** côté Android, versions alignées
- Un appareil Android 8+ avec Débogage USB activé
- Burp Suite (ou mitmproxy) sur le PC

### Vérifications rapides

```bash
python --version
pip --version
adb version
```

---

## 📝 Étape 1 — Installer Objection et Frida côté PC

### Option isolée (recommandée)

```bash
pip install --user pipx
pipx ensurepath
pipx install objection
```

### Ou via pip classique

```bash
pip install --upgrade objection frida frida-tools
```

### Vérifiez

```bash
objection --version
frida --version
python -c "import frida; print(frida.__version__)"
```

![Frida version](https://github.com/user-attachments/assets/876ced7e-0dce-4e76-babe-d90a9c6ef12a)

*Capture 1 : frida --version → 17.9.6*

![Objection help](https://github.com/user-attachments/assets/02a21a90-ece5-4246-ac50-68f2303ee468)

*Capture 2 : objection --help*

---

## 📱 Étape 2 — Préparer l'appareil et démarrer frida-server

### Activer le Débogage USB et vérifier la connexion

```bash
adb devices
```

### Identifier l'architecture CPU

```bash
adb shell getprop ro.product.cpu.abi
```

### Télécharger frida-server

Téléchargez `frida-server-<version>-android-<arch>.xz` depuis [GitHub Frida Releases](https://github.com/frida/frida/releases)

### Pousser et lancer frida-server

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

### Vérifier que l'appareil est visible

```bash
frida-ps -Uai
```

![Processus Android](https://github.com/user-attachments/assets/76d1e944-ef25-40cc-b58e-9608603960f3)

*Capture 3 : Liste des processus Android dont Diva (PID 6989)*

---

## 🔐 Étape 3 — Configurer le proxy et installer la CA

### Certificats installés sur l'appareil

![Certificats Trusted](https://github.com/user-attachments/assets/3e6c1fee-6019-4f82-838e-d3322b6cd872)

*Capture 4 : Certificats mitmproxy et PortSwigger CA dans "Trusted credentials"*

### Interface Burp Suite

![Burp CA Certificate](https://github.com/user-attachments/assets/d945a10f-f545-4b04-910e-88c14cb78fa7)

*Capture 5 : Interface Burp Suite avec "CA Certificate"*

### Configuration du proxy

1. **Lancer Burp** (ou mitmproxy) sur le PC, noter l'adresse/port (ex: `192.168.X.Y:8080`)

2. **Sur le téléphone, configurer le proxy Wi-Fi :**

![Configuration proxy Wi-Fi](https://github.com/user-attachments/assets/b8593cee-1ac9-4b88-a14a-ea79ee61c203)

*Capture 6 : Configuration - Proxy hostname: 10.0.2.2, Proxy port: 8080*

3. **Installer la CA du proxy :**
   - **Burp** : visiter `http://burp`
   - **mitmproxy** : visiter `http://mitm.it` → Android

---

## 🚀 Étape 4 — Lancer l'app avec Objection et désactiver le pinning

### Erreur avant désactivation

![Erreur SSL](https://github.com/user-attachments/assets/0357f27c-45b7-4fd9-98fa-6a8abd21f7f1)

*Capture 7 : Erreur NET:ERR_CERT_AUTHORITY_INVALID avant désactivation*

### Identifier le package

```bash
frida-ps -Uai
```

### Méthode SPAWN (recommandée)

```bash
objection -g com.example.app explore --startup-command "android sslpinning disable"
```

### Méthode ATTACH

```bash
objection -g com.example.app explore
# Dans la console Objection :
android sslpinning disable
```

### Pour Diva

```bash
objection -g diva explore --startup-command "android sslpinning disable"
```

---

## ✅ Étape 5 — Validation

1. Générer du trafic dans l'app (login, requêtes API)
2. Dans Burp/mitmproxy, vérifier que les requêtes HTTPS apparaissent
3. Plus d'erreur `ERR_CERT_AUTHORITY_INVALID`

---

## 🔧 Dépannage (FAQ)

| Problème | Solution |
|----------|----------|
| `unable to connect to remote frida-server` | Aligner les versions Frida, vérifier adb forward |
| Objection affiche des erreurs au spawn | Passer en mode attach |
| Le proxy ne voit rien | Vérifier IP/port, CA installée, même réseau |
| L'app utilise Cronet/pinning natif | Objection insuffisant → script Frida natif |

---

## 📌 Bonnes pratiques

- ✓ Commencez en spawn pour patcher tôt
- ✓ Ne surchargez pas `--startup-command`
- ✓ Nettoyez après le test : fermez frida-server, supprimez la CA

### Variante : tout en une ligne

```bash
objection -g com.example.app explore --startup-command "android sslpinning disable"
```

---

## ✔️ Annexe — Checklist rapide

- [ ] objection, frida, frida-server installés et alignés
- [ ] Proxy opérationnel et CA installée
- [ ] Lancement Objection avec `android sslpinning disable`
- [ ] Trafic HTTPS visible dans le proxy

---

**Fin du LAB16**
