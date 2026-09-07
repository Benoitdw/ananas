# 🍍 Ananas

Veille emploi geolocalisee. Tu choisis des entreprises sur une carte, Ananas
surveille leurs pages carrieres et t'envoie **une notification Telegram par
jour, uniquement quand il y a de nouvelles offres**.

Fait suite au POC `../job_an` (carte Dash des 81 partenaires BioWin, favoris en
CSV) : le repertoire d'entreprises et son geocodage sont repris, tout le reste
est neuf — comptes utilisateurs, Postgres, offres, detection de nouveaute,
notifications.

## Organisation

Ce depot est un **orchestrateur** : `api/` et `web/` sont des submodules git
(chacun un depot independant), et il ne contient lui-meme que le compose et
la configuration qui les fait tourner ensemble.

```
Ananas/
├── docker-compose.yml       # db + backend + worker + web, build local (dev)
├── docker-compose.prod.yml  # memes services, images ghcr.io pre-construites (NAS)
├── .env                     # copie de .env.example, jamais commite
├── api/                     -> submodule: FastAPI, worker, scrapers, notifiers
└── web/                     -> submodule: SvelteKit, carte MapLibre
```

Les deux depots ne partagent aucun code : ils ne se connaissent que par le
contrat HTTP de l'API.

Cloner ce depot avec les submodules :

```bash
git clone --recurse-submodules git@github.com:Benoitdw/ananas.git
# ou, si deja clone sans --recurse-submodules :
git submodule update --init --recursive
```

## Demarrage

```bash
cp .env.example .env
# renseigne JWT_SECRET (openssl rand -hex 32) et TELEGRAM_BOT_TOKEN

docker compose up --build -d
docker compose exec backend alembic upgrade head
docker compose exec backend python -m ananas.seed
```

Le seed charge trois repertoires : les 81 partenaires BioWin herites du POC
(`api/data/partners.csv`), 49 biotech/pharma belges (`api/data/biotech-belgique.csv`)
et les 137 membres et partenaires du BioPark de Charleroi
(`api/data/biopark.csv`, suivi dans `api/BIOPARK.md`). Chacun garde sa
provenance (`biowin`, `curated`, `biopark`) et se filtre separement sur la
carte.

Pour disposer d'un compte d'administration, renseigne `ADMIN_EMAIL` et
`ADMIN_PASSWORD` dans le `.env` puis :

```bash
docker compose exec backend python -m ananas.admin
```

- front : http://localhost:5173
- API et documentation interactive : http://localhost:8000/docs

Les ports hote sont configurables dans le `.env` (`POSTGRES_PORT`,
`BACKEND_PORT`, `WEB_PORT`) — 5432 est souvent deja pris par un autre projet.

## Deploiement (NAS, etc.)

Chaque submodule construit et publie son image a chaque push sur `master`
(voir `api/.github/workflows/docker-publish.yml` et
`web/.github/workflows/docker-publish.yml`) :

- `ghcr.io/benoitdw/ananas_backend` — sert au `backend` et au `worker`
- `ghcr.io/benoitdw/ananas_web` — build `production` du Dockerfile (adapter-node)

`docker-compose.prod.yml` consomme ces images au lieu de builder localement.
Sur le NAS :

```bash
git clone --recurse-submodules git@github.com:Benoitdw/ananas.git
cd ananas
cp .env.example .env   # renseigne les secrets, comme en dev

# si les packages GHCR sont prives (par defaut a la premiere publication) :
# un token classic avec le scope read:packages suffit
echo $GHCR_TOKEN | docker login ghcr.io -u <ton-user-github> --password-stdin

docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml exec backend alembic upgrade head
```

### Reverse proxy (Nginx Proxy Manager)

Le front est un **SPA** : le navigateur appelle l'API directement, jamais le
conteneur `web`. `http://backend:8000` ne resout donc pas cote front — il faut
**deux** proxy hosts NPM, l'un pour le front, l'autre pour l'API :

```bash
docker network create proxy          # une seule fois
# rattache le conteneur NPM au reseau `proxy`
```

`docker-compose.prod.yml` place deja `web` et `backend` sur ce reseau. Dans NPM :

| Proxy host                     | Forward         | Sert a                         |
| ------------------------------ | --------------- | ------------------------------ |
| `ananas.mondomaine.fr`         | `web:3000`      | le front                       |
| `api.ananas.mondomaine.fr`     | `backend:8000`  | l'API (fetch du navigateur)    |

Puis dans `.env` du NAS :

