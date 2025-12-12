# DATABASE_SCHEMA.md - Schéma Supabase Complet

## 📊 Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SUPABASE SCHEMA                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  USERS & AUTH          CHALLENGES           GAMIFICATION                   │
│  ────────────          ──────────           ────────────                   │
│  users                 challenges           user_cards                     │
│  user_stats            challenge_members    user_rewards                   │
│  user_settings         check_ins            spin_history                   │
│  friendships           challenge_chat       daily_challenges               │
│                        stakes (gages)                                       │
│                                                                             │
│  SOCIAL                MASCOTTES            LEAGUES                        │
│  ──────                ─────────            ───────                        │
│  reactions             mascots              leagues                        │
│  kudos                 mascot_skins         league_members                 │
│  notifications                              league_scores                  │
│                                                                             │
│  CONTENT               EVENTS                                              │
│  ───────               ──────                                              │
│  challenge_templates   seasonal_events                                     │
│  card_definitions      event_rewards                                       │
│  title_definitions     community_goals                                     │
│  badge_definitions                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Tables Détaillées

### 1. USERS & AUTH

```sql
-- Extension pour UUID
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Table principale utilisateurs (extends auth.users)
CREATE TABLE public.users (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE NOT NULL,
  display_name TEXT NOT NULL,
  avatar_url TEXT,
  bio TEXT,
  date_of_birth DATE NOT NULL,
  
  -- Progression
  level INTEGER DEFAULT 1,
  xp INTEGER DEFAULT 0,
  kudos_balance INTEGER DEFAULT 0,
  
  -- Streaks
  daily_streak INTEGER DEFAULT 0,
  max_daily_streak INTEGER DEFAULT 0,
  challenge_streak INTEGER DEFAULT 0,
  max_challenge_streak INTEGER DEFAULT 0,
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  last_active_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Constraints
  CONSTRAINT username_length CHECK (char_length(username) >= 3 AND char_length(username) <= 20),
  CONSTRAINT username_format CHECK (username ~ '^[a-zA-Z0-9_]+$'),
  CONSTRAINT age_check CHECK (date_of_birth <= CURRENT_DATE - INTERVAL '16 years')
);

-- Stats détaillées utilisateur
CREATE TABLE public.user_stats (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  
  -- Défis
  challenges_created INTEGER DEFAULT 0,
  challenges_joined INTEGER DEFAULT 0,
  challenges_completed INTEGER DEFAULT 0,
  challenges_failed INTEGER DEFAULT 0,
  
  -- Check-ins
  total_check_ins INTEGER DEFAULT 0,
  photos_posted INTEGER DEFAULT 0,
  videos_posted INTEGER DEFAULT 0,
  
  -- Social
  kudos_given INTEGER DEFAULT 0,
  kudos_received INTEGER DEFAULT 0,
  reactions_given INTEGER DEFAULT 0,
  reactions_received INTEGER DEFAULT 0,
  friends_count INTEGER DEFAULT 0,
  
  -- Gamification
  cards_collected INTEGER DEFAULT 0,
  cards_used INTEGER DEFAULT 0,
  spins_total INTEGER DEFAULT 0,
  jackpots_won INTEGER DEFAULT 0,
  
  -- Titres
  titles_earned INTEGER DEFAULT 0,
  
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Paramètres utilisateur
CREATE TABLE public.user_settings (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  
  -- Notifications
  notif_challenge_reminders BOOLEAN DEFAULT TRUE,
  notif_deadline_warnings BOOLEAN DEFAULT TRUE,
  notif_friend_activity BOOLEAN DEFAULT TRUE,
  notif_kudos BOOLEAN DEFAULT TRUE,
  notif_chat_messages BOOLEAN DEFAULT TRUE,
  notif_daily_challenge BOOLEAN DEFAULT TRUE,
  notif_level_up BOOLEAN DEFAULT TRUE,
  
  -- Rappels
  morning_reminder_time TIME DEFAULT '08:00',
  quiet_hours_start TIME DEFAULT '23:00',
  quiet_hours_end TIME DEFAULT '07:00',
  
  -- Confidentialité
  profile_visibility TEXT DEFAULT 'friends' CHECK (profile_visibility IN ('public', 'friends', 'private')),
  show_stats_publicly BOOLEAN DEFAULT TRUE,
  show_challenges_publicly BOOLEAN DEFAULT FALSE,
  
  -- Préférences
  theme TEXT DEFAULT 'system' CHECK (theme IN ('light', 'dark', 'system')),
  language TEXT DEFAULT 'fr',
  
  -- Cosmétiques actifs
  active_avatar_frame TEXT,
  active_celebration_animation TEXT DEFAULT 'confetti',
  
  -- Titres affichés (max 3)
  displayed_titles TEXT[] DEFAULT '{}',
  
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Amitiés
CREATE TABLE public.friendships (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  friend_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'blocked')),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  accepted_at TIMESTAMPTZ,
  
  UNIQUE(user_id, friend_id),
  CHECK (user_id != friend_id)
);

-- Index
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_level ON users(level);
CREATE INDEX idx_friendships_user ON friendships(user_id);
CREATE INDEX idx_friendships_friend ON friendships(friend_id);
CREATE INDEX idx_friendships_status ON friendships(status);
```

