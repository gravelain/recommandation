# Infrastructure MVP — Next.js + PostgreSQL

Ce document décrit l'infrastructure recommandée pour le MVP.

L'objectif est de disposer d'une architecture :

- simple à maintenir
- gratuite ou quasi gratuite au stade MVP
- sans serveur à administrer
- sans Docker obligatoire
- sans Supabase
- compatible avec Next.js
- basée sur un vrai PostgreSQL
- prête à gérer l'authentification et les OTP par email
- facilement évolutive vers une vraie infrastructure de production

---

## 1. Architecture recommandée

```text
                         ┌─────────────────────┐
                         │       GitHub        │
                         │   Repository / CI   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │       Vercel        │
                         │      Next.js        │
                         │                     │
                         │  - Frontend         │
                         │  - API Routes       │
                         │  - Server Actions   │
                         │  - SSR              │
                         └──────┬─────────┬────┘
                                │         │
                    PostgreSQL  │         │ Email / OTP
                                │         │
                                ▼         ▼
                    ┌───────────────┐   ┌───────────────┐
                    │     Neon      │   │    Resend     │
                    │   PostgreSQL  │   │ Transactional │
                    │               │   │     Email     │
                    └───────────────┘   └───────────────┘

                         ┌─────────────────────┐
                         │     Better Auth     │
                         │                     │
                         │ - Sessions          │
                         │ - Users             │
                         │ - Email OTP         │
                         │ - Verification      │
                         └─────────────────────┘
```

### Stack

| Besoin           | Technologie                    | Rôle                           |
| ---------------- | ------------------------------ | ------------------------------ |
| Application      | Next.js                        | Frontend + backend             |
| Hébergement      | Vercel                         | Déploiement Next.js            |
| Base de données  | Neon PostgreSQL                | PostgreSQL managé              |
| ORM              | Drizzle ORM                    | Accès et migrations DB         |
| Authentification | Better Auth                    | Sessions, utilisateurs, OTP    |
| Emails           | Resend                         | Emails transactionnels / OTP   |
| Code             | GitHub                         | Repository + historique        |
| Domaine / DNS    | Cloudflare ou registrar actuel | DNS uniquement au départ       |

---

## 2. Pourquoi cette architecture ?

L'objectif n'est pas de construire l'infrastructure d'une entreprise qui a déjà plusieurs millions d'utilisateurs.

Le MVP doit avoir le minimum de composants possible, mais chaque composant doit être suffisamment sérieux pour pouvoir rester en place lorsque le produit grandira.

On évite donc :

- Kubernetes
- VPS à administrer
- Docker obligatoire
- reverse proxy Nginx
- Redis dès le départ
- RabbitMQ / Kafka
- serveur SMTP maison
- base PostgreSQL installée manuellement
- architecture microservices
- Supabase

Le principe est :

> Un monolithe Next.js + une base PostgreSQL managée + un fournisseur d'email.

C'est largement suffisant pour un MVP.

---

## 3. Hébergement : Vercel

### Pourquoi Vercel ?

Pour une application Next.js, Vercel permet de déployer directement depuis GitHub.

À chaque push :

```text
git push
    ↓
GitHub
    ↓
Vercel
    ↓
Build Next.js
    ↓
Deployment
```

Il n'y a donc pas de serveur Linux à maintenir.

Pas besoin de :

```text
SSH
systemd
nginx
PM2
Docker
certbot
```

Vercel fournit également les déploiements automatiques et les previews.

Documentation :

- https://vercel.com/docs
- https://nextjs.org/learn/pages-router/deploying-nextjs-app-deploy

### Attention au gratuit

Le plan Hobby est gratuit, mais il est destiné aux projets personnels / non commerciaux.

**Pour un MVP personnel / prototype :**

```text
Vercel Hobby
     +
Neon Free
     +
Resend Free
     =
0 $ / mois
```

**Pour un MVP commercial :**

Ne pas essayer de contourner les conditions de Vercel Hobby. Prévoir :

```text
Vercel Pro
     +
Neon
     +
Resend
```

ou migrer l'hébergement Next.js vers une autre plateforme lorsque le produit devient commercial.

---

## 4. Base de données : Neon PostgreSQL

### Choix

Utiliser :

```text
Neon + PostgreSQL
```

et non une base PostgreSQL installée sur un serveur.

Documentation : https://neon.com/docs