```
PUBLIC_API_URL=https://api.ananas.mondomaine.fr
CORS_ORIGINS=https://ananas.mondomaine.fr
```

Les deux domaines partageant le suffixe `mondomaine.fr`, le cookie de session
(`SameSite=Lax`) passe sans reglage supplementaire.

Pour rendre les packages publics (evite le `docker login` sur le NAS) :
GitHub → onglet **Packages** du compte → paquet → *Package settings* →
*Change visibility* → Public, pour `ananas_backend` et `ananas_web`.

Pour mettre a jour apres un nouveau push :

```bash
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

`git submodule update --remote` n'est utile ici que pour faire avancer les
pointeurs de submodule dans ce depot (traçabilite) — les images tournant sur
le NAS viennent de GHCR, pas d'un `git pull` local.

## Le parcours

1. `/` → creer un compte (email + mot de passe, actif immediatement, aucun mail
   a valider).
2. `/map` → 125 entreprises geolocalisees sur 130. Clic sur un marqueur, puis
   **Enregistrer**. Les favoris passent en orange.
3. **Proposer une entreprise** absente du repertoire depuis la carte : nom,
   tags et adresse suffisent. Elle est geocodee a l'envoi, devient visible et
   filtrable par tout le monde, et peut etre ajoutee a ta veille dans la
   foulee. Les doublons sont detectes par nom et par domaine du site.
4. `/settings` → **renseigner son domicile** (une ville suffit) et son
   perimetre de recherche. L'adresse est geocodee a l'enregistrement; la carte
   trace alors un cercle en pointilles autour du point de reference, chaque
   entreprise et chaque offre affiche sa distance a vol d'oiseau, et on peut
   filtrer « a moins de N km » ou trier par proximite depuis la carte comme
   depuis la page Offres. Le curseur de la carte fait varier le rayon sans
   toucher au reglage enregistre. Une case — decochee par defaut — applique
   aussi ce perimetre a la notification quotidienne : une entreprise dont la
   position est inconnue reste notifiee, et le message annonce en pied combien
   d'offres le rayon a ecartees, pour qu'un filtre ne se confonde jamais avec
   une journee sans offre.
5. `/settings` → **importer son CV en PDF** (ou coller le texte) et decrire ses
   aspirations. Ananas en extrait un
   profil structure (et te montre ce qu'il en a compris), puis donne a chaque
   offre un score de correspondance. **Ce qui a ete compris s'edite chip par
   chip** — retirer une competence mal lue, ajouter une langue oubliee,
   corriger le niveau : les scores sont recalcules a l'enregistrement, sans
   repasser par le CV. Tu regles le seuil de pertinence et tu peux ne recevoir
   que les offres au-dessus.
6. `/settings` → **Connecter Telegram** : un bouton ouvre le bot (ou un QR code
   a scanner depuis le telephone), un appui sur **Démarrer**, et la page se lie
   toute seule. Rien a chercher, aucun identifiant a recopier.
7. Chaque nuit a minuit, le worker scrape les entreprises suivies et soumet un
   lot de caracterisation a Gemini (moitie prix, mais sans delai garanti) ; a
   7h il recolte ce qui est pret et envoie les offres qui te concernent,
   **triees par score**. Rien de neuf → aucun message. Une entreprise que tu
   suis mais sans scraper est signalee en pied de notification — c'est le
   seul moyen de le savoir, personne ne l'a jamais verifiee pour toi.

## Corriger une fiche

Le repertoire est partage : une adresse fausse ou un nom mal orthographie sont
visibles de tous. Deux personnes peuvent corriger une fiche, et c'est le
serveur qui tranche :

- **l'administration**, sur n'importe quelle fiche, depuis `/admin` ;
- **l'utilisateur qui a propose une entreprise**, sur la sienne uniquement —
  un bouton *Modifier cette fiche* apparait alors sur sa fiche.

Les deux passent par le meme ecran et la meme route (`PATCH /api/companies/{id}`) :
deux formulaires auraient fini par diverger, et c'est toujours le plus
permissif qui aurait fait foi. L'adresse n'est relocalisee que si elle change,
pour qu'une correction de telephone ne fasse pas perdre a une fiche BioWin la
precision heritee du POC.

`/admin` ajoute la liste des comptes : date d'inscription, entreprises suivies,
fiches proposees, notification configuree, profil analyse. **En lecture seule
et sans le contenu** — ni CV, ni aspirations, ni identifiant Telegram : ces
donnees n'aident pas a moderer un repertoire, et les afficher ferait de la page
une cible.

Le droit d'administration ne s'accorde pas par HTTP. `python -m ananas.admin`
est la seule facon de le poser, donc depuis le serveur : une elevation de
privilege demande un acces machine, pas une faille dans un formulaire.

## Correspondance des offres avec ton profil

Un modele structure **une fois par offre** ses caracteristiques (metier,
seniorite, competences, secteur, langues, lieu) — c'est commun a tous les
utilisateurs. La comparaison a ton profil est ensuite du **Python
deterministe** : gratuite, stable d'une execution a l'autre, et capable de dire
*pourquoi* une offre te correspond.

Consequence pratique : tous les scores sont calcules d'avance, donc filtrer les
offres pertinentes d'une entreprise est instantane. Et passer a un modele local
ne demande de reecrire que l'extraction — voir `api/README.md`.

Le CV s'importe en PDF : le texte est extrait localement, et pour un PDF
scanne le modele prend le relais. Dans les deux cas le texte remplit le champ
pour que tu le relises avant l'analyse.

Configure `GEMINI_API_KEY` dans le `.env`. Sans cle, tout le reste fonctionne :
les offres s'affichent simplement sans score.

## Configuration du bot Telegram

Un seul bot cote serveur, partage par tous les utilisateurs. Cree-le aupres de
[@BotFather](https://t.me/BotFather) et colle le token dans `TELEGRAM_BOT_TOKEN`.
Sans token, l'application fonctionne : la page Parametres le signale et la
connexion renvoie une erreur explicite.

**Le token ne sort jamais du serveur.** C'est un secret partage : qui le detient
peut piloter le bot et ecrire a tous les comptes. La liaison d'un utilisateur
passe donc par un lien profond a usage unique — un bot Telegram ne pouvant pas
ecrire le premier, c'est l'utilisateur qui envoie `/start <code>`, et le serveur
seul lit le `chat.id`. Voir `api/ananas/telegram_link.py`.

## Tester sans attendre le lendemain

```bash
docker compose exec worker python -m ananas.worker.main --list-scrapers
docker compose exec worker python -m ananas.worker.main --scrape-only --company gsk-wavre
docker compose exec worker python -m ananas.worker.main --now
```

Relancer `--now` une seconde fois n'envoie **rien** : c'est la garantie
anti-doublon (`sent_job_notifications`).

## Etat

**Fait** — comptes, carte, favoris, proposition d'entreprises par les
utilisateurs (avec tags partages et geocodage), correction d'une fiche par son
auteur ou par l'administration, page d'administration (repertoire et comptes),
profil professionnel (correction manuelle du profil extrait comprise) et score
de correspondance des offres, domicile de
reference avec perimetre affiche sur la carte (filtre et tri par distance sur
la carte et dans la page Offres), connexion Telegram,
worker planifie, detection des nouvelles offres, notification triee par
pertinence, une page Offres qui montre soit ta veille soit tout le repertoire
scrape (avec suivi d'une entreprise en un clic depuis une offre, groupes
repliables, et possibilite d'ecarter une offre qui ne t'interesse pas), sept
scrapers reels — QbD Group (Teamtailor), GSK Wavre et Rixensart, UCB
Braine-l'Alleud et Anderlecht (Phenom People), Sanofi Belgium (Radancy),
AbbVie Belgium (Attrax) — 230 offres reelles.

**A faire par toi** — les autres scrapers. Voir `api/README.md`, section
*Ecrire un scraper* : une seule methode a implementer, le fichier depose est
detecte automatiquement.

**Plus tard** — un agent IA qui ecrit le scraper d'une entreprise proposee et
ouvre une pull request (la page carrieres est deja collectee par le
formulaire); d'autres canaux de notification (email, Discord); moderation des
propositions si le repertoire s'ouvre largement.

## Honnetete sur les donnees

Le geocodage herite du POC n'est pas parfait, et l'interface le signale plutot
que de faire semblant :

| `geo_precision` | n | signification |
|---|---|---|
| `adresse` | 62 | rue + numero trouves |
| `ville` | 59 | seule la commune a ete resolue |
| `approx. (nom)` | 4 | position deduite du nom, aucune adresse publiee |
| `non localise` | 5 | listees dans la sidebar, absentes de la carte |

Les 49 entreprises du repertoire Ananas n'ont pour l'instant que leur commune :
elles tombent toutes sur le point de celle-ci. La carte les eclate en eventail
autour de ce point pour qu'elles restent cliquables — dix se superposeraient
sinon a Leuven — mais la fiche continue d'afficher `ville`, et leur site web
comme leur page carrieres restent a renseigner.
