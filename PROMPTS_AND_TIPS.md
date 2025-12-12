# PROMPTS_AND_TIPS.md - Guide Complet pour Claude Code

## 🎯 Pourquoi Claude Code plutôt que Gemini ?

| Critère | Claude Code | Gemini AI Studio |
|---------|-------------|------------------|
| **Contexte long** | ★★★★★ (200k tokens) | ★★★☆☆ (perdu sur gros projets) |
| **Suivi d'instructions** | ★★★★★ | ★★★☆☆ |
| **Qualité React Native** | ★★★★★ | ★★★★☆ |
| **Consistance** | ★★★★★ | ★★☆☆☆ (oublie le contexte) |
| **Refus de coder** | Rare | Fréquent |

**Verdict** : Pour un projet complexe comme ChallengePact, Claude Code est nettement supérieur.

---

## 🛠️ Configuration de Claude Code

### Installation

```bash
# Installer Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Vérifier l'installation
claude --version

# Se connecter (API key requise)
claude auth
```

### Lancer dans ton Projet

```bash
cd /path/to/challengepact

# Lancer Claude Code
claude

# Ou avec un fichier d'init
claude --init  # Crée CLAUDE.md si absent
```

---

## 📁 Fichiers ESSENTIELS à Préparer

### Structure Recommandée

```
/challengepact
├── CLAUDE.md              # ⭐ CRITIQUE - Instructions système
├── PROJECT_SPEC.md        # ⭐ CRITIQUE - Specs techniques  
├── DATABASE_SCHEMA.md     # ⭐ CRITIQUE - Schéma Supabase
├── IMPLEMENTATION_PLAN.md # Plan de développement
├── instruction.md         # Description produit complète
├── package.json
├── .env.example
└── /src
    └── ... (ton code)
```

### Le fichier CLAUDE.md est ROI

Claude Code lit **automatiquement** `CLAUDE.md` à la racine. C'est là que tu mets :

```markdown
# CLAUDE.md

## Stack Technique
- Frontend: React Native + Expo SDK 50+
- Backend: Supabase
- State: Zustand
- Styling: NativeWind
- Animations: Reanimated + Lottie

## Conventions de Code
- TypeScript strict, jamais de `any`
- Composants fonctionnels uniquement
- Un composant = un fichier, max 200 lignes
- Imports absolus avec @/

## Patterns à Suivre
[exemples de code]

## Erreurs à Éviter
[anti-patterns]

## Structure du Projet
[arborescence]
```

---

## 📝 Prompts Optimisés par Phase

### PHASE 1 : Setup Projet

```markdown
## Tâche : Setup initial ChallengePact

### Contexte
Lis CLAUDE.md pour les conventions et PROJECT_SPEC.md pour les dépendances.

### À faire
1. Créer projet Expo avec TypeScript et expo-router
2. Installer toutes les dépendances de PROJECT_SPEC.md
3. Configurer :
   - tailwind.config.js (couleurs du design system)
   - tsconfig.json (path aliases @/)
   - app.json (scheme: challengepact)
4. Créer la structure de dossiers de CLAUDE.md
5. Créer /src/lib/supabase.ts

### Validation
- `npm start` fonctionne sans erreur
- Structure correcte
```

### PHASE 2 : Authentification

```markdown
## Tâche : Système d'authentification complet

### Référence
- Schéma DB : DATABASE_SCHEMA.md section "USERS & AUTH"
- Conventions : CLAUDE.md

### À créer (dans cet ordre)

1. **Types** : /src/types/auth.ts
   - User, AuthState, LoginCredentials, RegisterData

2. **Store Zustand** : /src/stores/authStore.ts
   - State: user, session, isLoading, error
   - Actions: login, register, logout, resetPassword
   - Persist avec AsyncStorage

3. **Hook** : /src/hooks/useAuth.ts
   - Wrapper du store
   - Auto-refresh session
   - Listener changements Supabase

4. **Composants** : /src/components/auth/
   - LoginForm.tsx
   - RegisterForm.tsx
   - SocialAuthButtons.tsx (Apple + Google)

5. **Screens** : /app/(auth)/
   - login.tsx
   - register.tsx
   - onboarding.tsx (3 slides + profile setup)

6. **Layouts** :
   - /app/_layout.tsx (root avec auth check)
   - /app/(auth)/_layout.tsx (stack sans header)
   - /app/(tabs)/_layout.tsx (tab bar)

### Contraintes
- Validation: username 3-20 chars alphanumeric
- Age minimum: 16 ans
- Erreurs en français
- Loading states partout
```

### PHASE 3 : Création de Défi

```markdown
## Tâche : Flow création de défi (wizard 4 étapes)

### Référence
- DB: DATABASE_SCHEMA.md section "CHALLENGES"
- Produit: instruction.md section "Création de Défi"

### Service à créer : /src/services/challengeService.ts
```typescript
interface CreateChallengeInput {
  name: string;
  description?: string;
  emoji: string;
  duration_days: 7 | 14 | 21 | 30;
  proof_type: 'photo' | 'video' | 'text' | 'check';
  deadline_time: string;
  active_days: string[];
  mode: 'strict' | 'tolerant';
  stakes_enabled: boolean;
  start_date: string;
}

