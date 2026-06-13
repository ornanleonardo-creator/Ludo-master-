# 🚀 Déploiement Backend sur Render.com

> Guide pas à pas pour déployer le backend Ludo Master Pro
> sur **Render.com** (gratuit au démarrage)

---

## 1. Préparer le repo GitHub

### Structure attendue sur GitHub

```
ton-repo/
├── backend/          ← dossier du backend Node.js
│   ├── package.json
│   ├── .env.example
│   └── src/
│       └── server.js
├── android/
├── admin/
└── README.md
```

### Pousser sur GitHub

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/TON_USERNAME/ludo-master-pro.git
git push -u origin main
```

---

## 2. Créer le compte Render

1. Aller sur **https://render.com**
2. Cliquer **Get Started for Free**
3. Se connecter avec **GitHub** (recommandé)

---

## 3. Créer le Web Service

1. Dashboard Render → **New +** → **Web Service**
2. Connecter ton repo GitHub : `ludo-master-pro`
3. Configurer :

| Paramètre | Valeur |
|-----------|--------|
| **Name** | `ludo-master-api` |
| **Region** | `Frankfurt (EU Central)` — plus proche de Kinshasa |
| **Branch** | `main` |
| **Root Directory** | `backend` |
| **Runtime** | `Node` |
| **Build Command** | `npm install` |
| **Start Command** | `node src/server.js` |
| **Plan** | `Free` (512 MB RAM) |

4. Cliquer **Create Web Service**

---

## 4. Ajouter les Variables d'Environnement

Dans le dashboard du service → onglet **Environment** → **Add Environment Variable**

Ajouter une par une :

```
NODE_ENV              = production
PORT                  = 3000
MONGO_URI             = mongodb+srv://USER:PASS@cluster.mongodb.net/ludomaster
JWT_SECRET            = mettre_ici_une_chaine_aleatoire_de_32_caracteres
JWT_EXPIRES_IN        = 7d
WONYAPAY_BASE_URL     = https://api.wonyasoft.com/v1
WONYAPAY_API_KEY      = sk_live_xxxxxxxxxx
WONYAPAY_SECRET       = secret_xxxxxxxxxx
WONYAPAY_MERCHANT_ID  = MCH_xxxxxxxxxx
WONYAPAY_CALLBACK_URL = https://ludo-master-api.onrender.com/api/payment/callback
PLATFORM_FEE_PERCENT  = 10
MIN_DEPOSIT           = 500
MAX_DEPOSIT           = 1000000
MIN_BET               = 200
MAX_BET               = 50000
```

> ⚠️ **Remplace `ludo-master-api` par le nom exact** que Render t'a attribué.
> L'URL finale ressemblera à : `https://ludo-master-api.onrender.com`

---

## 5. Générer un JWT_SECRET sécurisé

Dans n'importe quel terminal (ou https://codebeautify.org/generate-random-string) :

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
# Exemple de sortie :
# a3f8c2d1e4b5a9f0c3d2e1b4a5f8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6
```

Copier cette valeur dans `JWT_SECRET`.

---

## 6. Ajouter render.yaml (déploiement automatique)

Créer ce fichier à la **racine du repo** pour tout configurer en une fois :

```yaml
# render.yaml — à la racine du repo
services:
  - type: web
    name: ludo-master-api
    runtime: node
    rootDir: backend
    buildCommand: npm install
    startCommand: node src/server.js
    region: frankfurt
    plan: free
    envVars:
      - key: NODE_ENV
        value: production
      - key: PORT
        value: 3000
      # Les secrets sont ajoutés manuellement dans le dashboard
      - key: MONGO_URI
        sync: false
      - key: JWT_SECRET
        sync: false
      - key: WONYAPAY_API_KEY
        sync: false
      - key: WONYAPAY_SECRET
        sync: false
      - key: WONYAPAY_MERCHANT_ID
        sync: false
      - key: WONYAPAY_CALLBACK_URL
        sync: false
      - key: PLATFORM_FEE_PERCENT
        value: "10"