### 2. CHALLENGES

```sql
-- Défis
CREATE TABLE public.challenges (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- Infos de base
  name TEXT NOT NULL,
  description TEXT,
  emoji TEXT DEFAULT '🎯',
  color TEXT DEFAULT '#7C3AED',
  cover_image_url TEXT,
  
  -- Configuration
  duration_days INTEGER NOT NULL CHECK (duration_days IN (7, 14, 21, 30)),
  proof_type TEXT DEFAULT 'photo' CHECK (proof_type IN ('photo', 'photo_gallery', 'video', 'text', 'check')),
  deadline_time TIME DEFAULT '23:59',
  active_days TEXT[] DEFAULT '{mon,tue,wed,thu,fri,sat,sun}',
  mode TEXT DEFAULT 'tolerant' CHECK (mode IN ('strict', 'tolerant')),
  
  -- Gamification
  stakes_enabled BOOLEAN DEFAULT FALSE,
  is_hardcore BOOLEAN DEFAULT FALSE,
  
  -- État
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'completed', 'failed', 'cancelled')),
  
  -- Dates
  start_date DATE,
  end_date DATE,
  join_deadline TIMESTAMPTZ,
  
  -- Stats
  current_day INTEGER DEFAULT 0,
  lives_remaining INTEGER,
  
  -- Mascotte
  mascot_id UUID REFERENCES mascots(id),
  mascot_name TEXT,
  mascot_level INTEGER DEFAULT 1,
  
  -- Template source (si créé depuis un template)
  template_id UUID REFERENCES challenge_templates(id),
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Membres d'un défi
CREATE TABLE public.challenge_members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- Rôle
  role TEXT DEFAULT 'member' CHECK (role IN ('creator', 'member')),
  
  -- État
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'declined', 'removed')),
  
  -- Gage (si activé)
  stake_text TEXT,
  stake_validated BOOLEAN DEFAULT FALSE,
  
  -- Stats du membre pour ce défi
  check_ins_count INTEGER DEFAULT 0,
  missed_days INTEGER DEFAULT 0,
  kudos_received INTEGER DEFAULT 0,
  
  -- Titres gagnés dans ce défi
  titles_earned TEXT[] DEFAULT '{}',
  
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(challenge_id, user_id)
);

-- Check-ins quotidiens
CREATE TABLE public.check_ins (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- Date du check-in
  check_in_date DATE NOT NULL,
  
  -- Preuve
  proof_type TEXT NOT NULL,
  proof_url TEXT, -- Pour photo/video
  proof_text TEXT, -- Pour texte
  caption TEXT, -- Légende optionnelle
  
  -- Métadonnées
  metadata JSONB DEFAULT '{}', -- timestamp_original, device_info, location (opt)
  
  -- Validation
  validated_at TIMESTAMPTZ DEFAULT NOW(),
  is_late BOOLEAN DEFAULT FALSE, -- Validé dans la dernière heure
  
  -- Mini-défi
  daily_challenge_completed BOOLEAN DEFAULT FALSE,
  daily_challenge_bonus_xp INTEGER DEFAULT 0,
  
  -- Récompenses obtenues
  xp_earned INTEGER DEFAULT 0,
  spin_result JSONB, -- Résultat de la roue
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(challenge_id, user_id, check_in_date)
);

-- Chat de groupe
CREATE TABLE public.challenge_chat (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  message_type TEXT DEFAULT 'text' CHECK (message_type IN ('text', 'image', 'gif', 'system')),
  content TEXT NOT NULL,
  media_url TEXT,
  
  -- Pour les messages système
  system_event TEXT, -- 'user_joined', 'day_completed', 'milestone', etc.
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Gages
CREATE TABLE public.stakes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  stake_text TEXT NOT NULL,
  
  -- Validation
  is_triggered BOOLEAN DEFAULT FALSE, -- Le user a fait échouer le défi
  proof_url TEXT, -- Preuve que le gage a été fait
  proof_submitted_at TIMESTAMPTZ,
  
  -- Deadline
  deadline TIMESTAMPTZ,
  is_honored BOOLEAN, -- null = en attente, true = fait, false = non fait
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_challenges_creator ON challenges(creator_id);
CREATE INDEX idx_challenges_status ON challenges(status);
CREATE INDEX idx_challenges_start_date ON challenges(start_date);
CREATE INDEX idx_challenge_members_challenge ON challenge_members(challenge_id);
CREATE INDEX idx_challenge_members_user ON challenge_members(user_id);
CREATE INDEX idx_check_ins_challenge ON check_ins(challenge_id);
CREATE INDEX idx_check_ins_user ON check_ins(user_id);
CREATE INDEX idx_check_ins_date ON check_ins(check_in_date);
CREATE INDEX idx_challenge_chat_challenge ON challenge_chat(challenge_id);
```

