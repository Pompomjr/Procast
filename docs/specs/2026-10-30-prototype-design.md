# Procast — prototype de démo : conception

Date : 30 octobre 2026 · Validé avec Mattia (morceaux 1 et 2 en conversation, « lance le déploiement »).

## But

Un vrai site qui marche, à montrer aux financeurs (Innoviris, etc.) en rendez-vous. Comptes et vidéos de démo
créés par l'équipe et ses proches — **pas d'inscription d'inconnus** à ce stade, donc RGPD « léger » (hébergement
en Europe, vidéos privées, suppression possible), le RGPD complet viendra avant un pilote ouvert.

Site web d'abord (téléphone pour le candidat, ordinateur pour le recruteur). L'appli mobile viendra ensuite
(React Native) en réutilisant le même serveur Supabase.

Référence visuelle : la maquette v1 (`maquette/`, https://claude.ai/artifact/QoM6FiXyVLAz8wNAYPSobe).

## Décisions produit (rappel)

- Pas de swipe : l'entreprise publie une offre → le candidat postule avec ses capsules → l'entreprise choisit.
- Capsules guidées de **60 s** chacune, arrêt automatique à 60 s, on peut refaire autant qu'on veut.
  - Candidat (5) : Qui je suis · Mon diplôme · Mon expérience · Mes langues (une seule capsule, ~20 s par langue) · Hors du travail.
  - Entreprise (4) : Le lieu · Le poste · L'équipe · Nos valeurs.
- Interface en français uniquement pour le prototype.

## Pile technique

| Rôle | Choix |
|---|---|
| Site | Next.js (App Router, TypeScript), Tailwind CSS, police Manrope |
| Comptes, base, fichiers, temps réel | Supabase, région **eu-central-1 (Francfort)** |
| Hébergement du site | Vercel |
| Tests | Vitest (logique pure), script d'intégration RLS contre le projet Supabase |

Secours vidéo : si le test iPhone/Android échoue (format non lisible d'un appareil à l'autre), ajouter Mux ou
Cloudflare Stream pour convertir les vidéos. Rien d'autre ne change.

Dépôt : `C:\dev\projects\Procast` (git), site dans `app/`, migrations SQL dans `app/supabase/migrations/`.

## Écrans

### Candidat (téléphone d'abord)
1. **Inscription / connexion** — e-mail + mot de passe ; puis profil : nom, prénom, date de naissance, ville,
   domaine, langues, lien LinkedIn, CV (PDF ≤ 5 Mo), photo ; case « Visible dans la base de talents » (décochée).
2. **Mes capsules** — 5 capsules, état (à tourner / tournée), progression.
3. **Tournage** — caméra, consigne et pistes, compte à rebours 60 s, arrêt auto, revoir / garder / refaire.
4. **Explorer** — offres publiées, filtre par domaine, capsule « Le poste » de l'entreprise.
5. **Offre** — 4 capsules entreprise, conditions, « Postuler avec mes capsules » (exige 5/5 capsules).
6. **Mes candidatures** — statut (envoyée → vue → entretien proposé → décision) ; prochains entretiens en tête.
7. **Messages** — une conversation par candidature ou invitation ; choix d'un créneau d'entretien ; bouton
   « Ajouter à mon calendrier » (.ics).
8. **Mon profil** — tel que les entreprises le voient ; modifier.

### Entreprise (ordinateur d'abord)
1. **Page entreprise** — infos + 4 capsules.
2. **Publier une offre** — formulaire, brouillon / publiée / fermée.
3. **Candidatures reçues** — par offre, statut : à regarder / retenu / pas retenu / entretien.
   Ouvrir une candidature la marque « vue » côté candidat.
4. **Fiche candidat** — 5 capsules, CV, notes privées de l'équipe, « Proposer un entretien » (2–3 créneaux,
   sur place ou visio), « Pas cette fois ».
5. **Base de talents** — candidats visibles uniquement ; filtres domaine, ville, langues, disponibilité ;
   capsule « Qui je suis » ; « Inviter à postuler » (ouvre une conversation proposant une offre).
6. **Agenda** — semaine : entretiens confirmés + créneaux en attente ; clic → fiche candidat ; .ics.
7. **Messages**.

Hors prototype : export PDF du profil, NL/EN, e-mails de notification, synchro Outlook/Google, plusieurs
recruteurs gérés finement, RGPD complet, modération.

## Données

