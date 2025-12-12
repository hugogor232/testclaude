# CLAUDE.md - Instructions pour Claude Code

## 🎯 Contexte du Projet

Tu développes **ChallengePact**, une application mobile de défis entre amis avec gamification.

**Stack technique :**
- **Frontend** : React Native avec Expo (SDK 50+)
- **Backend** : Supabase (Auth, Database, Storage, Realtime, Edge Functions)
- **State Management** : Zustand
- **Navigation** : Expo Router (file-based routing)
- **UI** : NativeWind (TailwindCSS pour React Native)
- **Animations** : React Native Reanimated + Lottie
- **Notifications** : Expo Notifications + Supabase Edge Functions
- **Analytics** : Mixpanel ou Amplitude (optionnel)

---

## 🧠 Règles de Développement (IMPORTANT)

### Architecture

```
/app                    # Expo Router - Pages
  /(auth)              # Routes non-authentifiées
    /login.tsx
    /register.tsx
    /onboarding.tsx
  /(tabs)              # Routes principales (tab bar)
    /index.tsx         # Home - Défis en cours
    /create.tsx        # Créer un défi
    /social.tsx        # Amis & Social
    /profile.tsx       # Profil & Stats
  /challenge/[id].tsx  # Détail d'un défi
  /spin.tsx            # Roue de la fortune
  /_layout.tsx         # Root layout

/src
  /components          # Composants réutilisables
    /ui                # Boutons, inputs, cards...
    /challenge         # Composants liés aux défis
    /gamification      # Roue, cartes, mascotte...
    /social            # Réactions, kudos, chat...
  /hooks               # Custom hooks
  /stores              # Zustand stores
  /services            # API calls & business logic
  /utils               # Helpers, constants
  /types               # TypeScript types

/supabase
  /migrations          # SQL migrations
  /functions           # Edge Functions
  /seed.sql            # Données de test
```

### Conventions de Code

1. **TypeScript strict** : Toujours typer, jamais de `any`
2. **Composants fonctionnels** : Pas de classes
3. **Hooks personnalisés** : Extraire la logique dans `/hooks`
4. **Naming** :
   - Composants : PascalCase (`ChallengeCard.tsx`)
   - Hooks : camelCase avec prefix `use` (`useChallenge.ts`)
   - Utils : camelCase (`formatDate.ts`)
   - Types : PascalCase avec suffix (`ChallengeType.ts`)
5. **Fichiers** : Un composant = un fichier, max 200 lignes
6. **Imports** : Absolus avec alias `@/` pour `/src`

### Patterns Obligatoires

```typescript
// ✅ BON - Hook avec Supabase
export function useChallenge(id: string) {
  const [challenge, setChallenge] = useState<Challenge | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchChallenge = async () => {
      try {
        const { data, error } = await supabase
          .from('challenges')
          .select('*, participants(*), mascot(*)')
          .eq('id', id)
          .single();
        
        if (error) throw error;
        setChallenge(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchChallenge();
  }, [id]);

  return { challenge, loading, error };
}

// ✅ BON - Composant avec NativeWind
export function ChallengeCard({ challenge }: { challenge: Challenge }) {
  return (
    <Pressable className="bg-white rounded-2xl p-4 shadow-sm active:scale-98">
      <View className="flex-row items-center gap-3">
        <Text className="text-2xl">{challenge.emoji}</Text>
        <View className="flex-1">
          <Text className="text-lg font-bold text-gray-900">
            {challenge.name}
          </Text>
          <Text className="text-sm text-gray-500">
            {challenge.duration} jours • {challenge.participants.length} participants
          </Text>
        </View>
      </View>
    </Pressable>
  );
}

// ✅ BON - Store Zustand
interface ChallengeStore {
  challenges: Challenge[];
  currentChallenge: Challenge | null;
  setChallenges: (challenges: Challenge[]) => void;
  setCurrentChallenge: (challenge: Challenge | null) => void;
  addCheckIn: (challengeId: string, checkIn: CheckIn) => void;
}

export const useChallengeStore = create<ChallengeStore>((set) => ({
  challenges: [],
  currentChallenge: null,
  setChallenges: (challenges) => set({ challenges }),
  setCurrentChallenge: (challenge) => set({ currentChallenge: challenge }),
  addCheckIn: (challengeId, checkIn) =>
    set((state) => ({
      challenges: state.challenges.map((c) =>
        c.id === challengeId
          ? { ...c, checkIns: [...c.checkIns, checkIn] }
          : c
      ),
    })),
}));
```

