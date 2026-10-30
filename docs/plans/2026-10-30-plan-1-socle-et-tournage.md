# Procast — Plan 1 : socle et tournage vidéo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Un site Next.js en ligne où l'on crée un compte candidat ou recruteur, tourne ses capsules de 60 s dans le navigateur, et les revoit — sur une base Supabase complète (toutes les tables et règles d'accès du prototype).

**Architecture:** Next.js (App Router, TypeScript) dans `app/`, Supabase (Postgres + RLS + Storage, région Francfort) appelé avec la session de l'utilisateur — jamais avec la clé secrète dans le site. Les vidéos partent directement du navigateur vers un bucket privé ; la lecture passe par des URL signées d'une heure, que la base n'accorde qu'aux personnes autorisées (RLS sur `storage.objects`).

**Tech Stack:** Next.js (dernière version stable via `create-next-app@latest`), React, TypeScript, Tailwind CSS v4, `@supabase/supabase-js`, `@supabase/ssr`, Vitest, Vercel.

**Spec:** `docs/specs/2026-10-30-prototype-design.md` (à lire avant toute tâche).

## Global Constraints

- Dépôt git : `C:\dev\projects\Procast` (remote `https://github.com/Pompomjr/Procast`). Le site vit dans `app/`. Toutes les commandes npm se lancent depuis `C:\dev\projects\Procast\app`.
- Interface **en français uniquement**. Tutoiement côté candidat, vouvoiement côté entreprise.
- Couleurs (reprises de la maquette) : ground `#F4F7F3`, card `#FFFFFF`, ink `#0F2A20`, muted `#53655B`, line `#DDE5DF`, mint `#8FE3B4`, mint-strong `#1B7A4C`, mint-soft `#E3F6EB`, forest `#0D3B2C`, forest-2 `#14503D`, forest-text-2 `#B5CEC2`, video `#17251F`, rec `#D93A3A`. Police **Manrope**. Candidat = mint, entreprise = forest.
- Capsules de **60 s maximum**, arrêt automatique. Candidat : 5 capsules ; entreprise : 4.
- Variables d'environnement : `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` (site) ; `SUPABASE_SECRET_KEY` (scripts uniquement, **jamais** importée depuis `src/`).
- L'agent ne crée aucun compte et ne saisit aucune clé : Mattia remplit `app/.env.local` et lance lui-même `npx supabase login` / `npx vercel login`.
- Fichiers vidéo ≤ 25 Mo ; CV ≤ 5 Mo (PDF) ; photo ≤ 3 Mo.
- Messages de commit en français, terminés par `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Chaque tâche finit par `npm run test` et `npm run build` verts.

## Structure des fichiers (après ce plan)

```
app/
  .env.example
  package.json · vitest.config.ts · tsconfig.json · next.config.ts
  supabase/migrations/
    20261030000001_schema.sql        tables, types, déclencheur d'inscription
    20261030000002_rls.sql           fonctions d'accès + règles RLS
    20261030000003_storage.sql       buckets privés + règles d'accès aux fichiers
  scripts/check-rls.ts               vérifie « qui voit quoi » contre le vrai projet
  src/
    app/layout.tsx · globals.css · page.tsx (accueil)
    app/inscription/page.tsx · app/connexion/page.tsx · app/(auth)/actions.ts
    app/candidat/capsules/page.tsx                 liste + lecture
    app/candidat/capsules/[slot]/page.tsx          tournage
    app/candidat/capsules/actions.ts               saveCapsule
    app/entreprise/capsules/page.tsx · [slot]/page.tsx (mêmes composants, côté entreprise)
    components/Logo.tsx · components/CapsuleRecorder.tsx · components/CapsulePlayer.tsx
    lib/env.ts                                      lecture/validation des variables
    lib/supabase/client.ts · server.ts · session.ts
    lib/capsules/slots.ts (+ .test.ts)              définitions et consignes
    lib/video/mime.ts (+ .test.ts)                  choix du format d'enregistrement
    lib/video/useCapsuleRecorder.ts                 caméra + MediaRecorder + 60 s
    lib/storage/paths.ts (+ .test.ts)               chemins de fichiers
    middleware.ts  (ou proxy.ts si Next ≥ 16)       rafraîchit la session, protège les routes
```

---

### Task 1: Socle Next.js aux couleurs de Procast

**Files:**
- Create: `app/` (via create-next-app), `app/vitest.config.ts`, `app/src/components/Logo.tsx`, `app/.env.example`
- Modify: `app/src/app/globals.css`, `app/src/app/layout.tsx`, `app/src/app/page.tsx`, `app/package.json`
- Test: `app/src/lib/smoke.test.ts`

**Interfaces:**
- Produces: classes Tailwind `bg-ground text-ink bg-mint text-mint-strong bg-mint-soft bg-forest bg-forest-2 text-forest-text-2 border-line text-muted bg-video bg-rec` ; `<Logo variant="candidate" | "company" | "onForest" />` ; scripts npm `dev`, `build`, `test`.

- [ ] **Step 1: Créer l'application**

Run (depuis `C:\dev\projects\Procast`):
```bash
npx create-next-app@latest app --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --no-turbopack --yes
```
Expected: dossier `app/` créé, `npm run build` possible. Noter la version de Next installée (`npx next --version`) : si ≥ 16, le fichier de middleware s'appelle `src/proxy.ts` et exporte `proxy` au lieu de `middleware` (Task 3).

- [ ] **Step 2: Installer Vitest et les dépendances Supabase**

```bash
cd app
npm install @supabase/supabase-js @supabase/ssr
npm install -D vitest @vitejs/plugin-react jsdom tsx dotenv
```

`app/vitest.config.ts` :
```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],
  test: { environment: 'node', include: ['src/**/*.test.ts', 'src/**/*.test.tsx'] },
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
});
```
Dans `app/package.json`, scripts : `"test": "vitest run"`, `"test:watch": "vitest"`, `"check:rls": "tsx scripts/check-rls.ts"`.

- [ ] **Step 3: Test de fumée**

`app/src/lib/smoke.test.ts` :
```ts
import { describe, it, expect } from 'vitest';
describe('socle', () => { it('lance les tests', () => { expect(1 + 1).toBe(2); }); });
```
Run: `npm run test` → Expected: 1 passed.

- [ ] **Step 4: Couleurs et police**

`app/src/app/globals.css` (remplacer le contenu) :
```css
@import "tailwindcss";

@theme {
  --color-ground: #F4F7F3;
  --color-card: #FFFFFF;
  --color-ink: #0F2A20;
  --color-muted: #53655B;
  --color-line: #DDE5DF;
  --color-mint: #8FE3B4;
  --color-mint-strong: #1B7A4C;
  --color-mint-soft: #E3F6EB;
  --color-forest: #0D3B2C;
  --color-forest-2: #14503D;
  --color-forest-text-2: #B5CEC2;
  --color-video: #17251F;
  --color-rec: #D93A3A;
  --color-warn: #9A5B00;
  --color-warn-soft: #FDF1DC;
  --font-sans: var(--font-manrope), system-ui, sans-serif;
}

body { background: var(--color-ground); color: var(--color-ink); }
:focus-visible { outline: 3px solid var(--color-mint-strong); outline-offset: 2px; }
```

`app/src/app/layout.tsx` :
```tsx
import type { Metadata, Viewport } from 'next';
import { Manrope } from 'next/font/google';
import './globals.css';

const manrope = Manrope({ subsets: ['latin'], variable: '--font-manrope' });

export const metadata: Metadata = { title: 'Procast — Motion your future', description: 'Le recrutement par capsules vidéo, des deux côtés.' };
export const viewport: Viewport = { width: 'device-width', initialScale: 1, themeColor: '#F4F7F3' };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="fr" className={manrope.variable}>
      <body className="min-h-dvh font-sans antialiased">{children}</body>
    </html>
  );
}
```

- [ ] **Step 5: Logo**

`app/src/components/Logo.tsx` :
```tsx
type Variant = 'candidate' | 'company' | 'onForest';
const markColor: Record<Variant, string> = { candidate: '#2FBF71', company: '#0D3B2C', onForest: '#8FE3B4' };

export function Logo({ variant = 'candidate', size = 28 }: { variant?: Variant; size?: number }) {
  const text = variant === 'onForest' ? 'text-white' : 'text-ink';
  return (
    <span className="inline-flex items-center gap-2">
      <svg width={size} height={size} viewBox="0 0 40 40" fill="none" aria-hidden="true" style={{ color: markColor[variant] }}>
        <path d="M4 15V7a3 3 0 0 1 3-3h11" stroke="currentColor" strokeWidth="3.5" strokeLinecap="round" />
        <path d="M36 25v8a3 3 0 0 1-3 3H22" stroke="currentColor" strokeWidth="3.5" strokeLinecap="round" />
        <path d="M15 12.5v15l13-7.5z" fill="currentColor" stroke="currentColor" strokeWidth="2.5" strokeLinejoin="round" />
      </svg>
      <span className={`text-xl font-extrabold tracking-tight ${text}`}>ProCast</span>
    </span>
  );
}
```

- [ ] **Step 6: Page d'accueil (reprend l'écran Accueil de la maquette)**

`app/src/app/page.tsx` :
```tsx
import Link from 'next/link';
import { Logo } from '@/components/Logo';