Neon fournit un véritable PostgreSQL managé et est particulièrement adapté aux applications serverless.

Le point intéressant pour un MVP est notamment le *scale-to-zero*, qui évite de payer pour une base qui ne travaille pas lorsque l'application est inactive.

Le Free Plan actuel donne notamment :

- 1 GB de stockage PostgreSQL par projet
- 100 CU-hours de compute par mois
- autoscaling
- plusieurs branches

Documentation : https://neon.com/

> Les limites des plans gratuits évoluent : vérifier les conditions actuelles sur le site officiel avant de s'engager.

### Pourquoi pas Supabase ?

Ce projet n'a pas besoin de Supabase.

Supabase apporte beaucoup de choses :

```text
Postgres
Auth
Storage
Realtime
API
Edge Functions
...
```

Mais ici cela ferait essentiellement doublon avec :

```text
Next.js
Better Auth
Neon
Resend
```

On préfère garder chaque responsabilité séparée.

---

## 5. ORM : Drizzle

Pour communiquer avec PostgreSQL : **Drizzle ORM**

Documentation : https://orm.drizzle.team/

Architecture :

```text
Next.js
   │
   ▼
Drizzle
   │
   ▼
PostgreSQL / Neon
```

### Pourquoi Drizzle ?

- TypeScript natif
- très léger
- SQL relativement transparent
- migrations explicites
- très adapté à PostgreSQL
- peu de magie
- fonctionne bien dans des environnements serverless
- permet de conserver une bonne maîtrise du schéma SQL

Le schéma de données doit rester dans le repository.

Exemple :

```text
src/
  db/
    schema/
      users.ts
      organizations.ts
      ...
    index.ts

drizzle/
  migrations/
```

Les migrations doivent être versionnées dans Git.

---

## 6. Authentification : Better Auth

Pour l'authentification : **Better Auth**

Documentation : https://better-auth.com/docs

Better Auth s'intègre directement avec Next.js via une route du type :

```text
/api/auth/[...all]
```

Documentation Next.js : https://better-auth.com/docs/1.6/integrations/next

Il permet notamment de gérer :

- utilisateurs
- sessions
- authentification
- vérification email
- password reset
- OAuth
- OTP
- etc.

Cela évite de développer soi-même un système d'authentification.

---

## 7. OTP : privilégier l'email au début

Il faut distinguer deux choses.

### OTP par email

```text
Utilisateur
    │
    │ demande connexion
    ▼
Next.js / Better Auth
    │
    ▼
Resend
    │
    ▼
Email contenant le code
    │
    ▼
Utilisateur
    │
    │ saisit 6 chiffres
    ▼
Better Auth
    │
    ▼
Session
```

C'est ce qui est recommandé pour le MVP si le produit n'a pas explicitement besoin d'un numéro de téléphone.

Better Auth possède un plugin officiel **Email OTP** qui gère notamment :

- OTP de connexion
- vérification d'email
- récupération de compte
- expiration
- nombre maximum de tentatives
- renouvellement des codes

Par défaut, l'OTP est de 6 chiffres, expire après 5 minutes et le nombre de tentatives est limité.

Documentation : https://better-auth.com/docs/plugins/email-otp

---

## 8. Emails : Resend

Pour envoyer les emails : **Resend**

Documentation : https://resend.com/docs

Ne surtout pas héberger son propre serveur SMTP pour un MVP.

Cela apporte immédiatement des problèmes de :

- réputation IP
- SPF
- DKIM
- DMARC
- délivrabilité
- blacklist
- monitoring
- bounce
- retry
- réputation du domaine

Resend s'occupe de cette partie.

Le plan gratuit actuel permet notamment :

- 3 000 emails / mois
- 100 emails / jour
- 3 domaines vérifiés

Ce qui est largement suffisant pour un petit MVP.

---

## 9. OTP avec Resend

L'intégration logique est :

```text
Better Auth
      │
      │ génère OTP
      ▼
sendVerificationOTP()
      │
      ▼
Resend API
      │
      ▼
Email utilisateur
```

Il ne faut pas créer une deuxième logique OTP dans l'application.

**Better Auth** est responsable de :

- génération du code
- expiration
- validation
- nombre d'essais
- invalidation
- session

**Resend** est uniquement responsable de :

- transporter l'email

Cette séparation est importante.

---

## 10. Si l'OTP est par SMS

Si par « OTP » il s'agit en réalité d'un OTP SMS, l'architecture change légèrement :

