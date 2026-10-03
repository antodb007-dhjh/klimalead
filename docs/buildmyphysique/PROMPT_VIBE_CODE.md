# BuildMyPhysique — prompt de vibe code (version complète)

> Comment l'utiliser : colle le **prompt maître** (section 1) une seule fois au début dans ton outil
> (Claude Code, Cursor, Lovable, Bolt, v0…). Ensuite, envoie **un prompt de phase à la fois** (section 2)
> et teste chaque phase avant de passer à la suivante. Ne colle pas tout d'un bloc : les outils de vibe code
> livrent beaucoup mieux par lots courts et vérifiables.
>
> Remplace les `[CROCHETS]` par tes vraies valeurs. Si ton app existe déjà dans une autre stack,
> garde la tienne et dis-le à l'outil dans `[STACK ACTUELLE]`.

---

## 1. Prompt maître (contexte permanent)

```text
Tu es un senior full-stack engineer et product designer spécialisé dans les apps fitness
grand public US à abonnement (références : Cal AI, MacroFactor, Fitbod, Ladder, Future, BetterMe,
Mad Muscles). Tu vas FINALISER mon SaaS "BuildMyPhysique" (https://buildmyphysique.vercel.app).

## Le produit
L'utilisateur importe une photo de son corps + ses objectifs. L'app génère :
1. une projection visuelle réaliste de lui dans 3 / 6 / 12 mois ("future you"),
2. un plan d'entraînement détaillé, semaine par semaine,
3. un plan nutritionnel détaillé (calories, macros, repas, liste de courses),
4. une feuille de route mois par mois pour atteindre l'objectif.

## Le vrai problème à résoudre
La landing et la promesse convertissent correctement. Ce qui manque, c'est le PRODUIT APRÈS LE PAIEMENT.
Aujourd'hui l'utilisateur paie, reçoit un plan, et n'a plus de raison de revenir → churn.
L'objectif est une app qu'on ouvre CHAQUE JOUR : séance du jour, repas du jour, suivi,
check-in hebdomadaire qui ajuste le plan, preuves visibles de progression. Le MRR vient de la rétention.

## Stack
[STACK ACTUELLE — sinon utilise :]
- Next.js 15 (App Router) + TypeScript strict + Tailwind + shadcn/ui + Framer Motion
- Supabase : Auth (email magic link + Google + Apple), Postgres avec RLS sur TOUTES les tables,
  Storage en buckets PRIVÉS (URLs signées courtes) pour les photos
- Stripe : Checkout + Billing (abonnements), Customer Portal, webhooks idempotents
- IA texte/vision : un LLM avec sortie JSON validée par schéma Zod (analyse photo, plans, coach, check-ins)
- IA image : un modèle d'édition d'image (image-to-image) pour la projection "future you",
  derrière une interface `ImageProvider` interchangeable (variable d'env), jamais codé en dur
- Jobs asynchrones (Inngest ou Trigger.dev, ou file Supabase) pour toute génération > 10 s
- Emails transactionnels : Resend + React Email
- Analytics produit : PostHog (événements funnel + rétention)
- PWA installable (manifest, icônes, push web notifications)
- Déploiement Vercel. Toutes les clés en variables d'environnement, fichier `.env.example` à jour.

## Direction artistique (voir mon visuel de référence)
- Dark mode premium : fond #0A0A0A / cartes #141414, accent lime néon #C6FF3D, texte blanc/gris.
- Typo grotesque bold et serrée pour les titres (ex. "Inter Tight" / "Clash Display"), chiffres en tabular-nums.
- Mobile-first absolu (390 px), desktop = app centrée + sidebar. Gros CTA arrondis lime pleine largeur.
- Micro-animations : scan laser sur la photo, anneaux de progression, slider avant/après, confettis sobres
  sur les records. Haptique-like via animations, jamais lourdes.
- Ton : motivant, direct, "Discipline today. A stronger tomorrow." UI en anglais (marché US),
  i18n prête (next-intl) avec le français en 2e langue.

## Règles non négociables
- Projections RÉALISTES et calibrées : perte de gras 0,5–1 % du poids/semaine, prise de muscle
  selon niveau (débutant ~0,5–1 kg/mois, intermédiaire ~0,25–0,5, avancé moins). Le chiffre affiché
  ("87 kg → 72 kg, 28 % → 12 %") est calculé par une fonction déterministe, PAS inventé par l'IA.
  L'image est générée pour correspondre à ces chiffres. Mention "AI visualization, results vary".
- Sécurité : 18+ uniquement, refus des photos de mineurs/nudité (modération avant génération),
  photos privées, suppression de compte + données en 1 clic (RGPD/CCPA), aucune photo dans les logs.
- Santé : pas de conseil médical, questionnaire de contre-indications (blessures, grossesse,
  troubles alimentaires → on n'affiche pas de déficit agressif et on renvoie vers un pro),
  plancher calorique (jamais < 1500 kcal homme / 1200 femme), disclaimer clair.
- Chaque fonctionnalité payante est protégée côté SERVEUR par l'entitlement Stripe, pas seulement côté UI.
- Toute sortie IA passe par un schéma Zod ; si invalide → retry, puis fallback propre.
- Code propre, typé, composants réutilisables, pas de TODO laissés silencieusement.
  À la fin de chaque phase : liste ce qui est fait, ce qui est testé, et ce qui reste.
```

