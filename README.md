# 🎲 Ludo Master Pro — Édition Bookmaker

> Kotlin · Jetpack Compose · Canvas 3D · WonyaPay · Tournois en argent réel

---

## 🏗️ Structure du projet

```
LudoMasterPro/
├── android/                    ← Application Android (Kotlin + Compose)
│   ├── app/src/main/java/com/ludomasterpro/
│   │   ├── engine/
│   │   │   ├── LudoEngine.kt       ← Règles, IA, mécanique corrigée
│   │   │   └── GameViewModel.kt    ← MVVM + StateFlow + DataStore
│   │   ├── ui/
│   │   │   ├── theme/Theme.kt
│   │   │   ├── components/
│   │   │   │   ├── BoardCanvas.kt  ← Plateau 3D Canvas natif
│   │   │   │   └── DiceView.kt     ← Dé doré animé + haptique
│   │   │   └── screens/
│   │   │       ├── MenuScreen.kt           ← Accueil Solo/Tournoi
│   │   │       ├── AuthWalletLobby.kt      ← Auth + Portefeuille + Lobby
│   │   │       └── GameAndPodiumScreens.kt ← Jeu + Podium
│   │   └── MainActivity.kt
│   └── .github/workflows/build-apk.yml
│
├── backend/                    ← API Node.js Express
│   └── src/
│       ├── models/   (User, Transaction, Competition)
│       ├── routes/   (auth, payment, competition, admin)
│       ├── services/ (wonyapay.js)
│       ├── middleware/(auth.js)
│       └── server.js
│
├── admin/
│   └── index.html             ← Dashboard admin web complet
│
└── README.md
```

---

## 🎮 Mécanique du jeu (corrigée)

| Règle | Implémentation |
|-------|---------------|
| Sortie de base | Dé = 6 uniquement |
| Rejouer | Dé = 6 OU capture réussie |
| Arrivée exacte | Entrée au centre = exacte sinon rebond |
| Cases sûres (★) | Pas de capture possible |
| Forteresse | 2 pions même couleur = imprenable |
| IA | 3 niveaux : Facile / Normal / Expert |

---

## 💰 Système Bookmaker (WonyaPay)

```
Joueur dépose (M-Pesa / Orange / Airtel)
         ↓
    Wallet CDF
         ↓
    Rejoindre tournoi (entryFee débité)
         ↓
    Jouer la partie (temps réel via Socket.IO)
         ↓
    Résultats → Distribution automatique
    1er 60% + 2ème 30% + 3ème 10%
    (- 10% frais plateforme)
         ↓
    Retrait vers Mobile Money
```

### Flux financier
- **Dépôt** : POST /api/payment/deposit → WonyaPay Collect
- **Retrait** : POST /api/payment/withdraw → WonyaPay Disbursement  
- **Callbacks** : /api/payment/callback/deposit | /callback/withdraw
- **Signature** : HMAC-SHA256 sur chaque requête WonyaPay

---

## 🚀 Déploiement Backend

```bash
cd backend
cp .env.example .env     # Remplir MONGO_URI, WONYAPAY_*, JWT_SECRET
npm install
npm start
```

**Variables requises dans `.env` :**

| Variable | Description |
|----------|-------------|
| `MONGO_URI` | MongoDB Atlas connection string |
| `WONYAPAY_API_KEY` | Clé API WonyaSoft |
| `WONYAPAY_SECRET` | Secret pour signature HMAC |
| `WONYAPAY_MERCHANT_ID` | ID marchand WonyaSoft |
| `WONYAPAY_CALLBACK_URL` | URL publique de votre serveur |
| `JWT_SECRET` | Secret JWT (32+ chars) |
| `PLATFORM_FEE_PERCENT` | Frais plateforme (défaut: 10) |

---

## 📱 Build APK (GitHub Actions)

```bash
# 1. Push → APK généré automatiquement dans Actions > Artifacts
git push origin main

# 2. Release officielle
git tag v1.0.0 && git push origin v1.0.0
```

**Signature éphémère** : Clé RSA 2048 auto-générée dans le runner CI,  
utilisée pour signer l'APK, puis immédiatement supprimée. Zéro secret à gérer.

---

## 🖥️ Dashboard Admin

Ouvrir `admin/index.html` dans le navigateur.

**Fonctionnalités :**
- 📊 KPIs : revenus, dépôts, retraits, utilisateurs
- 🏆 Créer/gérer des compétitions avec distribution des gains
- 👥 Gérer les joueurs (activer/désactiver, ajuster solde)
- 💰 Voir toutes les transactions
- 📈 Rapports par opérateur Mobile Money
- ⚡ Annuler une compétition avec remboursement automatique

---

## 🤖 Intelligence Artificielle

| Niveau | Stratégie |
|--------|-----------|
| 😊 Facile | Aléatoire |
| 🤖 Normal | Priorité : Arrivée > Capture > Avancement > Sortie |
| 🧠 Expert | Idem + évite les cases à portée d'adversaires |

---

## 📱 Compatibilité Android

- minSdk : 24 (Android 7.0+)
- targetSdk : 35 (Android 15)
- Architecture : arm64-v8a, armeabi-v7a, x86_64