### 3. GAMIFICATION

```sql
-- Définitions des cartes (statique, géré par admin)
CREATE TABLE public.card_definitions (
  id TEXT PRIMARY KEY, -- 'shield', 'joker', 'xp_boost', etc.
  name TEXT NOT NULL,
  description TEXT NOT NULL,
  effect_type TEXT NOT NULL, -- 'protection', 'boost', 'social', 'cosmetic'
  effect_data JSONB NOT NULL, -- Paramètres de l'effet
  rarity TEXT NOT NULL CHECK (rarity IN ('common', 'rare', 'epic', 'legendary')),
  icon_url TEXT,
  is_active BOOLEAN DEFAULT TRUE
);

-- Cartes possédées par les utilisateurs
CREATE TABLE public.user_cards (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  card_id TEXT NOT NULL REFERENCES card_definitions(id),
  
  quantity INTEGER DEFAULT 1,
  
  obtained_from TEXT, -- 'spin', 'achievement', 'event', 'level_up', 'purchase'
  obtained_at TIMESTAMPTZ DEFAULT NOW()
);

-- Historique d'utilisation des cartes
CREATE TABLE public.card_usage_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  card_id TEXT NOT NULL REFERENCES card_definitions(id),
  challenge_id UUID REFERENCES challenges(id),
  
  used_at TIMESTAMPTZ DEFAULT NOW(),
  effect_applied JSONB -- Détails de l'effet appliqué
);

-- Historique des spins
CREATE TABLE public.spin_history (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  check_in_id UUID REFERENCES check_ins(id),
  
  spin_type TEXT DEFAULT 'daily' CHECK (spin_type IN ('daily', 'weekly_super', 'event')),
  
  -- Résultat
  result_type TEXT NOT NULL, -- 'xp_5', 'xp_10', 'card_common', 'jackpot', 'nothing', etc.
  result_value JSONB NOT NULL, -- Détails du gain
  
  -- Probabilités utilisées (pour audit)
  probabilities_snapshot JSONB,
  
  spun_at TIMESTAMPTZ DEFAULT NOW()
);

-- Mini-défis quotidiens
CREATE TABLE public.daily_challenges (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Définition
  challenge_type TEXT NOT NULL, -- 'early_bird', 'first', 'storyteller', etc.
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  condition JSONB NOT NULL, -- Conditions à remplir
  xp_reward INTEGER NOT NULL,
  
  -- Disponibilité
  is_active BOOLEAN DEFAULT TRUE,
  weight INTEGER DEFAULT 10, -- Probabilité relative d'apparition
  
  -- Catégorie
  category TEXT CHECK (category IN ('timing', 'content', 'social', 'special'))
);

-- Mini-défis assignés aux utilisateurs chaque jour
CREATE TABLE public.user_daily_challenges (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  daily_challenge_id UUID NOT NULL REFERENCES daily_challenges(id),
  challenge_id UUID NOT NULL REFERENCES challenges(id), -- Le défi concerné
  
  assigned_date DATE NOT NULL,
  
  is_completed BOOLEAN DEFAULT FALSE,
  completed_at TIMESTAMPTZ,
  xp_earned INTEGER,
  
  UNIQUE(user_id, challenge_id, assigned_date)
);

-- Définitions des titres
CREATE TABLE public.title_definitions (
  id TEXT PRIMARY KEY, -- 'early_bird', 'clutch', 'mvp', etc.
  name TEXT NOT NULL,
  icon TEXT NOT NULL,
  description TEXT NOT NULL,
  category TEXT CHECK (category IN ('timing', 'social', 'performance', 'fun')),
  condition JSONB NOT NULL, -- Conditions pour obtenir le titre
  is_active BOOLEAN DEFAULT TRUE
);

-- Titres obtenus par les utilisateurs
CREATE TABLE public.user_titles (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title_id TEXT NOT NULL REFERENCES title_definitions(id),
  
  -- Contexte d'obtention
  challenge_id UUID REFERENCES challenges(id),
  earned_count INTEGER DEFAULT 1, -- Combien de fois obtenu
  
  first_earned_at TIMESTAMPTZ DEFAULT NOW(),
  last_earned_at TIMESTAMPTZ DEFAULT NOW()
);

-- Définitions des badges
CREATE TABLE public.badge_definitions (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  icon TEXT NOT NULL,
  description TEXT NOT NULL,
  category TEXT, -- 'progression', 'category', 'special'
  condition JSONB NOT NULL,
  rarity TEXT CHECK (rarity IN ('common', 'rare', 'epic', 'legendary', 'mythic')),
  is_active BOOLEAN DEFAULT TRUE
);

-- Badges obtenus
CREATE TABLE public.user_badges (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  badge_id TEXT NOT NULL REFERENCES badge_definitions(id),
  
  earned_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, badge_id)
);

-- Récompenses diverses (XP, kudos gagnés)
CREATE TABLE public.user_rewards (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  reward_type TEXT NOT NULL, -- 'xp', 'kudos', 'card'
  amount INTEGER NOT NULL,
  source TEXT NOT NULL, -- 'check_in', 'spin', 'achievement', 'challenge_complete', etc.
  source_id UUID, -- ID de la source
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_user_cards_user ON user_cards(user_id);
CREATE INDEX idx_spin_history_user ON spin_history(user_id);
CREATE INDEX idx_user_daily_challenges_user ON user_daily_challenges(user_id);
CREATE INDEX idx_user_daily_challenges_date ON user_daily_challenges(assigned_date);
CREATE INDEX idx_user_titles_user ON user_titles(user_id);
CREATE INDEX idx_user_badges_user ON user_badges(user_id);
CREATE INDEX idx_user_rewards_user ON user_rewards(user_id);
```

