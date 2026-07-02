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
8. [Cas d'usage fonctionnels](#8-cas-dusage-fonctionnels)
9. [Installation & lancement](#9-installation--lancement)
10. [Variables d'environnement](#10-variables-denvironnement)

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

## 8. Cas d'usage fonctionnels

### Acteurs du système

| Acteur | Description | Application |
|---|---|---|
| **Membre** | Client inscrit à la salle de sport | Mobile |
| **Coach** | Animateur de cours | Web |
| **Admin** | Gestionnaire de la salle | Web |
| **Staff** | Personnel d'accueil | Scanner |
| **Système** | API SmartGym | Backend |
| **Stripe** | Service de paiement externe | — |

### Sommaire des cas d'usage

| ID | Cas d'usage | Acteur | Application |
|---|---|---|---|
| UC01 | S'inscrire | Membre | Mobile |
| UC02 | Se connecter | Tous | Mobile / Web / Scanner |
| UC03 | Consulter les cours | Membre | Mobile |
| UC04 | Réserver une séance | Membre | Mobile |
| UC05 | Annuler une réservation | Membre | Mobile |
| UC06 | Générer un QR code de check-in | Membre | Mobile |
| UC07 | Valider un check-in par QR | Staff | Scanner |
| UC08 | Souscrire à un abonnement | Membre | Mobile |
| UC09 | Gérer son abonnement | Membre | Mobile |
| UC10 | Démarrer un entraînement | Membre | Mobile |
| UC11 | Consulter l'historique d'entraînement | Membre | Mobile |
| UC12 | Modifier son profil | Membre | Mobile |
| UC13 | Créer un cours | Coach / Admin | Web |
| UC14 | Planifier une séance | Coach / Admin | Web |
| UC15 | Consulter ses séances (coach) | Coach | Web |
| UC16 | Gérer les membres | Admin | Web |
| UC17 | Consulter les réservations | Admin | Web |
| UC18 | Consulter les check-ins du jour | Admin | Web |
| UC19 | Gérer les abonnements | Admin | Web |
| UC20 | Consulter le tableau de bord | Admin | Web |

---

### UC01 — S'inscrire

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : L'utilisateur n'a pas de compte SmartGym

**Scénario nominal**
1. L'utilisateur ouvre l'application mobile
2. Il appuie sur "Créer un compte"
3. Il saisit son prénom, nom, adresse e-mail et mot de passe
4. Il soumet le formulaire
5. Le système valide les données et crée le compte avec le rôle "Client"
6. Le système retourne un token JWT
7. L'utilisateur est redirigé vers l'écran d'accueil

**Scénarios alternatifs**
- **E1 — E-mail déjà utilisé** : Le système retourne une erreur 409, un message "E-mail déjà utilisé" s'affiche
- **E2 — Données invalides** : Le formulaire affiche les erreurs de validation en temps réel

---

### UC02 — Se connecter

**Acteur principal** : Membre / Coach / Admin / Staff | **Application** : Mobile, Web, Scanner  
**Préconditions** : L'utilisateur possède un compte actif

**Scénario nominal**
1. L'utilisateur saisit son e-mail et mot de passe
2. Le système vérifie les credentials (BCrypt)
3. Le système génère un JWT valide 8 heures
4. L'utilisateur est redirigé selon son rôle : Membre → Accueil mobile, Admin → Dashboard web, Coach → Planning web, Staff → Écran scanner

**Scénarios alternatifs**
- **E1 — Credentials incorrects** : Message "Email ou mot de passe incorrect"
- **E2 — Coach/Admin sur mobile** : Accès bloqué, message "Cette application est réservée aux membres"
- **E3 — Token expiré** : Déconnexion automatique et retour à l'écran de connexion

---

### UC03 — Consulter les cours

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté

**Scénario nominal**
1. Le membre accède à l'onglet "Cours"
2. Le système charge la liste de tous les cours disponibles
3. Le membre voit les cartes cours (nom, durée, coach, niveau)
4. Il peut rechercher un cours par nom via la barre de recherche
5. Il appuie sur un cours pour voir son détail et ses prochaines séances

**Scénarios alternatifs**
- **E1 — Aucun cours** : Message "Aucun cours disponible"
- **E2 — Recherche sans résultat** : Message "Aucun cours trouvé pour cette recherche"

---

### UC04 — Réserver une séance

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté, la séance doit avoir des places disponibles

**Scénario nominal**
1. Le membre consulte le détail d'un cours (UC03)
2. Il voit la liste des prochaines séances (futures uniquement)
3. Il sélectionne une séance et appuie sur "Réserver"
4. Le système vérifie les places disponibles (`currentBookings < maxCapacity`)
5. Le système crée la réservation et incrémente `currentBookings`
6. Une confirmation s'affiche : "Réservation confirmée !"
7. La séance apparaît dans l'onglet "Réservations"

**Scénarios alternatifs**
- **E1 — Séance complète** : Le bouton "Réserver" est désactivé, message "Complet"
- **E2 — Déjà réservé** : Message "Vous avez déjà réservé cette séance"

---

### UC05 — Annuler une réservation

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté, avoir une réservation active

**Scénario nominal**
1. Le membre accède à l'onglet "Réservations"
2. Il appuie sur "Annuler" sur une réservation future
3. Le système marque la réservation comme annulée (`isCancelled = true`)
4. Le système décrémente `currentBookings` de la séance
5. La réservation disparaît de la liste "À venir"

**Scénarios alternatifs**
- **E1 — Réservation passée** : Le bouton d'annulation n'est pas affiché

---

### UC06 — Générer un QR code de check-in

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté

**Scénario nominal**
1. Le membre accède à l'onglet "Check-In"
2. Le système génère un token unique via `POST /api/checkins/generate-qr`
3. Un QR code est affiché à l'écran avec un compte à rebours de 60 secondes
4. Le membre présente le QR code au staff à l'accueil
5. À l'expiration (60 s), un nouveau QR code est automatiquement régénéré

**Scénarios alternatifs**
- **E1 — Perte de connexion** : Message d'erreur, bouton "Réessayer" affiché

---

### UC07 — Valider un check-in par QR

**Acteur principal** : Staff | **Application** : Scanner  
**Préconditions** : Le staff est connecté, le membre a un QR code valide (UC06)

**Scénario nominal**
1. Le staff ouvre l'application Scanner et se connecte
2. La caméra s'active en mode scan continu
3. Le staff pointe la caméra vers le QR code du membre
4. Le système appelle `POST /api/checkins/validate-qr`
5. Le backend vérifie le token (existence, expiration < 60 s)
6. L'écran affiche en vert : nom du membre + "Check-in validé !"
7. Le check-in est enregistré en base avec l'heure d'entrée

**Scénarios alternatifs**
- **E1 — Token expiré** : Écran rouge "QR code expiré"
- **E2 — Token invalide** : Écran rouge "QR code invalide"
- **E3 — Déjà utilisé** : Écran rouge "Ce QR code a déjà été utilisé"

---

### UC08 — Souscrire à un abonnement

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté, ne pas avoir d'abonnement actif

**Scénario nominal**
1. Le membre accède à son profil et appuie sur "S'abonner"
2. Le backend crée une session Stripe Checkout et retourne l'URL
3. Une WebView in-app s'ouvre sur la page de paiement Stripe
4. Le membre valide le paiement
5. Stripe appelle le webhook backend qui crée l'abonnement en base
6. La carte d'abonnement s'affiche sur le profil (plan, date d'expiration)

**Scénarios alternatifs**
- **E1 — Paiement refusé** : Stripe affiche un message d'erreur, la WebView reste ouverte
- **E2 — Fermeture de la WebView** : Le membre revient au profil sans abonnement

---

### UC09 — Gérer son abonnement

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté, avoir un abonnement actif

**Scénario nominal**
1. Le membre appuie sur "Gérer mon abonnement" depuis son profil
2. Une WebView s'ouvre sur le portail Stripe Customer Portal
3. Le membre peut modifier son plan, ses moyens de paiement ou résilier
4. Stripe notifie le backend via webhook qui met à jour l'abonnement en base

**Scénarios alternatifs**
- **E1 — Résiliation** : Le statut passe à "Expiré" sur le profil après la date de fin

---

### UC10 — Démarrer un entraînement

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté

**Scénario nominal**
1. Le membre appuie sur "Démarrer un entraînement" dans l'onglet Entraînement
2. Un écran modal s'ouvre avec un chronomètre
3. Il ajoute des exercices via le sélecteur (recherche + filtre par groupe musculaire)
4. Pour chaque exercice, il saisit poids et répétitions par série
5. Le système affiche le poids/reps du dernier entraînement (mémoire des performances)
6. Il coche ✓ pour valider chaque série
7. Il appuie sur "Terminer" — le log et tous les exercices sont enregistrés via l'API

**Scénarios alternatifs**
- **E1 — Aucun exercice** : Le bouton "Terminer" est désactivé
- **E2 — Suppression d'une série** : Appui long sur la ligne → suppression
- **E3 — Suppression d'un exercice** : Bouton poubelle sur le bloc

---

### UC11 — Consulter l'historique d'entraînement

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté

**Scénario nominal**
1. Le membre accède à l'onglet "Entraînement"
2. Le système charge l'historique trié par date décroissante
3. Chaque carte affiche : date, exercices, séries, volume total en kg

**Scénarios alternatifs**
- **E1 — Aucun entraînement** : Message "Démarre ton premier entraînement"

---

### UC12 — Modifier son profil

**Acteur principal** : Membre | **Application** : Mobile  
**Préconditions** : Être connecté

**Scénario nominal**
1. Le membre accède à l'onglet "Profil" et appuie sur "Modifier"
2. Il modifie prénom, nom ou e-mail
3. Le système appelle `PUT /api/users/{id}` et met à jour les données

**Scénarios alternatifs**
- **E1 — E-mail déjà utilisé** : Message "Cet e-mail est déjà associé à un autre compte"

---

### UC13 — Créer un cours

**Acteur principal** : Coach / Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Coach ou Admin

**Scénario nominal**
1. Le coach clique sur "Nouveau cours"
2. Il remplit le formulaire (nom, description, durée, capacité max, coach assigné)
3. Le système appelle `POST /api/courses` et crée le cours

**Scénarios alternatifs**
- **E1 — Champs manquants** : Validation FluentValidation, erreurs affichées

---

### UC14 — Planifier une séance

**Acteur principal** : Coach / Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Coach ou Admin, au moins un cours existant

**Scénario nominal**
1. Le coach clique sur "Nouvelle séance"
2. Il sélectionne un cours, une date et une heure de début
3. Le système crée la séance avec `currentBookings = 0`

**Scénarios alternatifs**
- **E1 — Date passée** : Message "La date doit être dans le futur"

---

### UC15 — Consulter ses séances (coach)

**Acteur principal** : Coach | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Coach

**Scénario nominal**
1. Le coach accède à "Mes réservations"
2. Le système charge les réservations liées à ses cours
3. Il voit les membres inscrits par séance avec le taux de remplissage

---

### UC16 — Gérer les membres

**Acteur principal** : Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Admin

**Scénario nominal**
1. L'admin accède à la page "Membres"
2. Il peut rechercher par nom ou e-mail
3. Il peut supprimer un membre via `DELETE /api/users/{id}`

**Scénarios alternatifs**
- **E1 — Membre avec réservations actives** : La suppression annule les réservations en cascade

---

### UC17 — Consulter les réservations

**Acteur principal** : Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Admin

**Scénario nominal**
1. L'admin accède à "Réservations"
2. Il voit toutes les réservations avec nom du membre et cours
3. Il peut filtrer par date, cours ou statut (annulées)

---

### UC18 — Consulter les check-ins du jour

**Acteur principal** : Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Admin

**Scénario nominal**
1. L'admin accède à la page "Check-In"
2. Le système charge tous les check-ins du jour via `GET /api/checkins/today`
3. L'admin voit : nom du membre, heure d'entrée, compteur total

---

### UC19 — Gérer les abonnements

**Acteur principal** : Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Admin

**Scénario nominal**
1. L'admin accède à "Abonnements"
2. Il voit chaque abonnement avec le nom complet du membre (jointure Users), plan, dates, progression, statut
3. Il peut filtrer par plan ou par statut et supprimer un abonnement

**Scénarios alternatifs**
- **E1 — Abonnement Stripe actif** : La suppression en base ne résilie pas Stripe (à gérer manuellement)

---

### UC20 — Consulter le tableau de bord

**Acteur principal** : Admin | **Application** : Web  
**Préconditions** : Être connecté avec le rôle Admin

**Scénario nominal**
1. L'admin accède au Dashboard
2. Le système retourne via `GET /api/stats/dashboard` :
   - Nombre total de membres, séances du jour, réservations du jour, check-ins du jour
   - Taux de remplissage moyen (%)
   - Courbe des réservations sur les 7 derniers jours
   - Prochaines séances du jour

---

### Matrice acteurs / cas d'usage

|  | Membre | Coach | Admin | Staff |
|---|---|---|---|---|
| UC01 S'inscrire | ✅ | | | |
| UC02 Se connecter | ✅ | ✅ | ✅ | ✅ |
| UC03 Consulter les cours | ✅ | | | |
| UC04 Réserver une séance | ✅ | | | |
| UC05 Annuler une réservation | ✅ | | | |
| UC06 Générer QR code | ✅ | | | |
| UC07 Valider QR code | | | | ✅ |
| UC08 Souscrire | ✅ | | | |
| UC09 Gérer abonnement | ✅ | | | |
| UC10 Démarrer entraînement | ✅ | | | |
| UC11 Historique entraînement | ✅ | | | |
| UC12 Modifier profil | ✅ | | | |
| UC13 Créer un cours | | ✅ | ✅ | |
| UC14 Planifier une séance | | ✅ | ✅ | |
| UC15 Consulter ses séances | | ✅ | | |
| UC16 Gérer les membres | | | ✅ | |
| UC17 Consulter réservations | | | ✅ | |
| UC18 Consulter check-ins | | | ✅ | |
| UC19 Gérer abonnements | | | ✅ | |
| UC20 Tableau de bord | | | ✅ | |

---

## 9. Installation & lancement

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

## 10. Variables d'environnement

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
