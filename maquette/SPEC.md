# Procast — brief commun de la maquette (à lire en entier avant d'écrire un écran)

## Le produit (ce qu'on sait, ne rien inventer d'autre)

Procast — « Motion your future ». Plateforme belge de recrutement par **capsules vidéo guidées**.
Les candidats ET les entreprises se présentent en vidéo : c'est un match dans les deux sens.

- Cible candidats : diplômés en recherche active d'emploi (pas seulement les jeunes), postes à profil défini
  dans des multinationales (économie, juridique, RH, IT, ingénieur). Lancement à Bruxelles / Belgique, FR-NL-EN.
- **Pas de swipe à la Tinder** : jugé trop ludique pour une décision de vie. Le flux est sérieux :
  l'entreprise publie une offre → le candidat postule en envoyant ses capsules → l'entreprise choisit qui rencontrer.
  Plus tard, l'entreprise pourra aussi chercher dans une base de talents.
- **Capsules guidées, durée égale : 60 secondes chacune**, avec une consigne affichée pendant le tournage.
  - Candidat : 1. Qui je suis · 2. Mon diplôme · 3. Mon expérience · 4. Mes langues · 5. Hors du travail (hobbies).
  - Entreprise : 1. Le lieu · 2. Le poste · 3. L'équipe · 4. Nos valeurs. (+ rémunération et conditions en texte.)
