# IMPLEMENTATION_PLAN.md - Plan de Développement par Phases

## 🎯 Vue d'Ensemble

**Objectif** : MVP fonctionnel en 8-10 semaines

```
PHASE 1 (Semaines 1-2)     PHASE 2 (Semaines 3-4)     PHASE 3 (Semaines 5-6)
─────────────────────      ─────────────────────      ─────────────────────
Setup + Auth + DB          Défis Core                 Gamification Base
                           Check-ins                   Roue + XP
                           
PHASE 4 (Semaines 7-8)     PHASE 5 (Semaines 9-10)    POST-MVP
─────────────────────      ─────────────────────      ─────────────────────
Social                     Polish                      Cartes avancées
Notifications              Tests                       Ligues
Mascotte                   Bug fixes                   Events
```

---

## 📋 PHASE 1 : Fondations (Semaines 1-2)

### Objectifs
- [x] Setup projet Expo + TypeScript
- [x] Configuration Supabase
- [x] Authentification complète
- [x] Navigation de base
- [x] Design system

### Tâches Détaillées

#### 1.1 Setup Projet (Jour 1-2)

```bash
# Commandes à exécuter
npx create-expo-app@latest challengepact --template tabs
cd challengepact
npx expo install expo-router expo-camera expo-image-picker
npx expo install @supabase/supabase-js
npm install zustand nativewind
npm install react-native-reanimated lottie-react-native
npm install -D tailwindcss
```

**Fichiers à créer :**
```
/app
  /_layout.tsx           # Root layout avec providers
  /(auth)
    /_layout.tsx         # Auth layout
    /login.tsx
    /register.tsx
  /(tabs)
    /_layout.tsx         # Tab layout
    /index.tsx           # Home placeholder
    /profile.tsx         # Profile placeholder
    
/src
  /lib
    /supabase.ts         # Client Supabase
  /stores
    /authStore.ts        # Auth state
  /types
    /index.ts            # Types de base
```

**Configuration :**
- `app.json` - Config Expo
- `tailwind.config.js` - Couleurs, fonts
- `tsconfig.json` - Paths aliases
- `.env` - Variables Supabase

#### 1.2 Supabase Setup (Jour 2-3)

```sql
-- Exécuter dans Supabase SQL Editor

-- Tables de base Phase 1
CREATE TABLE users (...);
CREATE TABLE user_stats (...);
CREATE TABLE user_settings (...);
CREATE TABLE friendships (...);

-- Voir DATABASE_SCHEMA.md pour le SQL complet
```

**Fichiers :**
```
/supabase
  /migrations
    /001_users.sql
    /002_user_stats.sql
  /seed.sql
```

#### 1.3 Authentification (Jour 3-5)

**Composants :**
```typescript
// /src/components/auth/
LoginForm.tsx
RegisterForm.tsx
SocialAuthButtons.tsx    // Apple + Google
ForgotPasswordForm.tsx
```

**Hooks :**
```typescript
// /src/hooks/
useAuth.ts              // Login, register, logout, session
useUser.ts              // Get current user data
```

**Screens :**
```typescript
// /app/(auth)/
login.tsx               // Email + Social login
register.tsx            // Register flow
onboarding.tsx          // 3 slides + profile setup
```

**Flow d'auth :**
```
App Launch
    │
    ├── Session exists? ─── Yes ──→ /(tabs)/
    │
    └── No ──→ /(auth)/login
                    │
                    ├── Login success ──→ /(tabs)/
                    │
                    └── Register ──→ /onboarding ──→ /(tabs)/
```

#### 1.4 Design System (Jour 5-7)

**Composants UI de base :**
```typescript
// /src/components/ui/
Button.tsx              // Primary, secondary, ghost, sizes
Input.tsx               // Text input avec états
Avatar.tsx              // Image + fallback initiales
Card.tsx                // Container avec shadow
Badge.tsx               // XP, niveau, etc.
Modal.tsx               // Bottom sheet modal
Toast.tsx               // Notifications in-app
LoadingSpinner.tsx
EmptyState.tsx
```