### 4. MASCOTTES

```sql
-- Types de mascottes disponibles
CREATE TABLE public.mascot_types (
  id TEXT PRIMARY KEY, -- 'piu', 'mochi', 'buddy', etc.
  name TEXT NOT NULL,
  default_name TEXT NOT NULL,
  description TEXT,
  base_image_url TEXT NOT NULL,
  
  -- Images par niveau (1-5)
  level_images JSONB NOT NULL, -- {"1": "url", "2": "url", ...}
  
  -- États émotionnels
  emotion_images JSONB NOT NULL, -- {"happy": "url", "sad": "url", ...}
  
  is_active BOOLEAN DEFAULT TRUE
);

-- Mascottes actives dans les défis
CREATE TABLE public.mascots (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  
  mascot_type_id TEXT NOT NULL REFERENCES mascot_types(id),
  name TEXT NOT NULL,
  
  -- Progression
  level INTEGER DEFAULT 1 CHECK (level >= 1 AND level <= 5),
  happiness INTEGER DEFAULT 50 CHECK (happiness >= 0 AND happiness <= 100),
  
  -- État actuel
  current_emotion TEXT DEFAULT 'happy',
  
  -- Skin actif
  active_skin_id TEXT REFERENCES mascot_skins(id),
  
  -- Interactions
  last_fed_at TIMESTAMPTZ,
  last_pet_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Skins de mascottes
CREATE TABLE public.mascot_skins (
  id TEXT PRIMARY KEY, -- 'halloween_piu', 'christmas_mochi', etc.
  mascot_type_id TEXT NOT NULL REFERENCES mascot_types(id),
  
  name TEXT NOT NULL,
  description TEXT,
  image_url TEXT NOT NULL,
  
  -- Obtention
  unlock_type TEXT NOT NULL, -- 'event', 'achievement', 'loyalty', 'card'
  unlock_condition JSONB,
  
  is_active BOOLEAN DEFAULT TRUE
);

-- Skins débloqués par utilisateur
CREATE TABLE public.user_mascot_skins (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  skin_id TEXT NOT NULL REFERENCES mascot_skins(id),
  
  unlocked_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, skin_id)
);

-- Galerie des mascottes évoluées (souvenirs)
CREATE TABLE public.mascot_gallery (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  mascot_type_id TEXT NOT NULL,
  mascot_name TEXT NOT NULL,
  final_level INTEGER NOT NULL,
  skin_id TEXT,
  
  -- Contexte
  challenge_id UUID REFERENCES challenges(id),
  challenge_name TEXT NOT NULL,
  challenge_completed_at TIMESTAMPTZ NOT NULL,
  
  -- Autres membres
  squad_members JSONB, -- [{user_id, username, avatar_url}]
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 5. SOCIAL

```sql
-- Réactions aux check-ins
CREATE TABLE public.reactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  check_in_id UUID NOT NULL REFERENCES check_ins(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  reaction_type TEXT NOT NULL, -- 'fire', 'heart', 'laugh', 'wow', 'strong', 'clap', 'mindblown'
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(check_in_id, user_id, reaction_type)
);