- But : réduire l'anxiété de postuler, briser la glace avant l'entretien, mieux matcher la personne et l'environnement.
- Profil candidat (croquis d'origine) : photo, nom prénom, date de naissance, lien LinkedIn, CV (PDF), export PDF du profil.
- Sections profil candidat : infos perso, formation, expérience, langues, centres d'intérêt, hobbies.
- Sections fiche entreprise : logo, adresse, secteur, poste proposé, lieu de travail, valeurs, offre, collègues, rémunération.

## Données d'exemple (fictives — n'utilise AUCUNE vraie marque d'entreprise)

- Candidat principal : **Jacques Moets**, 32 ans, ingénieur industriel, Bruxelles (Ixelles). Master ingénieur industriel
  (ECAM Bruxelles, 2016). 6 ans chez un fabricant d'équipements (amélioration continue, Lean). Langues : FR natif, NL courant,
  EN courant. Hobbies : vélo de route, restauration de vieilles radios. Initiales « JM ».
- Autres candidats (côté entreprise) : Camille Dubois (24, juriste junior, Liège→Bruxelles), Thomas Peeters (35, développeur
  back-end, Gand), Isabelle Renard (41, en reconversion vers les RH), Yassine El Amrani (27, ingénieur industriel, Charleroi),
  Lotte Janssens (29, ingénieure qualité, Louvain).
- Entreprise principale : **Veltra Industries** — fabrication d'équipements industriels, siège à Evere (Bruxelles), ~1 200
  personnes en Belgique. Recruteuse : **Sarah Lambert**, Talent Acquisition. Offre phare : « Ingénieur·e industriel·le —
  amélioration continue (M/F/X) », CDI, Evere, hybride 2 j/sem, 3 900 – 4 500 € brut/mois + voiture de société, FR + NL.
- Autres entreprises : Lumen Legal (cabinet d'avocats d'affaires, Bruxelles), Brightwave (éditeur logiciel, Gand),
  Nordhal Bank (banque, Bruxelles), Celestra Pharma (pharma, Wavre).
- Dates : on est fin octobre 2026. Entretiens proposés en novembre 2026.

## Identité visuelle (obligatoire, identique sur tous les écrans)

Police unique : **Manrope** (Google Fonts), 400/500/600/700/800. Dans `<helmet>` :
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="">
<link href="https://fonts.googleapis.com/css2?family=Manrope:wght@400;500;600;700;800&amp;display=swap" rel="stylesheet">
```

Couleurs (valeurs en dur dans les `style="…"`, pas de variables CSS) :

| rôle | hex | usage |
|---|---|---|
| ground | `#F4F7F3` | fond d'écran clair |
| card | `#FFFFFF` | cartes |
| ink | `#0F2A20` | texte principal |
| muted | `#53655B` | texte secondaire (≥ 4.5:1 sur ground et card) |
| line | `#DDE5DF` | bordures 1px |
| **mint** (vert clair, CANDIDAT) | `#8FE3B4` | aplats candidat : bouton principal, onglet actif, puces. Texte posé dessus = ink |
| mint-strong | `#1B7A4C` | texte/icône vert sur fond clair (liens, statuts positifs) |
| mint-soft | `#E3F6EB` | fonds de puces / badges candidat |
| **forest** (vert foncé, ENTREPRISE) | `#0D3B2C` | aplats entreprise : barre latérale, bouton principal entreprise. Texte dessus = `#FFFFFF` |
| forest-2 | `#14503D` | cartes / survol sur fond forest |
| forest-text-2 | `#B5CEC2` | texte secondaire sur forest |
| video | `#17251F` | fond des vignettes vidéo |
| rec | `#D93A3A` | point d'enregistrement uniquement |
| warn | `#9A5B00` sur `#FDF1DC` | statut « en attente » |

Règle de dualité : **côté candidat tout ce qui est « marque » est mint ; côté entreprise c'est forest.**

Formes : cartes radius 16px, bordure 1px line, pas d'ombre lourde (au plus `0 1px 2px rgba(15,42,32,.06)`).
Boutons : hauteur 52px (mobile) / 44px (desktop), radius 999px, font 600 16px. Puces : radius 999px, 13px 600.
Échelle typo : titre d'écran 26px/800, titre de section 18px/700, corps 15px/500, légende 13px/500. Interligne 1.4.
Espacement : multiples de 4 ; marge latérale mobile 20px.
Interdits : dégradés, emoji, bordure gauche colorée sur les cartes, ombres néon, fausses barres d'état (heure/batterie),
lorem ipsum, scores de « match » en %.

### Logo (inline SVG, couleur via `color`)
Candidat : `color:#2FBF71`. Entreprise : `color:#0D3B2C` sur clair, `#8FE3B4` sur forest.
```html
<div style="display: flex; align-items: center; gap: 8px">
  <svg width="28" height="28" viewBox="0 0 40 40" fill="none" aria-hidden="true" style="color: #2FBF71"><path d="M4 15V7a3 3 0 0 1 3-3h11" stroke="currentColor" stroke-width="3.5" stroke-linecap="round"></path><path d="M36 25v8a3 3 0 0 1-3 3H22" stroke="currentColor" stroke-width="3.5" stroke-linecap="round"></path><path d="M15 12.5v15l13-7.5z" fill="currentColor" stroke="currentColor" stroke-width="2.5" stroke-linejoin="round"></path></svg>
  <span style="font-size: 20px; font-weight: 800; letter-spacing: -0.02em; color: #0F2A20">ProCast</span>
</div>
```

### Vignette vidéo (capsule) — composant de référence
Fond `#17251F`, radius 16px, ratio 9:16 (portrait) ou 16:9 selon la place. Au centre un rond blanc 48px avec triangle play
forest. En bas à gauche une puce « 1 · Qui je suis » (fond rgba(255,255,255,.14), texte blanc). En bas à droite la durée
« 1:00 » (texte blanc 12px 700). Pour évoquer la personne/le lieu sans photo : grandes initiales (« JM ») en
`rgba(255,255,255,.12)` 96px 800, ou une silhouette en traits. Jamais d'image externe.
```html
<div style="position: relative; width: 100%; aspect-ratio: 16 / 9; background: #17251F; border-radius: 16px; overflow: hidden; display: flex; align-items: center; justify-content: center">
  <span aria-hidden="true" style="position: absolute; font-size: 96px; font-weight: 800; color: rgba(255,255,255,.10); letter-spacing: -0.04em">JM</span>
  <span style="position: relative; width: 48px; height: 48px; border-radius: 999px; background: #FFFFFF; display: flex; align-items: center; justify-content: center"><svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true"><path d="M7 4.5v15l13-7.5z" fill="#0D3B2C"></path></svg></span>
  <span style="position: absolute; left: 12px; bottom: 12px; padding: 4px 10px; border-radius: 999px; background: rgba(255,255,255,.14); color: #FFFFFF; font-size: 12px; font-weight: 700">1 · Qui je suis</span>
  <span style="position: absolute; right: 12px; bottom: 12px; color: #FFFFFF; font-size: 12px; font-weight: 700">1:00</span>
</div>
```

### Icônes
Inline SVG au trait, style Lucide : `viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"
stroke-linecap="round" stroke-linejoin="round"`, 20–24px, `aria-hidden="true"` (et `aria-label` sur le bouton qui les porte).
Tu peux reprendre les tracés Lucide de mémoire (compass, inbox, message-circle, user, video, check, clock, map-pin,
briefcase, calendar, search, chevron-right, arrow-left, upload, file-text, linkedin → mets plutôt un carré arrondi avec « in »).

## Navigation

### Candidat — appli mobile 390×844 (taille FIXE)
Barre d'onglets en bas (hauteur 72px, fond #FFFFFF, bordure haute line) avec 4 liens :
Explorer → `C-Explore.dc.html` · Candidatures → `C-Candidatures.dc.html` · Messages → `C-Messages.dc.html` ·
Profil → `C-Profil.dc.html`. Onglet actif : icône dans une pastille mint 56×32 radius 999, libellé ink 700 ; inactifs muted.
Chaque onglet est un `<a href>` de 44px min de haut. Écrans du parcours d'inscription (création profil, capsules,
enregistrement) : pas de barre d'onglets, un bouton retour en haut à gauche à la place.
Racine : `<div style="width: 390px; height: 844px; position: relative; overflow: hidden; background: #F4F7F3; font-family: Manrope, system-ui, sans-serif; color: #0F2A20; display: flex; flex-direction: column">`
avec une zone de contenu `flex: 1; overflow-y: auto` et la barre d'onglets en bas.