---

## 2. Prompts de phase (à envoyer un par un)

### Phase 1 — Modèle de données, auth, entitlements

```text
Phase 1. Mets en place les fondations.

Tables Supabase (avec RLS "owner only") :
- profiles (id, sex, birth_year, height_cm, units metric/imperial, timezone, onboarding_completed)
- goals (user_id, goal_type: lose_fat | build_muscle | recomp | get_lean_athletic,
  target_weight, target_bf, horizon_months 3|6|12, experience beginner|intermediate|advanced,
  days_per_week, session_minutes, equipment: gym|home_dumbbells|bodyweight, injuries text[],
  diet: omnivore|vegetarian|vegan|halal|no_pork…, allergies text[], disliked_foods text[], meals_per_day)
- body_scans (user_id, photo_path, est_weight, est_bf_pct, muscle_scores jsonb {chest, shoulders,
  arms, back, abs, legs}, posture_notes, symmetry_notes, created_at)
- projections (scan_id, horizon_months, image_path, projected_weight, projected_bf, status)
- programs (user_id, version, starts_on, phases jsonb), workouts, workout_exercises
- exercises (bibliothèque : nom, muscles, équipement, consignes, vidéo/gif, alternatives[])
- workout_logs + set_logs (reps, poids, RPE, completed_at)
- nutrition_plans (version, kcal, protein_g, carbs_g, fat_g, meal_templates jsonb)
- meal_logs (photo_path?, description, kcal, p, c, f, source: plan|photo|manual|barcode)
- weigh_ins, progress_photos, check_ins (weekly), streaks, achievements
- subscriptions (stripe_customer_id, status, plan, current_period_end, trial_end)
- ai_jobs (type, status, input_hash, output, error) pour suivre les générations async

Auth : magic link + Google + Apple. Helper serveur `requirePro()` qui lit l'entitlement.
Livrables : migrations SQL, types générés, seed de 150 exercices réalistes avec alternatives,
middleware de protection des routes /app/*.
```

### Phase 2 — Onboarding "quiz" + scan + paywall (le funnel)

```text
Phase 2. Refais le funnel d'acquisition comme les meilleures apps US (BetterMe / Mad Muscles / Cal AI) :
un quiz long, engageant, une question par écran, barre de progression en haut.

Écrans (≈ 20) : objectif principal → zone à améliorer (body map cliquable) → âge → taille →
poids actuel → poids cible (avec message réaliste dynamique "achievable by [date]") → niveau →
jours/semaine → durée séance → lieu/équipement → blessures → régime → allergies →
repas/jour → sommeil → stress → "ce qui t'a fait abandonner avant" → écran social proof
intercalé (témoignage + stat) tous les 4–5 écrans → upload photo.

Upload photo : guide de pose (silhouette en surimpression, lumière, plein cadre, tenue de sport),
vérification qualité côté client (résolution, flou), consentement explicite, modération serveur.

Écran "Scanning…" exactement comme mon visuel : photo avec grille 3D lime, ligne laser qui balaie,
liste qui se coche (Body composition, Muscle mass, Body fat %, Posture, Symmetry), anneau en %.
Pendant ce temps : vrai appel vision → body_scans + calcul déterministe des projections
+ job de génération d'image lancé en arrière-plan.

Reveal "This is you in 6 months 👀" : split avant/après avec trait lumineux, poids et % de gras
avant → après, 4 bénéfices (More Energy, A Stronger You, Better Health, Real Results).
L'image "après" est visible mais partiellement floutée / watermarkée tant que non payé.

Écran "Your 6-Month Plan is Ready" : timeline verticale Month 1…6 (Build the foundation,
Increase strength, See the changes, Build momentum, Transform, A new you) avec une vignette
de projection par étape, puis CTA "Start My Plan →".

Paywall : 3 offres, annuelle mise en avant avec essai gratuit 3 jours, prix
[9,99 $/semaine], [29,99 $/mois], [59,99 $/an], ancrage "per day", liste des features,
garantie, avis, FAQ, timer d'offre discret honnête (pas de faux compte à rebours qui se reset).
Si l'utilisateur ferme : une offre de rattrapage unique (-X % annuel). Stripe Checkout,
retour sur /app/welcome. Tracke chaque écran dans PostHog pour voir où ça décroche.
```