```text
Next.js
   │
Better Auth / Auth service
   │
   ▼
SMS Provider
   │
   ▼
Téléphone utilisateur
```

Dans ce cas, il faut choisir un fournisseur SMS comme :

- Twilio
- Vonage
- autre fournisseur SMS adapté au marché cible

Contrairement à l'email, le SMS n'est généralement pas réellement gratuit à volume réel.

Donc pour un MVP :

> Si le produit peut fonctionner avec un OTP par email, choisir l'email.

Si le numéro de téléphone est une exigence fonctionnelle, le SMS devient une dépendance payante à prévoir dans le budget.

---

## 11. Variables d'environnement

Aucune clé secrète ne doit être commitée dans Git.

Exemple :

```env
DATABASE_URL="postgresql://..."

BETTER_AUTH_SECRET="..."
BETTER_AUTH_URL="http://localhost:3000"

RESEND_API_KEY="re_..."
EMAIL_FROM="My App <noreply@example.com>"
```

Le fichier `.env.local` doit être dans `.gitignore`.

Exemple :

```text
.env
.env.local
.env.*.local
```

Les variables de production sont configurées directement dans Vercel.

---

## 12. Environnements

Même pour un MVP, il faut séparer au minimum :

```text
local
   │
   ▼
development
   │
   ▼
production
```

Idéalement :

```text
Neon
├── production
└── staging
```

et :

```text
Vercel
├── Preview
└── Production
```

L'intérêt est d'éviter qu'une migration ou un test détruise la vraie base.

Neon permet d'utiliser des branches de base de données, ce qui est particulièrement intéressant pour isoler les environnements et tester les migrations.

---

## 13. Workflow Git

Le workflow peut rester extrêmement simple.

```text
feature/*
      │
      ▼
Pull Request
      │
      ▼
Vercel Preview
      │
      ▼
Tests
      │
      ▼
merge main
      │
      ▼
Production
```

Chaque Pull Request obtient une preview.

Cela permet de tester une fonctionnalité sans toucher à la production.

---

## 14. Structure du projet

Une structure raisonnable :

```text
project/
├── app/
│   ├── (auth)/
│   ├── dashboard/
│   ├── api/
│   │   └── auth/
│   │       └── [...all]/
│   └── ...
│
├── components/
│
├── lib/
│   ├── auth.ts
│   ├── email.ts
│   └── ...
│
├── db/
│   ├── index.ts
│   └── schema/
│
├── drizzle/
│   └── migrations/
│
├── public/
│
├── .env.example
├── .gitignore
├── drizzle.config.ts
├── package.json
└── README.md
```

Il n'est pas nécessaire de créer :

```text
backend/
frontend/
worker/
auth-service/
email-service/
```

Le MVP doit rester un **monolithe modulaire**.

---

## 15. Ce qu'on ne met PAS maintenant

### Pas de Docker

Pas nécessaire. Le développeur lance simplement :

```bash
npm install
npm run dev
```

ou avec pnpm :

```bash
pnpm install
pnpm dev
```

Docker pourra être ajouté plus tard si l'équipe a réellement besoin d'un environnement reproductible.

### Pas de Redis

Ne pas ajouter Redis « parce qu'une application sérieuse en utilise ». Le MVP n'en a probablement pas besoin.

Si un jour il faut :

- rate limiting distribué
- cache
- queue
- sessions distribuées spécifiques
- jobs asynchrones

alors Redis pourra être ajouté.

### Pas de Kubernetes

Clairement pas nécessaire.

### Pas de microservices

Le backend reste dans Next.js.

```text
Next.js
├── UI
├── Server Components
├── Server Actions
├── Route Handlers
├── Auth
└── Business Logic
```

C'est beaucoup plus simple à maintenir.

---

## 16. Architecture applicative recommandée

Il faut toutefois éviter de mettre toute la logique métier dans les composants React.

Préférer :

```text
UI
 │
 ▼
Server Action / Route Handler
 │
 ▼
Service métier
 │
 ├── Auth
 ├── Validation
 ├── Business logic
 │
 ▼
Repository / Drizzle
 │
 ▼
PostgreSQL
```

Par exemple :

```text
app/
  dashboard/
    page.tsx

lib/
  services/
    user.service.ts
    project.service.ts

db/
  schema/
    users.ts
    projects.ts
```

