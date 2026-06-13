# 📋 CONFIGURATION & FONCTIONNEMENT
## Ludo Master Pro — Édition Bookmaker

> Guide technique complet : installation, configuration, déploiement, flux de paiement, règles du jeu.

---

## 📑 TABLE DES MATIÈRES

1. [Vue d'ensemble de l'architecture](#1-vue-densemble)
2. [Prérequis](#2-prérequis)
3. [Configuration Backend (Node.js)](#3-configuration-backend)
4. [Intégration WonyaPay](#4-intégration-wonyapay)
5. [Configuration Android (Kotlin)](#5-configuration-android)
6. [Build & Déploiement APK (GitHub Actions)](#6-build--déploiement-apk)
7. [Dashboard Admin](#7-dashboard-admin)
8. [Règles du jeu — Mécanique complète](#8-règles-du-jeu)
9. [Flux financier — Pas à pas](#9-flux-financier)
10. [Flux Compétition — Pas à pas](#10-flux-compétition)
11. [Sécurité](#11-sécurité)
12. [Variables d'environnement — Référence complète](#12-variables-denvironnement)
13. [API REST — Référence des endpoints](#13-api-rest)
14. [Dépannage](#14-dépannage)

---

## 1. Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────┐
│                    LUDO MASTER PRO                           │
├─────────────────┬────────────────────┬───────────────────────┤
│  📱 APP ANDROID │  ⚙️  BACKEND NODE  │  🖥️  ADMIN WEB       │
│  Kotlin/Compose │  Express + MongoDB │  HTML/CSS/JS          │
├─────────────────┼────────────────────┼───────────────────────┤
│ • Plateau 3D    │ • API REST         │ • Dashboard KPIs      │
│ • IA 3 niveaux  │ • Socket.IO        │ • Créer tournois      │
│ • Auth JWT      │ • WonyaPay         │ • Gérer joueurs       │
│ • Wallet        │ • MongoDB Atlas    │ • Rapports financiers │
│ • Lobbby        │ • JWT Auth         │ • Transactions        │
└─────────────────┴────────────────────┴───────────────────────┘
           │                │
           └────────────────┘
                WonyaPay API
         (M-Pesa / Orange / Airtel)
              Agréé BCC — RDC
```

---

## 2. Prérequis

### Pour le Backend
| Outil | Version minimale | Installation |
|-------|-----------------|--------------|
| Node.js | 18+ | https://nodejs.org |
| MongoDB Atlas | Gratuit (M0) | https://cloud.mongodb.com |
| Compte WonyaSoft | Requis | https://wonyasoft.com |
| Serveur (VPS/Cloud) | Ubuntu 20.04+ | OVH, DigitalOcean, Railway… |

### Pour l'Application Android
| Outil | Version | Note |
|-------|---------|------|
| Android Studio | Hedgehog 2023.1.1+ | Développement local |
| JDK | 17 (Temurin) | `apt install openjdk-17-jdk` |
| Android SDK | API 35 | Via Android Studio |
| Gradle | 8.7 | Via wrapper (automatique) |

### Pour le Dashboard Admin
- Navigateur web moderne (Chrome, Firefox)
- Accès à l'URL du backend

---

## 3. Configuration Backend

### 3.1 Installation

```bash
# Cloner ou copier le dossier backend/
cd backend
npm install

# Copier et configurer les variables d'environnement
cp .env.example .env
nano .env        # Remplir toutes les variables (voir §12)
```

### 3.2 Configuration MongoDB Atlas

1. Créer un compte sur https://cloud.mongodb.com
2. Créer un cluster M0 (gratuit)
3. **Database Access** → Créer un utilisateur : `ludomaster` / mot de passe fort
4. **Network Access** → Ajouter `0.0.0.0/0` (ou l'IP de votre serveur)
5. **Connect** → Choisir "Connect your application" → copier l'URI

```
MONGO_URI=mongodb+srv://ludomaster:MOT_DE_PASSE@cluster0.xxxxx.mongodb.net/ludomaster?retryWrites=true&w=majority
```

### 3.3 Créer le compte Admin initial

Après avoir démarré le serveur, créer l'admin via l'API :

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "email": "admin@ludomaster.cd",
    "password": "MotDePasseAdmin123!",
    "phone": "+243812345678"
  }'
```

Puis promouvoir en admin directement dans MongoDB :
```javascript
// Dans MongoDB Compass ou Atlas Data Explorer
db.users.updateOne(
  { email: "admin@ludomaster.cd" },
  { $set: { role: "admin" } }
)
```

### 3.4 Démarrage

```bash
# Développement (avec rechargement automatique)
npm run dev

# Production
npm start

# Avec PM2 (recommandé en production)
npm install -g pm2
pm2 start src/server.js --name ludo-master
pm2 save
pm2 startup
```

### 3.5 Déploiement sur Railway (simple et gratuit)

```bash
# Installer Railway CLI
npm install -g @railway/cli
railway login
railway init
railway add --plugin mongodb

# Configurer les variables d'env dans le dashboard Railway
railway up
```

---

## 4. Intégration WonyaPay

### 4.1 Obtenir les clés API

1. Aller sur **https://wonyasoft.com**
2. Créer un compte marchand
3. Compléter la vérification KYC entreprise
4. Dans le **Portail Développeur** :
   - Récupérer `API_KEY`, `SECRET`, `MERCHANT_ID`
5. Configurer l'URL de callback :
   ```
   https://votre-domaine.com/api/payment/callback/deposit
   https://votre-domaine.com/api/payment/callback/withdraw
   ```

### 4.2 Variables WonyaPay dans `.env`

```env
WONYAPAY_BASE_URL=https://api.wonyasoft.com/v1
WONYAPAY_API_KEY=sk_live_xxxxxxxxxxxxxxxxxxxx
WONYAPAY_SECRET=secret_xxxxxxxxxxxxxxxxxxxx
WONYAPAY_MERCHANT_ID=MCH_xxxxxxxxxx
WONYAPAY_CALLBACK_URL=https://votre-domaine.com/api/payment/callback
```

### 4.3 Opérateurs supportés

| Opérateur | Code dans l'API | Préfixes DRC |
|-----------|-----------------|--------------|
| M-Pesa (Vodacom) | `mpesa` | 081, 082, 083, 084, 085 |
| Orange Money | `orange_money` | 084, 085, 086, 087, 088, 089 |
| Airtel Money | `airtel_money` | 097, 098, 099 |

### 4.4 Format du numéro de téléphone

```
Entrée acceptée :  0812345678  |  +243812345678  |  243812345678
Format normalisé : +243812345678
```

### 4.5 Tester en mode sandbox

WonyaPay fournit un environnement de test :
```env
WONYAPAY_BASE_URL=https://sandbox.api.wonyasoft.com/v1
```
Utiliser les numéros de test fournis dans leur documentation.

---

## 5. Configuration Android

### 5.1 Pointer l'app vers votre backend

Ouvrir `android/app/src/main/java/com/ludomasterpro/MainActivity.kt` :

```kotlin
// Ligne ~15 — remplacer par l'URL de votre serveur
val API = "https://votre-domaine.com/api"
```

Pour le développement local avec un émulateur Android :
```kotlin
val API = "http://10.0.2.2:3000/api"   // 10.0.2.2 = localhost depuis l'émulateur
```

Pour un téléphone physique sur le même Wi-Fi :
```kotlin
val API = "http://192.168.1.X:3000/api"  // IP locale de votre PC
```

### 5.2 Configurer l'applicationId

Dans `android/app/build.gradle.kts`, changer :
```kotlin
applicationId = "com.ludomasterpro"   // → com.votrenom.ludomaster
```

### 5.3 Build local

```bash
cd android
chmod +x gradlew

# Build debug (pas besoin de keystore)
./gradlew assembleDebug

# Installer sur téléphone connecté
./gradlew installDebug
```

L'APK debug se trouve dans :
```
android/app/build/outputs/apk/debug/app-debug.apk
```

---

## 6. Build & Déploiement APK

### 6.1 Via GitHub Actions (recommandé — zéro configuration)

Le workflow `.github/workflows/build-apk.yml` :
- **Se déclenche** sur chaque push vers `main`
- **Génère** un APK signé avec une clé éphémère (RSA 2048, auto-générée, jamais stockée)
- **Publie** dans les Artifacts du workflow

```
┌─────────────┐    push     ┌──────────────────────────────────────┐
│   GitHub    │ ──────────► │         GitHub Actions Runner         │
│  (main)     │             │                                       │
└─────────────┘             │  1. Checkout du code                  │
                            │  2. Setup Java 17                     │
                            │  3. Cache Gradle                      │
                            │  4. Build APK (./gradlew assembleRelease)│
                            │  5. openssl genrsa   → clé éphémère   │
                            │  6. openssl req -x509 → certificat    │
                            │  7. zipalign → aligne l'APK           │
                            │  8. apksigner → signe l'APK           │
                            │  9. rm clé éphémère (sécurité)        │
                            │  10. Upload Artifact (30 jours)       │
                            └──────────────────────────────────────┘
```

### 6.2 Télécharger l'APK après un build

1. Aller sur votre repo GitHub
2. Onglet **Actions**
3. Cliquer sur le dernier workflow réussi
4. Section **Artifacts** en bas de page
5. Télécharger `LudoMasterPro-vX.X.X-buildN.apk`

### 6.3 Créer une Release officielle

```bash
# Taguer une version → crée automatiquement une GitHub Release avec l'APK
git tag v1.0.0
git push origin v1.0.0
```

### 6.4 Installer l'APK sur Android

1. Transférer le fichier `.apk` sur le téléphone (WhatsApp, USB, Drive…)
2. **Paramètres** → **Applications** → **Accès spéciaux** → **Installer des applis inconnues**
3. Activer pour votre gestionnaire de fichiers
4. Ouvrir le fichier `.apk` → Installer

---

## 7. Dashboard Admin

### 7.1 Accès

Ouvrir `admin/index.html` dans un navigateur.

Au premier démarrage, saisir :
- **URL API** : `https://votre-domaine.com/api` (ou `http://localhost:3000/api`)
- **Email** : l'email admin créé au §3.3
- **Mot de passe** : le mot de passe admin

### 7.2 Fonctionnalités

| Section | Description |
|---------|-------------|
| 📊 Dashboard | KPIs en temps réel : revenus, dépôts, retraits, utilisateurs |
| 🏆 Compétitions | Liste de tous les tournois avec filtrage par statut |
| ➕ Nouvelle Compét. | Formulaire de création avec prévisualisation des gains |
| 👥 Joueurs | Recherche, activation/désactivation, ajustement de solde |
| 💰 Transactions | Historique complet de tous les mouvements financiers |
| 📈 Rapports | Top joueurs, flux par opérateur Mobile Money |

### 7.3 Créer une Compétition — Exemple

```
Titre         : Tournoi du Vendredi Soir
Mise d'entrée : 500 CDF
Nb joueurs    : 4
Distribution  : 1er 60% / 2ème 30% / 3ème 10%

Calcul automatique :
  Cagnotte brute   = 500 × 4 = 2 000 CDF
  Frais plateforme = 2 000 × 10% = 200 CDF
  Prize Pool net   = 1 800 CDF
  → 1er gagne : 1 080 CDF
  → 2ème gagne : 540 CDF
  → 3ème gagne : 180 CDF
```

---

## 8. Règles du Jeu — Mécanique Complète

### 8.1 Objectif
Faire arriver les **4 pions** de votre couleur au **centre du plateau** en premier.

### 8.2 Le Plateau

```
┌─────────────────────────────────────────────┐
│  🟢 VERT       │  Chemin  │  🔴 ROUGE       │
│  (cases 26-51) │ principal │  (cases 0-25)   │
│                │  52 cases │                 │
├────────────────┼───────────┼─────────────────┤
│  Couloir ← ← ←│  🏆 Centre│→ → → Couloir    │
├────────────────┼───────────┼─────────────────┤
│  🟡 JAUNE      │  Chemin  │  🔵 BLEU        │
│  (cases 39-51) │ principal │  (cases 13-38)  │
└─────────────────────────────────────────────┘
```

### 8.3 Déroulement d'un tour

```
1. Lancer le dé (1-6)
        │
        ├─── Résultat = 6 ──────► Sortir 1 pion de la base (si besoin)
        │                         ET rejouer (nouveau tour)
        │
        ├─── Aucun coup possible ► Passer le tour
        │
        └─── Choisir un pion ───► Déplacer du nombre de cases obtenu
                                          │
                                          ├── Case occupée (adversaire seul) ► CAPTURE
                                          │       → Pion adverse retourne en base
                                          │       → Vous REJOUEZ
                                          │
                                          ├── Case ★ (rosette) ──────────────► SÛRE
                                          │       → Pas de capture possible
                                          │
                                          ├── Forteresse (2 pions alliés) ──► BLOQUÉE
                                          │       → Impossible d'y atterrir
                                          │
                                          └── Centre (arrivée exacte) ───────► ARRIVÉ
                                                  → Score +1
```

### 8.4 Règles importantes

| Règle | Détail |
|-------|--------|
| **Sortie de base** | Uniquement avec un 6 |
| **Rejouer si 6** | Toujours, même si aucun pion ne peut bouger |
| **Rejouer si capture** | Oui — capture + nouveau tour |
| **Arrivée exacte** | Le décompte doit tomber pile sur le centre, pas de dépassement |
| **Forteresse** | 2 pions de même couleur sur la même case = imprenable |
| **Cases sûres (★)** | Cases 0, 8, 13, 21, 26, 34, 39, 47 — aucune capture |
| **Couloir final** | Accessible uniquement pour la couleur correspondante |

### 8.5 Intelligence Artificielle

| Niveau | Comportement | Score calcul |
|--------|-------------|--------------|
| 😊 Facile | Aléatoire parmi les coups valides | — |
| 🤖 Normal | Optimise : Arrivée > Capture > Avancement > Sortie | Position + 400pts/capture |
| 🧠 Expert | Idem + évite les cases dangereuses proches d'adversaires | Position + 600pts/capture − 80pts/danger |

---

## 9. Flux Financier — Pas à Pas

### 9.1 Dépôt

```
Joueur                  App Android            Backend              WonyaPay
  │                         │                     │                     │
  │── Saisit montant ──────►│                     │                     │
  │   téléphone, opérateur  │                     │                     │
  │                         │── POST /payment ───►│                     │
  │                         │     /deposit        │── POST /collect ───►│
  │                         │                     │◄── {reference} ─────│
  │                         │◄── {txId, ref} ─────│                     │
  │◄── "Confirmez sur       │                     │                     │
  │     votre téléphone" ───│                     │                     │
  │                         │                     │                     │
  │── Confirme sur M-Pesa ──────────────────────────────────────────────│
  │                         │                     │◄── Callback ────────│
  │                         │                     │    {status:success} │
  │                         │                     │                     │
  │                         │                     │ Crédite le wallet   │
  │                         │                     │ Sauvegarde tx       │
  │◄── Notification solde ──│◄── Solde mis à jour─│                     │
```

**Statuts possibles :**
- `pending` — En attente de confirmation du client
- `success` — Paiement confirmé, solde crédité
- `failed` — Échec (solde insuffisant, timeout…)

### 9.2 Retrait

```
Joueur                  App Android            Backend              WonyaPay
  │                         │                     │                     │
  │── Demande retrait ─────►│                     │                     │
  │                         │── POST /payment ───►│                     │
  │                         │     /withdraw       │ Débite wallet       │
  │                         │                     │── POST /disburse ──►│
  │                         │◄── {txId, ref} ─────│◄── {reference} ─────│
  │                         │                     │                     │
  │                         │                     │◄── Callback ────────│
  │                         │                     │    {status:success} │
  │◄── Reçoit l'argent ─────────────────────────────────────────────────│
  │    sur son téléphone    │                     │                     │
```

**Sécurité :** En cas d'échec du retrait, le montant est automatiquement remboursé sur le wallet.

---

## 10. Flux Compétition — Pas à Pas

```
ADMIN                   Backend              JOUEURS (1-4)
  │                        │                     │
  │─ Crée compétition ────►│                     │
  │  (titre, mise, distrib)│ status: "open"      │
  │                        │                     │
  │                        │◄─ Rejoindre ────────│ (x4)
  │                        │  (userId, couleur)  │
  │                        │ Débite entryFee     │
  │                        │ status: "full"      │
  │                        │                     │
  │                        │──► Notif Socket.IO ─►│
  │                        │    "game_start"      │
  │                        │                     │
  │                    ┌───┴──────────────────────┴───┐
  │                    │   PARTIE EN COURS (temps-réel)│
  │                    │   Socket.IO : dice_roll,      │
  │                    │   piece_move, game_over       │
  │                    └───┬──────────────────────┬───┘
  │                        │                     │
  │                        │◄─ POST /result ─────│ (serveur/admin)
  │                        │  [{userId, rank}]   │
  │                        │                     │
  │                        │ Distribution gains  │
  │                        │ 1er → 60% prizePool │
  │                        │ 2ème → 30%          │
  │                        │ 3ème → 10%          │
  │                        │ Crédite les wallets │
  │                        │──► Notif résultats ─►│
```

### Calcul du Prize Pool

```
Exemple : 4 joueurs × 500 CDF de mise

Cagnotte brute    = 4 × 500      = 2 000 CDF
Frais plateforme  = 2 000 × 10%  =   200 CDF  (revenus de la plateforme)
Prize Pool net    = 2 000 - 200  = 1 800 CDF

Distribution :
  🥇 1er  (60%) = 1 800 × 0.60 = 1 080 CDF
  🥈 2ème (30%) = 1 800 × 0.30 =   540 CDF
  🥉 3ème (10%) = 1 800 × 0.10 =   180 CDF
  4ème    ( 0%) =   0 CDF (ne gagne rien)
```

---

## 11. Sécurité

### 11.1 Authentification
- Tokens **JWT** (HS256) avec expiration 7 jours
- Mots de passe hashés avec **bcrypt** (12 rounds)
- Rate limiting : 10 tentatives de connexion / 15 min

### 11.2 Paiements
- Callbacks WonyaPay vérifiés via **signature HMAC-SHA256**
- Chaque callback vérifié avant toute modification du solde
- Solde bloqué immédiatement à la demande de retrait
- Remboursement automatique en cas d'échec du retrait

### 11.3 APK Android
- Signature **RSA 2048 éphémère** : clé générée dans le runner CI, utilisée, puis supprimée
- Aucun keystore ou secret stocké dans le repository
- Conforme pour la distribution APK directe

### 11.4 Backend
- **Helmet.js** : headers de sécurité HTTP
- **CORS** : origines whitelist configurables
- **Rate limiting** : 100 requêtes / 15 min par IP
- Variables sensibles dans `.env` (jamais dans le code)

---

## 12. Variables d'Environnement — Référence Complète

Fichier : `backend/.env` (copier depuis `backend/.env.example`)

```env
# ── SERVEUR ──────────────────────────────────────────────────
PORT=3000
NODE_ENV=production          # development | production

# ── MONGODB ──────────────────────────────────────────────────
MONGO_URI=mongodb+srv://USER:PASSWORD@cluster.mongodb.net/ludomaster

# ── JWT ──────────────────────────────────────────────────────
JWT_SECRET=au_moins_32_caracteres_aleatoires_ici_XXXXXXXXXXX
JWT_EXPIRES_IN=7d

# ── WONYAPAY ─────────────────────────────────────────────────
WONYAPAY_BASE_URL=https://api.wonyasoft.com/v1
WONYAPAY_API_KEY=sk_live_xxxxxxxxxxxxxxxxxxxxxxxx
WONYAPAY_SECRET=secret_xxxxxxxxxxxxxxxxxxxxxxxx
WONYAPAY_MERCHANT_ID=MCH_xxxxxxxxxx
WONYAPAY_CALLBACK_URL=https://votre-domaine.com/api/payment/callback

# ── ADMIN ────────────────────────────────────────────────────
ADMIN_EMAIL=admin@votreapp.cd
ADMIN_PASSWORD=MotDePasseTresSecurise123!

# ── RÈGLES FINANCIÈRES ───────────────────────────────────────
PLATFORM_FEE_PERCENT=10      # % prélevé sur chaque prize pool
MIN_DEPOSIT=500               # CDF minimum par dépôt
MAX_DEPOSIT=1000000           # CDF maximum par dépôt
MIN_BET=200                   # CDF minimum pour rejoindre un tournoi
MAX_BET=50000                 # CDF maximum par tournoi

# ── ORIGINES AUTORISÉES (CORS) ───────────────────────────────
ALLOWED_ORIGINS=https://admin.votreapp.cd,https://votreapp.cd
```

---

## 13. API REST — Référence des Endpoints

### Auth
| Méthode | Endpoint | Description | Auth |
|---------|----------|-------------|------|
| POST | `/api/auth/register` | Créer un compte | — |
| POST | `/api/auth/login` | Se connecter | — |
| GET | `/api/auth/me` | Profil courant | JWT |
| PATCH | `/api/auth/momo` | Config Mobile Money | JWT |

### Paiement
| Méthode | Endpoint | Description | Auth |
|---------|----------|-------------|------|
| POST | `/api/payment/deposit` | Initier un dépôt | JWT |
| POST | `/api/payment/withdraw` | Initier un retrait | JWT |
| GET | `/api/payment/status/:ref` | Statut d'une transaction | JWT |
| GET | `/api/payment/history` | Historique | JWT |
| POST | `/api/payment/callback/deposit` | Webhook WonyaPay (dépôt) | Signature |
| POST | `/api/payment/callback/withdraw` | Webhook WonyaPay (retrait) | Signature |

### Compétitions
| Méthode | Endpoint | Description | Auth |
|---------|----------|-------------|------|
| GET | `/api/competitions` | Liste des tournois ouverts | JWT |
| GET | `/api/competitions/:id` | Détail d'un tournoi | JWT |
| POST | `/api/competitions/:id/join` | Rejoindre un tournoi | JWT |
| POST | `/api/competitions` | Créer un tournoi | Admin |
| POST | `/api/competitions/:id/result` | Soumettre les résultats | Admin |
| PATCH | `/api/competitions/:id` | Modifier un tournoi | Admin |
| DELETE | `/api/competitions/:id` | Annuler + rembourser | Admin |

### Admin
| Méthode | Endpoint | Description |
|---------|----------|-------------|
| GET | `/api/admin/dashboard` | KPIs généraux |
| GET | `/api/admin/users` | Liste des joueurs |
| PATCH | `/api/admin/users/:id/wallet` | Ajuster solde |
| PATCH | `/api/admin/users/:id/status` | Activer/Désactiver |
| GET | `/api/admin/reports/daily` | Rapport journalier |
| GET | `/api/admin/reports/top-players` | Classement gains |

---

## 14. Dépannage

### ❌ `chmod: cannot access 'gradlew'`
```bash
# Vérifier que gradlew existe
ls android/gradlew

# Le recréer si absent
cat > android/gradlew << 'EOF'
#!/bin/sh
APP_HOME=$(cd "$(dirname "$0")" && pwd)
CLASSPATH=$APP_HOME/gradle/wrapper/gradle-wrapper.jar
exec "${JAVA_HOME:-}/bin/java" -classpath "$CLASSPATH" org.gradle.wrapper.GradleWrapperMain "$@"
EOF
chmod +x android/gradlew
```

### ❌ `WonyaPay: Signature invalide`
- Vérifier que `WONYAPAY_SECRET` est identique dans `.env` et dans votre portail WonyaSoft
- S'assurer que le body du callback n'est pas modifié avant la vérification
- Utiliser `express.json({ strict: false })` pour éviter les problèmes de parsing

### ❌ `MongoDB: Authentication failed`
- Vérifier le nom d'utilisateur et mot de passe dans l'URI
- Confirmer que l'IP du serveur est dans la whitelist Network Access
- Tester l'URI avec MongoDB Compass

### ❌ `Solde pas crédité après un dépôt`
1. Vérifier les logs du callback : `GET /api/payment/history`
2. Vérifier que l'URL de callback est accessible depuis internet
3. Tester avec ngrok en développement : `ngrok http 3000`

### ❌ `L'APK refuse de s'installer`
- Désactiver **Google Play Protect** temporairement
- Vérifier que "Sources inconnues" est activé pour le gestionnaire de fichiers utilisé
- Essayer d'installer via ADB : `adb install LudoMasterPro.apk`

### ❌ `Le jeu se bloque / pion ne bouge pas`
- Vérifier les logs du `GameViewModel` (Android Studio Logcat)
- S'assurer que `applyMove()` est appelé à la fin de chaque animation
- Vérifier que `LudoRules.animPath()` retourne un tableau non vide

---

## 📞 Support

Pour obtenir de l'aide sur WonyaPay :
- 🌐 https://wonyasoft.com
- 📧 Contacter le support WonyaSoft via leur portail développeur

---

*Document généré pour Ludo Master Pro — Édition Bookmaker*
*Kotlin · Jetpack Compose · Node.js · WonyaPay · MongoDB*