-- Kudos
CREATE TABLE public.kudos (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  from_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  to_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  check_in_id UUID REFERENCES check_ins(id) ON DELETE CASCADE,
  
  amount INTEGER NOT NULL CHECK (amount > 0),
  is_super BOOLEAN DEFAULT FALSE, -- Super kudos (x5)
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Nudges
CREATE TABLE public.nudges (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  from_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  to_user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  
  nudge_date DATE NOT NULL,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Notifications
CREATE TABLE public.notifications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  type TEXT NOT NULL, -- 'challenge_invite', 'check_in', 'kudos', 'nudge', 'level_up', etc.
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  data JSONB, -- Données additionnelles pour deep linking
  
  is_read BOOLEAN DEFAULT FALSE,
  read_at TIMESTAMPTZ,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_reactions_check_in ON reactions(check_in_id);
CREATE INDEX idx_kudos_to_user ON kudos(to_user_id);
CREATE INDEX idx_kudos_check_in ON kudos(check_in_id);
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_read ON notifications(is_read);
```

### 6. LEAGUES (Mode Hardcore)

```sql
-- Ligues
CREATE TABLE public.leagues (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Identifiant unique pour le matchmaking
  template_id UUID REFERENCES challenge_templates(id),
  duration_days INTEGER NOT NULL,
  start_date DATE NOT NULL,
  
  name TEXT NOT NULL, -- Auto-généré : "30j Sport - Dec 2024 #3"
  
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'completed')),
  
  -- Stats
  total_groups INTEGER DEFAULT 0,
  groups_remaining INTEGER DEFAULT 0,
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);

-- Groupes dans une ligue
CREATE TABLE public.league_members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  league_id UUID NOT NULL REFERENCES leagues(id) ON DELETE CASCADE,
  challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
  
  -- Score
  score INTEGER DEFAULT 0,
  rank INTEGER,
  
  -- Statut
  is_eliminated BOOLEAN DEFAULT FALSE,
  eliminated_at TIMESTAMPTZ,
  eliminated_day INTEGER,
  
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(league_id, challenge_id)
);