Cela permet de conserver un monolithe propre sans tomber dans le microservice.

---

## 17. Validation

Utiliser **Zod** pour valider les entrées.

```text
Request
   │
   ▼
Zod validation
   │
   ├── invalid → 400
   │
   └── valid
          │
          ▼
      Business logic
```

Ne jamais faire confiance aux données venant du client.

Même si TypeScript indique :

```ts
email: string
```

cela ne signifie pas que l'utilisateur a réellement envoyé une valeur valide.

---

## 18. Sécurité minimale dès le MVP

Même pour un prototype, mettre en place :

### Secrets

- jamais de secret dans Git
- `.env.local` ignoré
- secrets dans Vercel

### Auth

- Better Auth
- sessions sécurisées
- OTP avec expiration
- limitation des tentatives

### Database

- utilisateur applicatif avec permissions minimales
- pas de PostgreSQL exposé directement au navigateur

### API

- validation des inputs
- vérification systématique de l'utilisateur connecté
- contrôle des permissions côté serveur

### Email

- domaine vérifié
- SPF
- DKIM
- DMARC

---

## 19. Domaine

Au début, il peut utiliser :

```text
mon-app.vercel.app
```

Aucun achat de domaine n'est nécessaire pour commencer.

Quand le produit devient sérieux :

```text
app.example.com
```

et éventuellement :

```text
example.com
```

Pour le DNS, Cloudflare peut être utilisé sans ajouter de complexité applicative.

---

## 20. Coût cible du MVP

### Prototype personnel

| Service       | Coût       |
| ------------- | ---------- |
| Vercel Hobby  | 0 $        |
| Neon Free     | 0 $        |
| Resend Free   | 0 $        |
| Better Auth   | 0 $        |
| Drizzle       | 0 $        |
| GitHub        | 0 $        |
| Domaine       | optionnel  |
| **Total**     | **0 $ / mois** |

Les limites doivent évidemment être surveillées.

---

## 21. Quand commencer à payer ?

Ne pas optimiser le coût à l'extrême. Le bon principe est :

> Payer quand le produit commence à avoir de la valeur, pas avant.

Par exemple :

```text
MVP
 │
 ├── quelques utilisateurs
 ├── validation produit
 └── 0 $/mois
       │
       ▼
Premiers utilisateurs réels
       │
       ▼
Revenus / usage significatif
       │
       ▼
Infrastructure payante
```

Le premier coût devrait être accepté lorsque le produit justifie réellement le coût.

---

## 22. Première évolution de l'infrastructure

Quand le MVP commence à fonctionner :

```text
                    ┌──────────────┐
                    │   Vercel     │
                    │    Next.js   │
                    └──────┬───────┘
                           │
              ┌────────────┼─────────────┐
              │            │             │
              ▼            ▼             ▼
           Neon         Resend       Monitoring
         PostgreSQL       Email         / Logs
```

On ne change toujours pas d'architecture. On augmente simplement les ressources.

---

## 23. Deuxième évolution

Si le produit devient réellement important :

```text
                       ┌───────────────┐
                       │    Vercel     │
                       │    Next.js    │
                       └───────┬───────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                ▼              ▼              ▼
             Postgres        Redis          Queue
              Neon           cache          / Jobs
                │                             │
                └──────────────┬──────────────┘
                               ▼
                        Workers / Jobs
```

À ce moment seulement, on introduit des composants supplémentaires.

---

## 24. Monitoring

Pour le tout début :

- logs Vercel
- logs applicatifs structurés
- monitoring Neon
- logs Resend

Pas besoin de construire une plateforme d'observabilité complète.

Quand le nombre d'utilisateurs augmente, ajouter un outil comme **Sentry** pour les erreurs applicatives devient pertinent.

---

## 25. Backups

Ne pas considérer le MVP comme « sans risque » simplement parce que Neon est managé.

Il faut :

- vérifier la politique de rétention
- tester une restauration
- conserver les migrations dans Git
- ne jamais utiliser la production pour faire des tests destructifs

Neon fournit des mécanismes de restauration et de branching (une fenêtre de restauration de quelques heures sur le plan gratuit), mais pour une vraie production il faudra prévoir une stratégie de sauvegarde adaptée aux exigences métier.

---

## 26. Checklist de mise en ligne

### GitHub

- [ ] Repository créé
- [ ] `main` protégée
- [ ] `.env.local` dans `.gitignore`
- [ ] `.env.example` présent
- [ ] README présent

