# 🎱 CueScore Explorer - Guide & Documentation Technique

**CueScore Explorer** est une application web autonome (*single-file*) développée en React, Tailwind CSS et JavaScript. Elle permet de consulter, filtrer et suivre en direct les tournois et matchs de billard hébergés sur la plateforme **CueScore** (ex: *Championnat Bretagne DR4C*), à partir de l'API officielle de CueScore.

## 1. Description Générale

L'application a été conçue pour offrir aux joueurs, capitaines d'équipe et supporters une interface claire, réactive et optimisée sur mobile comme sur ordinateur.

Plutôt que de naviguer sur l'interface parfois complexe du site officiel, **CueScore Explorer** synthétise les données essentielles :

* **L'agenda et les heures de convocations** (converties en format et fuseau horaire français).

* **La localisation précise des matchs** (nom de la salle, adresse physique, ville et numéro de table d'arbitrage).

* **Le filtrage intelligent par équipe et favori** (mémorisation de votre club ou équipe de cœur).

* **Les résultats et scores en direct**.

## 2. Explication des Composants & Fonctionnalités

### 2.1. Barre de Sélection du Tournoi

Située en haut de la page, cette section permet de définir la compétition à charger :

* **Liste déroulante des tournois pré-configurés** : Permet de basculer instantanément entre des tournois enregistrés (par exemple le *Championnat Bretagne DR4C 2026/2027* avec l'ID `#88770526`).

* **Saisie d'un ID CueScore sur-mesure** : Un champ texte permet de saisir n'importe quel ID de tournoi CueScore pour l'interroger à la volée.

* **Réinitialisation automatique** : Chaque changement de tournoi réinitialise automatiquement les filtres de recherche actifs afin d'éviter tout conflit de données.

### 2.2. Indicateurs de Statut API

L'application communique directement avec les serveurs CueScore via des requêtes HTTP :

* **Indicateur de chargement** : Affiche une animation visuelle pendant le rafraîchissement des données.

* **Gestionnaire d'erreurs** : Si un ID est inexistant ou si l'API est temporairement indisponible, un message explicatif s'affiche en rouge sans faire planter l'application.

### 2.3. Bannière du Tournoi & Métriques Globales

Une fois les données chargées, la bannière présente un résumé haut de niveau :

* Intitulé du tournoi, discipline (*8-Ball, 9-Ball, Blackball, etc.*), nom de l'organisateur et lieu principal.

* **Cartes statistiques** : Compteurs temps réel indiquant le nombre total de matchs, les matchs à venir, ceux en cours (*LIVE*) et ceux déjà terminés.

### 2.4. Panneau de Filtrage Avancé

Ce composant offre 5 niveaux de filtres combinables :

1. **Équipe / Club** : Filtre la liste pour n'afficher que les matchs impliquant l'équipe sélectionnée (`playerA` ou `playerB`).

2. **Gestion de l'Équipe Favorite (⭐)** :

   * Un bouton d'étoile permet de marquer l'équipe sélectionnée comme **favorite**.

   * L'équipe favorite est sauvegardée dans le navigateur (`localStorage`).

   * Au chargement du tournoi, l'application applique automatiquement le filtre sur cette équipe.

   * Un bouton rapide *"Mon équipe : \[Nom\]"* permet de ré-appliquer le filtre en un clic.

3. **Lieu / Salle** : Permet d'isoler les rencontres se déroulant dans un club ou une salle spécifique.

4. **Joueur** : Filtre les matchs attribués à un joueur individuel.

5. **Statut du Match** : Filtre selon l'état du match (*Tous*, *À venir*, *En cours*, *Terminés*).

6. **Recherche libre** : Recherche textuelle instantanée dans le nom des équipes, des joueurs, des salles, des adresses ou des tours.

### 2.5. Modes d'Affichage (Grille vs Chronologique)

* **Mode Grille (Défaut)** : Affiche les rencontres sous forme de cartes d'informations rectangulaires, idéales pour repérer rapidement les équipes et les lieux.

* **Mode Chronologique (Timeline)** : Organise les matchs le long d'une ligne du temps verticale, facilitant le suivi du déroulement de la journée de compétition.

### 2.6. Modale "Fiche du Match & Convocation"

En cliquant sur le bouton **"Détails"** d'un match :

* Une fenêtre surgissante affiche l'ensemble des détails du match.

* **Date & Heure exacte de convocation**.

* **Lieu & Table attribuée**.

* **Bouton Google Maps** : Ouvre directement le GPS/Google Maps configuré avec l'adresse physique de la salle.

* **Lien Direct CueScore** : Permet d'ouvrir la page officielle du match (`https://cuescore.com/match/?matchId=...`).

## 3. Architecture Technique pour les Développeurs

### 3.1. Stack Technique

* **Format** : Fichier unique `index.html` (Application web monopage autonome / Single Page Application).

* **React 18** : Librairie UI chargée via CDN (`umd/react.production.min.js`).

* **Babel Standalone** : Transpilation JSX à la volée dans le navigateur (`@babel/standalone`).

* **Tailwind CSS** : Framework CSS utilitaire chargé via CDN script.

* **FontAwesome 6** : Kit d'icônes vectorielles.

### 3.2. Flux de Données & Gestion du CORS (Cross-Origin Resource Sharing)

L'API officielle de CueScore (`https://api.cuescore.com/tournament/?id=...`) peut bloquer les requêtes `fetch()` en provenance de navigateurs web externes en raison des politiques de sécurité Same-Origin policy (CORS).

Pour garantir un fonctionnement 100% fiable en toutes circonstances (en local ou hébergé), la fonction `loadTournamentById()` intègre un **fallback multi-proxy cascade** :

```
const targetUrl = `https://api.cuescore.com/tournament/?id=${cleanId}`;
const fetchUrls = [
  targetUrl,                                                  // 1. Essai direct HTTP/HTTPS
  `https://corsproxy.io/?${encodeURIComponent(targetUrl)}`,    // 2. Proxy de secours N°1
  `https://api.allorigins.win/raw?url=${encodeURIComponent(targetUrl)}` // 3. Proxy de secours N°2
];

```

L'application essaie chaque source séquentiellement jusqu'à obtenir un résultat JSON valide, avec un délai d'expiration (*timeout*) de 5 secondes par tentative via `AbortController`.

### 3.3. Parsing & Normalisation des Données (`parseCueScoreJson`)

L'API CueScore renvoie des structures JSON qui peuvent varier selon le type de tournoi (tableau individuel, championnat par équipe, phases de poules).

La fonction `parseCueScoreJson()` normalise ces variations vers un modèle objet unifié :

```
interface NormalizedMatch {
  id: string;
  cuescoreUrl: string;
  roundName: string;
  starttime: string;
  table: {
    id: string | null;
    name: string;
    venue: {
      id: string | null;
      name: string;
      placename: string;
      address: string;
    }
  };
  playerA: { id: string | null; name: string; url: string; player: string };
  playerB: { id: string | null; name: string; url: string; player: string };
  scoreA: number;
  scoreB: number;
  finished: boolean;
  status: 'upcoming' | 'running' | 'finished';
}

```

#### Points clés du parser :

* **Identification des équipes** : Extraction prioritaire des sous-objets `match.playerA` et `match.playerB` (`name`, `teamId`, `url`).

* **Localisation** : Inspection profonde de `match.table.venue` (`name`, `placename`, `address`).

* **Génération des URLs d'équipe** : Si l'API ne fournit pas d'URL explicite pour une équipe, l'application la reconstruit via la structure `https://cuescore.com/team/{Nom}/{TeamId}`.

### 3.4. Gestion des États & Persistance (Hooks React)

* `currentTournamentId` : Identifiant du tournoi actif.

* `favTeam` : Équipe favorite stockée dans le `localStorage` sous la clé `'cuescore_fav_team'`.

* `selectedTeam` : Filtre d'équipe actuellement appliqué.

* **Résolution tolérante des noms d'équipes (Fuzzy Matching)** :
  Au chargement de nouvelles données, un `useEffect` compare le nom de l'équipe favorite (`favTeam`) avec la liste des équipes réellement présentes dans le tournoi pour ajuster automatiquement le filtre, même en cas de légère différence de frappe ou de casse.

### 3.5. Formatting & Localisation (`formatFrenchDateTime`)

Les dates fournies au format ISO ou UTC par l'API sont formatées en français standard via l'API native `Intl.DateTimeFormat` / `toLocaleDateString('fr-FR')` :

* Format Date : *"Samedi 10 octobre 2026"*

* Format Heure : *"14h30"*

## 4. Hébergement et Déploiement

L'application étant entièrement contenue dans le fichier `index.html`, elle ne nécessite **aucun serveur Node.js, compilation préalable ou base de données**.

### Options d'Hébergement Gratuit :

1. **GitHub Pages** : Déposez le fichier `index.html` sur un dépôt GitHub public et activez *GitHub Pages* dans les options.

2. **Netlify Drop / Vercel** : Glissez-déposez le fichier sur [app.netlify.com/drop](https://app.netlify.com/drop) pour obtenir un lien sécurisé HTTPS en quelques secondes.