### Entreprise — application web, PAGE fluide (boards `w` 1440, `expand: "fill"`)
Barre latérale forest (largeur 248px, logo variante mint sur forest, puis liens) : Profil entreprise → `E-Profil.dc.html` ·
Publier une offre → `E-Publier.dc.html` · Candidatures → `E-Candidatures.dc.html` · Base de talents →
`E-Base.dc.html` · Agenda → `E-Agenda.dc.html` · Aide (lien `#`). Lien actif : fond forest-2, texte blanc 700 ;
inactifs forest-text-2. En bas de la barre : avatar « SL » + « Sarah Lambert · Veltra Industries ».
Mise en page fluide OBLIGATOIRE : racine `<div style="min-height: 100vh; display: flex; flex-wrap: wrap; background: #F4F7F3; font-family: Manrope, system-ui, sans-serif; color: #0F2A20">`,
barre latérale `flex: 1 1 248px; max-width: 100%` (PAS sticky, PAS 100vh — sur téléphone elle passe au-dessus),
contenu `flex: 999 1 560px; min-width: 0; padding: 32px 40px`, conteneur interne `max-width: 1100px`.
Grilles en `repeat(N, minmax(0, 1fr))` ou `repeat(auto-fill, minmax(240px, 1fr))` ; tableaux larges dans une boîte
`overflow-x: auto`. Pas de largeur en px sur la racine.

## Format de fichier .dc.html (règles qui cassent en silence)

Squelette exact :
```html
<!doctype html>
<html lang="fr">
<head>
<meta charset="utf-8">
<title>Procast — Explorer</title>
<script src="./support.js"></script>
</head>
<body>
<x-dc>
<helmet>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="">
<link href="https://fonts.googleapis.com/css2?family=Manrope:wght@400;500;600;700;800&amp;display=swap" rel="stylesheet">
<style>
body{margin:0}
a{color:#1B7A4C}a:hover{color:#0F2A20}
</style>
</helmet>
<div style="…racine…">…</div>
</x-dc>
<script type="text/x-dc" data-dc-script data-props='{"$preview":{"width":390,"height":844}}'>
class Component extends DCLogic {
renderVals() {
return {};
}
}
</script>
</body>
</html>
```
- Garde `<script src="./support.js"></script>` EXACTEMENT. Toujours le bloc `<script type="text/x-dc" data-dc-script>`, JS classique.
- Ferme TOUS les éléments non vides (`<path …></path>`, pas `<path/>`), mets des guillemets à TOUS les attributs.
- Mise en forme en `style="…"` inline. `<helmet><style>` seulement pour `body{margin:0}`, `a`/`a:hover`, et au besoin
  `:focus-visible` et un reset `button{font:inherit}`.