### Neon

- [ ] Projet PostgreSQL créé
- [ ] Database créée
- [ ] `DATABASE_URL` récupérée
- [ ] Migration initiale exécutée
- [ ] Environnement staging séparé si nécessaire

### Better Auth

- [ ] Secret généré
- [ ] Auth configurée
- [ ] Sessions testées
- [ ] OTP testé
- [ ] Expiration OTP testée
- [ ] Rate limiting / nombre d'essais vérifié

### Resend

- [ ] Compte créé
- [ ] Domaine vérifié
- [ ] SPF configuré
- [ ] DKIM configuré
- [ ] DMARC configuré
- [ ] API key créée
- [ ] Email OTP testé

### Vercel

- [ ] Repository GitHub connecté
- [ ] Variables d'environnement configurées
- [ ] Preview deployment testé
- [ ] Production deployment testé
- [ ] Domaine configuré si nécessaire

---

## 27. Variables d'environnement de production

Exemple :

```env
DATABASE_URL=...

BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://app.example.com

RESEND_API_KEY=...
EMAIL_FROM=My App <noreply@example.com>
```

Les vraies valeurs ne doivent jamais apparaître dans le repository.

---

## 28. Résumé de la décision technique

```text
┌─────────────────────────────────────────┐
│                FRONTEND                 │
│                 Next.js                 │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│                 VERCEL                  │
│       Hosting / CDN / Functions         │
└───────────────┬─────────────────────────┘
                │
       ┌────────┴─────────┐
       │                  │
       ▼                  ▼
┌──────────────┐    ┌──────────────┐
│    Neon      │    │    Resend    │
│ PostgreSQL   │    │    Email     │
└──────┬───────┘    └──────────────┘
       │
       ▼
┌──────────────┐
│    Drizzle   │
│     ORM      │
└──────────────┘

Authentication:
Better Auth
       │
       ├── Sessions
       ├── Users
       └── Email OTP
```

---

## 29. Philosophie d'architecture

Le point le plus important n'est pas le choix individuel de chaque outil. C'est de ne pas sur-architecturer le MVP.

L'infrastructure initiale doit être :

```text
1 repository
1 application Next.js
1 PostgreSQL
1 fournisseur email
1 hébergeur
```

C'est tout.

Si le produit fonctionne, on aura alors un problème intéressant :

> « Comment faire évoluer une application qui a des utilisateurs ? »

Et non :

> « Comment maintenir les 14 services que nous avons créés avant d'avoir un utilisateur ? »

---

## 30. Stack finale recommandée

### Obligatoire

- Next.js
- Vercel
- Neon PostgreSQL
- Drizzle ORM
- Better Auth
- Resend
- GitHub

### Optionnel plus tard

- Sentry
- Redis
- Queue / background jobs
- stockage objet
- Cloudflare
- monitoring avancé

### À éviter au stade MVP

- Kubernetes
- Docker obligatoire
- VPS
- microservices
- Redis sans besoin concret
- serveur SMTP maison
- Supabase
- architecture distribuée

---

## 31. Documentation officielle

| Outil                    | Lien                                                  |
| ------------------------ | ----------------------------------------------------- |
| Next.js                  | https://nextjs.org/docs                               |
| Vercel                   | https://vercel.com/docs                               |
| Neon                     | https://neon.com/docs                                 |
| Drizzle ORM              | https://orm.drizzle.team/docs/overview                |
| Better Auth              | https://better-auth.com/docs                          |
| Better Auth — Next.js    | https://better-auth.com/docs/1.6/integrations/next    |
| Better Auth — Email OTP  | https://better-auth.com/docs/plugins/email-otp        |
| Resend                   | https://resend.com/docs                               |

---

## Conclusion

Pour le MVP, partir sur :

**Next.js + Vercel + Neon + Drizzle + Better Auth + Resend.**

C'est une architecture suffisamment simple pour être mise en place en quelques heures, mais suffisamment propre pour ne pas avoir à tout réécrire dès que le MVP trouve ses premiers utilisateurs.

La seule réserve importante concerne Vercel Hobby : s'il s'agit déjà d'un produit commercial, il ne faut pas considérer le plan gratuit comme une solution de production commerciale. Dans ce cas, il faut budgéter l'hébergement dès le départ ou choisir une alternative d'hébergement compatible avec l'usage commercial.