### Supabase Patterns

```typescript
// ✅ BON - Service avec gestion d'erreur
export const challengeService = {
  async create(data: CreateChallengeInput): Promise<Challenge> {
    const { data: challenge, error } = await supabase
      .from('challenges')
      .insert(data)
      .select()
      .single();

    if (error) {
      console.error('Error creating challenge:', error);
      throw new Error('Impossible de créer le défi');
    }

    return challenge;
  },

  async getById(id: string): Promise<Challenge> {
    const { data, error } = await supabase
      .from('challenges')
      .select(`
        *,
        participants:challenge_participants(
          *,
          user:users(id, username, avatar_url)
        ),
        check_ins(*),
        mascot:mascots(*)
      `)
      .eq('id', id)
      .single();

    if (error) throw new Error('Défi introuvable');
    return data;
  },

  // Realtime subscription
  subscribeToChallenge(id: string, callback: (payload: any) => void) {
    return supabase
      .channel(`challenge:${id}`)
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'check_ins',
          filter: `challenge_id=eq.${id}`,
        },
        callback
      )
      .subscribe();
  },
};
```

### Animations avec Reanimated

```typescript
// ✅ BON - Animation de la roue
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  Easing,
} from 'react-native-reanimated';

export function FortuneWheel({ onResult }: { onResult: (prize: Prize) => void }) {
  const rotation = useSharedValue(0);
  const isSpinning = useSharedValue(false);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }));

  const spin = () => {
    if (isSpinning.value) return;
    isSpinning.value = true;

    const randomRotation = 1800 + Math.random() * 1800; // 5-10 tours
    rotation.value = withTiming(rotation.value + randomRotation, {
      duration: 4000,
      easing: Easing.bezier(0.2, 0.8, 0.2, 1),
    }, () => {
      isSpinning.value = false;
      // Calculate prize based on final rotation
      runOnJS(onResult)(calculatePrize(rotation.value % 360));
    });
  };

  return (
    <Animated.View style={animatedStyle}>
      {/* Wheel content */}
    </Animated.View>
  );
}
```

---

## 📋 Checklist par Feature

### Check-in Quotidien
- [ ] Prise de photo in-app avec Camera
- [ ] Upload vers Supabase Storage
- [ ] Sauvegarde check-in en DB
- [ ] Trigger roue de la fortune
- [ ] Mise à jour XP utilisateur
- [ ] Mise à jour streak
- [ ] Notification aux autres membres
- [ ] Animation de célébration

### Roue de la Fortune
- [ ] Animation de spin (Reanimated)
- [ ] Calcul probabilités côté serveur (Edge Function)
- [ ] Distribution des récompenses
- [ ] Sons et haptics
- [ ] Affichage du gain

### Système de Cartes
- [ ] Inventaire utilisateur
- [ ] Utilisation de carte
- [ ] Effets des cartes (Edge Functions)
- [ ] Animation d'ouverture de carte

### Mascotte
- [ ] Affichage avec états
- [ ] Animations Lottie par état
- [ ] Évolution selon progression
- [ ] Interactions (tap, nourrir)

### Notifications
- [ ] Push locales (rappels)
- [ ] Push distantes (Supabase Edge Functions)
- [ ] Deep links vers les bonnes pages
- [ ] Badges sur l'app

---

## ⚠️ Erreurs à Éviter