-- Historique des scores quotidiens
CREATE TABLE public.league_scores (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  league_member_id UUID NOT NULL REFERENCES league_members(id) ON DELETE CASCADE,
  
  day_number INTEGER NOT NULL,
  date DATE NOT NULL,
  
  -- Points du jour
  base_points INTEGER DEFAULT 0,
  bonus_points INTEGER DEFAULT 0,
  total_points INTEGER DEFAULT 0,
  
  -- Détails
  all_validated BOOLEAN DEFAULT FALSE,
  average_validation_time TIME,
  sync_bonus BOOLEAN DEFAULT FALSE, -- Tous validé dans la même heure
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_leagues_start_date ON leagues(start_date);
CREATE INDEX idx_league_members_league ON league_members(league_id);
CREATE INDEX idx_league_members_score ON league_members(score DESC);
```

### 7. EVENTS

```sql
-- Événements saisonniers
CREATE TABLE public.seasonal_events (
  id TEXT PRIMARY KEY, -- 'halloween_2024', 'christmas_2024'
  
  name TEXT NOT NULL,
  description TEXT,
  theme_color TEXT,
  banner_image_url TEXT,
  
  start_date TIMESTAMPTZ NOT NULL,
  end_date TIMESTAMPTZ NOT NULL,
  
  -- Objectif communautaire
  community_goal_type TEXT, -- 'check_ins', 'challenges_completed'
  community_goal_target INTEGER,
  community_goal_current INTEGER DEFAULT 0,
  community_goal_reached BOOLEAN DEFAULT FALSE,
  
  -- Récompenses
  rewards JSONB, -- Liste des récompenses disponibles
  
  is_active BOOLEAN DEFAULT TRUE
);

-- Templates de défis spéciaux pour events
CREATE TABLE public.event_challenge_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  event_id TEXT NOT NULL REFERENCES seasonal_events(id),
  template_id UUID NOT NULL REFERENCES challenge_templates(id),
  
  is_featured BOOLEAN DEFAULT FALSE
);

-- Progression utilisateur dans un event
CREATE TABLE public.user_event_progress (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  event_id TEXT NOT NULL REFERENCES seasonal_events(id),
  
  -- Points d'event
  event_points INTEGER DEFAULT 0,
  
  -- Contribution au goal communautaire
  contribution INTEGER DEFAULT 0,
  
  -- Récompenses réclamées
  claimed_rewards TEXT[] DEFAULT '{}',
  
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  UNIQUE(user_id, event_id)
);
```

### 8. TEMPLATES

```sql
-- Templates de défis
CREATE TABLE public.challenge_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Infos
  name TEXT NOT NULL,
  description TEXT NOT NULL,
  emoji TEXT NOT NULL,
  
  -- Catégorie
  category TEXT NOT NULL, -- 'sport', 'learning', 'finance', 'wellness', 'creativity', 'social'
  
  -- Configuration par défaut
  default_duration INTEGER NOT NULL,
  default_proof_type TEXT NOT NULL,
  default_mode TEXT DEFAULT 'tolerant',
  
  -- Popularité
  times_used INTEGER DEFAULT 0,
  
  -- Affichage
  is_featured BOOLEAN DEFAULT FALSE,
  display_order INTEGER DEFAULT 100,
  
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 9. COSMÉTIQUES

```sql
-- Définitions des cosmétiques
CREATE TABLE public.cosmetic_definitions (
  id TEXT PRIMARY KEY,
  
  type TEXT NOT NULL CHECK (type IN ('color', 'avatar_frame', 'celebration_animation', 'emoji_pack')),
  name TEXT NOT NULL,
  description TEXT,
  preview_url TEXT,
  
  -- Obtention
  unlock_type TEXT NOT NULL, -- 'level', 'kudos', 'event', 'achievement'
  unlock_requirement JSONB, -- {"level": 5} ou {"kudos": 100}
  kudos_price INTEGER, -- Si achetable avec kudos
  
  is_active BOOLEAN DEFAULT TRUE
);

-- Cosmétiques débloqués
CREATE TABLE public.user_cosmetics (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  cosmetic_id TEXT NOT NULL REFERENCES cosmetic_definitions(id),
  
  unlocked_at TIMESTAMPTZ DEFAULT NOW(),
  unlock_source TEXT, -- 'level_up', 'kudos_purchase', 'event', 'achievement'
  
  UNIQUE(user_id, cosmetic_id)
);
```

---

## 🔐 Row Level Security (RLS)