### Phase 3 — Génération du plan (le cœur de la valeur)

```text
Phase 3. Génération des plans, en job asynchrone, avec écran d'attente animé.

Nutrition (déterministe d'abord, IA ensuite) :
- BMR Mifflin-St Jeor → TDEE avec facteur d'activité → déficit/surplus selon objectif et vitesse réaliste.
- Protéines 1,6–2,2 g/kg, lipides ≥ 0,6 g/kg, reste en glucides. Planchers de sécurité.
- L'IA ne fait QUE composer : 7 jours de menus respectant kcal/macros (±5 %), régime, allergies,
  aliments détestés, budget, temps de préparation ; recettes avec quantités en g ;
  liste de courses agrégée par rayon ; bouton "swap" sur chaque repas (même macros).

Entraînement :
- Choix du split selon jours/semaine (Full body 3j, Upper/Lower 4j, PPL 5–6j), équipement, blessures.
- Pour chaque séance : échauffement, exercices issus de la table `exercises` (jamais inventés),
  séries × reps × RPE/RIR, repos, consignes, alternatives si matériel occupé.
- Périodisation sur 6 mois en blocs de 4 semaines + deload, cohérente avec la timeline Month 1–6.
- Progression automatique (double progression) basée sur les logs réels.

Roadmap : par mois, objectifs mesurables (poids, tour de taille, charges cibles sur 3 lifts clés),
habitudes (pas/jour, sommeil, eau) et la projection visuelle correspondante.
Tout est versionné (programs.version, nutrition_plans.version) pour pouvoir ajuster sans perdre l'historique.
```

### Phase 4 — L'app quotidienne (ce qui fait payer chaque semaine)

```text
Phase 4. Construis l'expérience post-paiement. Navigation en bas : Today · Train · Eat · Progress · Coach.

TODAY (écran d'accueil) :
- Salutation + streak 🔥, anneau calories restantes + 3 barres macros, séance du jour (ou repos actif),
  checklist d'habitudes (pas, eau, sommeil, protéines), prochain check-in, carte "Future you"
  avec le % d'avancement vers la projection.

TRAIN :
- Lecteur de séance plein écran : exercice courant, gif/vidéo, séries à cocher, champs poids/reps
  pré-remplis avec la cible du jour (progressive overload), minuteur de repos auto avec son/vibration,
  swap d'exercice, notes. Fonctionne hors ligne et synchronise ensuite.
- Fin de séance : récap volume, records personnels (PR) avec animation, partage en story.
- Historique et graphiques par exercice (1RM estimé, volume).

EAT :
- Plan du jour avec cases "mangé", swap de repas, recettes.
- Logging rapide : photo du repas → estimation IA kcal/macros éditable (style Cal AI),
  recherche texte, scan code-barres (Open Food Facts), favoris, "comme hier".
- Liste de courses de la semaine cochable.

PROGRESS :
- Courbe de poids lissée (moyenne mobile 7 j), mensurations, % gras estimé.
- Photos de progression : timeline + slider avant/après comparé à la PROJECTION ("you vs. future you").
- Badges et paliers (premier mois, 10 séances, -5 kg, etc.).

COACH (IA) :
- Chat contextuel qui connaît le profil, le plan, les logs des 14 derniers jours.
  Peut proposer des modifications (ex. "remplace le squat, j'ai mal au genou") que l'utilisateur
  valide en 1 tap → nouvelle version du plan. Garde-fous santé, quota raisonnable par jour.
```