export default function Home() {
  return (
    <main className="mx-auto flex min-h-dvh max-w-md flex-col px-5 pb-7 pt-6">
      <Logo />
      <section className="mt-10">
        <p className="text-[13px] font-bold uppercase tracking-[.08em] text-mint-strong">Motion your future</p>
        <h1 className="mt-2 text-3xl font-extrabold leading-tight tracking-tight">Présentez-vous en vidéo. Des deux côtés.</h1>
        <p className="mt-3 text-[15px] font-medium leading-relaxed text-muted">
          Des capsules guidées d'une minute : la personne derrière le CV, l'équipe derrière l'offre.
          Vous vous rencontrez déjà un peu avant l'entretien.
        </p>
      </section>
      <div className="mt-auto flex flex-col gap-3 pt-10">
        <Link href="/inscription?role=candidate" className="flex h-[52px] items-center justify-center rounded-full bg-mint font-bold text-ink">Je cherche un emploi</Link>
        <Link href="/inscription?role=recruiter" className="flex h-[52px] items-center justify-center rounded-full bg-forest font-bold text-white">Je recrute</Link>
        <p className="mt-1 text-center text-sm font-medium text-muted">Déjà inscrit ? <Link href="/connexion" className="font-bold text-mint-strong underline">Se connecter</Link></p>
      </div>
    </main>
  );
}
```

- [ ] **Step 7: Modèle d'environnement**

`app/.env.example` :
```
# Copier en .env.local et remplir depuis Supabase > Project Settings > API Keys
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_xxx
# Scripts uniquement (seed, check-rls). Ne jamais l'utiliser dans src/.
SUPABASE_SECRET_KEY=sb_secret_xxx
```

- [ ] **Step 8: Vérifier**

Run: `npm run test && npm run build` → Expected: tests verts, build OK. `npm run dev` puis ouvrir http://localhost:3000 : logo, titre, deux boutons verts.

- [ ] **Step 9: Commit**

```bash
git add app
git commit -m "Prototype : socle Next.js aux couleurs de Procast"
```

---

### Task 2: Base de données complète, règles d'accès et stockage

**Files:**
- Create: `app/supabase/migrations/20261030000001_schema.sql`, `app/supabase/migrations/20261030000002_rls.sql`, `app/supabase/migrations/20261030000003_storage.sql`, `app/scripts/check-rls.ts`

**Interfaces:**
- Produces: tables du spec ; à l'inscription, `auth.users.raw_user_meta_data` doit contenir `role` (`candidate`|`recruiter`) et, pour un recruteur, `full_name` et `company_name` — le déclencheur crée `profiles` + `candidates`, ou `profiles` + `companies` + `recruiters`. Fonctions SQL : `public.my_company_id() → uuid`, `public.can_view_candidate(uuid) → boolean`, `public.choose_interview_slot(interview uuid, slot_index int) → void`. Buckets privés `capsules`, `cvs`, `photos`, `logos`, chemin `{propriétaire}/…` où propriétaire = id du candidat ou id de l'entreprise.
- Écart volontaire au spec : les créneaux d'entretien sont en `jsonb` (`[{ "start": iso, "end": iso }]`) + `chosen_start`/`chosen_end`, plus simples à lire en JavaScript que des `tstzrange[]`.

- [ ] **Step 1: Schéma**

`app/supabase/migrations/20261030000001_schema.sql` :
```sql
create type public.user_role as enum ('candidate', 'recruiter');
create type public.capsule_owner as enum ('candidate', 'company');
create type public.offer_status as enum ('draft', 'published', 'closed');
create type public.application_status as enum ('sent', 'seen', 'shortlisted', 'interview', 'rejected');
create type public.interview_mode as enum ('onsite', 'video');
create type public.interview_status as enum ('proposed', 'confirmed', 'cancelled');

create table public.profiles (
  id uuid primary key references auth.users on delete cascade,
  role public.user_role not null,
  created_at timestamptz not null default now()
);

create table public.companies (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  sector text,
  address text,
  size_label text,
  "values" text[] not null default '{}',
  logo_path text,
  created_at timestamptz not null default now()
);

create table public.recruiters (
  id uuid primary key references public.profiles on delete cascade,
  company_id uuid not null references public.companies on delete cascade,
  full_name text not null,
  job_title text
);

create table public.candidates (
  id uuid primary key references public.profiles on delete cascade,
  first_name text not null default '',
  last_name text not null default '',
  birth_date date,
  city text,
  domain text,
  languages text[] not null default '{}',
  linkedin_url text,
  cv_path text,
  photo_path text,
  available boolean not null default true,
  visible_in_talent_base boolean not null default false
);

create table public.capsules (
  id uuid primary key default gen_random_uuid(),
  owner_kind public.capsule_owner not null,
  owner_id uuid not null,
  slot smallint not null,
  video_path text not null,
  mime_type text not null,
  duration_s numeric(5,2) not null check (duration_s > 0 and duration_s <= 61),
  updated_at timestamptz not null default now(),
  unique (owner_kind, owner_id, slot),
  check ((owner_kind = 'candidate' and slot between 1 and 5) or (owner_kind = 'company' and slot between 1 and 4))
);

create table public.offers (
  id uuid primary key default gen_random_uuid(),
  company_id uuid not null references public.companies on delete cascade,
  title text not null,
  domain text not null,
  location text not null,
  contract text not null,
  remote text,
  salary_min integer,
  salary_max integer,
  languages text[] not null default '{}',
  missions text not null default '',
  status public.offer_status not null default 'draft',
  published_at timestamptz,
  created_at timestamptz not null default now()
);

create table public.applications (
  id uuid primary key default gen_random_uuid(),
  offer_id uuid not null references public.offers on delete cascade,
  candidate_id uuid not null references public.candidates on delete cascade,
  status public.application_status not null default 'sent',
  created_at timestamptz not null default now(),
  seen_at timestamptz,
  unique (offer_id, candidate_id)
);

create table public.notes (
  id uuid primary key default gen_random_uuid(),
  application_id uuid not null references public.applications on delete cascade,
  author_id uuid not null references public.recruiters on delete cascade,
  body text not null check (length(body) between 1 and 2000),
  created_at timestamptz not null default now()
);

create table public.conversations (
  id uuid primary key default gen_random_uuid(),
  candidate_id uuid not null references public.candidates on delete cascade,
  company_id uuid not null references public.companies on delete cascade,
  application_id uuid references public.applications on delete cascade,
  invited_offer_id uuid references public.offers on delete set null,
  created_at timestamptz not null default now()
);

create table public.messages (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references public.conversations on delete cascade,
  sender_id uuid not null references public.profiles on delete cascade,
  body text not null check (length(body) between 1 and 4000),
  created_at timestamptz not null default now()
);

create table public.interviews (
  id uuid primary key default gen_random_uuid(),
  application_id uuid not null references public.applications on delete cascade,
  mode public.interview_mode not null,
  location text,
  proposed_slots jsonb not null check (jsonb_typeof(proposed_slots) = 'array' and jsonb_array_length(proposed_slots) between 1 and 3),
  chosen_start timestamptz,
  chosen_end timestamptz,
  status public.interview_status not null default 'proposed',
  created_at timestamptz not null default now()
);

create index on public.offers (company_id);
create index on public.applications (offer_id);
create index on public.applications (candidate_id);
create index on public.messages (conversation_id, created_at);
create index on public.interviews (application_id);

-- Inscription : crée le profil et la fiche selon le rôle choisi.
create function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = public as $$
declare
  r public.user_role := (new.raw_user_meta_data ->> 'role')::public.user_role;
  new_company uuid;
begin
  insert into public.profiles (id, role) values (new.id, r);
  if r = 'candidate' then
    insert into public.candidates (id, first_name, last_name)
    values (new.id, coalesce(new.raw_user_meta_data ->> 'first_name', ''), coalesce(new.raw_user_meta_data ->> 'last_name', ''));
  else
    insert into public.companies (name) values (coalesce(new.raw_user_meta_data ->> 'company_name', 'Mon entreprise'))
    returning id into new_company;
    insert into public.recruiters (id, company_id, full_name)
    values (new.id, new_company, coalesce(new.raw_user_meta_data ->> 'full_name', ''));
  end if;
  return new;
end $$;

create trigger on_auth_user_created after insert on auth.users
for each row execute function public.handle_new_user();

-- Une conversation est ouverte automatiquement pour chaque candidature.
create function public.open_conversation_for_application() returns trigger
language plpgsql security definer set search_path = public as $$
begin
  insert into public.conversations (candidate_id, company_id, application_id)
  select new.candidate_id, o.company_id, new.id from public.offers o where o.id = new.offer_id;
  return new;
end $$;

create trigger on_application_created after insert on public.applications
for each row execute function public.open_conversation_for_application();
```

- [ ] **Step 2: Règles d'accès**

`app/supabase/migrations/20261030000002_rls.sql` :
```sql
create function public.my_company_id() returns uuid
language sql stable security definer set search_path = public as $$
  select company_id from public.recruiters where id = auth.uid()
$$;

create function public.can_view_candidate(cid uuid) returns boolean
language sql stable security definer set search_path = public as $$
  select cid = auth.uid() or (
    public.my_company_id() is not null and (
      exists (select 1 from public.candidates c where c.id = cid and c.visible_in_talent_base)
      or exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
                 where a.candidate_id = cid and o.company_id = public.my_company_id())
      or exists (select 1 from public.conversations v where v.candidate_id = cid and v.company_id = public.my_company_id())
    )
  )
$$;

create function public.candidate_has_all_capsules(cid uuid) returns boolean
language sql stable security definer set search_path = public as $$
  select count(*) = 5 from public.capsules where owner_kind = 'candidate' and owner_id = cid
$$;

alter table public.profiles enable row level security;
alter table public.companies enable row level security;
alter table public.recruiters enable row level security;
alter table public.candidates enable row level security;
alter table public.capsules enable row level security;
alter table public.offers enable row level security;
alter table public.applications enable row level security;
alter table public.notes enable row level security;
alter table public.conversations enable row level security;
alter table public.messages enable row level security;
alter table public.interviews enable row level security;

create policy "profil : soi-même" on public.profiles for select to authenticated using (id = auth.uid());

create policy "entreprises : lisibles par tous les connectés" on public.companies for select to authenticated using (true);
create policy "entreprises : modifiées par leurs recruteurs" on public.companies for update to authenticated
  using (id = public.my_company_id()) with check (id = public.my_company_id());