create(input: CreateChallengeInput): Promise<Challenge>
getById(id: string): Promise<Challenge>
getMyActive(): Promise<Challenge[]>
```

### Wizard : /app/create/
- index.tsx (Step 1: nom, description, emoji)
- schedule.tsx (Step 2: durée, jours, date, heure)
- rules.tsx (Step 3: preuve, mode, gages)
- review.tsx (Step 4: récap, créer, inviter)

### Composants : /src/components/challenge/
- StepIndicator.tsx
- EmojiPicker.tsx
- DurationSelector.tsx
- DayToggle.tsx
- ModeSelector.tsx
- ChallengePreviewCard.tsx

### UX
- Transitions animées entre steps
- Retour arrière possible
- Draft sauvé en local
- Preview temps réel
```

### PHASE 4 : Check-in

```markdown
## Tâche : Flow check-in avec photo

### Flow
1. Tap "Valider" → ouvre caméra plein écran
2. Capture photo
3. Preview avec Retake/Use
4. Caption optionnel
5. Upload Supabase Storage
6. Créer check_in en DB
7. +XP, +streak
8. Animation succès
9. Redirect vers roue (ou retour défi)

### Composants : /src/components/checkin/
- CameraView.tsx (expo-camera, flash, flip, capture)
- PhotoPreview.tsx (image, retake, confirm)
- CaptionInput.tsx (textarea avec compteur)
- CheckInSuccess.tsx (animation Lottie, +XP)

### Services
```typescript
// /src/services/storageService.ts
uploadCheckInPhoto(challengeId, userId, photo): Promise<string>

// /src/services/checkInService.ts
create(challengeId, photoUrl, caption?): Promise<CheckIn>
getTodayForChallenge(challengeId): Promise<CheckIn[]>
hasCheckedInToday(challengeId, userId): Promise<boolean>
```

### Screen : /app/challenge/[id]/checkin.tsx
```

### PHASE 5 : Roue de la Fortune

```markdown
## Tâche : Roue de la fortune animée

### Edge Function : /supabase/functions/spin-wheel/index.ts
- Input: { userId, checkInId, spinType }
- Vérifie check-in existe et appartient au user
- Vérifie pas de spin déjà fait pour ce check-in
- Calcule probabilités (avec bonus streak)
- Sélection aléatoire pondérée
- Applique récompense (XP, carte, etc.)
- Save dans spin_history
- Output: { prizeType, prizeValue }

Probabilités de base:
- xp_5: 25%
- xp_10: 15%
- xp_20: 8%
- card_common: 20%
- card_rare: 5%
- cosmetic: 3%
- joker: 2%
- xp_double: 2%
- jackpot: 0.5%
- nothing: 19.5%

### Screen : /app/spin.tsx
1. Affiche roue prête
2. User tap "SPIN!"
3. Call Edge Function
4. Pendant call: animation spin commence
5. Au retour: calcule angle final
6. Animation décélération vers cet angle
7. Haptic au stop
8. Reveal prix avec animation
9. Bouton "Continuer"

### Composant : /src/components/gamification/FortuneWheel.tsx
- 10 segments colorés
- Rotation avec Reanimated
- Easing personnalisé (bezier)
- Durée: 4 secondes
- Min 5 tours + angle final

### PrizeReveal.tsx
- Fond assombri
- Modal scale 0→1
- Icône bounce
- Confettis si rare+
```

### PHASE 6 : Mascotte

```markdown
## Tâche : Mascotte avec états émotionnels

### Types
```typescript
type MascotType = 'piu' | 'mochi' | 'buddy' | 'kitsune' | 'panda' | 
                  'sparkle' | 'drake' | 'bolt' | 'pixel' | 'sprout';

type MascotEmotion = 'sleeping' | 'happy' | 'excited' | 'euphoric' | 
                     'worried' | 'stressed' | 'sad' | 'devastated' | 'victorious';

interface Mascot {
  id: string;
  type: MascotType;
  name: string;
  level: 1 | 2 | 3 | 4 | 5;
  happiness: number;
  currentEmotion: MascotEmotion;
}
```

### Assets Lottie
/assets/lottie/mascots/{type}/{emotion}.json

### Composant MascotDisplay.tsx
- Affiche animation Lottie selon type + émotion
- Loop sur idle
- Transition smooth entre états
- Tap = bounce + son

### Hooks
```typescript
// Détermine émotion selon état du défi
useMascotEmotion(challenge: Challenge): MascotEmotion

// Détermine niveau selon jour du défi
useMascotEvolution(challenge: Challenge): 1 | 2 | 3 | 4 | 5
```

### Logique émotion
- 00h-06h: sleeping
- Tout le monde validé: euphoric
- Quelqu'un vient de valider: excited
- 2h avant deadline + quelqu'un manque: worried
- 30min avant + quelqu'un manque: stressed
- Vie perdue: sad
- Défi échoué: devastated
- Défi réussi: victorious
- Sinon: happy
```

