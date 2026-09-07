# RegWatch

Outil de veille réglementaire sur la transposition de **NIS 2** et de **REC** dans les
États membres de l'Union européenne.

RegWatch rassemble en un seul endroit l'état de transposition des 27 États membres (plus le
Royaume-Uni et la Norvège en comparateurs), les fiches pays détaillées, la chronologie des
événements réglementaires, les indicateurs de maturité et le registre des sources. Une
chaîne de collecte automatisée alimente une file de veille que l'équipe valide avant toute
publication.

Prototype interne Wavestone.

---

## Ouvrir l'outil

L'application tient dans **un seul fichier HTML**. Aucun serveur, aucune installation,
aucune connexion réseau ne sont nécessaires pour la consulter.

**Double-cliquez sur `regwatch.html`.** Le fichier peut aussi être copié sur une clé USB,
partagé dans Teams ou envoyé en pièce jointe : il fonctionne partout, tel quel.

Deux versions sont produites par la même construction :

| Version | Fichier | Taille | Destinataires |
|---|---|---|---|
| Client | `regwatch.html`, `index.html` | 1,6 Mo | ce qui est diffusé |
| Équipe | `regwatch-equipe.html` | 2,0 Mo | usage interne, rôle Développeur en plus |

Les validations effectuées dans l'outil sont enregistrées **dans le navigateur**
(localStorage). Chaque poste possède son propre état ; rien n'est partagé ni transmis.

---

## Ce que contient l'outil

* **Carte de maturité** interactive des 29 pays suivis, avec export PNG, en thème clair ou
  sombre.
* **Fiches pays** : statut de transposition, autorités compétentes, cadre applicable,
  chronologie des événements, sources citées.
* **File de veille** : chaque élément détecté par la chaîne de collecte, accompagné de la
  phrase de la source qui l'a déclenché. Rien ne rejoint une fiche pays sans validation.
* **Indicateurs et export Excel** natif, graphiques compris.
* **Assistant** conversationnel ancré sur les données de l'outil, qui cite ses sources.
* **Registre des sources** typées (officielles, presse, communautaires).
* **Bilingue** français et anglais, bascule immédiate.

Le classeur SharePoint reste la source de vérité. RegWatch en est un miroir en lecture :
les seules écritures possibles sont la validation d'un élément de veille et l'ajout d'une
source.

---

## Les rôles

Le sélecteur situé en haut à droite change ce qui est visible. Il ne protège rien : c'est un
confort de lecture, pas un contrôle d'accès. En production, cette fonction reviendrait au
SSO.

* **Lecteur** : les fiches pays et l'historique. La file de veille lui est masquée, afin que
  rien d'incertain ne lui soit montré.
* **Validateur** : voit chaque élément détecté par la chaîne de collecte, la phrase de la
  source qui l'a déclenché, et tranche. Aucune fiche ne bouge sans son accord.
* **Développeur** (version équipe uniquement) : un onglet supplémentaire, **Assistant
  technique**, qui répond sur l'architecture en lisant la documentation et le code
  embarqués.

L'onglet **Diagnostic**, qui affiche l'environnement du navigateur courant sans rien
transmettre, ne dépend pas du rôle. Il est replié derrière la roue crantée située à côté du
sélecteur et s'ouvre pour n'importe quel rôle. C'est l'information qu'on demande à une
personne qui signale un problème : l'obliger à changer de rôle pour la lire n'aurait pas de
sens. L'onglet reste absent de la barre tant que la roue n'a pas été pressée. La roue
n'existe que dans la version équipe.

---

## Publier sur GitHub Pages

Le dépôt contient un workflow prêt à l'emploi (`.github/workflows/static.yml`). Activez
Pages sur la branche par défaut et la publication devient automatique à chaque poussée.

Deux points à connaître avant de diffuser une URL.

**Pages sert tout l'arbre du dépôt.** Même si le dépôt est privé, le code source, les
fichiers de `data/` et les README répondent 200 à qui connaît l'URL. Seuls les fichiers non
versionnés, `.env` en tête, sont hors d'atteinte. Si le code doit rester confidentiel,
publiez depuis une branche ne contenant que les fichiers HTML.

**L'assistant exige que la nouvelle origine soit autorisée sur le proxy.** Le proxy Azure
n'accepte que les origines déclarées. Tant que l'URL Pages du nouveau dépôt n'y figure pas,
l'assistant répondra par une erreur alors que le reste de l'outil fonctionnera normalement.
Deux réglages sont nécessaires, décrits dans `azure-proxy/README.md` :

```sh
az functionapp config appsettings set -n regwatch-proxy -g rg-regwatch \
  --settings REGWATCH_ALLOWED_ORIGINS="https://<organisation>.github.io"

az functionapp cors add -n regwatch-proxy -g rg-regwatch \
  --allowed-origins "https://<organisation>.github.io"
```

Les emplacements où l'URL publique doit être renseignée portent tous le marqueur
`<organisation>.github.io`.

---

## Reconstruire

```sh
zsh src/build.sh
```

La commande produit les deux versions et les copie à la racine du dépôt. Cette copie fait
partie de la construction : il n'existe pas d'étape manuelle à ne pas oublier.

Le script exige zsh et se relance de lui-même s'il est appelé autrement. Ses listes de
fichiers sont des tableaux, dont bash ne prendrait que le premier élément, ce qui produisait
un fichier amputé que `node --check` validait sans rien signaler. La construction compare
donc les octets écrits à la somme des entrées et échoue si la concaténation est partielle.