### Phase 5 — Boucle d'adaptation & rétention

```text
Phase 5. Rends le plan vivant.

Check-in hebdo (dimanche, 2 min) : poids moyen, photo optionnelle (même pose guidée),
énergie/faim/sommeil/adhérence (1–5). Algorithme :
- perte trop lente 2 semaines de suite → -100 à -150 kcal ; trop rapide → +100 ;
- charges qui stagnent → ajuste volume ou deload ;
- adhérence faible → simplifie (moins de séances, repas plus simples) au lieu de culpabiliser.
L'IA rédige un résumé "This week" + 3 actions. Nouvelle photo → re-scan + projection mise à jour.

Re-engagement :
- Push web + emails Resend : rappel séance à l'heure choisie, check-in, streak en danger,
  récap hebdo, "1 month in: here's your before/after" (moment émotionnel fort = partage).
- Rapport mensuel visuel (PDF/image partageable) : avant / maintenant / projection.
- Programme de parrainage : 1 mois offert aux deux.

Billing :
- Customer Portal Stripe, gestion de l'essai, emails de fin d'essai J-1.
- Flow d'annulation : raison → offre adaptée (pause 1 mois, -50 % 3 mois, passage annuel) → annulation réelle possible.
- Webhooks : checkout.session.completed, customer.subscription.updated/deleted,
  invoice.payment_failed (dunning + bannière in-app). Accès coupé proprement à la fin de période.
```

### Phase 6 — Qualité, sécurité, lancement

```text
Phase 6. Prépare la mise en prod.
- Tests : unitaires sur les calculs (TDEE, macros, projections, progression), tests d'intégration
  des webhooks Stripe (Stripe CLI), un parcours E2E Playwright : quiz → photo fixture → paywall
  (mode test) → Today → logger une séance → check-in.
- Sécurité : RLS vérifiée table par table, buckets privés, rate limiting sur les routes IA,
  modération photo, suppression de compte qui purge Storage + DB + Stripe customer.
- Perf : images optimisées, LCP < 2,5 s sur la landing, skeletons partout, états vides et erreurs soignés.
- Légal : CGU, politique de confidentialité, disclaimer santé, consentement cookies.
- Admin minimal (/admin, rôle protégé) : utilisateurs, abonnements, jobs IA en échec, coût IA par user.
- Dashboard PostHog : funnel quiz → paywall → essai → payé, rétention J1/J7/J30, churn par cohorte.
Donne-moi enfin la checklist de lancement et les variables d'env à remplir.
```

---

## 3. Pourquoi cette structure (ce que font les apps US qui retiennent)

| Levier | Ce que ça apporte | Où dans le prompt |
|---|---|---|
| Quiz long avant paywall | Investissement émotionnel + données pour personnaliser → conversion plus forte | Phase 2 |
| Reveal "future you" flouté | Le désir est créé avant le prix, le flou donne la raison de payer | Phase 2 |
| Chiffres calculés, pas inventés | Crédibilité, moins de remboursements et de plaintes | Maître + Phase 3 |
| Séance guidée + logging | Raison d'ouvrir l'app 3–6 fois/semaine | Phase 4 |
| Logging repas par photo | Raison d'ouvrir l'app chaque jour | Phase 4 |
| Check-in qui ajuste le plan | Le plan "vit" → l'abonnement a du sens au-delà du premier PDF | Phase 5 |
| Avant/après vs. projection | Preuve de progrès + contenu partageable (acquisition gratuite) | Phases 4–5 |
| Flow d'annulation avec offres | Récupère une partie du churn | Phase 5 |

## 4. Conseils d'exécution

- Commence par la Phase 1 même si ton app existe : demande d'abord à l'outil d'**auditer ton code
  actuel** par rapport au prompt maître et de lister les écarts avant d'écrire du code.
- Garde le modèle d'image derrière une interface : les modèles et prix changent vite, et la
  génération d'image est ton poste de coût n° 1. Mets les projections en cache (une par horizon et par scan).
- L'offre hebdo à 9,99 $ convertit bien en impulsion mais churne vite ; c'est l'annuel avec essai
  qui fait le LTV. Mesure les deux dans PostHog avant de trancher.
- Vérifie la conformité des stores et de Stripe pour les contenus "body transformation"
  (pas de promesses de résultats garantis, disclaimer visible).