- `{{trou}}` = simple lecture pointée dans `renderVals()`, jamais une expression. Boucles : `<sc-for list="{{items}}" as="it" hint-placeholder-count="3">…{{it.name}}…</sc-for>`.
  Conditions : `<sc-if value="{{flag}}" hint-placeholder-val="{{ true }}">…</sc-if>`.
- Le texte rédactionnel reste en dur dans le balisage (pas de props). Les données répétées (listes de candidats, d'offres)
  peuvent venir de `renderVals()`.
- Interactions possibles (bienvenues, sobres) : état dans `this.state`, gestionnaires renvoyés par `renderVals()` et liés
  par `onClick="{{ choisir }}"`. Ex. filtres sélectionnables, onglets de section, case cochée.
- Liens entre écrans : `<a href="C-Offre.dc.html" style="…">Voir l'offre</a>` — style le `<a>` lui-même comme un bouton,
  jamais de `<button>` dans un `<a>`.
- Accessibilité : vrais `<button>`, `<a href>`, `<input>` + `<label>` ; `aria-label` sur boutons-icônes ; contraste 4.5:1.
  Cibles tactiles ≥ 44px.
- Interdits : `<iframe>`, `<object>`, `<embed>`, `innerHTML`, `appendChild`, réseau (sauf la police Google), emoji,
  `data:` URI, images externes, écouteurs clavier globaux.
- `lang="fr"`, tout le texte en français.

## Écrans (fichier → contenu)

Candidat (mobile 390×844) :
- `Main.dc.html` — accueil : logo, « Motion your future », choix « Je cherche un emploi » / « Je recrute ». (fait par le coordinateur)
- `C-Inscription.dc.html` — création du profil (croquis d'origine) : photo, nom prénom, date de naissance, LinkedIn, CV PDF, ville, domaine.
- `C-Capsules.dc.html` — « Mes capsules » : les 5 capsules guidées, état (tournée / à tourner), progression 3/5, conseil.
- `C-Tournage.dc.html` — tournage d'une capsule : cadre caméra (fond video), consigne guidée, minuteur 0:42 / 1:00, bouton rec.
- `C-Explore.dc.html` — fil d'offres : chaque offre avec sa capsule entreprise, filtres (domaine, ville, langue). PAS de swipe.
- `C-Offre.dc.html` — détail d'une offre Veltra : capsules entreprise (lieu, poste, équipe, valeurs), conditions, « Postuler avec mes capsules ».
- `C-Candidatures.dc.html` — suivi : envoyée / vue / entretien proposé / pas retenue, avec frise.
- `C-Messages.dc.html` — conversation avec Sarah Lambert (Veltra) qui propose un entretien, créneaux à choisir.
- `C-Profil.dc.html` — profil de Jacques vu comme les entreprises le voient : capsules, infos, export PDF, LinkedIn.

Entreprise (web, PAGE fluide, `w` 1440) :
- `E-Profil.dc.html` — page entreprise Veltra : 4 capsules, infos (adresse, secteur, taille), valeurs, offres ouvertes, collègues.
- `E-Publier.dc.html` — publier une offre : formulaire + capsule « Le poste » à joindre + aperçu côté candidat.
- `E-Candidatures.dc.html` — candidatures reçues pour l'offre phare : liste avec vignette capsule, statut, filtres.
- `E-Fiche.dc.html` — fiche d'un candidat (Jacques) : lecteur des 5 capsules, CV, notes privées, « Proposer un entretien » / « Pas cette fois ».
- `E-Agenda.dc.html` — agenda des entretiens de la semaine + créneaux proposés.
- `E-Base.dc.html` — base de talents (recherche de profils) avec bandeau « Bientôt » / phase 2.

Chaque écran doit être soigné, dense en vrai contenu, cohérent avec ce brief. Ne vérifie rien dans un navigateur.