```typescript
// ❌ MAUVAIS - Pas de gestion d'erreur
const { data } = await supabase.from('challenges').select();

// ✅ BON
const { data, error } = await supabase.from('challenges').select();
if (error) throw error;

// ❌ MAUVAIS - Logique métier dans le composant
function ChallengeScreen() {
  const handleCheckIn = async () => {
    const today = new Date();
    const { data: existingCheckIn } = await supabase
      .from('check_ins')
      .select()
      .eq('user_id', user.id)
      .eq('date', today.toISOString().split('T')[0]);
    
    if (existingCheckIn.length > 0) {
      alert('Déjà validé !');
      return;
    }
    // ... 50 lignes de plus
  };
}

// ✅ BON - Extraire dans un hook/service
function ChallengeScreen() {
  const { checkIn, isLoading, error } = useCheckIn(challengeId);
  
  return (
    <Button onPress={checkIn} loading={isLoading}>
      Valider
    </Button>
  );
}

// ❌ MAUVAIS - Styles inline
<View style={{ padding: 16, backgroundColor: 'white', borderRadius: 12 }}>

// ✅ BON - NativeWind
<View className="p-4 bg-white rounded-xl">

// ❌ MAUVAIS - Pas de TypeScript
function calculateScore(challenge, user) {
  return challenge.points * user.multiplier;
}

// ✅ BON
function calculateScore(challenge: Challenge, user: User): number {
  return challenge.points * user.multiplier;
}
```

---

## 🚀 Commandes Utiles

```bash
# Démarrer le projet
npx expo start

# Créer une migration Supabase
supabase migration new nom_migration

# Appliquer les migrations
supabase db push

# Générer les types TypeScript depuis Supabase
supabase gen types typescript --local > src/types/database.ts

# Lancer les Edge Functions en local
supabase functions serve

# Build pour les stores
eas build --platform ios
eas build --platform android
```

---

## 📱 Configuration Expo

```json
// app.json
{
  "expo": {
    "name": "ChallengePact",
    "slug": "challengepact",
    "scheme": "challengepact",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#7C3AED"
    },
    "plugins": [
      "expo-router",
      "expo-camera",
      "expo-image-picker",
      "expo-notifications",
      [
        "expo-build-properties",
        {
          "ios": {
            "useFrameworks": "static"
          }
        }
      ]
    ],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

---

## 🎨 Design System

### Couleurs (à utiliser dans tailwind.config.js)

```javascript
colors: {
  primary: {
    50: '#F5F3FF',
    100: '#EDE9FE',
    200: '#DDD6FE',
    300: '#C4B5FD',
    400: '#A78BFA',
    500: '#8B5CF6',
    600: '#7C3AED', // Principal
    700: '#6D28D9',
    800: '#5B21B6',
    900: '#4C1D95',
  },
  secondary: {
    500: '#F97316', // Orange
  },
  success: '#10B981',
  warning: '#F59E0B',
  error: '#EF4444',
}
```

### Typography

```javascript
fontFamily: {
  sans: ['Nunito', 'sans-serif'],
  'sans-bold': ['Nunito-Bold', 'sans-serif'],
}
```

---

## 📝 Prompts pour Développer

Quand tu développes une feature, suis cette structure :

1. **D'abord la DB** : Crée/modifie le schéma Supabase
2. **Ensuite les types** : Génère les types TypeScript
3. **Puis les services** : Logique métier et API calls
4. **Puis les hooks** : Interface entre services et UI
5. **Puis les composants** : UI réutilisables
6. **Enfin les écrans** : Pages complètes

---

## ✅ Definition of Done

Une feature est terminée quand :
- [ ] Le code compile sans erreur
- [ ] Les types TypeScript sont complets
- [ ] La gestion d'erreur est en place
- [ ] Le loading state est géré
- [ ] L'UI est responsive
- [ ] Les animations sont fluides (60fps)
- [ ] Les Edge Cases sont gérés
- [ ] Le code est lisible et documenté