create policy "recruteurs : lisibles par les connectés" on public.recruiters for select to authenticated using (true);
create policy "recruteurs : soi-même" on public.recruiters for update to authenticated using (id = auth.uid()) with check (id = auth.uid());

create policy "candidats : lecture autorisée" on public.candidates for select to authenticated using (public.can_view_candidate(id));
create policy "candidats : soi-même" on public.candidates for update to authenticated using (id = auth.uid()) with check (id = auth.uid());

create policy "capsules : lecture" on public.capsules for select to authenticated using (
  (owner_kind = 'company') or (owner_kind = 'candidate' and public.can_view_candidate(owner_id)));
create policy "capsules : écriture par le propriétaire" on public.capsules for all to authenticated
  using ((owner_kind = 'candidate' and owner_id = auth.uid()) or (owner_kind = 'company' and owner_id = public.my_company_id()))
  with check ((owner_kind = 'candidate' and owner_id = auth.uid()) or (owner_kind = 'company' and owner_id = public.my_company_id()));

create policy "offres : publiées ou les miennes" on public.offers for select to authenticated
  using (status = 'published' or company_id = public.my_company_id());
create policy "offres : écriture par l'entreprise" on public.offers for all to authenticated
  using (company_id = public.my_company_id()) with check (company_id = public.my_company_id());

create policy "candidatures : les deux parties" on public.applications for select to authenticated using (
  candidate_id = auth.uid()
  or exists (select 1 from public.offers o where o.id = offer_id and o.company_id = public.my_company_id()));
create policy "candidatures : le candidat postule" on public.applications for insert to authenticated with check (
  candidate_id = auth.uid() and status = 'sent'
  and exists (select 1 from public.offers o where o.id = offer_id and o.status = 'published')
  and public.candidate_has_all_capsules(auth.uid()));
create policy "candidatures : l'entreprise change le statut" on public.applications for update to authenticated
  using (exists (select 1 from public.offers o where o.id = offer_id and o.company_id = public.my_company_id()))
  with check (exists (select 1 from public.offers o where o.id = offer_id and o.company_id = public.my_company_id()));

create policy "notes : l'entreprise seulement" on public.notes for select to authenticated using (
  exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
          where a.id = application_id and o.company_id = public.my_company_id()));
create policy "notes : écrites par un recruteur de l'entreprise" on public.notes for insert to authenticated with check (
  author_id = auth.uid() and exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
          where a.id = application_id and o.company_id = public.my_company_id()));

create policy "conversations : les deux parties" on public.conversations for select to authenticated
  using (candidate_id = auth.uid() or company_id = public.my_company_id());
create policy "conversations : invitation par l'entreprise" on public.conversations for insert to authenticated
  with check (company_id = public.my_company_id() and application_id is null and invited_offer_id is not null);

create policy "messages : les deux parties" on public.messages for select to authenticated using (
  exists (select 1 from public.conversations v where v.id = conversation_id
          and (v.candidate_id = auth.uid() or v.company_id = public.my_company_id())));
create policy "messages : envoyer" on public.messages for insert to authenticated with check (
  sender_id = auth.uid() and exists (select 1 from public.conversations v where v.id = conversation_id
          and (v.candidate_id = auth.uid() or v.company_id = public.my_company_id())));

create policy "entretiens : les deux parties" on public.interviews for select to authenticated using (
  exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
          where a.id = application_id and (a.candidate_id = auth.uid() or o.company_id = public.my_company_id())));
create policy "entretiens : l'entreprise propose" on public.interviews for all to authenticated
  using (exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
          where a.id = application_id and o.company_id = public.my_company_id()))
  with check (exists (select 1 from public.applications a join public.offers o on o.id = a.offer_id
          where a.id = application_id and o.company_id = public.my_company_id()));

-- Le candidat choisit un créneau : seule écriture qu'il fait sur un entretien.
create function public.choose_interview_slot(interview uuid, slot_index int) returns void
language plpgsql security definer set search_path = public as $$
declare i public.interviews; s jsonb;
begin
  select * into i from public.interviews where id = interview;
  if i.id is null or not exists (select 1 from public.applications a where a.id = i.application_id and a.candidate_id = auth.uid()) then
    raise exception 'entretien introuvable';
  end if;
  if i.status <> 'proposed' then raise exception 'entretien déjà confirmé ou annulé'; end if;
  s := i.proposed_slots -> slot_index;
  if s is null then raise exception 'créneau inconnu'; end if;
  update public.interviews set chosen_start = (s ->> 'start')::timestamptz, chosen_end = (s ->> 'end')::timestamptz, status = 'confirmed'
  where id = interview;
end $$;

-- Realtime pour la messagerie (plan 3).
alter publication supabase_realtime add table public.messages;
```

- [ ] **Step 3: Stockage**

`app/supabase/migrations/20261030000003_storage.sql` :
```sql
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types) values
  ('capsules', 'capsules', false, 26214400, array['video/mp4', 'video/webm', 'video/quicktime']),
  ('cvs', 'cvs', false, 5242880, array['application/pdf']),
  ('photos', 'photos', false, 3145728, array['image/jpeg', 'image/png', 'image/webp']),
  ('logos', 'logos', false, 3145728, array['image/jpeg', 'image/png', 'image/webp', 'image/svg+xml']);

-- Dossier de premier niveau = id du candidat (auth.uid) ou id de l'entreprise.
create function public.storage_owner_is_me(object_name text) returns boolean
language sql stable security definer set search_path = public as $$
  select (storage.foldername(object_name))[1] in (auth.uid()::text, coalesce(public.my_company_id()::text, ''))
$$;

create function public.storage_can_read(bucket text, object_name text) returns boolean
language sql stable security definer set search_path = public as $$
  select public.storage_owner_is_me(object_name)
    or (bucket in ('capsules', 'logos') and exists (select 1 from public.companies c where c.id::text = (storage.foldername(object_name))[1]))
    or (bucket in ('capsules', 'cvs', 'photos') and exists (
          select 1 from public.candidates c where c.id::text = (storage.foldername(object_name))[1] and public.can_view_candidate(c.id)))
$$;

create policy "fichiers : lecture autorisée" on storage.objects for select to authenticated
  using (bucket_id in ('capsules', 'cvs', 'photos', 'logos') and public.storage_can_read(bucket_id, name));
create policy "fichiers : dépôt par le propriétaire" on storage.objects for insert to authenticated
  with check (bucket_id in ('capsules', 'cvs', 'photos', 'logos') and public.storage_owner_is_me(name));
create policy "fichiers : remplacement par le propriétaire" on storage.objects for update to authenticated
  using (bucket_id in ('capsules', 'cvs', 'photos', 'logos') and public.storage_owner_is_me(name));
create policy "fichiers : suppression par le propriétaire" on storage.objects for delete to authenticated
  using (bucket_id in ('capsules', 'cvs', 'photos', 'logos') and public.storage_owner_is_me(name));
```

- [ ] **Step 4: Appliquer les migrations (fait par Mattia, une fois)**

Mattia, dans un terminal depuis `C:\dev\projects\Procast\app` :
```bash
npx supabase login
npx supabase link --project-ref <ref-du-projet>
npx supabase db push
```
Expected : les 3 migrations appliquées sans erreur. (L'agent prépare les fichiers et attend ; il ne saisit ni jeton ni mot de passe.)

- [ ] **Step 5: Script de vérification des accès**

`app/scripts/check-rls.ts` — crée des comptes de test avec la clé secrète, puis agit **avec leurs propres sessions** :
```ts
import 'dotenv/config';
import { config } from 'dotenv';
import { createClient, type SupabaseClient } from '@supabase/supabase-js';

config({ path: '.env.local' });
const url = process.env.NEXT_PUBLIC_SUPABASE_URL!;
const publishable = process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!;
const admin = createClient(url, process.env.SUPABASE_SECRET_KEY!, { auth: { persistSession: false } });
const stamp = Date.now();
const password = `Rls-${stamp}-ok!`;
let failures = 0;

function check(label: string, ok: boolean) {
  console.log(`${ok ? 'ok  ' : 'ÉCHEC'} ${label}`);
  if (!ok) failures++;
}

async function user(email: string, meta: Record<string, string>): Promise<{ id: string; db: SupabaseClient }> {
  const { data, error } = await admin.auth.admin.createUser({ email, password, email_confirm: true, user_metadata: meta });
  if (error) throw error;
  const db = createClient(url, publishable, { auth: { persistSession: false } });
  const { error: e2 } = await db.auth.signInWithPassword({ email, password });
  if (e2) throw e2;
  return { id: data.user.id, db };
}

