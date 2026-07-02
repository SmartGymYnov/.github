# SmartGym — Documentation fonctionnelle & technique

> Projet fil rouge B3 DEV — Ynov Informatique  
> Gestion complète d'une salle de sport : réservations, check-in, abonnements, suivi d'entraînement.

---

## Sommaire

1. [Présentation du projet](#1-présentation-du-projet)
2. [Architecture globale](#2-architecture-globale)
3. [Backend — API REST (.NET 10)](#3-backend--api-rest-net-10)
4. [Frontend Web — Dashboard Admin (React)](#4-frontend-web--dashboard-admin-react)
5. [Application Mobile (React Native / Expo)](#5-application-mobile-react-native--expo)
6. [Application Scanner (Expo)](#6-application-scanner-expo)
7. [Base de données](#7-base-de-données)
8. [Installation & lancement](#8-installation--lancement)
9. [Variables d'environnement](#9-variables-denvironnement)

---

## 1. Présentation du projet

**SmartGym** est une solution numérique complète pour la gestion d'une salle de sport, composée de **4 applications indépendantes** qui communiquent toutes via une API REST centralisée.

### Besoin métier couvert

| Fonctionnalité | Public cible |
|---|---|
| Inscription, connexion sécurisée | Tous |
| Consultation et réservation de séances | Membres (mobile) |
| Check-in par QR code à l'entrée | Membres (mobile) + Staff (scanner) |
| Gestion des abonnements + paiement Stripe | Membres (mobile) |
| Suivi d'entraînement (style Hevy) | Membres (mobile) |
| Dashboard statistiques, gestion membres/cours | Admins (web) |
| Vue planning et liste des présents | Coachs (web) |
| Scan QR de validation à l'accueil | Staff (scanner) |

### Équipe & rôles

- **Admin** : accès complet (dashboard, membres, abonnements, cours, réservations, check-ins)
- **Coach** : accès au planning et à ses séances / réservations
- **Client** : accès à l'application mobile uniquement

---

## 2. Architecture globale

```
┌─────────────────────────────────────────────────────────────────┐
│                        SMARTGYM ECOSYSTEM                       │
│                                                                 │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│   │  Mobile App  │   │  Web Admin   │   │  Scanner App     │   │
│   │ React Native │   │    React     │   │  Expo (camera)   │   │
│   │    (Expo)    │   │    + Vite    │   │  Staff only      │   │
│   └──────┬───────┘   └──────┬───────┘   └────────┬─────────┘   │
│          │                  │                    │              │
│          └──────────────────┼────────────────────┘              │
│                             │  HTTPS + JWT Bearer               │
│                             ▼                                   │
│              ┌──────────────────────────┐                       │
│              │     API REST — .NET 10   │                       │
│              │     Clean Architecture  │                       │
│              │  Domain / App / Infra   │                       │
│              └──────────────┬───────────┘                       │
│                             │  EF Core                          │
│                             ▼                                   │
│              ┌──────────────────────────┐                       │
│              │      SQL Server          │                       │
│              │  10 tables, migrations   │                       │
│              └──────────────────────────┘                       │
│                                                                 │
│              ┌──────────────────────────┐                       │
│              │         Stripe           │                       │
│              │  Checkout + Portal +     │                       │
│              │  Webhooks                │                       │
│              └──────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### Stack technologique

| Couche | Technologie |
|---|---|
| Backend API | .NET 10, ASP.NET Core, EF Core |
| Frontend Web | React 19, Vite, TanStack Query, Zustand, Framer Motion |
| Mobile | React Native 0.81, Expo 54, React Navigation |
| Scanner | Expo 54, expo-camera (CameraView) |
| Base de données | SQL Server (EF Core migrations) |
| Auth | JWT Bearer, BCrypt |
| Paiement | Stripe (Checkout + Customer Portal + Webhooks) |
| Versioning | Git + GitHub |

---

## 3. Backend — API REST (.NET 10)

### Structure du projet

```
backend/
├── SmartGym.Domain/           # Entités, interfaces, enums — aucune dépendance
│   ├── Entities/
│   └── Interfaces/
├── SmartGym.Application/      # Services, DTOs, validateurs (FluentValidation)
│   ├── DTOs/
│   ├── Interfaces/
│   └── Services/
├── SmartGym.Infrastructure/   # EF Core, repositories, migrations, Stripe, BCrypt
│   ├── Data/
│   │   ├── Configurations/
│   │   └── Migrations/
│   ├── Repositories/
│   └── Services/
└── SmartGym.API/              # Controllers, middleware, Program.cs
    ├── Controllers/
    └── Middleware/
```

### Architecture — Clean Architecture

Le backend respecte les principes **SOLID** et **Clean Architecture** :

- **Domain** : entités métier pures, interfaces de repository → zéro dépendance externe
- **Application** : logique métier, orchestration, DTOs → dépend uniquement de Domain
- **Infrastructure** : implémentation concrète (EF Core, BCrypt, Stripe) → dépend de Application
- **API** : exposition HTTP, injection de dépendances → dépend de Application

### Authentification & Sécurité

- **JWT Bearer tokens** — durée de vie : 8 heures
- **BCrypt** pour le hashage des mots de passe
- **Middleware global** de gestion d'erreurs → retourne des `ApiResponse<T>` standardisés
- **CORS** configuré pour autoriser le frontend web
- **Policies d'autorisation** :
  - `RequireAdminRole` → Admin uniquement
  - `RequireCoachRole` → Coach ou Admin

### Endpoints API

#### Auth — `/api/auth`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Créer un compte |
| POST | `/api/auth/login` | Public | Connexion, retourne JWT |

#### Utilisateurs — `/api/users`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/users` | Admin | Liste tous les membres |
| GET | `/api/users/{id}` | Authentifié | Détail d'un utilisateur |
| PUT | `/api/users/{id}` | Authentifié | Modifier son profil |
| DELETE | `/api/users/{id}` | Admin | Supprimer un membre |

#### Cours — `/api/courses`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/courses` | Authentifié | Liste tous les cours |
| GET | `/api/courses/{id}` | Authentifié | Détail d'un cours |
| POST | `/api/courses` | Coach / Admin | Créer un cours |
| PUT | `/api/courses/{id}` | Coach / Admin | Modifier un cours |
| DELETE | `/api/courses/{id}` | Admin | Supprimer un cours |

#### Séances — `/api/sessions`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/sessions` | Authentifié | Prochaines séances (> maintenant) |
| GET | `/api/sessions/today` | Authentifié | Séances du jour |
| GET | `/api/sessions/{id}` | Authentifié | Détail d'une séance |
| POST | `/api/sessions` | Coach / Admin | Créer une séance |
| PUT | `/api/sessions/{id}` | Coach / Admin | Modifier une séance |
| DELETE | `/api/sessions/{id}` | Admin | Supprimer une séance |

#### Réservations — `/api/bookings`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/bookings` | Admin | Toutes les réservations |
| GET | `/api/bookings/user/{userId}` | Authentifié | Réservations d'un membre |
| GET | `/api/bookings/coach/{coachId}` | Coach / Admin | Réservations d'un coach |
| POST | `/api/bookings` | Authentifié | Créer une réservation |
| DELETE | `/api/bookings/{id}` | Authentifié | Annuler une réservation |

#### Check-ins — `/api/checkins`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/checkins/today` | Admin | Check-ins du jour |
| POST | `/api/checkins/generate-qr` | Authentifié | Générer un token QR (valide 60 s) |
| POST | `/api/checkins/validate-qr` | Authentifié | Valider un token QR et enregistrer l'entrée |

#### Abonnements — `/api/subscriptions`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/subscriptions` | Admin | Tous les abonnements (avec nom membre) |
| GET | `/api/subscriptions/me` | Authentifié | Mon abonnement |
| POST | `/api/subscriptions` | Admin | Créer un abonnement |
| PUT | `/api/subscriptions/{id}` | Admin | Modifier un abonnement |
| DELETE | `/api/subscriptions/{id}` | Admin | Supprimer un abonnement |
| POST | `/api/subscriptions/checkout` | Authentifié | Créer session Stripe Checkout |
| POST | `/api/subscriptions/portal` | Authentifié | Créer session Stripe Customer Portal |
| GET | `/api/subscriptions/success` | Public | Page de succès Stripe |
| GET | `/api/subscriptions/cancel` | Public | Page d'annulation Stripe |

#### Entraînements — `/api/workoutlogs`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/workoutlogs/user/{userId}` | Authentifié | Historique des workouts |
| POST | `/api/workoutlogs` | Authentifié | Créer un log d'entraînement |
| DELETE | `/api/workoutlogs/{id}` | Authentifié | Supprimer un log |
| GET | `/api/workoutlogs/{logId}/exercises` | Authentifié | Exercices d'un log |
| POST | `/api/workoutlogs/{logId}/exercises` | Authentifié | Ajouter un exercice |
| PUT | `/api/workoutlogs/{logId}/exercises/{exerciseId}` | Authentifié | Modifier un exercice |
| DELETE | `/api/workoutlogs/{logId}/exercises/{exerciseId}` | Authentifié | Supprimer un exercice |

#### Exercices — `/api/exercises`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/exercises` | Authentifié | Liste (filtre optionnel `?muscleGroup=`) |
| GET | `/api/exercises/{id}` | Authentifié | Détail |
| POST | `/api/exercises` | Admin | Créer un exercice |
| DELETE | `/api/exercises/{id}` | Admin | Supprimer un exercice |

#### Statistiques — `/api/stats`

| Méthode | Route | Accès | Description |
|---|---|---|---|
| GET | `/api/stats/dashboard` | Admin | KPIs : membres, séances du jour, check-ins, taux de remplissage, courbe 7 jours |

### Format de réponse standard

Toutes les réponses suivent le format `ApiResponse<T>` :

```json
{
  "success": true,
  "message": "Opération réussie",
  "data": { ... },
  "errors": []
}
```

---

## 4. Frontend Web — Dashboard Admin (React)

### Structure du projet

```
frontend/
├── src/
│   ├── features/
│   │   ├── auth/          # Login, Register
│   │   ├── dashboard/     # KPIs, graphiques
│   │   ├── members/       # Gestion membres
│   │   ├── courses/       # Gestion des cours
│   │   ├── sessions/      # Planning des séances
│   │   ├── bookings/      # Liste des réservations
│   │   ├── checkin/       # Check-ins du jour
│   │   ├── subscriptions/ # Gestion abonnements
│   │   ├── coach/         # Vue coach (ses réservations)
│   │   └── profile/       # Profil utilisateur
│   ├── services/          # Appels API (axios)
│   ├── store/             # Zustand (auth persisté localStorage)
│   ├── styles/            # CSS modules globaux
│   └── types/             # Interfaces TypeScript
```

### Stack

| Package | Version | Rôle |
|---|---|---|
| React | 19 | UI |
| Vite | 8 | Build & dev server |
| TypeScript | 6 | Typage |
| TanStack Query | 5 | Fetching, cache, mutations |
| Zustand | 5 | State auth persisté |
| Framer Motion | 12 | Animations |
| Axios | 1 | Client HTTP |

### Pages & accès

| URL | Page | Rôle requis |
|---|---|---|
| `/login` | Connexion | Public |
| `/register` | Inscription | Public |
| `/dashboard` | KPIs & graphiques | Admin |
| `/members` | Liste des membres | Admin |
| `/courses` | Gestion des cours | Admin |
| `/sessions` | Planning des séances | Admin / Coach |
| `/bookings` | Toutes les réservations | Admin |
| `/coach/bookings` | Réservations du coach | Coach / Admin |
| `/checkin` | Check-ins du jour (tableau) | Admin |
| `/subscriptions` | Gestion abonnements | Admin |
| `/profile` | Mon profil | Tous |

### Fonctionnalités principales

- **Dashboard** : nombre total de membres, séances du jour, taux de remplissage moyen, courbe des réservations sur 7 jours
- **Membres** : tableau paginé, recherche, suppression
- **Cours** : CRUD complet avec assignation à un coach
- **Sessions** : planning avec filtres, création de créneaux
- **Réservations** : vue globale avec filtres par date / cours
- **Check-ins** : liste en temps réel des entrées du jour avec nom du membre
- **Abonnements** : tableau avec progression, statut (actif / expire bientôt / expiré), nom du membre (JOIN), filtre par plan
- **Auth** : JWT stocké en `localStorage`, rôle vérifié à chaque route protégée

---

## 5. Application Mobile (React Native / Expo)

### Structure du projet

```
mobile/
└── src/
    ├── api/               # Modules axios par domaine (auth, sessions, bookings…)
    ├── components/
    │   └── ui/            # AppText, LoadingState, ErrorState…
    ├── context/           # AuthContext (expo-secure-store)
    ├── navigation/
    │   ├── RootNavigator.tsx
    │   ├── AppNavigator.tsx   # Bottom tabs
    │   └── types.ts
    ├── screens/           # 14 écrans
    ├── theme/             # Couleurs, espacement, radius, ombres
    ├── types/             # Interfaces TypeScript
    └── utils/             # JWT parser, formatters
```

### Navigation

```
RootNavigator (NativeStack)
│
├── [Auth non connecté]
│   ├── LoginScreen
│   └── RegisterScreen
│
└── [Auth connecté]
    ├── AppTabs (Bottom Tab Navigator)
    │   ├── Home          → Accueil / séances du jour
    │   ├── Courses       → Catalogue des cours
    │   ├── CheckIn       → QR code d'entrée
    │   ├── Reservations  → Mes réservations
    │   ├── Workout       → Historique d'entraînement
    │   └── Profile       → Profil & abonnement
    │
    ├── CourseDetail      → Détail cours + séances disponibles
    ├── SessionDetail     → Détail séance + bouton réserver
    ├── StripeWebView     → WebView Stripe (paiement / portail)
    └── ActiveWorkout     → Séance d'entraînement en cours (modal)
```

### Écrans détaillés

#### HomeScreen
- Séances disponibles du jour (filtrées côté API)
- Accès rapide : Check-in, Mes réservations
- Message de bienvenue personnalisé

#### CoursesScreen
- Liste de tous les cours avec recherche par nom
- Carte par cours : nom, durée, coach, niveau de difficulté

#### CourseDetailScreen
- Informations complètes du cours
- Liste des prochaines séances avec taux de remplissage
- Bouton de réservation par séance

#### SessionDetailScreen
- Détail de la séance (horaire, places restantes, coach)
- Bouton Réserver / Annuler selon l'état

#### ReservationsScreen
- Onglets Upcoming / Passées
- Annulation possible sur les réservations futures

#### CheckInScreen
- Génération automatique d'un QR code JWT (valide 60 secondes)
- Compte à rebours + régénération automatique
- Affiché à l'accueil pour scan par le staff

#### WorkoutScreen
- Historique des séances d'entraînement
- Carte par workout : date, nombre d'exercices, volume total (kg)
- Bouton "Démarrer un entraînement"

#### ActiveWorkoutScreen
- Chronomètre en temps réel
- Ajout d'exercices via un sélecteur (recherche + filtre par groupe musculaire)
- Saisie des séries / répétitions / poids avec rappel du précédent best
- Validation de chaque série avec toggle ✓
- Suppression de série par appui long
- Enregistrement du workout complet à la fin

#### ProfileScreen
- Informations personnelles avec modification
- Carte abonnement (plan, jours restants, statut Stripe)
- Accès au portail Stripe pour gérer l'abonnement
- Déconnexion

#### StripeWebViewScreen
- WebView in-app pour le parcours de paiement Stripe
- Détection automatique de l'URL de retour et fermeture de la WebView

### Sécurité mobile

- Token JWT stocké dans **expo-secure-store** (chiffré natif iOS Keychain / Android Keystore)
- Intercepteur Axios qui injecte automatiquement le Bearer token
- Coachs et Admins bloqués à la connexion (application réservée aux membres)

---

## 6. Application Scanner (Expo)

### Rôle

Application **dédiée au personnel** (coachs, admins) installée sur une tablette ou un téléphone à l'accueil de la salle. Elle permet de valider les entrées des membres en scannant leur QR code.

### Structure

```
scanner/
└── App.tsx    # Application complète en un seul fichier
```

### Flux utilisateur

```
┌─────────────────┐
│   Écran Login   │  Email + mot de passe staff
└────────┬────────┘
         │ POST /api/auth/login (JWT stocké en mémoire)
         ▼
┌─────────────────┐
│  Écran Scanner  │  Caméra en continu (QR only)
└────────┬────────┘
         │ QR code détecté → POST /api/checkins/validate-qr
         ▼
┌─────────────────────────────────────────────┐
│  ✅ Succès  │  Nom du membre + "Check-in validé !"  │
│  ❌ Erreur  │  Message d'erreur (token expiré, etc.) │
└─────────────────────────────────────────────┘
         │ Bouton "Scanner un autre"
         ▼
   Retour caméra
```

### Stack

| Package | Rôle |
|---|---|
| Expo 54 | Runtime |
| expo-camera (CameraView) | Scan QR code |
| fetch natif | Appels API (pas d'axios) |
| useState | State machine locale (login / scan) |

### Sécurité

- JWT du staff stocké en mémoire locale (pas persisté)
- Chaque validation de QR appelle le backend qui vérifie le token (valide 60 s max, à usage unique)

---

## 7. Base de données

### Schéma entités-relations

```
Users (id PK, firstName, lastName, email, passwordHash, roleId FK, createdAt)
  │
  ├─── Roles (id PK, name)
  │
  ├─── Subscriptions (id PK, userId FK, planName, startDate, endDate, isActive,
  │                   stripeCustomerId, stripeSubscriptionId, stripeStatus)
  │
  ├─── Bookings (id PK, userId FK, sessionId FK, bookedAt, isCancelled)
  │
  ├─── CheckIns (id PK, userId FK, checkedInAt, qrToken)
  │
  ├─── WorkoutLogs (id PK, userId FK, sessionId FK nullable, notes, loggedAt)
  │         └─── WorkoutExercises (id PK, workoutLogId FK, exerciseId FK,
  │                                sets, reps, weightKg, notes)
  │                     └─── Exercises (id PK, name, muscleGroup, description)
  │
  └─── Courses (id PK, name, description, coachId FK, durationMinutes, maxCapacity)
            └─── Sessions (id PK, courseId FK, startTime, currentBookings, coachId FK nullable)
```

### Règles métier en base

| Règle | Implémentation |
|---|---|
| Un utilisateur peut avoir 0 ou 1 abonnement | FK nullable + contrainte unique |
| Une séance peut avoir 0 ou N réservations | `currentBookings` auto-incrémenté |
| Un check-in QR token est à usage unique (60 s) | Validé côté API, pas en base |
| Un workout peut être lié à une séance ou libre | `SessionId` nullable dans `WorkoutLog` |
| Suppression d'une séance : les workout logs gardent l'historique | `ON DELETE SET NULL` sur `WorkoutLog.SessionId` |
| Filtrage sessions : uniquement les futures | `WHERE StartTime > DateTime.UtcNow` côté repo |

### Migrations EF Core

```
InitialCreate                 — Schéma initial complet
AddStripeToSubscription       — Ajout StripeCustomerId, StripeSubscriptionId, StripeStatus
WorkoutLogOptionalSession     — SessionId nullable + ON DELETE SET NULL
```

---

## 8. Installation & lancement

### Prérequis

- .NET 10 SDK
- Node.js 20+
- SQL Server (ou SQL Server Express)
- Expo CLI (`npm install -g expo-cli`)
- Compte Stripe (clés API)

### 1. Backend

```bash
cd backend
# Configurer appsettings.json (ConnectionStrings, JwtSettings, Stripe)
dotnet ef database update --project SmartGym.Infrastructure --startup-project SmartGym.API
dotnet run --project SmartGym.API
# API disponible sur http://localhost:5064
```

### 2. Frontend Web

```bash
cd frontend
npm install
# Configurer .env (VITE_API_URL)
npm run dev
# Disponible sur http://localhost:5173
```

### 3. Application Mobile

```bash
cd mobile
npm install
# Configurer src/config.ts (apiUrl)
npx expo start
# Scanner le QR code avec Expo Go (iOS/Android)
```

### 4. Scanner

```bash
cd scanner
npm install
# Configurer EXPO_PUBLIC_API_URL dans .env
npx expo start
```

---

## 9. Variables d'environnement

### Backend — `appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=SmartGymDb;..."
  },
  "JwtSettings": {
    "Secret": "votre_secret_jwt_256bits",
    "Issuer": "SmartGym",
    "Audience": "SmartGymUsers"
  },
  "Stripe": {
    "SecretKey": "sk_test_...",
    "WebhookSecret": "whsec_...",
    "PriceId": "price_..."
  }
}
```

### Frontend Web — `.env`

```
VITE_API_URL=http://localhost:5064/api
```

### Mobile — `src/config.ts`

```typescript
export const config = {
  apiUrl: 'http://192.168.x.x:5064/api'  // IP locale pour Expo Go
};
```

### Scanner — `.env`

```
EXPO_PUBLIC_API_URL=http://192.168.x.x:5064/api
```