---

## 🔧 Prompts de Debugging

### Erreur de Compilation

```markdown
## Bug : Erreur de compilation

### Erreur exacte
```
[COLLER L'ERREUR COMPLÈTE ICI]
```

### Fichier concerné
/src/components/xxx.tsx

### Contexte
J'essayais de [DÉCRIRE]

### Demande
1. Analyse l'erreur
2. Explique la cause
3. Montre le code corrigé
```

### Erreur Supabase

```markdown
## Bug : Erreur Supabase

### Requête
```typescript
const { data, error } = await supabase
  .from('challenges')
  .select('*')
  .eq('id', id);
```

### Erreur
```json
{ "message": "...", "code": "..." }
```

### Schéma de la table
```sql
CREATE TABLE challenges (...);
```

### RLS Policies
```sql
CREATE POLICY "..." ON challenges ...;
```
```

### Bug UI/Animation

```markdown
## Bug : Animation saccadée

### Composant
FortuneWheel.tsx

### Comportement attendu
Rotation fluide 60fps

### Comportement actuel
Saccades, freeze

### Code actuel
```typescript
[CODE]
```

### Device testé
iPhone 13 / Android Pixel 6
```

---

## 💡 Tips Avancés

### Tip 1 : Checkpoints réguliers

```markdown
## Checkpoint : Fin Phase 2

### ✅ Fait
- Auth complet
- Création défi
- Invitations

### ❌ Reste
- Feed check-ins
- Chat groupe

### Prochaine tâche
Créer CheckInFeed.tsx
```

### Tip 2 : Demande des reviews

```markdown
## Review : FortuneWheel.tsx

```typescript
[TON CODE]
```

### Questions
1. Animation optimisée ? (60fps)
2. Memory leaks potentiels ?
3. Suit les conventions CLAUDE.md ?
4. Améliorations possibles ?
```

### Tip 3 : Génère les types Supabase

```markdown
## Tâche : Générer types TypeScript

1. Lance: `supabase gen types typescript --local`
2. Sauvegarde dans /src/types/database.ts
3. Crée des alias pratiques dans /src/types/index.ts :
   - Database['public']['Tables']['challenges']['Row'] → Challenge
   - Database['public']['Tables']['challenges']['Insert'] → CreateChallengeInput
```

### Tip 4 : Tests automatiques

```markdown
## Tâche : Tests pour challengeService

### À tester
- create() - nominal + erreurs validation
- getById() - existant + inexistant
- getMyActive() - avec résultats + vide

### Framework
Jest + @testing-library/react-native

### Mocks
- Supabase client
- Auth user
```

---

## 🔄 Workflow Recommandé

### Session : Nouvelle Feature

```
1. Ouvre Claude Code dans le projet
2. "Lis CLAUDE.md et PROJECT_SPEC.md"
3. Décris la feature
4. Demande d'abord un plan
5. Valide le plan
6. Implémente étape par étape
7. Teste chaque étape
8. Review finale
```

### Session : Bug Fix

```
1. Décris le bug + étapes de repro
2. Colle l'erreur exacte
3. Montre le code concerné
4. Demande l'analyse
5. Applique la correction
6. Confirme résolu
```

### Session : Refactoring

```
1. Montre le code actuel
2. Explique le problème (perf, lisibilité...)
3. Demande des suggestions
4. Choisis l'approche
5. Implémente
6. Vérifie rien cassé
```

---

## ⚠️ Erreurs Courantes

### ❌ Prompt trop vague
```
"Fais le système de gamification"
```
### ✅ Prompt précis
```
"Implémente la roue de la fortune selon instruction.md section 7, 
avec les composants dans /src/components/gamification/"
```

### ❌ Pas de contexte
```
"Crée un Button"
```
### ✅ Avec contexte
```
"Crée Button.tsx dans /src/components/ui/ selon le design system CLAUDE.md"
```

### ❌ Tout d'un coup
```
"Crée toute l'app"
```
### ✅ Par étapes
```
"Phase 1: Setup. Puis Phase 2: Auth. Etc."
```

### ❌ Ignorer les erreurs
```
[Erreur] → "Continue"
```
### ✅ Corriger d'abord
```
[Erreur] → "Explique et corrige avant de continuer"
```

---

## 📋 Checklist Avant Session

- [ ] CLAUDE.md à la racine
- [ ] PROJECT_SPEC.md avec specs
- [ ] DATABASE_SCHEMA.md avec schéma
- [ ] package.json avec dépendances
- [ ] .env.example avec variables
- [ ] Projet compile (`npm start`)
- [ ] Plan clair de la session

---

## 🎯 Commandes Claude Code Utiles

```bash
# Lancer Claude Code
claude

# Avec fichier spécifique
claude --file src/components/Button.tsx

# Mode verbose
claude --verbose

# Limiter les tokens
claude --max-tokens 4000
```

---

*Guide pour utiliser Claude Code efficacement sur ChallengePact*
*Mise à jour : Décembre 2024*
