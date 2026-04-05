# Documentation du Workflow n8n : Scraper d'Offres d'Emploi (Welcome to the Jungle)

## Description du Projet
Ce workflow n8n est une solution d'automatisation complète conçue pour extraire quotidiennement des offres d'emploi ciblées (Alternance Développeur à Paris) depuis la plateforme Welcome to the Jungle. 

Il intègre un système d'extraction par requêtes HTTP directes sur l'API de recherche (Algolia), un mécanisme de déduplication strict basé sur Google Sheets, et un système de notification Discord optimisé pour respecter les limites de requêtes (rate limiting) des API externes.

## Genèse et Choix Techniques
Ce flux de travail est le résultat d'une itération technique visant à maximiser la pertinence et la qualité des données extraites.

Dans sa première phase, le système a été développé sous la forme d'un Proof of Concept (POC) s'appuyant sur l'agrégateur Google Jobs. L'objectif initial était de valider la logique d'automatisation, de déduplication et de notification. Cependant, l'analyse des premiers résultats a mis en évidence une qualification insuffisante des offres pour le secteur informatique, notamment en raison de la présence d'annonces redondantes, obsolètes ou émanant d'intermédiaires non pertinents.

L'analyse des sources disponibles a permis de conclure que la donnée présente sur la plateforme Welcome to the Jungle était beaucoup plus qualifiée, structurée et adaptée aux standards de l'écosystème Tech.

La plateforme ne proposant pas d'API publique ouverte aux développeurs externes, un pivot architectural a été opéré. Il a fallu procéder au reverse-engineering du moteur de recherche interne du site (propulsé par Algolia). Cette démarche a impliqué :
* L'analyse du trafic réseau et l'interception des requêtes XHR/Fetch.
* L'identification des clés d'API internes et des identifiants d'application.
* La reconstitution du Payload JSON pour manipuler les paramètres de pagination et forcer le renvoi de 50 résultats par appel.
* Le contournement des protections basiques côté serveur par la manipulation des en-têtes HTTP (forçage du User-Agent, Origin et Referer).

**Note sur le partage des clés :** Les identifiants Algolia fournis dans le nœud HTTP sont des clés publiques de type "Search-Only" extraites du front-end de Welcome to the Jungle. Elles sont volontairement laissées en clair dans le workflow afin de proposer une architecture "plug-and-play" et directement testable par la communauté, sans présenter de risque de sécurité (droits de lecture seule).

Cette approche globale garantit aujourd'hui l'extraction d'un flux de données hautement qualifié, directement à la source, sans dépendance à un service de scraping tiers ou payant.

## Architecture et Flux de Données
Le workflow fonctionne selon une architecture en deux branches parallèles après l'étape de filtrage des doublons. Voici le rôle de chaque nœud :

1. **Schedule Trigger** : Point d'entrée du workflow. Conçu pour s'exécuter à intervalle régulier (ex: tous les matins). Il déclenche simultanément la requête de scraping et la lecture de la base de données.
2. **HTTP Request (Scraper WTTJ)** : Effectue une requête POST vers les serveurs Algolia de Welcome to the Jungle. Le payload est configuré pour extraire jusqu'à 50 résultats par page. Les en-têtes (Headers) incluent un faux User-Agent et un Referer pour passer les sécurités basiques.
3. **Split Out** : Isole la grappe de données utile. Extrait le tableau d'objets situé dans le chemin `results[0].hits` pour permettre le traitement individuel de chaque annonce.
4. **Edit Fields (Traducteur)** : Normalise les données brutes de l'API en un format standardisé prêt à être exporté. Les champs générés sont : `id`, `title`, `company_name`, `location`, et la reconstitution dynamique de l'URL `share_link`.
5. **Google Sheets (Get rows)** : Récupère l'intégralité des identifiants des offres déjà traitées lors des exécutions précédentes.
6. **Compare Datasets** : Le cœur du système anti-doublons. Compare la clé `id` (source WTTJ) avec la colonne `ID` (source Google Sheets). Seules les nouvelles annonces ressortent par le port "In A Only".