async function main() {
  const c1 = await user(`rls-c1-${stamp}@example.test`, { role: 'candidate', first_name: 'Test', last_name: 'Un' });
  const c2 = await user(`rls-c2-${stamp}@example.test`, { role: 'candidate', first_name: 'Test', last_name: 'Deux' });
  const r1 = await user(`rls-r1-${stamp}@example.test`, { role: 'recruiter', full_name: 'Recruteur Un', company_name: 'Société Un' });
  const r2 = await user(`rls-r2-${stamp}@example.test`, { role: 'recruiter', full_name: 'Recruteur Deux', company_name: 'Société Deux' });

  const { data: co1 } = await r1.db.from('recruiters').select('company_id').eq('id', r1.id).single();
  check('le recruteur 1 a une entreprise', !!co1?.company_id);

  const { data: c2seenByC1 } = await c1.db.from('candidates').select('id').eq('id', c2.id);
  check('un candidat ne voit pas un autre candidat', (c2seenByC1 ?? []).length === 0);

  const { data: c1seenByR1 } = await r1.db.from('candidates').select('id').eq('id', c1.id);
  check('un recruteur ne voit pas un candidat invisible qui n\'a pas postulé', (c1seenByR1 ?? []).length === 0);

  await c1.db.from('candidates').update({ visible_in_talent_base: true }).eq('id', c1.id);
  const { data: c1visible } = await r1.db.from('candidates').select('id').eq('id', c1.id);
  check('un recruteur voit un candidat visible dans la base de talents', (c1visible ?? []).length === 1);

  const { error: capsuleForOther } = await c1.db.from('capsules').insert({ owner_kind: 'candidate', owner_id: c2.id, slot: 1, video_path: 'x', mime_type: 'video/mp4', duration_s: 10 });
  check('un candidat ne peut pas écrire la capsule d\'un autre', !!capsuleForOther);

  const { data: offer } = await r1.db.from('offers').insert({ company_id: co1!.company_id, title: 'Offre test', domain: 'Ingénierie', location: 'Bruxelles', contract: 'CDI', status: 'published', published_at: new Date().toISOString() }).select('id').single();
  check('le recruteur 1 publie une offre', !!offer?.id);

  const { error: applyWithoutCapsules } = await c2.db.from('applications').insert({ offer_id: offer!.id, candidate_id: c2.id });
  check('impossible de postuler sans les 5 capsules', !!applyWithoutCapsules);

  for (let slot = 1; slot <= 5; slot++) {
    await c2.db.from('capsules').insert({ owner_kind: 'candidate', owner_id: c2.id, slot, video_path: `${c2.id}/${slot}.mp4`, mime_type: 'video/mp4', duration_s: 30 });
  }
  const { data: app, error: applyErr } = await c2.db.from('applications').insert({ offer_id: offer!.id, candidate_id: c2.id }).select('id').single();
  check('postuler avec les 5 capsules', !applyErr && !!app?.id);

  const { error: noteErr } = await r1.db.from('notes').insert({ application_id: app!.id, author_id: r1.id, body: 'Note privée' });
  check('le recruteur 1 écrit une note', !noteErr);

  const { data: notesSeenByCandidate } = await c2.db.from('notes').select('id');
  check('le candidat ne lit aucune note', (notesSeenByCandidate ?? []).length === 0);

  const { data: appsSeenByR2 } = await r2.db.from('applications').select('id').eq('id', app!.id);
  check('une autre entreprise ne voit pas la candidature', (appsSeenByR2 ?? []).length === 0);

  const { data: notesSeenByR2 } = await r2.db.from('notes').select('id');
  check('une autre entreprise ne lit pas les notes', (notesSeenByR2 ?? []).length === 0);

  const { data: conv } = await c2.db.from('conversations').select('id').eq('application_id', app!.id).single();
  check('une conversation est ouverte à la candidature', !!conv?.id);

  const { error: statusByCandidate } = await c2.db.from('applications').update({ status: 'shortlisted' }).eq('id', app!.id).select('id').single();
  check('le candidat ne change pas lui-même son statut', !!statusByCandidate);

  for (const u of [c1, c2, r1, r2]) await admin.auth.admin.deleteUser(u.id);
  await admin.from('companies').delete().in('name', ['Société Un', 'Société Deux']);

  console.log(failures === 0 ? '\nTOUT EST VERT' : `\n${failures} règle(s) en échec`);
  process.exit(failures === 0 ? 0 : 1);
}

main().catch((e) => { console.error(e); process.exit(1); });
```

- [ ] **Step 6: Lancer la vérification**

Run (après Step 4 et `.env.local` rempli) : `npm run check:rls`
Expected : toutes les lignes `ok`, puis `TOUT EST VERT`. Toute ligne `ÉCHEC` = corriger la migration RLS (nouvelle migration, ne pas modifier celle déjà appliquée), `npx supabase db push`, relancer.

- [ ] **Step 7: Commit**

```bash
git add app/supabase app/scripts app/package.json
git commit -m "Prototype : base de données, règles d'accès et stockage"
```

---

### Task 3: Connexion à Supabase et comptes

**Files:**
- Create: `app/src/lib/env.ts`, `app/src/lib/env.test.ts`, `app/src/lib/supabase/client.ts`, `app/src/lib/supabase/server.ts`, `app/src/lib/supabase/session.ts`, `app/src/middleware.ts` (ou `app/src/proxy.ts`), `app/src/app/(auth)/actions.ts`, `app/src/app/inscription/page.tsx`, `app/src/app/connexion/page.tsx`, `app/src/app/candidat/page.tsx`, `app/src/app/entreprise/page.tsx`

**Interfaces:**
- Consumes: déclencheur `handle_new_user` (Task 2) et ses clés de métadonnées.
- Produces: `readPublicEnv(): { url: string; publishableKey: string }` ; `createBrowserSupabase()` ; `createServerSupabase(): Promise<SupabaseClient>` ; `getCurrentUser(): Promise<{ id: string; role: 'candidate' | 'recruiter' } | null>` ; actions `signUp(formData)`, `signIn(formData)`, `signOut()` ; routes `/candidat` et `/entreprise` protégées.

- [ ] **Step 1: Test de lecture de l'environnement**

`app/src/lib/env.test.ts` :
```ts
import { describe, it, expect } from 'vitest';
import { parsePublicEnv } from './env';

describe('parsePublicEnv', () => {
  it('renvoie l\'URL et la clé publique', () => {
    expect(parsePublicEnv({ NEXT_PUBLIC_SUPABASE_URL: 'https://a.supabase.co', NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY: 'sb_publishable_x' }))
      .toEqual({ url: 'https://a.supabase.co', publishableKey: 'sb_publishable_x' });
  });
  it('explique quoi faire si une variable manque', () => {
    expect(() => parsePublicEnv({ NEXT_PUBLIC_SUPABASE_URL: 'https://a.supabase.co' })).toThrow(/NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY.*\.env\.local/);
  });
});
```
Run: `npm run test` → Expected: FAIL (`./env` introuvable).

- [ ] **Step 2: Implémenter**

`app/src/lib/env.ts` :
```ts
type Raw = Record<string, string | undefined>;

export function parsePublicEnv(raw: Raw) {
  const url = raw.NEXT_PUBLIC_SUPABASE_URL;
  const publishableKey = raw.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY;
  for (const [name, value] of [['NEXT_PUBLIC_SUPABASE_URL', url], ['NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY', publishableKey]] as const) {
    if (!value) throw new Error(`${name} manquante : copier app/.env.example en app/.env.local et la remplir.`);
  }
  return { url: url!, publishableKey: publishableKey! };
}

export function readPublicEnv() {
  return parsePublicEnv({
    NEXT_PUBLIC_SUPABASE_URL: process.env.NEXT_PUBLIC_SUPABASE_URL,
    NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY,
  });
}
```
Run: `npm run test` → Expected: PASS.

- [ ] **Step 3: Clients Supabase**

`app/src/lib/supabase/client.ts` :
```ts
'use client';
import { createBrowserClient } from '@supabase/ssr';
import { readPublicEnv } from '@/lib/env';

export function createBrowserSupabase() {
  const { url, publishableKey } = readPublicEnv();
  return createBrowserClient(url, publishableKey);
}
```

`app/src/lib/supabase/server.ts` :
```ts
import 'server-only';
import { cookies } from 'next/headers';
import { createServerClient } from '@supabase/ssr';
import { readPublicEnv } from '@/lib/env';

export async function createServerSupabase() {
  const { url, publishableKey } = readPublicEnv();
  const store = await cookies();
  return createServerClient(url, publishableKey, {
    cookies: {
      getAll: () => store.getAll(),
      setAll: (list) => {
        try { list.forEach(({ name, value, options }) => store.set(name, value, options)); } catch { /* appelé depuis un composant serveur : le middleware s'en charge */ }
      },
    },
  });
}

export async function getCurrentUser(): Promise<{ id: string; role: 'candidate' | 'recruiter' } | null> {
  const supabase = await createServerSupabase();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return null;
  const { data } = await supabase.from('profiles').select('role').eq('id', user.id).single();
  return data ? { id: user.id, role: data.role } : null;
}
```
(`npm install server-only` si absent.)

`app/src/lib/supabase/session.ts` :
```ts
import { NextResponse, type NextRequest } from 'next/server';
import { createServerClient } from '@supabase/ssr';
import { readPublicEnv } from '@/lib/env';

const PROTECTED = ['/candidat', '/entreprise'];

export async function refreshSession(request: NextRequest) {
  let response = NextResponse.next({ request });
  const { url, publishableKey } = readPublicEnv();
  const supabase = createServerClient(url, publishableKey, {
    cookies: {
      getAll: () => request.cookies.getAll(),
      setAll: (list) => {
        list.forEach(({ name, value }) => request.cookies.set(name, value));
        response = NextResponse.next({ request });
        list.forEach(({ name, value, options }) => response.cookies.set(name, value, options));
      },
    },
  });
  const { data: { user } } = await supabase.auth.getUser();
  if (!user && PROTECTED.some((p) => request.nextUrl.pathname.startsWith(p))) {
    const login = request.nextUrl.clone();
    login.pathname = '/connexion';
    return NextResponse.redirect(login);
  }
  return response;
}
```

`app/src/middleware.ts` (Next ≤ 15 ; pour Next ≥ 16 : fichier `src/proxy.ts`, fonction `proxy`) :
```ts
import type { NextRequest } from 'next/server';
import { refreshSession } from '@/lib/supabase/session';

export function middleware(request: NextRequest) { return refreshSession(request); }

export const config = { matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|webp)$).*)'] };
```

- [ ] **Step 4: Actions d'inscription et de connexion**

`app/src/app/(auth)/actions.ts` :
```ts
'use server';
import { redirect } from 'next/navigation';
import { createServerSupabase } from '@/lib/supabase/server';

export type AuthState = { error?: string };

const home = (role: string) => (role === 'recruiter' ? '/entreprise' : '/candidat');