**Theme :**
```typescript
// /src/utils/theme.ts
export const colors = {
  primary: {...},
  secondary: {...},
  // etc.
};
```

### Livrables Phase 1
- [ ] App qui compile sans erreur
- [ ] Auth fonctionnel (email + social)
- [ ] Navigation entre pages
- [ ] Profil utilisateur affiché
- [ ] Design system utilisable

---

## 📋 PHASE 2 : Défis Core (Semaines 3-4)

### Objectifs
- [ ] Créer un défi
- [ ] Inviter des amis
- [ ] Rejoindre un défi
- [ ] Check-in quotidien avec photo
- [ ] Voir les check-ins des autres

### Tâches Détaillées

#### 2.1 Base de Données Défis (Jour 1)

```sql
-- Tables Phase 2
CREATE TABLE challenges (...);
CREATE TABLE challenge_members (...);
CREATE TABLE check_ins (...);
CREATE TABLE challenge_chat (...);
```

#### 2.2 Création de Défi (Jour 2-4)

**Screens :**
```typescript
// /app/
create.tsx              // Wizard de création

// ou multi-step
/create
  /step1.tsx           // Nom, description, emoji
  /step2.tsx           // Durée, jours actifs
  /step3.tsx           // Mode, preuve, deadline
  /step4.tsx           // Récap + invitations
```

**Composants :**
```typescript
// /src/components/challenge/
CreateChallengeWizard.tsx
StepIndicator.tsx
EmojiPicker.tsx
DurationSelector.tsx
ModeSelector.tsx
ProofTypeSelector.tsx
DatePicker.tsx
ChallengePreview.tsx
```

**Service :**
```typescript
// /src/services/challengeService.ts
export const challengeService = {
  create(data: CreateChallengeInput): Promise<Challenge>,
  getById(id: string): Promise<Challenge>,
  getMyActiveChallenges(): Promise<Challenge[]>,
  updateStatus(id: string, status: ChallengeStatus): Promise<void>,
};
```

#### 2.3 Invitations (Jour 4-5)

**Composants :**
```typescript
InviteModal.tsx          // Modal d'invitation
ShareButtons.tsx         // WhatsApp, iMessage, etc.
InviteLinkCard.tsx       // Lien copiable
FriendSelector.tsx       // Sélection d'amis existants
```

**Deep Linking :**
```typescript
// Configuration dans app.json
{
  "scheme": "challengepact",
  "web": {
    "bundler": "metro"
  }
}

// Handling
// /app/join/[code].tsx
```

#### 2.4 Page de Défi (Jour 5-7)

**Screen :**
```typescript
// /app/challenge/[id].tsx
```

**Composants :**
```typescript
ChallengeHeader.tsx      // Emoji, nom, countdown
ChallengeStats.tsx       // Jour X/Y, vies, streak
ParticipantsList.tsx     // Avatars + status du jour
CheckInButton.tsx        // CTA principal
CheckInFeed.tsx          // Liste des check-ins
DayProgress.tsx          // Qui a validé aujourd'hui
```

**États de la page :**
```
Challenge Status:
├── pending (avant start_date)
│   └── Afficher countdown, participants, chat
├── active
│   ├── User a validé aujourd'hui
│   │   └── Afficher feed, réactions
│   └── User n'a pas validé
│       └── Afficher CTA check-in
├── completed
│   └── Afficher récap, stats, badges
└── failed
    └── Afficher récap, encouragements
```

#### 2.5 Check-in (Jour 7-10)

**Screen :**
```typescript
// /app/challenge/[id]/checkin.tsx
```

**Composants :**
```typescript
CameraView.tsx           // Prise de photo
PhotoPreview.tsx         // Aperçu + retake
CaptionInput.tsx         // Légende optionnelle
CheckInConfirmation.tsx  // Succès + animation
```