**Traitement des nouvelles annonces (En parallèle) :**
* **Branche Base de données** : Les nouvelles annonces passent par un nœud de pause (Wait) pour s'assurer de la synchronisation, puis sont envoyées au nœud **Google Sheets (Append row)** qui réalise une insertion de masse (Bulk Insert) dans le tableur.
* **Branche Notification** : Les annonces entrent dans un **Loop Over Items** (taille de lot de 1). Chaque annonce passe par un **Wait** (délai d'attente) avant d'être envoyée au webhook **Discord**. Cette boucle garantit que l'API Discord ne bloquera pas les envois massifs via une erreur 429 (Too Many Requests).

## Prérequis et Configuration
Pour exécuter ce workflow correctement, vous devez disposer des éléments suivants :

### 1. Structure du fichier Google Sheets
Créez un fichier Google Sheets et nommez la première feuille "Feuille 1" (ou mettez à jour le nœud de destination). Le tableau doit impérativement contenir les entêtes de colonnes exacts suivants à la première ligne :
* `ID`
* `TITRE`
* `ENTREPRISE`
* `LIEN`

### 2. Identifiants et Connexions (Credentials)
* **Google Sheets OAuth2 API** : Nécessite une connexion active avec les permissions de lecture et d'écriture sur le Google Drive cible.
* **Discord Webhook** : Nécessite une URL de Webhook générée depuis les paramètres d'intégration d'un canal Discord.

## Déploiement
1. Dans votre instance n8n, créez un nouveau workflow.
2. Ouvrez le menu des options en haut à droite et sélectionnez "Import from File" ou copiez-collez directement le contenu du fichier JSON fourni sur le canevas vierge.
3. Double-cliquez sur les nœuds **Google Sheets** (Get et Append) pour relier vos "Credentials" et sélectionner votre propre document (Document ID).
4. Double-cliquez sur le nœud **Discord** pour sélectionner vos identifiants Webhook.
5. Modifiez le nœud **Schedule Trigger** selon l'heure ou la fréquence de déclenchement souhaitée.
6. Activez le workflow à l'aide du commutateur "Active" en haut à droite.

## Maintenance et Personnalisation
* **Modifier les critères de recherche** : Pour changer de poste ou de ville, il faut modifier la valeur de la requête URL-encodée (`query=alternance%20développeur%20Paris`) située dans le champ JSON du paramètre `jsonBody` du nœud HTTP Request.
* **Ajuster la limite Discord** : Si les notifications s'affichent trop lentement, vous pouvez réduire le délai dans le nœud **Wait** situé dans la boucle (actuellement configuré à 10 secondes ou équivalent). Veillez à ne pas descendre sous la limite stricte de l'API Discord.

## Axes d'Amélioration (Roadmap)
Afin de faire évoluer ce projet, plusieurs pistes d'optimisation peuvent être envisagées :

* **Gestion des erreurs (Error Handling)** : Implémentation d'un nœud *Error Trigger* global. En cas de modification inopinée de l'API cible (changement de la structure JSON) ou d'échec de la requête HTTP, le workflow enverrait une alerte technique sur un canal Discord d'administration.
* **Enrichissement des données (Data Enrichment)** : Ajout d'une étape de scraping sur le lien de l'offre finale pour récupérer le texte complet de l'annonce, couplée à une analyse par un modèle de langage (LLM) pour extraire automatiquement les technologies requises (stack technique) et le salaire mentionné.
* **Requêtage dynamique** : Remplacement de la requête statique dans le payload par des variables dynamiques. Le workflow pourrait lire une liste de mots-clés et de villes depuis un fichier de configuration pour effectuer plusieurs recherches successives.
* **Purge automatisée de la base de données** : Création d'un workflow secondaire s'exécutant hebdomadairement pour vérifier le statut HTTP des liens stockés dans le tableur. Les annonces renvoyant une erreur 404 (offre pourvue ou supprimée) seraient automatiquement effacées afin de conserver une base de prospection propre.