export async function signUp(_: AuthState, form: FormData): Promise<AuthState> {
  const role = form.get('role') === 'recruiter' ? 'recruiter' : 'candidate';
  const email = String(form.get('email') ?? '').trim();
  const password = String(form.get('password') ?? '');
  if (password.length < 8) return { error: 'Le mot de passe doit faire au moins 8 caractères.' };
  const data = role === 'candidate'
    ? { role, first_name: String(form.get('first_name') ?? ''), last_name: String(form.get('last_name') ?? '') }
    : { role, full_name: String(form.get('full_name') ?? ''), company_name: String(form.get('company_name') ?? '') };
  const supabase = await createServerSupabase();
  const { error } = await supabase.auth.signUp({ email, password, options: { data } });
  if (error) return { error: error.message === 'User already registered' ? 'Un compte existe déjà avec cet e-mail.' : 'Inscription impossible. Réessaie dans un instant.' };
  redirect(home(role));
}

export async function signIn(_: AuthState, form: FormData): Promise<AuthState> {
  const supabase = await createServerSupabase();
  const { data, error } = await supabase.auth.signInWithPassword({ email: String(form.get('email') ?? ''), password: String(form.get('password') ?? '') });
  if (error || !data.user) return { error: 'E-mail ou mot de passe incorrect.' };
  const { data: profile } = await supabase.from('profiles').select('role').eq('id', data.user.id).single();
  redirect(home(profile?.role ?? 'candidate'));
}

export async function signOut() {
  const supabase = await createServerSupabase();
  await supabase.auth.signOut();
  redirect('/');
}
```
Note pour Mattia : dans Supabase > Authentication > Sign In / Providers > Email, **désactiver « Confirm email »** pour la démo (sinon chaque compte attend un e-mail).

- [ ] **Step 5: Pages inscription et connexion**

`app/src/app/inscription/page.tsx` :
```tsx
'use client';
import { useActionState } from 'react';
import { useSearchParams } from 'next/navigation';
import Link from 'next/link';
import { Logo } from '@/components/Logo';
import { signUp, type AuthState } from '../(auth)/actions';

const field = 'h-12 w-full rounded-xl border border-line bg-card px-4 text-[15px] font-medium';

export default function Inscription() {
  const role = useSearchParams().get('role') === 'recruiter' ? 'recruiter' : 'candidate';
  const [state, action, pending] = useActionState<AuthState, FormData>(signUp, {});
  const candidate = role === 'candidate';
  return (
    <main className="mx-auto max-w-md px-5 py-6">
      <Logo variant={candidate ? 'candidate' : 'company'} />
      <h1 className="mt-8 text-[26px] font-extrabold">{candidate ? 'Crée ton compte' : 'Créez le compte de votre entreprise'}</h1>
      <form action={action} className="mt-6 flex flex-col gap-4">
        <input type="hidden" name="role" value={role} />
        {candidate ? (
          <div className="grid grid-cols-2 gap-3">
            <label className="text-sm font-semibold">Prénom<input name="first_name" required className={field} autoComplete="given-name" /></label>
            <label className="text-sm font-semibold">Nom<input name="last_name" required className={field} autoComplete="family-name" /></label>
          </div>
        ) : (
          <>
            <label className="text-sm font-semibold">Votre nom<input name="full_name" required className={field} autoComplete="name" /></label>
            <label className="text-sm font-semibold">Entreprise<input name="company_name" required className={field} autoComplete="organization" /></label>
          </>
        )}
        <label className="text-sm font-semibold">E-mail<input name="email" type="email" required className={field} autoComplete="email" /></label>
        <label className="text-sm font-semibold">Mot de passe (8 caractères minimum)<input name="password" type="password" minLength={8} required className={field} autoComplete="new-password" /></label>
        {state.error && <p role="alert" className="rounded-xl bg-warn-soft px-4 py-3 text-sm font-semibold text-warn">{state.error}</p>}
        <button disabled={pending} className={`h-[52px] rounded-full font-bold ${candidate ? 'bg-mint text-ink' : 'bg-forest text-white'} disabled:opacity-60`}>
          {pending ? 'Création…' : 'Créer mon compte'}
        </button>
        <p className="text-center text-sm text-muted">Déjà inscrit ? <Link href="/connexion" className="font-bold text-mint-strong underline">Se connecter</Link></p>
      </form>
    </main>
  );
}
```
Envelopper l'export dans `<Suspense>` si le build Next l'exige pour `useSearchParams` (message d'erreur explicite au build) : renommer le composant `InscriptionForm` et exporter `export default function Page() { return <Suspense><InscriptionForm /></Suspense>; }`.

`app/src/app/connexion/page.tsx` : même structure avec `signIn`, champs e-mail + mot de passe (`autoComplete="current-password"`), titre « Se connecter », lien « Pas encore de compte ? » → `/`.

- [ ] **Step 6: Accueils provisoires protégés**

`app/src/app/candidat/page.tsx` :
```tsx
import Link from 'next/link';
import { redirect } from 'next/navigation';
import { getCurrentUser } from '@/lib/supabase/server';
import { signOut } from '../(auth)/actions';
import { Logo } from '@/components/Logo';

export default async function CandidatHome() {
  const me = await getCurrentUser();
  if (!me) redirect('/connexion');
  if (me.role !== 'candidate') redirect('/entreprise');
  return (
    <main className="mx-auto max-w-md px-5 py-6">
      <Logo />
      <h1 className="mt-8 text-[26px] font-extrabold">Bienvenue</h1>
      <Link href="/candidat/capsules" className="mt-6 flex h-[52px] items-center justify-center rounded-full bg-mint font-bold">Mes capsules</Link>
      <form action={signOut} className="mt-4"><button className="h-11 w-full rounded-full border border-line font-semibold">Se déconnecter</button></form>
    </main>
  );
}
```
`app/src/app/entreprise/page.tsx` : identique avec `role !== 'recruiter'` → `/candidat`, logo `company`, lien « Capsules de l'entreprise » → `/entreprise/capsules`, bouton forest.

- [ ] **Step 7: Vérifier dans le navigateur**

Run: `npm run test && npm run build`, puis `npm run dev`. Créer un compte candidat → arrive sur `/candidat`. Se déconnecter, ouvrir `/candidat` → redirigé vers `/connexion`. Créer un compte recruteur → `/entreprise` ; dans Supabase, tables `companies` et `recruiters` remplies. Capture d'écran de `/candidat`.

- [ ] **Step 8: Commit**

```bash
git add app/src
git commit -m "Prototype : comptes candidat et recruteur"
```

---

### Task 4: Définition des capsules et choix du format vidéo

**Files:**
- Create: `app/src/lib/capsules/slots.ts`, `app/src/lib/capsules/slots.test.ts`, `app/src/lib/video/mime.ts`, `app/src/lib/video/mime.test.ts`, `app/src/lib/storage/paths.ts`, `app/src/lib/storage/paths.test.ts`

**Interfaces:**
- Produces:
  - `type OwnerKind = 'candidate' | 'company'`
  - `type CapsuleSlot = { slot: number; title: string; prompt: string; tips: string[] }`
  - `CAPSULE_SLOTS: Record<OwnerKind, CapsuleSlot[]>` ; `getSlot(kind: OwnerKind, slot: number): CapsuleSlot | null` ; `MAX_CAPSULE_SECONDS = 60`
  - `pickRecordingFormat(isSupported: (type: string) => boolean): { mimeType: string; extension: 'mp4' | 'webm' } | null`
  - `capsuleObjectPath(ownerId: string, slot: number, extension: string, now: number): string` ; `MAX_CAPSULE_BYTES = 25 * 1024 * 1024`

- [ ] **Step 1: Tests**

`app/src/lib/capsules/slots.test.ts` :
```ts
import { describe, it, expect } from 'vitest';
import { CAPSULE_SLOTS, getSlot, MAX_CAPSULE_SECONDS } from './slots';

describe('capsules', () => {
  it('5 capsules candidat et 4 entreprise, numérotées dans l\'ordre', () => {
    expect(CAPSULE_SLOTS.candidate.map((s) => s.slot)).toEqual([1, 2, 3, 4, 5]);
    expect(CAPSULE_SLOTS.company.map((s) => s.slot)).toEqual([1, 2, 3, 4]);
  });
  it('les titres suivent la conception', () => {
    expect(CAPSULE_SLOTS.candidate.map((s) => s.title)).toEqual(['Qui je suis', 'Mon diplôme', 'Mon expérience', 'Mes langues', 'Hors du travail']);
    expect(CAPSULE_SLOTS.company.map((s) => s.title)).toEqual(['Le lieu', 'Le poste', "L'équipe", 'Nos valeurs']);
  });
  it('chaque capsule a une consigne et au moins 2 pistes', () => {
    for (const s of [...CAPSULE_SLOTS.candidate, ...CAPSULE_SLOTS.company]) {
      expect(s.prompt.length).toBeGreaterThan(20);
      expect(s.tips.length).toBeGreaterThanOrEqual(2);
    }
  });
  it('getSlot refuse un numéro inexistant', () => {
    expect(getSlot('company', 5)).toBeNull();
    expect(getSlot('candidate', 4)?.title).toBe('Mes langues');
  });
  it('60 secondes maximum', () => { expect(MAX_CAPSULE_SECONDS).toBe(60); });
});
```

`app/src/lib/video/mime.test.ts` :
```ts
import { describe, it, expect } from 'vitest';
import { pickRecordingFormat } from './mime';

describe('pickRecordingFormat', () => {
  it('préfère le MP4 H.264 (lisible sur iPhone et Android)', () => {
    expect(pickRecordingFormat(() => true)).toEqual({ mimeType: 'video/mp4;codecs=avc1', extension: 'mp4' });
  });
  it('se rabat sur le MP4 générique', () => {
    expect(pickRecordingFormat((t) => t === 'video/mp4')).toEqual({ mimeType: 'video/mp4', extension: 'mp4' });
  });
  it('sinon WebM', () => {
    expect(pickRecordingFormat((t) => t.startsWith('video/webm'))).toEqual({ mimeType: 'video/webm;codecs=vp9', extension: 'webm' });
  });
  it('null si le navigateur ne sait rien enregistrer', () => {
    expect(pickRecordingFormat(() => false)).toBeNull();
  });
});
```

`app/src/lib/storage/paths.test.ts` :
```ts
import { describe, it, expect } from 'vitest';
import { capsuleObjectPath } from './paths';