**Flow :**
```
Tap "Valider" button
       │
       ▼
   CameraView
       │
       ▼
   Take Photo ──→ PhotoPreview ──→ Retake?
       │                               │
       │                               ▼
       │                          Retake ──→ CameraView
       │
       ▼
   Add Caption (optional)
       │
       ▼
   Upload to Supabase Storage
       │
       ▼
   Create check_in record
       │
       ▼
   Update XP + Streak
       │
       ▼
   [PHASE 3: Trigger Wheel]
       │
       ▼
   Success Animation
       │
       ▼
   Return to Challenge Page
```

**Service Upload :**
```typescript
// /src/services/storageService.ts
export const storageService = {
  async uploadCheckInPhoto(
    challengeId: string,
    userId: string,
    photo: ImagePickerAsset
  ): Promise<string> {
    const fileName = `${challengeId}/${userId}/${Date.now()}.jpg`;
    const { data, error } = await supabase.storage
      .from('check-ins')
      .upload(fileName, photo.base64, {
        contentType: 'image/jpeg',
      });
    
    if (error) throw error;
    return supabase.storage.from('check-ins').getPublicUrl(fileName).data.publicUrl;
  },
};
```

### Livrables Phase 2
- [ ] Créer un défi complet
- [ ] Inviter via lien/partage
- [ ] Rejoindre via deep link
- [ ] Check-in avec photo
- [ ] Voir le feed du groupe
- [ ] Chat basique fonctionnel

---

## 📋 PHASE 3 : Gamification Base (Semaines 5-6)

### Objectifs
- [ ] Système XP + Niveaux
- [ ] Roue de la fortune
- [ ] Mini-défis quotidiens
- [ ] Badges de base

### Tâches Détaillées

#### 3.1 Système XP (Jour 1-2)

**Edge Function :**
```typescript
// /supabase/functions/award-xp/index.ts
// Calcule et attribue les XP après un check-in
```

**Hook :**
```typescript
// /src/hooks/useXP.ts
export function useXP() {
  const addXP = async (amount: number, source: string) => {...};
  const getCurrentLevel = () => {...};
  const getProgressToNextLevel = () => {...};
  return { addXP, level, xp, progress };
}
```

**Composants :**
```typescript
XPBar.tsx                // Barre de progression
LevelBadge.tsx           // Affichage du niveau
LevelUpModal.tsx         // Animation level up
```

#### 3.2 Roue de la Fortune (Jour 2-5)

**Screen :**
```typescript
// /app/spin.tsx
// Accessible après check-in
```

**Composants :**
```typescript
// /src/components/gamification/
FortuneWheel.tsx         // La roue animée
WheelSegment.tsx         // Segment individuel
SpinButton.tsx           // Bouton de spin
PrizeReveal.tsx          // Animation du gain
```

**Animation (Reanimated) :**
```typescript
// Rotation avec easing personnalisé
// Durée: 3-4 secondes
// Décélération progressive
// Haptics au stop
```

**Edge Function :**
```typescript
// /supabase/functions/spin-wheel/index.ts
// Calcule le résultat (côté serveur pour éviter triche)
// Retourne: { prize_type, prize_value, probabilities_used }
```

**Probabilités :**
```typescript
const PROBABILITIES = {
  'xp_5': 0.25,
  'xp_10': 0.15,
  'xp_20': 0.08,
  'card_common': 0.20,
  'card_rare': 0.05,
  'cosmetic': 0.03,
  'joker': 0.02,
  'xp_double': 0.02,
  'jackpot': 0.005,
  'nothing': 0.195,
};
```

#### 3.3 Mini-Défis Quotidiens (Jour 5-7)

**Edge Function :**
```typescript
// /supabase/functions/assign-daily-challenge/index.ts
// Appelé à minuit, assigne un mini-défi à chaque user actif
```

**Composants :**
```typescript
DailyChallengeCard.tsx   // Affichage du mini-défi
DailyChallengeStatus.tsx // Complété ou non
```

**Hook :**
```typescript
// /src/hooks/useDailyChallenge.ts
export function useDailyChallenge(challengeId: string) {
  const { dailyChallenge, isCompleted, complete } = ...;
  return { dailyChallenge, isCompleted, complete };
}
```