```sql
-- Activer RLS sur toutes les tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE challenges ENABLE ROW LEVEL SECURITY;
ALTER TABLE challenge_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE check_ins ENABLE ROW LEVEL SECURITY;
-- ... etc pour toutes les tables

-- Policies pour users
CREATE POLICY "Users can view their own profile"
  ON users FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update their own profile"
  ON users FOR UPDATE
  USING (auth.uid() = id);

-- Policies pour challenges
CREATE POLICY "Anyone can view active challenges they're part of"
  ON challenges FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM challenge_members
      WHERE challenge_members.challenge_id = challenges.id
      AND challenge_members.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can create challenges"
  ON challenges FOR INSERT
  WITH CHECK (auth.uid() = creator_id);

-- Policies pour check_ins
CREATE POLICY "Members can view check-ins of their challenges"
  ON check_ins FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM challenge_members
      WHERE challenge_members.challenge_id = check_ins.challenge_id
      AND challenge_members.user_id = auth.uid()
    )
  );

CREATE POLICY "Users can create their own check-ins"
  ON check_ins FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- ... Ajouter les policies pour chaque table
```

---

## 🔄 Functions & Triggers

```sql
-- Fonction pour mettre à jour updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Appliquer le trigger
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

-- Fonction pour calculer le niveau basé sur XP
CREATE OR REPLACE FUNCTION calculate_level(xp INTEGER)
RETURNS INTEGER AS $$
BEGIN
  RETURN CASE
    WHEN xp >= 10000 THEN 10
    WHEN xp >= 6000 THEN 9
    WHEN xp >= 4000 THEN 8
    WHEN xp >= 2500 THEN 7
    WHEN xp >= 1500 THEN 6
    WHEN xp >= 1000 THEN 5
    WHEN xp >= 600 THEN 4
    WHEN xp >= 300 THEN 3
    WHEN xp >= 100 THEN 2
    ELSE 1
  END;
END;
$$ LANGUAGE plpgsql;

-- Trigger pour mettre à jour le niveau quand XP change
CREATE OR REPLACE FUNCTION update_user_level()
RETURNS TRIGGER AS $$
BEGIN
  NEW.level = calculate_level(NEW.xp);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_level_on_xp_change
  BEFORE UPDATE OF xp ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_user_level();

-- Fonction pour vérifier si un check-in est valide
CREATE OR REPLACE FUNCTION validate_check_in()
RETURNS TRIGGER AS $$
DECLARE
  challenge_record RECORD;
  existing_check_in RECORD;
BEGIN
  -- Récupérer le défi
  SELECT * INTO challenge_record
  FROM challenges
  WHERE id = NEW.challenge_id;
  
  -- Vérifier que le défi est actif
  IF challenge_record.status != 'active' THEN
    RAISE EXCEPTION 'Le défi n''est pas actif';
  END IF;
  
  -- Vérifier qu'il n'y a pas déjà un check-in pour aujourd'hui
  SELECT * INTO existing_check_in
  FROM check_ins
  WHERE challenge_id = NEW.challenge_id
    AND user_id = NEW.user_id
    AND check_in_date = NEW.check_in_date;
  
  IF FOUND THEN
    RAISE EXCEPTION 'Check-in déjà effectué pour aujourd''hui';
  END IF;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_check_in_before_insert
  BEFORE INSERT ON check_ins
  FOR EACH ROW
  EXECUTE FUNCTION validate_check_in();
```

---

## 📊 Vues Utiles

```sql
-- Vue pour les stats d'un défi
CREATE VIEW challenge_stats AS
SELECT
  c.id as challenge_id,
  c.name,
  c.status,
  c.current_day,
  c.duration_days,
  c.lives_remaining,
  COUNT(DISTINCT cm.user_id) as member_count,
  COUNT(DISTINCT ci.id) as total_check_ins,
  COUNT(DISTINCT CASE WHEN ci.check_in_date = CURRENT_DATE THEN ci.id END) as today_check_ins
FROM challenges c
LEFT JOIN challenge_members cm ON c.id = cm.challenge_id AND cm.status = 'accepted'
LEFT JOIN check_ins ci ON c.id = ci.challenge_id
GROUP BY c.id;

-- Vue pour le leaderboard kudos
CREATE VIEW kudos_leaderboard AS
SELECT
  u.id,
  u.username,
  u.avatar_url,
  u.level,
  us.kudos_received,
  RANK() OVER (ORDER BY us.kudos_received DESC) as rank
FROM users u
JOIN user_stats us ON u.id = us.user_id
ORDER BY us.kudos_received DESC;
```

---

*Schéma complet pour ChallengePact - Supabase*
*Version 1.0*