Deux corpus doivent être régénérés lorsque leur source change. La construction les intègre
s'ils existent et s'en passe sinon :

```sh
python3 tools/build_devdocs.py   # documentation -> src/data_devdocs.js
python3 tools/build_devcode.py   # code source   -> src/data_devcode.js
```

---

## La chaîne de veille

La collecte s'exécute sur un poste, pas sur GitHub Pages, qui ne sert que des fichiers
statiques.

```sh
python3 -m venv venv
./venv/bin/pip install -r requirements.txt
cp .env.example .env        # puis renseigner les identifiants Azure OpenAI
```

Le fichier `.env` n'est jamais versionné. `python3 agent-veille/check_azure.py` vérifie la
configuration avant toute collecte.

```sh
python3 agent-veille/health.py     # état de la file, et ce qui a bougé depuis la référence
python3 agent-veille/collect.py    # une passe de collecte sur le registre de sources
python3 tools/veille_to_watchitems.py <classeur.xlsx>   # classeur vers la file de veille
zsh src/build.sh                   # intégrer le résultat dans les fichiers publiés
```

`health.py` signale le vieillissement de la file. C'est une surveillance utile : les
pourcentages affichés restent identiques pendant que la veille s'arrête, symptôme qui ne se
voit pas autrement.

`agent-veille/README.md` détaille l'organisation de la chaîne, et `tools/README-sync.md` la
synchronisation avec le classeur SharePoint.

**Une dépendance externe à connaître.** Le moteur de collecte d'origine, `nis2_agent_v2.py`,
vit dans un dépôt distinct, celui de son auteur, et ne figure pas ici. Ce dépôt porte tout ce
qui l'entoure et qui suffit à faire tourner une collecte : le registre de sources,
`collect.py` qui est autonome, la conversion du classeur vers la file de veille, le routage
des champs, le score de fiabilité et les mesures. Les fichiers `regwatch_fields.py` et
`regwatch_push.py` sont destinés à être copiés à côté du moteur d'origine pour le brancher
sur la file de validation ; leurs en-têtes indiquent les points d'accroche.

---

## Assistant et proxy

Les deux assistants, métier et technique, passent par un proxy Azure Functions qui porte la
clé du cabinet. Cette clé n'est **jamais** présente dans la page publiée, qui est par nature
téléchargeable.

Conséquence à connaître : le proxy n'accepte que l'origine publiée, donc **les assistants ne
fonctionnent pas depuis un fichier ouvert en local**. Le message affiché explique alors
comment lancer un proxy local (`python3 tools/chat_proxy.py`). Tout le reste de l'outil
fonctionne hors ligne.

`azure-proxy/README.md` décrit le déploiement, ainsi que ce qui protège réellement
l'endpoint et ce qui n'en donne que l'apparence.

---

## Organisation du dépôt

| Chemin | Contenu |
|---|---|
| `regwatch.html`, `index.html` | la version client, à distribuer telle quelle |
| `regwatch-equipe.html` | la version équipe |
| `src/` | les sources et le script de construction |
| `docs/` | cahier des charges et schémas d'architecture |
| `tools/` | conversion des classeurs, indicateurs, export Excel, comparaisons |
| `agent-veille/` | la chaîne de collecte |
| `azure-proxy/` | le proxy portant la clé du cabinet |
| `data/` | états intermédiaires versionnés : file de veille, caches, mesures |
| `requirements.txt` | dépendances Python de la collecte et des outils |

Dans `src/` :

* `shell_top.html` : squelette de page et feuille de style, conforme à la charte Wavestone.
* `app_*.js` : l'application, soit le routage, la carte, les fiches, la file de veille, les
  indicateurs et les assistants.
* `reg/nis2/`, `reg/rec/` : les données propres à chaque réglementation. Ajouter DORA revient
  à ajouter un dossier et une ligne dans `build.sh`.
* `map_data.js` : fond de carte, généré par `convert_map.py`.
* `build.sh` : la construction.

---

## Charte graphique

Violet `#451DC7` pour les titres et l'accent principal, vert `#04F06A` en accentuation
limitée à 5 % de la surface et jamais sous du texte blanc, `#250F6B` en accent profond.
Couleurs fonctionnelles : `#4682B4` pour les infographies, `#FFCA4A` et `#FF2A49` pour les
statuts. Police Aptos, repli Segoe UI.

Les palettes de la carte et des graphiques sont validées en contraste et en lisibilité pour
les daltonismes, en thème clair comme en thème sombre.

---

## Périmètre et limites

Il s'agit d'un **prototype** destiné à démontrer l'usage et à obtenir un accord interne.

* Les validations sont enregistrées localement dans le navigateur, sans persistance partagée
  ni backend.
* Le sélecteur de rôle est un confort de lecture, pas un contrôle d'accès.
* L'édition manuelle des fiches a été retirée volontairement : le classeur SharePoint est la
  source de vérité unique, et deux surfaces d'écriture pour une même donnée produisaient des
  divergences que rien n'arbitrait.
* Les notifications Teams, les exports PowerPoint et l'intégration complète de la chaîne de
  collecte relèvent des versions suivantes.

`docs/cahier-des-charges.md` présente le produit cible, l'architecture de production
envisagée et une couverture point par point de ce que le prototype livre réellement.

---

## Développement assisté par IA

Une partie du code et de la documentation de ce dépôt a été produite avec l'aide d'outils
d'assistance par intelligence artificielle. L'ensemble a été relu, testé et validé par
l'équipe. Les données réglementaires proviennent des sources officielles citées dans l'outil
et dans le registre des sources.