#### 3.4 Badges (Jour 7-8)

**Service :**
```typescript
// /src/services/badgeService.ts
export const badgeService = {
  checkAndAwardBadges(userId: string): Promise<Badge[]>,
  getUserBadges(userId: string): Promise<Badge[]>,
};
```

**Composants :**
```typescript
BadgeGrid.tsx            // Grille de badges
BadgeCard.tsx            // Badge individuel
BadgeUnlockModal.tsx     // Animation déblocage
```

### Livrables Phase 3
- [ ] XP gagnés après check-in
- [ ] Level up fonctionnel
- [ ] Roue qui tourne et donne des prix
- [ ] Mini-défi affiché chaque jour
- [ ] Badges qui se débloquent

---

## 📋 PHASE 4 : Social + Mascotte (Semaines 7-8)

### Objectifs
- [ ] Réactions aux check-ins
- [ ] Système de Kudos
- [ ] Notifications push
- [ ] Mascotte avec états

### Tâches Détaillées

#### 4.1 Réactions (Jour 1-2)

**Composants :**
```typescript
ReactionBar.tsx          // Barre de réactions
ReactionButton.tsx       // Bouton emoji
ReactionCount.tsx        // Compteur par type
```

**Realtime :**
```typescript
// Subscription aux nouvelles réactions
supabase
  .channel('reactions')
  .on('postgres_changes', { 
    event: 'INSERT', 
    schema: 'public', 
    table: 'reactions',
    filter: `check_in_id=eq.${checkInId}`
  }, handleNewReaction)
  .subscribe();
```

#### 4.2 Kudos (Jour 2-3)

**Composants :**
```typescript
KudosButton.tsx          // Bouton pour donner
KudosModal.tsx           // Sélection montant
KudosBalance.tsx         // Solde affiché
KudosStore.tsx           // Boutique cosmétiques
```

**Service :**
```typescript
// /src/services/kudosService.ts
export const kudosService = {
  give(toUserId: string, amount: number, checkInId?: string): Promise<void>,
  getBalance(userId: string): Promise<number>,
  purchase(cosmeticId: string): Promise<void>,
};
```

#### 4.3 Notifications (Jour 3-5)

**Setup Expo Notifications :**
```typescript
// /src/services/notificationService.ts
import * as Notifications from 'expo-notifications';

export const notificationService = {
  async registerForPushNotifications(): Promise<string>,
  async scheduleLocalReminder(challengeId: string, time: Date): Promise<void>,
  async cancelAllReminders(): Promise<void>,
};
```

**Edge Function pour Push :**
```typescript
// /supabase/functions/send-notification/index.ts
// Utilise Expo Push API
```

**Types de notifications :**
```typescript
type NotificationType = 
  | 'challenge_invite'
  | 'challenge_starting'
  | 'deadline_warning_2h'
  | 'deadline_warning_30m'
  | 'friend_checked_in'
  | 'kudos_received'
  | 'nudge'
  | 'level_up'
  | 'badge_unlocked';
```

#### 4.4 Mascotte (Jour 5-8)

**Composants :**
```typescript
// /src/components/mascot/
Mascot.tsx               // Composant principal
MascotDisplay.tsx        // Affichage avec animation
MascotSelector.tsx       // Choix initial
MascotEvolution.tsx      // Animation d'évolution
MascotInteraction.tsx    // Tap, feed, etc.
```

**Animations Lottie :**
```
/assets/lottie/mascots/
  piu/
    idle.json
    happy.json
    sad.json
    excited.json
    sleeping.json
    evolve.json
```

**Hook :**
```typescript
// /src/hooks/useMascot.ts
export function useMascot(challengeId: string) {
  const { mascot, emotion, level, interact, feed } = ...;
  return { mascot, emotion, level, interact, feed };
}
```

