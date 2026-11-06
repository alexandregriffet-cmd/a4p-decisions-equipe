# Décisions d'équipe · A4P Mental (entreprise)
Jeu de décision pour managers d'équipes en entreprise : quatre rôles (Dirigeant, Responsable RH, Manager, Chef de projet), une trajectoire de projet en 12 échéances, comptes, nombre d'accès par compte, parties enregistrées, débrief en ligne et export PDF.

Ce projet est **indépendant** : son propre dépôt GitHub, son propre projet Supabase, son propre projet Vercel.

| Fichier | Rôle |
|---|---|
| `index.html` | Le jeu : connexion, mes parties, partie, débrief, PDF |
| `admin.html` | Console d'administration : comptes et accès |
| `config.js` | URL et clé publique de votre projet Supabase (à remplir) |
| `api/admin.js` | Fonction Vercel qui crée les comptes en sécurité |
| `supabase/schema.sql` | La base de données |
| `package.json` | Demande à Vercel d'utiliser Node 26 |

## Les accès
1 accès = 1 partie lancée. Reprendre ou rouvrir une partie ne consomme rien. Par compte : nombre d'accès de 0 à n ou « Illimité », suspension, date d'expiration. Contrôle côté serveur.

## Étape 1 · GitHub
Nouveau dépôt privé `a4p-decisions-equipe`. Déposez `index.html`, `admin.html`, `package.json`, puis créez `config.js` et `api/admin.js` avec « Create new file » (les .js ne se téléchargent pas toujours).

## Étape 2 · Supabase (nouveau projet `a4p-decisions-equipe`)
1. SQL Editor, collez tout `supabase/schema.sql`, Run.
2. Authentication, Sign In / Providers, Email : désactivez « Allow new users to sign up ».
3. Authentication, Users, Add user : votre email, un mot de passe neuf, « Auto Confirm User » coché.
4. SQL Editor :
```sql
insert into public.profiles (id, email, display_name, role, quota, active)
select id, email, 'Alexandre Griffet', 'admin', null, true from auth.users where email = 'votre@email.fr'
on conflict (id) do update set role = 'admin', quota = null, active = true;
select email, role, quota, active from public.profiles;
```
5. Project Settings, API : relevez Project URL (sans /rest/v1/), clé publique (anon) et clé service_role (secrète).

## Étape 3 · config.js
```js
window.A4P_CONFIG = {
  SUPABASE_URL: "https://xxxx.supabase.co",
  SUPABASE_ANON_KEY: "clé publique"
};
```

## Étape 4 · Vercel (nouveau projet)
Import du dépôt, Framework « Other ». Variables : `SUPABASE_URL` et `SUPABASE_SERVICE_ROLE_KEY` (clé service_role, jamais dans GitHub). Deploy.

## Test
Connexion administrateur, Administration, compte test avec 1 accès, lancer une partie, vérifier que le second lancement est refusé.

## Problèmes
- 404 : `index.html` n'est pas à la racine du dépôt.
- Pas de connexion demandée : `config.js` contient encore « VOTRE ».
- Clé `sb_publishable_` refusée : utilisez la clé Legacy anon (`eyJ`).
- « Variables manquantes » : ajoutez-les dans Vercel puis Redeploy.
