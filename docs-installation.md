# Recherche d'emploi automatique avec n8n — guide d'installation

Chaque jour à 14h, ce workflow :

1. lit ton CV (un Google Doc) ; Gemini en déduit ton métier, ton niveau, ton pays et les mots-clés de recherche ;
2. cherche les offres de la semaine avec **JSearch** (Google for Jobs : LinkedIn, Indeed, sites locaux…) ;
3. filtre les offres hors sujet, trop seniors, les stages et celles déjà envoyées ;
4. fait noter chaque offre sur 100 par Gemini par rapport à ton CV ;
5. pour les meilleures, crée une copie de ton CV adaptée à l'offre (réordonnée, jamais inventée) ;
6. ajoute une ligne dans ton Google Sheet et t'envoie un email avec les liens.

**Il ne postule jamais à ta place** : tu relis le CV et tu postules toi-même.

Tout est gratuit (n8n en local, Gemini gratuit, JSearch gratuit jusqu'à 200 requêtes/mois ; le workflow en utilise ~120).

Compte environ 1 h 30 pour l'installation.

---

## 1. Installer n8n (Docker)

```bash
docker volume create n8n_data
docker run -d --name n8n --restart unless-stopped -p 5678:5678 \
  -e GENERIC_TIMEZONE=Africa/Casablanca -e TZ=Africa/Casablanca \
  -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

Ouvre http://localhost:5678 et crée ton compte. Si tu n'es pas au Maroc, remplace `Africa/Casablanca` par ton fuseau (ex. `Europe/Paris`).

## 2. Importer le workflow

Dans n8n : **Create workflow → ⋯ (en haut à droite) → Import from File** → choisis `recherche-emploi-automatique.json`.

## 3. Préparer ton modèle de CV (le plus important)

1. Mets ton CV dans Google Drive et ouvre-le avec Google Docs. Si c'est un .docx : **Fichier → Enregistrer au format Google Docs** (l'API ne lit pas les .docx).
2. Dans ce Google Doc, entoure chaque morceau que Gemini a le droit d'adapter avec un **marqueur** :

   ```
   {{CLÉ: le texte de ton CV}}
   ```

   - CLÉ en MAJUSCULES, chiffres et `_` uniquement, puis `:` et un espace.
   - Un marqueur reste dans **un seul paragraphe** (pas de retour à la ligne dedans).
   - Ne touche pas au reste (nom, contact, dates, titres de sections, photo) : ça ne change jamais.

3. Le nom de la clé dit à Gemini ce qu'il peut faire :

   | Clé | Ce que Gemini peut faire |
   |---|---|
   | `TITRE` | réordonner les mots-clés du titre selon l'offre |
   | `PROFIL` | réordonner les phrases du profil, mettre en avant ce que l'offre demande |
   | `COMP_...` (ex. `COMP_CLOUD`) | liste séparée par des virgules : **seulement changer l'ordre** |
   | `NOM_1`, `NOM_2`… (ex. `ORANGE_1`, `ORANGE_2`) | puces d'une même expérience : les réordonner, reformuler légèrement |
   | `PROJET_1`, `PROJET_2`… | réordonner les projets, texte inchangé |
   | autre clé | ne change rien |

   Exemple :

   ```
   {{TITRE: INGÉNIEUR DEVOPS | KUBERNETES | CI/CD}}
   {{PROFIL: Ingénieur DevOps junior avec un stage chez X. J'automatise les déploiements avec GitLab CI/CD.}}
   Compétences : {{COMP_OUTILS: Docker, Kubernetes, GitLab CI/CD, Terraform}}
   • {{ORANGE_1: Mise en place de pipelines GitLab CI/CD pour 5 applications.}}
   • {{ORANGE_2: Déploiement d'applications sur Kubernetes avec Helm.}}
   ```

4. Garde le lien de ce Google Doc.

**Sécurité anti-invention :** si Gemini écrit dans un champ un mot qui n'existe nulle part dans ton CV, le workflow garde le texte original de ce champ. Ton CV adapté ne contient donc jamais d'outil ou d'expérience que tu n'as pas.

## 4. Créer ton Google Sheet

Crée une feuille Google Sheets **vide**, renomme l'onglet du bas en **`Offres`**. Les en-têtes sont créés automatiquement au premier envoi. Garde le lien.

## 5. Créer les identifiants

Dans n8n : **Overview → Credentials → Create credential**. Ensuite, dans chaque nœud marqué « ⚠️ Choisis ton identifiant ici », sélectionne le bon identifiant.

### Gemini (gratuit)
1. https://aistudio.google.com/apikey → **Create API key**.
2. n8n : credential **Google Gemini (PaLM) API** → colle la clé.
3. Nœuds : `Gemini (analyse)`, `Gemini (note)`, `Gemini (CV)`.

### JSearch (gratuit, 200 requêtes/mois)
1. Crée un compte sur https://rapidapi.com, cherche **JSearch** (éditeur OpenWeb Ninja) → **Subscribe** au plan **Basic (gratuit)**.
2. Copie ta clé `X-RapidAPI-Key`.
3. n8n : credential **Header Auth** → Name : `X-RapidAPI-Key`, Value : ta clé.
4. Nœud : `JSearch`.

### Google (Docs, Drive, Sheets, Gmail)
1. https://console.cloud.google.com → crée un projet.
2. **APIs & Services → Library** : active **Google Docs API**, **Google Drive API**, **Google Sheets API**, **Gmail API**.
3. **OAuth consent screen** : type *External*, remplis le nom et ton email, ajoute ton email dans *Test users*.
4. **Credentials → Create credentials → OAuth client ID** → *Web application*. Dans *Authorized redirect URIs*, colle l'URL affichée par n8n quand tu crées le credential (en général `http://localhost:5678/rest/oauth2-credential/callback`).
5. Dans n8n, crée 4 credentials avec le Client ID et le Client Secret : **Google Docs OAuth2 API**, **Google Drive OAuth2 API**, **Google Sheets OAuth2 API**, **Gmail OAuth2 API** → **Sign in with Google** pour chacun.
6. Nœuds :
   - Google Docs → `Lire le modèle de CV`, `Remplir le CV` (nœuds HTTP Request : *Authentication = Predefined Credential Type → Google Docs OAuth2 API*) ;
   - Google Drive → `Copier le modèle` ;
   - Google Sheets → `Lire le Sheet`, `Ajouter au Sheet` ;
   - Gmail → `Envoyer l'email`.
7. **Important :** quand tout marche, retourne dans *OAuth consent screen → Audience → Publish app* (« In production »). En mode *Testing*, Google coupe l'accès tous les 7 jours et le workflow s'arrête.

## 6. Remplir le nœud ⚙️ Paramètres

Ouvre le nœud **Paramètres** et remplace :

- `email` → ton adresse ;
- `modeleCV` → le lien de ton Google Doc (étape 3) ;
- `sheet` → le lien de ton Google Sheet (étape 4).

Optionnel : `scoreMinimum` (60 par défaut), `offresParJour` (3), `motsCles` et `pays` si tu veux forcer ce que Gemini a déduit de ton CV.

## 7. Tester puis activer

1. Clique sur **Execute workflow**. Compte 2 à 4 minutes (pause de 10 s entre deux appels Gemini pour respecter la limite gratuite).
2. Clique sur le nœud **Profil du CV** : vérifie le métier, le niveau, le pays et les mots-clés trouvés.
3. Si une offre passe le score minimum : un CV `CV_<prénom>_<entreprise>_<poste>` apparaît dans ton Drive, une ligne dans le Sheet et tu reçois l'email. **Relis toujours le CV avant d'envoyer.**
4. Clique sur **Publish** : le workflow tourne tous les jours à 14h.

## Bon à savoir

- **PC éteint à 14h ?** Le workflow se rattrape tout seul au prochain démarrage de n8n (une seule fois par jour). Si tu changes l'heure du nœud « Programmation », change aussi `HEURE_PREVUE` dans le nœud « Déjà lancé aujourd'hui ? ».
- **Pas d'email un jour ?** Normal si aucune offre n'atteint le score minimum. Regarde le nœud **Lire le score** dans l'exécution pour voir les notes.
- **Erreur « Aucun marqueur {{CLÉ: texte}} trouvé »** : le lien du modèle est faux, ou les marqueurs sont mal écrits (il faut `{{CLÉ: texte}}` avec deux-points).
- **Erreur Gemini 429 / 503** : limite gratuite ou surcharge ; le workflow réessaie 3 fois. Baisse `offresAnalysees` si ça revient souvent.
- Le CV adapté est surtout **réordonné** (le plus pertinent pour l'offre passe en premier). C'est voulu : rien n'est inventé.