describe('capsuleObjectPath', () => {
  it('range la vidéo dans le dossier du propriétaire', () => {
    expect(capsuleObjectPath('abc', 3, 'mp4', 1700000000000)).toBe('abc/3-1700000000000.mp4');
  });
  it('refuse un numéro de capsule hors limites', () => {
    expect(() => capsuleObjectPath('abc', 0, 'mp4', 1)).toThrow();
    expect(() => capsuleObjectPath('abc', 6, 'mp4', 1)).toThrow();
  });
});
```
Run: `npm run test` → Expected: FAIL (modules introuvables).

- [ ] **Step 2: Implémenter**

`app/src/lib/capsules/slots.ts` :
```ts
export type OwnerKind = 'candidate' | 'company';
export type CapsuleSlot = { slot: number; title: string; prompt: string; tips: string[] };

export const MAX_CAPSULE_SECONDS = 60;

export const CAPSULE_SLOTS: Record<OwnerKind, CapsuleSlot[]> = {
  candidate: [
    { slot: 1, title: 'Qui je suis', prompt: 'Présente-toi comme si tu rencontrais ta future équipe autour d\'un café.', tips: ['Ton prénom et ce que tu fais', 'Ce qui te motive dans ton métier', 'Le genre de poste que tu cherches'] },
    { slot: 2, title: 'Mon diplôme', prompt: 'Raconte ta formation et ce que tu en as vraiment retenu.', tips: ['Ton diplôme et ton école', 'Un cours ou un projet qui t\'a marqué', 'Ce que tu en utilises encore aujourd\'hui'] },
    { slot: 3, title: 'Mon expérience', prompt: 'Parle d\'une expérience dont tu es fier ou fière, avec un exemple concret.', tips: ['Le contexte en une phrase', 'Ce que tu as fait, toi', 'Le résultat'] },
    { slot: 4, title: 'Mes langues', prompt: 'Présente-toi dans chacune de tes langues de travail, environ 20 secondes par langue.', tips: ['Commence par ta langue maternelle', 'Un exemple où tu as utilisé chaque langue', 'Une langue que tu apprends'] },
    { slot: 5, title: 'Hors du travail', prompt: 'Montre qui tu es en dehors du travail : une passion, un engagement, un projet.', tips: ['Ce que tu fais de ton temps libre', 'Pourquoi ça compte pour toi', 'Ce que ça dit de toi au travail'] },
  ],
  company: [
    { slot: 1, title: 'Le lieu', prompt: 'Faites visiter l\'endroit où la personne travaillera au quotidien.', tips: ['L\'entrée et l\'accès', 'Le poste de travail', 'Les espaces communs'] },
    { slot: 2, title: 'Le poste', prompt: 'Expliquez le poste comme vous le feriez à un ami : ce qu\'on y fait vraiment.', tips: ['Une journée type', 'Les défis des premiers mois', 'Ce qui fait réussir dans ce rôle'] },
    { slot: 3, title: "L'équipe", prompt: 'Présentez l\'équipe que la personne rejoindra, idéalement avec elle.', tips: ['Qui fait quoi', 'Comment vous travaillez ensemble', 'Un rituel d\'équipe'] },
    { slot: 4, title: 'Nos valeurs', prompt: 'Racontez une valeur de l\'entreprise à travers un exemple réel.', tips: ['La valeur en un mot', 'Une situation où elle a compté', 'Ce que vous attendez du candidat'] },
  ],
};

export function getSlot(kind: OwnerKind, slot: number): CapsuleSlot | null {
  return CAPSULE_SLOTS[kind].find((s) => s.slot === slot) ?? null;
}
```

`app/src/lib/video/mime.ts` :
```ts
const PREFERENCES = [
  { mimeType: 'video/mp4;codecs=avc1', extension: 'mp4' },
  { mimeType: 'video/mp4', extension: 'mp4' },
  { mimeType: 'video/webm;codecs=vp9', extension: 'webm' },
  { mimeType: 'video/webm', extension: 'webm' },
] as const;

export function pickRecordingFormat(isSupported: (type: string) => boolean): { mimeType: string; extension: 'mp4' | 'webm' } | null {
  const found = PREFERENCES.find((p) => isSupported(p.mimeType));
  return found ? { mimeType: found.mimeType, extension: found.extension } : null;
}
```

`app/src/lib/storage/paths.ts` :
```ts
export const MAX_CAPSULE_BYTES = 25 * 1024 * 1024;

export function capsuleObjectPath(ownerId: string, slot: number, extension: string, now: number): string {
  if (!Number.isInteger(slot) || slot < 1 || slot > 5) throw new Error(`Numéro de capsule invalide : ${slot}`);
  return `${ownerId}/${slot}-${now}.${extension}`;
}
```
Run: `npm run test` → Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add app/src/lib
git commit -m "Prototype : consignes des capsules et choix du format vidéo"
```

---

### Task 5: Tournage d'une capsule dans le navigateur

**Files:**
- Create: `app/src/lib/video/useCapsuleRecorder.ts`, `app/src/components/CapsuleRecorder.tsx`, `app/src/app/candidat/capsules/actions.ts`, `app/src/app/candidat/capsules/[slot]/page.tsx`, `app/src/app/entreprise/capsules/[slot]/page.tsx`

**Interfaces:**
- Consumes: `getSlot`, `MAX_CAPSULE_SECONDS`, `pickRecordingFormat`, `capsuleObjectPath`, `MAX_CAPSULE_BYTES`, `createBrowserSupabase`, `createServerSupabase`, `getCurrentUser`.
- Produces:
  - `useCapsuleRecorder(maxSeconds: number)` → `{ state: 'idle' | 'requesting' | 'ready' | 'recording' | 'review' | 'error'; error: string | null; elapsed: number; stream: MediaStream | null; recording: { blob: Blob; url: string; mimeType: string; extension: 'mp4' | 'webm'; durationS: number } | null; start(): Promise<void>; record(): void; stop(): void; retake(): void; release(): void }`
  - `<CapsuleRecorder kind={OwnerKind} ownerId={string} slot={number} backHref={string} />`
  - server action `saveCapsule(input: { kind: OwnerKind; slot: number; path: string; mimeType: string; durationS: number }): Promise<{ error?: string }>`

- [ ] **Step 1: Le hook d'enregistrement**

`app/src/lib/video/useCapsuleRecorder.ts` :
```ts
'use client';
import { useCallback, useEffect, useRef, useState } from 'react';
import { pickRecordingFormat } from './mime';

type State = 'idle' | 'requesting' | 'ready' | 'recording' | 'review' | 'error';
type Recording = { blob: Blob; url: string; mimeType: string; extension: 'mp4' | 'webm'; durationS: number };

export function useCapsuleRecorder(maxSeconds: number) {
  const [state, setState] = useState<State>('idle');
  const [error, setError] = useState<string | null>(null);
  const [elapsed, setElapsed] = useState(0);
  const [stream, setStream] = useState<MediaStream | null>(null);
  const [recording, setRecording] = useState<Recording | null>(null);
  const recorderRef = useRef<MediaRecorder | null>(null);
  const chunksRef = useRef<Blob[]>([]);
  const startedAtRef = useRef(0);
  const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const clearTimer = () => { if (timerRef.current) clearInterval(timerRef.current); timerRef.current = null; };

  const release = useCallback(() => {
    clearTimer();
    stream?.getTracks().forEach((t) => t.stop());
    setStream(null);
  }, [stream]);

  const start = useCallback(async () => {
    if (typeof MediaRecorder === 'undefined' || !navigator.mediaDevices?.getUserMedia) {
      setError('Ce navigateur ne permet pas de filmer. Essaie avec Chrome ou Safari à jour.');
      setState('error');
      return;
    }
    setState('requesting');
    try {
      const s = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: 'user', width: { ideal: 1280 }, height: { ideal: 720 } },
        audio: { echoCancellation: true, noiseSuppression: true },
      });
      setStream(s);
      setState('ready');
    } catch (e) {
      const denied = e instanceof DOMException && (e.name === 'NotAllowedError' || e.name === 'SecurityError');
      setError(denied
        ? 'L\'accès à la caméra a été refusé. Autorise la caméra et le micro pour ce site dans les réglages du navigateur, puis réessaie.'
        : 'Impossible d\'ouvrir la caméra. Vérifie qu\'aucune autre application ne l\'utilise.');
      setState('error');
    }
  }, []);

  const stop = useCallback(() => {
    clearTimer();
    if (recorderRef.current?.state === 'recording') recorderRef.current.stop();
  }, []);

  const record = useCallback(() => {
    if (!stream) return;
    const format = pickRecordingFormat((t) => MediaRecorder.isTypeSupported(t));
    if (!format) { setError('Ce navigateur ne sait pas enregistrer de vidéo.'); setState('error'); return; }
    chunksRef.current = [];
    const rec = new MediaRecorder(stream, { mimeType: format.mimeType, videoBitsPerSecond: 1_500_000, audioBitsPerSecond: 96_000 });
    rec.ondataavailable = (ev) => { if (ev.data.size > 0) chunksRef.current.push(ev.data); };
    rec.onstop = () => {
      const durationS = Math.min(maxSeconds, (Date.now() - startedAtRef.current) / 1000);
      const blob = new Blob(chunksRef.current, { type: format.mimeType.split(';')[0] });
      setRecording({ blob, url: URL.createObjectURL(blob), mimeType: blob.type, extension: format.extension, durationS: Math.round(durationS * 100) / 100 });
      setState('review');
    };
    recorderRef.current = rec;
    startedAtRef.current = Date.now();
    setElapsed(0);
    rec.start(1000);
    setState('recording');
    timerRef.current = setInterval(() => {
      const s = (Date.now() - startedAtRef.current) / 1000;
      setElapsed(Math.min(maxSeconds, s));
      if (s >= maxSeconds) stop();
    }, 200);
  }, [stream, maxSeconds, stop]);

  const retake = useCallback(() => {
    if (recording) URL.revokeObjectURL(recording.url);
    setRecording(null);
    setElapsed(0);
    setState(stream ? 'ready' : 'idle');
  }, [recording, stream]);

  useEffect(() => () => { clearTimer(); stream?.getTracks().forEach((t) => t.stop()); }, [stream]);

  return { state, error, elapsed, stream, recording, start, record, stop, retake, release };
}
```