```

---

## 7. Corriger le port pour Render

Render utilise la variable `PORT` automatiquement. Vérifier dans `backend/src/server.js` :

```javascript
// ✅ Déjà correct dans notre server.js :
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log(`🚀 Port ${PORT}`));
```

---

## 8. Mettre à jour l'URL dans l'app Android

Après le premier déploiement, copier l'URL Render :
```
https://ludo-master-api.onrender.com
```

Dans `android/app/src/main/java/com/ludomasterpro/MainActivity.kt`,
remplacer la ligne de config API :

```kotlin
// Ligne à modifier dans LudoApp ou un fichier Config.kt
const val API_BASE = "https://ludo-master-api.onrender.com/api"
```

---

## 9. Créer le compte Admin après déploiement

Une fois le service déployé et actif :

```bash
curl -X POST https://ludo-master-api.onrender.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "email": "admin@ludomaster.cd",
    "password": "TonMotDePasseAdmin123!",
    "phone": "+243812345678"
  }'
```

Puis dans **MongoDB Atlas** → **Browse Collections** → collection `users` :
```json
{ "$set": { "role": "admin" } }
```
sur l'utilisateur admin.

---

## 10. Dashboard Admin sur Render

L'admin web (`admin/index.html`) peut être servi comme **Static Site** sur Render :

1. **New +** → **Static Site**
2. Repo : même repo
3. Root Directory : `admin`
4. Build Command : *(vide)*
5. Publish Directory : `.`

URL admin : `https://ludo-master-admin.onrender.com`

Dans `admin/index.html`, changer la ligne :
```javascript
const API = localStorage.getItem('ludo_api')
         || 'https://ludo-master-api.onrender.com/api';
```

---

## 11. Plan Free — Limitations importantes

| Limitation | Détail |
|-----------|--------|
| **Sleep après 15 min** | Le service s'endort si inactif |
| **Réveil ~30 secondes** | Premier appel lent après sleep |
| **750h/mois** | Suffisant pour 1 service en continu |
| **512 MB RAM** | Suffisant pour notre backend |
| **Logs** | 7 jours de rétention |

### Éviter le sleep (optionnel)

Ajouter un **cron job** gratuit qui ping le serveur toutes les 14 minutes :

```bash
# Sur un service externe (cron-job.org — gratuit)
URL  : https://ludo-master-api.onrender.com/health
Every: 14 minutes
```

---

## 12. Upgrade vers le plan payant

Quand le jeu grossit :

| Plan | Prix | RAM | CPU | Sleep |
|------|------|-----|-----|-------|
| Free | 0$ | 512 MB | 0.1 | Oui |
| Starter | 7$/mois | 512 MB | 0.5 | Non |
| Standard | 25$/mois | 2 GB | 1 | Non |

---

## 13. Vérification du déploiement

```bash
# Health check
curl https://ludo-master-api.onrender.com/health
# Réponse attendue : {"status":"ok","env":"production"}

# Test inscription
curl -X POST https://ludo-master-api.onrender.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@test.com","password":"Test1234!","phone":"+243812345678"}'
```

---

## Récapitulatif des URLs

| Service | URL |
|---------|-----|
| **API Backend** | `https://ludo-master-api.onrender.com` |
| **Admin Dashboard** | `https://ludo-master-admin.onrender.com` |
| **Health Check** | `https://ludo-master-api.onrender.com/health` |
| **WonyaPay Callback Dépôt** | `https://ludo-master-api.onrender.com/api/payment/callback/deposit` |
| **WonyaPay Callback Retrait** | `https://ludo-master-api.onrender.com/api/payment/callback/withdraw` |

---

*Render.com — Agréé pour applications Node.js/Express*
*Hébergement européen (Frankfurt) — RGPD compatible*