**Logic d'état :**
```typescript
function getMascotEmotion(challenge: Challenge): MascotEmotion {
  const now = new Date();
  const hour = now.getHours();
  
  if (hour < 6) return 'sleeping';
  if (allCheckedInToday(challenge)) return 'euphoric';
  if (someoneJustCheckedIn) return 'excited';
  if (isNearDeadline && someoneMissing) return 'stressed';
  if (lifeWasLost) return 'sad';
  return 'happy';
}
```

### Livrables Phase 4
- [ ] Réactions fonctionnelles en temps réel
- [ ] Kudos donnables et comptabilisés
- [ ] Boutique Kudos basique
- [ ] Notifications push fonctionnelles
- [ ] Mascotte affichée avec états
- [ ] Mascotte qui évolue

---

## 📋 PHASE 5 : Polish + Tests (Semaines 9-10)

### Objectifs
- [ ] UI/UX polish
- [ ] Performance optimization
- [ ] Bug fixes
- [ ] Tests utilisateurs
- [ ] Préparation store

### Tâches

#### 5.1 UI Polish (Jour 1-3)

- [ ] Animations de transition entre pages
- [ ] Loading states partout
- [ ] Empty states design
- [ ] Error states design
- [ ] Haptic feedback
- [ ] Pull-to-refresh
- [ ] Skeleton loaders

#### 5.2 Performance (Jour 3-5)

- [ ] Optimisation images (compression, lazy loading)
- [ ] Pagination des listes
- [ ] Cache avec React Query ou SWR
- [ ] Réduction bundle size
- [ ] Profiling avec React DevTools

#### 5.3 Tests (Jour 5-7)

- [ ] Tests unitaires services
- [ ] Tests composants critiques
- [ ] Test auth flow
- [ ] Test check-in flow
- [ ] Test sur devices réels

#### 5.4 Beta Testing (Jour 7-10)

- [ ] Déploiement TestFlight (iOS)
- [ ] Déploiement Internal Testing (Android)
- [ ] 10-20 beta testeurs
- [ ] Collecte feedback
- [ ] Fix bugs critiques

#### 5.5 Store Preparation

**Assets requis :**
- [ ] App Icon (1024x1024)
- [ ] Screenshots (6.5", 5.5", iPad)
- [ ] Feature Graphic (Android)
- [ ] Description FR + EN
- [ ] Keywords
- [ ] Privacy Policy URL
- [ ] Support URL

### Livrables Phase 5
- [ ] App stable et performante
- [ ] 0 crash bloquant
- [ ] Feedback positif des beta testeurs
- [ ] Assets store prêts
- [ ] Build EAS configuré

---

## 🚀 POST-MVP : Features Avancées

### Sprint 1 : Cartes Avancées
- Système complet de cartes
- Utilisation des cartes
- Collection et inventaire

### Sprint 2 : Paris/Gages
- Configuration des gages
- Flow de validation
- Preuves de gage

### Sprint 3 : Ligues Hardcore
- Matchmaking
- Scoring
- Classements
- Récompenses

### Sprint 4 : Événements
- Infrastructure events
- Premier event saisonnier
- Récompenses exclusives

### Sprint 5 : Titres Dynamiques
- Attribution automatique
- Affichage sur profil
- Historique

---

## 📊 Métriques de Succès MVP

| Métrique | Objectif MVP |
|----------|--------------|
| Crash-free rate | > 99% |
| App load time | < 3s |
| Check-in completion rate | > 80% |
| D1 Retention | > 40% |
| D7 Retention | > 20% |
| Invites sent per user | > 2 |

---

## 🛠️ Outils Recommandés

| Besoin | Outil |
|--------|-------|
| IDE | VS Code + extensions React Native |
| Device testing | Expo Go + simulateurs |
| API testing | Supabase Studio + Postman |
| Design | Figma |
| Analytics | Mixpanel ou Amplitude |
| Crash reporting | Sentry |
| CI/CD | EAS Build |
| Project management | Linear ou Notion |

---

*Plan de développement ChallengePact*
*Estimation : 8-10 semaines pour MVP*