- [ ] **Step 2: Action d'enregistrement de la capsule**

`app/src/app/candidat/capsules/actions.ts` :
```ts
'use server';
import { revalidatePath } from 'next/cache';
import { createServerSupabase, getCurrentUser } from '@/lib/supabase/server';
import { getSlot, MAX_CAPSULE_SECONDS, type OwnerKind } from '@/lib/capsules/slots';

export async function saveCapsule(input: { kind: OwnerKind; slot: number; path: string; mimeType: string; durationS: number }): Promise<{ error?: string }> {
  const me = await getCurrentUser();
  if (!me) return { error: 'Session expirée, reconnecte-toi.' };
  if (!getSlot(input.kind, input.slot)) return { error: 'Capsule inconnue.' };
  if (input.durationS <= 0 || input.durationS > MAX_CAPSULE_SECONDS + 1) return { error: 'Durée invalide.' };
  const supabase = await createServerSupabase();
  const ownerId = input.kind === 'candidate'
    ? me.id
    : (await supabase.from('recruiters').select('company_id').eq('id', me.id).single()).data?.company_id;
  if (!ownerId || !input.path.startsWith(`${ownerId}/`)) return { error: 'Fichier non autorisé.' };

  const { data: previous } = await supabase.from('capsules').select('video_path')
    .eq('owner_kind', input.kind).eq('owner_id', ownerId).eq('slot', input.slot).maybeSingle();
  const { error } = await supabase.from('capsules').upsert(
    { owner_kind: input.kind, owner_id: ownerId, slot: input.slot, video_path: input.path, mime_type: input.mimeType, duration_s: input.durationS, updated_at: new Date().toISOString() },
    { onConflict: 'owner_kind,owner_id,slot' });
  if (error) return { error: 'Impossible d\'enregistrer la capsule. Réessaie.' };
  if (previous?.video_path && previous.video_path !== input.path) await supabase.storage.from('capsules').remove([previous.video_path]);
  revalidatePath(input.kind === 'candidate' ? '/candidat/capsules' : '/entreprise/capsules');
  return {};
}
```

- [ ] **Step 3: Le composant de tournage**

`app/src/components/CapsuleRecorder.tsx` :
```tsx
'use client';
import { useEffect, useRef, useState } from 'react';
import Link from 'next/link';
import { useRouter } from 'next/navigation';
import { useCapsuleRecorder } from '@/lib/video/useCapsuleRecorder';
import { getSlot, MAX_CAPSULE_SECONDS, CAPSULE_SLOTS, type OwnerKind } from '@/lib/capsules/slots';
import { capsuleObjectPath, MAX_CAPSULE_BYTES } from '@/lib/storage/paths';
import { createBrowserSupabase } from '@/lib/supabase/client';
import { saveCapsule } from '@/app/candidat/capsules/actions';

export function CapsuleRecorder({ kind, ownerId, slot, backHref }: { kind: OwnerKind; ownerId: string; slot: number; backHref: string }) {
  const def = getSlot(kind, slot)!;
  const total = CAPSULE_SLOTS[kind].length;
  const r = useCapsuleRecorder(MAX_CAPSULE_SECONDS);
  const live = useRef<HTMLVideoElement>(null);
  const router = useRouter();
  const [saving, setSaving] = useState(false);
  const [saveError, setSaveError] = useState<string | null>(null);
  const remaining = Math.max(0, Math.ceil(MAX_CAPSULE_SECONDS - r.elapsed));
  const mmss = (s: number) => `0:${String(Math.floor(s)).padStart(2, '0')}`;

  useEffect(() => { if (live.current && r.stream) live.current.srcObject = r.stream; }, [r.stream, r.state]);
  useEffect(() => { void r.start(); }, []); // eslint-disable-line react-hooks/exhaustive-deps

  async function keep() {
    if (!r.recording) return;
    if (r.recording.blob.size > MAX_CAPSULE_BYTES) { setSaveError('La vidéo dépasse 25 Mo. Refais une prise un peu plus courte.'); return; }
    setSaving(true); setSaveError(null);
    const path = capsuleObjectPath(ownerId, slot, r.recording.extension, Date.now());
    const supabase = createBrowserSupabase();
    const { error: upErr } = await supabase.storage.from('capsules').upload(path, r.recording.blob, { contentType: r.recording.mimeType, upsert: false });
    if (upErr) { setSaving(false); setSaveError('L\'envoi a échoué. Vérifie ta connexion et réessaie : ta prise est gardée.'); return; }
    const res = await saveCapsule({ kind, slot, path, mimeType: r.recording.mimeType, durationS: r.recording.durationS });
    setSaving(false);
    if (res.error) { setSaveError(res.error); return; }
    r.release();
    router.push(backHref);
  }

  return (
    <main className="relative mx-auto flex min-h-dvh max-w-md flex-col bg-video text-white">
      <header className="flex items-center justify-between px-4 pt-4">
        <Link href={backHref} onClick={() => r.release()} aria-label="Fermer" className="flex h-11 w-11 items-center justify-center rounded-full bg-white/10">✕</Link>
        <div className="text-center"><p className="text-xs font-semibold text-white/70">Capsule {slot}/{total}</p><p className="text-lg font-extrabold">{def.title}</p></div>
        <span className="w-11" />
      </header>

      <div className="relative mx-4 mt-4 flex-1 overflow-hidden rounded-2xl bg-black">
        {r.state === 'review' && r.recording
          ? <video src={r.recording.url} controls playsInline className="h-full w-full object-cover" />
          : <video ref={live} autoPlay muted playsInline className="h-full w-full -scale-x-100 object-cover" />}
        {r.state === 'recording' && (
          <div className="absolute left-1/2 top-3 flex -translate-x-1/2 items-center gap-2 rounded-full bg-black/60 px-3 py-1.5 text-sm font-bold">
            <span className="h-2.5 w-2.5 rounded-full bg-rec" aria-hidden="true" />REC {mmss(r.elapsed)} / 1:00
          </div>
        )}
        {(r.state === 'ready' || r.state === 'recording') && (
          <section className="absolute inset-x-3 bottom-3 rounded-2xl bg-black/55 p-4 backdrop-blur">
            <p className="text-xs font-bold text-mint">Consigne</p>
            <p className="mt-1 text-[15px] font-semibold leading-snug">{def.prompt}</p>
            <ul className="mt-2 list-disc pl-5 text-sm text-white/85">{def.tips.map((t) => <li key={t}>{t}</li>)}</ul>
          </section>
        )}
        {r.state === 'error' && <p role="alert" className="absolute inset-x-4 top-1/3 rounded-xl bg-warn-soft p-4 text-sm font-semibold text-warn">{r.error}</p>}
      </div>

      <footer className="flex flex-col items-center gap-3 px-4 pb-6 pt-4">
        {r.state === 'recording' && <p className="text-sm text-white/80" aria-live="polite">Encore {remaining} secondes</p>}
        {r.state === 'ready' && <button onClick={r.record} aria-label="Commencer l'enregistrement" className="h-20 w-20 rounded-full border-4 border-white"><span className="mx-auto block h-14 w-14 rounded-full bg-rec" /></button>}
        {r.state === 'recording' && <button onClick={r.stop} aria-label="Arrêter l'enregistrement" className="h-20 w-20 rounded-full border-4 border-white"><span className="mx-auto block h-8 w-8 rounded-md bg-rec" /></button>}
        {r.state === 'review' && (
          <div className="flex w-full gap-3">
            <button onClick={r.retake} disabled={saving} className="h-[52px] flex-1 rounded-full border border-white/40 font-bold">Refaire</button>
            <button onClick={keep} disabled={saving} className="h-[52px] flex-1 rounded-full bg-mint font-bold text-ink disabled:opacity-60">{saving ? 'Envoi…' : 'Garder cette prise'}</button>
          </div>
        )}
        {r.state === 'error' && <button onClick={() => void r.start()} className="h-[52px] w-full rounded-full bg-mint font-bold text-ink">Réessayer</button>}
        {saveError && <p role="alert" className="w-full rounded-xl bg-warn-soft p-3 text-sm font-semibold text-warn">{saveError}</p>}
      </footer>
    </main>
  );
}
```

- [ ] **Step 4: Pages de tournage**

`app/src/app/candidat/capsules/[slot]/page.tsx` :
```tsx
import { notFound, redirect } from 'next/navigation';
import { getCurrentUser } from '@/lib/supabase/server';
import { getSlot } from '@/lib/capsules/slots';
import { CapsuleRecorder } from '@/components/CapsuleRecorder';

export default async function Tournage({ params }: { params: Promise<{ slot: string }> }) {
  const me = await getCurrentUser();
  if (!me || me.role !== 'candidate') redirect('/connexion');
  const slot = Number((await params).slot);
  if (!getSlot('candidate', slot)) notFound();
  return <CapsuleRecorder kind="candidate" ownerId={me.id} slot={slot} backHref="/candidat/capsules" />;
}
```

`app/src/app/entreprise/capsules/[slot]/page.tsx` :
```tsx
import { notFound, redirect } from 'next/navigation';
import { createServerSupabase, getCurrentUser } from '@/lib/supabase/server';
import { getSlot } from '@/lib/capsules/slots';
import { CapsuleRecorder } from '@/components/CapsuleRecorder';

export default async function TournageEntreprise({ params }: { params: Promise<{ slot: string }> }) {
  const me = await getCurrentUser();
  if (!me || me.role !== 'recruiter') redirect('/connexion');
  const slot = Number((await params).slot);
  if (!getSlot('company', slot)) notFound();
  const supabase = await createServerSupabase();
  const { data } = await supabase.from('recruiters').select('company_id').eq('id', me.id).single();
  if (!data) redirect('/entreprise');
  return <CapsuleRecorder kind="company" ownerId={data.company_id} slot={slot} backHref="/entreprise/capsules" />;
}
```