| Table | Colonnes principales |
|---|---|
| `profiles` | `id` (= auth user), `role` (`candidate` \| `recruiter`), `created_at` |
| `candidates` | `id` → profiles, `first_name`, `last_name`, `birth_date`, `city`, `domain`, `languages text[]`, `linkedin_url`, `cv_path`, `photo_path`, `available bool`, `visible_in_talent_base bool default false` |
| `companies` | `id`, `name`, `sector`, `address`, `size_label`, `values text[]`, `logo_path` |
| `recruiters` | `id` → profiles, `company_id`, `full_name`, `job_title` |
| `capsules` | `id`, `owner_kind` (`candidate` \| `company`), `owner_id`, `slot` (1–5 ou 1–4), `video_path`, `mime_type`, `duration_s`, `updated_at` ; unique (`owner_kind`,`owner_id`,`slot`) |
| `offers` | `id`, `company_id`, `title`, `domain`, `location`, `contract`, `remote`, `salary_min`, `salary_max`, `languages text[]`, `missions`, `status` (`draft` \| `published` \| `closed`), `published_at` |
| `applications` | `id`, `offer_id`, `candidate_id`, `status` (`sent` \| `seen` \| `shortlisted` \| `interview` \| `rejected`), `created_at`, `seen_at` ; unique (`offer_id`,`candidate_id`) |
| `notes` | `id`, `application_id`, `author_id`, `body`, `created_at` |
| `conversations` | `id`, `candidate_id`, `company_id`, `application_id` (nullable), `invited_offer_id` (nullable) |
| `messages` | `id`, `conversation_id`, `sender_id`, `body`, `created_at` |
| `interviews` | `id`, `application_id`, `mode` (`onsite` \| `video`), `location`, `proposed_slots tstzrange[]`, `chosen_slot tstzrange`, `status` (`proposed` \| `confirmed` \| `cancelled`) |

### Qui voit quoi (RLS Supabase, appliqué par la base)
- Candidat : lit/écrit son profil et ses capsules ; lit les offres publiées et les entreprises ; lit ses
  candidatures, conversations, messages, entretiens ; choisit un créneau.
- Recruteur : lit/écrit son entreprise, ses offres, capsules entreprise ; lit les candidatures à ses offres
  et les profils + capsules de ces candidats ; lit les profils + capsules des candidats `visible_in_talent_base` ;
  notes de son entreprise uniquement ; propose des entretiens.
- **Notes** : jamais lisibles par un candidat.
- Fichiers : buckets privés `capsules`, `cvs`, `photos`, `logos` ; chemin `{owner_id}/…` ; écriture par le
  propriétaire ; lecture via **URL signée 1 h** générée côté serveur après contrôle d'accès.

## Vidéo

- Tournage : `getUserMedia` (caméra avant, 720p visé) + `MediaRecorder`. Format préféré dans l'ordre :
  `video/mp4;codecs=avc1` → `video/mp4` → `video/webm;codecs=vp9` → `video/webm`. Débit ~1,5 Mbit/s
  (~11 Mo par minute). Arrêt automatique à 60 s.
- Envoi direct du navigateur vers Supabase Storage (`capsules/{owner_id}/{slot}-{horodatage}.{ext}`), puis
  enregistrement de la ligne `capsules` ; l'ancienne vidéo est supprimée.
- Lecture : `<video playsinline controls>` sur URL signée.
- **Premier chantier = test réel** : tourner sur Android, lire sur iPhone, et l'inverse. Échec → secours Mux.
- Erreurs prévues : caméra refusée (message + comment l'autoriser), navigateur sans `MediaRecorder`, envoi
  interrompu (réessayer, la prise reste en mémoire), fichier > 25 Mo (refusé).

## Données de démo

Script `app/scripts/seed.ts` (clé service, exécuté par l'équipe) : 3 entreprises fictives (Veltra Industries,
Celestra Pharma, Brightwave), 5–6 offres, 5 candidats (dont Jacques Moets), candidatures à divers statuts,
messages, 2 entretiens. Les vidéos de démo sont lues depuis un dossier local `demo-videos/` (non versionné) ;
à défaut, des vidéos de remplacement générées.

## Tests et vérification

- Vitest : choix du format vidéo, validation des créneaux, génération .ics, règles de statut de candidature.
- `scripts/check-rls.ts` : crée deux candidats et deux recruteurs de test et vérifie chaque règle « qui voit quoi »
  (notamment : un candidat ne lit aucune note ; une entreprise ne lit pas les candidatures d'une autre).
- À chaque tranche : build Next.js sans erreur, parcours vérifié dans le navigateur, capture d'écran.

## Ordre de construction

1. Socle : dépôt, Next.js, Tailwind (couleurs de la maquette), Supabase (client, migrations, RLS), comptes.
2. Tournage vidéo + test réel iPhone/Android (déploiement Vercel nécessaire : caméra = HTTPS).
3. Profil candidat + capsules candidat.
4. Entreprise : page, capsules, offres.
5. Explorer, offre, candidature.
6. Candidatures reçues, fiche candidat, notes, statuts.
7. Messages (temps réel) + entretiens + .ics.
8. Agenda + base de talents + invitation.
9. Données de démo, passe visuelle, déploiement final.

## Ce que l'équipe doit faire (l'assistant ne crée pas de comptes et ne saisit pas de clés)

1. Créer un projet Supabase, région **Central EU (Frankfurt)** ; copier l'URL du projet, la clé publique (anon)
   et la clé service dans `app/.env.local` (modèle : `app/.env.example`).
2. Créer un compte Vercel ; dans un terminal : `npx vercel login`.
3. Facultatif : un dépôt GitHub pour partager le code avec Luca.