- [ ] **Step 5: Vérifier**

Run: `npm run test && npm run build`. Puis `npm run dev`, connecté en candidat, ouvrir http://localhost:3000/candidat/capsules/1 sur le PC (localhost autorise la caméra) : autoriser la caméra, enregistrer 5 s, arrêter, revoir, « Garder cette prise » → retour sur la liste (Task 6), fichier visible dans Supabase > Storage > capsules, ligne dans `capsules`. Laisser tourner une prise jusqu'au bout : arrêt automatique à 1:00. Refuser la caméra : message d'aide affiché. Capture d'écran pendant l'enregistrement.

- [ ] **Step 6: Commit**

```bash
git add app/src
git commit -m "Prototype : tournage d'une capsule dans le navigateur"
```

---

### Task 6: Liste des capsules et lecture sécurisée

**Files:**
- Create: `app/src/lib/storage/signed.ts`, `app/src/components/CapsulePlayer.tsx`, `app/src/components/CapsuleList.tsx`, `app/src/app/candidat/capsules/page.tsx`, `app/src/app/entreprise/capsules/page.tsx`

**Interfaces:**
- Consumes: `CAPSULE_SLOTS`, `createServerSupabase`, `getCurrentUser`.
- Produces: `signedUrls(bucket: 'capsules' | 'cvs' | 'photos' | 'logos', paths: string[]): Promise<Record<string, string>>` (URL valables 1 h, uniquement pour ce que la base autorise) ; `<CapsulePlayer src={string} title={string} />` ; `<CapsuleList kind ownerId basePath />` (composant serveur réutilisé par les plans suivants).

- [ ] **Step 1: URL signées**

`app/src/lib/storage/signed.ts` :
```ts
import 'server-only';
import { createServerSupabase } from '@/lib/supabase/server';

export async function signedUrls(bucket: 'capsules' | 'cvs' | 'photos' | 'logos', paths: string[]): Promise<Record<string, string>> {
  if (paths.length === 0) return {};
  const supabase = await createServerSupabase();
  const { data } = await supabase.storage.from(bucket).createSignedUrls(paths, 3600);
  const out: Record<string, string> = {};
  for (const item of data ?? []) if (item.signedUrl && item.path) out[item.path] = item.signedUrl;
  return out;
}
```

- [ ] **Step 2: Lecteur et liste**

`app/src/components/CapsulePlayer.tsx` :
```tsx
export function CapsulePlayer({ src, title }: { src: string; title: string }) {
  return <video src={src} controls playsInline preload="metadata" aria-label={title} className="aspect-[9/16] w-full rounded-2xl bg-video object-cover" />;
}
```

`app/src/components/CapsuleList.tsx` :
```tsx
import Link from 'next/link';
import { CAPSULE_SLOTS, type OwnerKind } from '@/lib/capsules/slots';
import { createServerSupabase } from '@/lib/supabase/server';
import { signedUrls } from '@/lib/storage/signed';
import { CapsulePlayer } from './CapsulePlayer';

export async function CapsuleList({ kind, ownerId, basePath }: { kind: OwnerKind; ownerId: string; basePath: string }) {
  const supabase = await createServerSupabase();
  const { data: rows } = await supabase.from('capsules').select('slot, video_path, duration_s').eq('owner_kind', kind).eq('owner_id', ownerId);
  const bySlot = new Map((rows ?? []).map((r) => [r.slot, r]));
  const urls = await signedUrls('capsules', (rows ?? []).map((r) => r.video_path));
  const done = bySlot.size;
  const total = CAPSULE_SLOTS[kind].length;
  const accent = kind === 'candidate' ? 'bg-mint' : 'bg-forest';
  return (
    <section>
      <p className="font-bold">{done} capsule{done > 1 ? 's' : ''} sur {total} tournée{done > 1 ? 's' : ''}</p>
      <div className="mt-2 flex gap-1.5" aria-hidden="true">{CAPSULE_SLOTS[kind].map((s) => <span key={s.slot} className={`h-1.5 flex-1 rounded-full ${bySlot.has(s.slot) ? accent : 'bg-line'}`} />)}</div>
      <ul className="mt-5 grid grid-cols-2 gap-3">
        {CAPSULE_SLOTS[kind].map((s) => {
          const row = bySlot.get(s.slot);
          return (
            <li key={s.slot} className="flex flex-col gap-2">
              {row && urls[row.video_path]
                ? <CapsulePlayer src={urls[row.video_path]} title={`${s.slot} · ${s.title}`} />
                : <div className="flex aspect-[9/16] items-center justify-center rounded-2xl border-2 border-dashed border-line bg-card text-sm font-semibold text-muted">À tourner</div>}
              <p className="text-sm font-bold">{s.slot} · {s.title}</p>
              <Link href={`${basePath}/${s.slot}`} className={`flex h-11 items-center justify-center rounded-full text-sm font-bold ${row ? 'border border-line' : `${accent} ${kind === 'candidate' ? 'text-ink' : 'text-white'}`}`}>
                {row ? 'Refaire' : 'Tourner'}
              </Link>
            </li>
          );
        })}
      </ul>
    </section>
  );
}
```

- [ ] **Step 3: Pages « Mes capsules »**

`app/src/app/candidat/capsules/page.tsx` :
```tsx
import Link from 'next/link';
import { redirect } from 'next/navigation';
import { getCurrentUser } from '@/lib/supabase/server';
import { CapsuleList } from '@/components/CapsuleList';

export default async function MesCapsules() {
  const me = await getCurrentUser();
  if (!me || me.role !== 'candidate') redirect('/connexion');
  return (
    <main className="mx-auto max-w-md px-5 py-6">
      <Link href="/candidat" className="text-sm font-semibold text-muted">← Retour</Link>
      <h1 className="mt-4 text-[26px] font-extrabold">Mes capsules</h1>
      <p className="mt-2 text-[15px] text-muted">5 capsules guidées de 60 secondes. Une consigne s'affiche pendant le tournage. Tu peux refaire autant que tu veux.</p>
      <div className="mt-6"><CapsuleList kind="candidate" ownerId={me.id} basePath="/candidat/capsules" /></div>
    </main>
  );
}
```
`app/src/app/entreprise/capsules/page.tsx` : même chose avec `role !== 'recruiter'`, `ownerId` = `company_id` lu dans `recruiters`, titre « Capsules de l'entreprise », texte au vouvoiement (« 4 capsules de 60 secondes : le lieu, le poste, l'équipe, vos valeurs. »), `kind="company"`, `basePath="/entreprise/capsules"`.

- [ ] **Step 4: Vérifier**

Run: `npm run test && npm run build`. Dans le navigateur : la capsule tournée en Task 5 se lit dans la liste ; « Refaire » remplace la vidéo et l'ancien fichier disparaît du bucket. Avec un **autre** compte candidat, ouvrir l'URL signée copiée depuis le premier : elle marche (c'est un lien temporaire) — mais `CapsuleList` du second compte ne liste jamais les capsules du premier. Relancer `npm run check:rls` → `TOUT EST VERT`. Capture de la liste.

- [ ] **Step 5: Commit**

```bash
git add app/src
git commit -m "Prototype : liste des capsules et lecture par liens temporaires"
```

---

### Task 7: Mise en ligne et test sur de vrais téléphones

**Files:**
- Create: `docs/tests/2026-10-30-test-video-telephones.md`

- [ ] **Step 1: Déployer (Mattia se connecte, l'agent lance)**

Mattia : `npx vercel login` (une fois). Puis, depuis `C:\dev\projects\Procast\app` :
```bash
npx vercel link --yes
npx vercel env add NEXT_PUBLIC_SUPABASE_URL production
npx vercel env add NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY production
npx vercel --prod
```
(Les deux `env add` demandent la valeur : c'est Mattia qui la colle.) Expected : une adresse `https://….vercel.app`. Dans Supabase > Authentication > URL Configuration, mettre cette adresse en « Site URL ».

- [ ] **Step 2: Test croisé**

Créer `docs/tests/2026-10-30-test-video-telephones.md` avec ce tableau, puis le remplir avec Mattia :

| # | Appareil qui filme | Navigateur | Format obtenu | Appareil qui lit | Lecture OK ? | Son OK ? | Remarque |
|---|---|---|---|---|---|---|---|
| 1 | iPhone | Safari | | Android Chrome | | | |
| 2 | Android | Chrome | | iPhone Safari | | | |
| 3 | PC | Chrome | | iPhone Safari | | | |
| 4 | iPhone | Safari | | PC Chrome | | | |

Format obtenu = colonne `mime_type` de la table `capsules`.

- [ ] **Step 3: Décider**

Si les 4 lignes sont OK : on garde l'approche A, plan 2 ensuite. Si une lecture échoue : ajouter le secours vidéo (Mux ou Cloudflare Stream) **avant** le plan 2 — rédiger alors un mini-plan dédié.

- [ ] **Step 4: Commit**

```bash
git add docs/tests
git commit -m "Prototype : en ligne, test vidéo sur téléphones"
git push
```

---

## Plans suivants (à écrire après le plan 1)

- **Plan 2** — Profil candidat complet (CV, photo, visibilité) ; page entreprise ; offres (publier, explorer, détail) ; candidature.
- **Plan 3** — Candidatures reçues, fiche candidat, notes, statuts ; messagerie temps réel ; entretiens et .ics.
- **Plan 4** — Agenda, base de talents et invitation ; données de démo ; passe visuelle d'après la maquette ; mise en ligne finale.
