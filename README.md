# Cartographe

Extension Firefox qui cartographie une **trajectoire d'investigation** dans la
littérature scientifique.

Le nœud du graphe est une **requête**, pas un article.

---

## Pourquoi

Une douzaine d'outils cartographient déjà les articles et leurs citations —
Connected Papers, Research Rabbit, Litmaps, Inciteful, Argo Scholar. Ils
répondent à la question « qu'est-ce qui est proche de cet article ? ».

Cartographe répond à une autre question : **par où êtes-vous passé ?**

Il enregistre chaque requête comme un événement daté et immuable, puis relie ces
événements entre eux : les bifurcations, les impasses, les reformulations, et
les moments où deux pistes lancées à plusieurs semaines d'écart convergent sur
le même article sans que vous l'ayez remarqué.

Le corpus n'est pas le sujet. Le chemin l'est.

## Deux partis pris

### 1. Un nœud est un acte, pas un concept

Aucune fusion de nœuds, jamais. Une même chaîne tapée deux fois à trois semaines
d'écart produit **deux nœuds** — c'est le motif le plus intéressant que l'outil
puisse révéler : le retour sur ses pas.

Deux couches strictement séparées :

- **couche événement** — canonique, immuable, append-only. La chaîne brute est
  conservée telle que tapée, jamais réécrite. C'est ce qui s'exporte et se
  rejoue.
- **couche vue** — dérivée, recalculable, jetable. Les événements partageant une
  même *clé de forme* s'affichent repliés en un nœud unique avec compteur
  (`×4`), dépliable. Lisibilité sans perte d'information.

La clé de forme combine la chaîne normalisée, les filtres et les sources.
Relancer la même requête avec une borne d'année différente est un acte
différent.

**La normalisation est délibérément faible** — minuscules, accents supprimés,
ponctuation retirée, tokens triés. Et rien d'autre :

| Interdit | Raison |
|---|---|
| stemming | `retraction` → `retractions` n'est pas un mouvement intellectuel. `misconduct` → `fabrication` en est un. Une normalisation agressive efface le second en même temps que le premier. |
| suppression de mots vides | même raison |
| synonymes, embeddings, seuils de similarité | c'est le point exact où l'outil commence à interpréter — et interpréter, ici, revient à mentir sur la trajectoire |

La parenté sémantique existe, mais comme **arête typée et filtrable**, jamais
comme fusion. Ces interdits sont couverts par des tests qui échouent si
quelqu'un les contourne.

### 2. La couverture des données est affichée, pas masquée

Les listes de références ouvertes ont des trous **non aléatoires** : ils se
concentrent sur les publications anciennes, les sciences humaines, le
non-anglophone et les éditeurs restés fermés. Un graphe qui ne le signale pas
est une carte trompeuse — deux requêtes apparaîtront non voisines alors
qu'elles le sont.

Cartographe affiche donc sur chaque nœud d'article un ratio **« X connues /
Y déclarées »**, et voile les zones à faible couverture.

Trois états distincts, jamais confondus :

- **mesurée** — opacité proportionnelle au ratio ;
- **non mesurée** — trame hachurée. Ni pleine ni vide : tant que la mesure n'est
  pas faite, l'absence d'arête ne prouve rien ;
- **indéterminable** — aucune source ne déclare de nombre de références.

La même règle s'applique aux **versions**. Un article est signalé `preprint`,
`version publiée`, ou `aucune relation déclarée` — jamais « autonome ». Les
relations de version sont déposées par les éditeurs et le sont très
inégalement : leur absence signifie que personne n'a déclaré de contrepartie,
pas qu'il n'en existe pas. Quand la contrepartie est déjà dans le corpus local,
une arête `version_of` la relie ; sinon, son DOI est résolu jusqu'au titre pour
rester exploitable.

Y n'est directement observable nulle part : le nombre réel de références d'un
article n'existe que dans son texte. On croise trois estimateurs, tous biaisés
à la baisse (Semantic Scholar, Crossref `reference-count`, OpenCitations), et
on retient le plus haut. **Le ratio affiché est donc une borne supérieure de la
couverture réelle** — on peut sous-estimer le trou, jamais le surestimer. Cette
asymétrie remonte jusqu'à l'interface et jusqu'aux exports.

## Installation

Aucune version publiée sur AMO pour l'instant.

**Depuis l'archive** — ouvrez `about:debugging#/runtime/this-firefox`,
« Charger un module complémentaire temporaire », sélectionnez
`cartographe-0.1.0.zip`. L'extension disparaît au redémarrage de Firefox.

**Depuis les sources** :

```bash
npm install
npm start          # compile et ouvre Firefox avec l'extension chargée
```

Firefox 140 ou plus récent. Pas de version Chrome, pas de portage prévu.

## Utilisation

Ouvrez le panneau latéral, tapez une requête, `Émettre`.

- Une requête **soumise et ayant retourné des résultats** crée un nœud. Un
  résultat vide n'en crée aucun, et le dit.
- Cliquez un nœud de requête pour voir ses émissions successives et ses voisines
  par recouvrement de résultats.
- Cliquez un nœud d'article pour `Mesurer la couverture` ou
  `Vérifier rétractation et version` — opérations à la demande, jamais automatiques.
- `Exporter` → `Restaurer un export natif…` recharge une sauvegarde ;
  `Tout effacer…` vide le graphe pour repartir de zéro. Voir *Sauvegarde,
  restauration et remise à zéro*.
- Les cases à cocher filtrent par type d'arête. Le chiffre à droite est le rang
  de fiabilité : `1` = factuel, `3` = décoratif.

### Types d'arêtes

| Type | Rang | Calcul |
|---|---|---|
| recouvrement de résultats | 1 | local, zéro appel réseau. L'ossature du graphe. |
| reformulation | 1 | local, lexical. Dirigé et chronologique. |
| appartenance requête → article | 1 | local |
| couplage bibliographique | 2 | **à la demande uniquement.** Deux requêtes pointant vers la même littérature amont sans partager de résultat. |
| citation | 3 | factuel, peu coûteux |
| preprint → publié | 3 | déclaré par l'éditeur chez Crossref, donc factuel — pas inféré |

## Sources de données et clés

Aucune source payante n'est requise. Tout fonctionne sans clé, mais mal — voyez
l'avertissement plus bas.

| Source | Rôle | Clé |
|---|---|---|
| **Semantic Scholar** | recherche, métadonnées, références | facultative, **fortement recommandée** |
| **Crossref** | rétractations (Retraction Watch), relations preprint ↔ version publiée | aucune |
| **OpenCitations** | références et citations ouvertes par DOI | jeton gratuit, facultatif |
| **OpenAlex** | enrichissement d'un nœud isolé | **obligatoire** depuis février 2026 |

Les clés se saisissent dans le panneau, section `Clés`, et restent en local dans
`browser.storage.local`. L'extension ne transmet rien à personne d'autre que ces
quatre API.

> **Sans clé Semantic Scholar, il n'existe aucun quota par utilisateur.** La
> documentation officielle est explicite : les appels anonymes partagent un pool
> unique entre tous les usagers du monde, susceptible d'être étranglé en période
> de charge. Ça tient très bien pour un usage personnel, et se dégraderait à
> l'échelle si l'extension était largement installée. Une clé donne 1 requête par
> seconde dédiée et s'obtient gratuitement.

Les appels sont mis en file par hôte, avec backoff exponentiel et respect de
`Retry-After`. Chaque geste utilisateur dispose d'un **budget d'appels borné**
(3 pour une recherche, 4 pour une mesure de couverture, 40 pour un couplage) :
le coût d'une action est prévisible, il ne croît pas avec la taille du graphe.

Un cache IndexedDB garantit qu'un nœud déjà résolu n'est jamais réinterrogé. Ce
cache est aussi le corpus local persistant : rien ne part sur un serveur, tout
reste dans le navigateur.

### Note sur OpenAlex

La grille tarifaire d'OpenAlex est par type d'opération, et **le singleton
`/works/{id}` est gratuit**. L'enrichissement d'un nœud isolé est exactement un
singleton : il ne coûte rien. Ce sont `list+filter` (0,10 $/1000) et `search`
(1 $/1000) qui sont facturés — d'où l'interdiction d'utiliser OpenAlex pour
l'expansion de graphe. Le module client n'expose structurellement que le
singleton.

Le champ `is_retracted` d'OpenAlex n'est **pas** repris : il agrège tout update
Crossref, corrections comprises, et classe donc des articles simplement corrigés
comme rétractés. Le statut est lu directement dans `updated-by` de Crossref, en
conservant le type et la source du signalement.

## Exports

Quatre familles, toutes accessibles depuis le panneau.

- **Bibliographique** — BibTeX, RIS, CSL-JSON. Compatible Zotero. La couverture
  et le statut de rétractation voyagent avec chaque référence : un export qui
  perdrait cette information transformerait la carte honnête en carte trompeuse
  dès qu'elle quitte l'extension.
- **Graphe** — GEXF (Gephi) et liste d'arêtes CSV. `networkx` lit les deux
  directement.
- **JSON natif** — schéma versionné, couche événement reproduite telle quelle,
  listes de références résolues comprises. Réimportable : voir *Sauvegarde et
  restauration*.
- **Markdown à wikilinks** — un fichier par article et par requête, liens
  `[[...]]`. Ouvre Obsidian.

Extrait réel d'un fichier de requête exporté :

```markdown
---
type: requete
raw_query: "Bernard Werber"
timestamp: 2026-08-05T23:42:29.867Z
shape_key: "v1 | bernard werber | limit=25 | semanticscholar"
results: 25
---
# Bernard Werber
Emise le 06/08/2026 01:42:29. 25 resultats geles.
## Resultats
- [[Le Pere de nos peres]]
- [[L'Arbre des possibles]]
- [[Troisieme humanite by Bernard Werber (review)]]
```

## Sauvegarde, restauration et remise à zéro

L'export **JSON natif** est une sauvegarde complète : événements, articles,
arêtes et listes de références résolues. `Restaurer un export natif…` le
recharge.

La restauration **remplace** la base entière, après une confirmation qui
affiche en chiffres ce qui arrive et ce qui sera perdu. Elle est atomique : si
une écriture échoue, rien n'est perdu.

Elle ne **fusionne** jamais deux graphes, et c'est délibéré. Fusionner
demanderait de répondre à des questions que le §11 ne tranche pas : que faire
d'un identifiant d'événement déjà pris avec un contenu différent, sachant que
la couche événement est immuable ? Et surtout — un graphe où vos actes et ceux
d'un tiers sont indiscernables cesse de montrer *votre* trajectoire, qui est
l'objet même du produit. Le cas réellement utile, changer de machine ou
retrouver un profil Firefox vidé, ne demande pas de fusion.

Si le normaliseur a changé de version entre l'export et la restauration, les
clés de forme sont recalculées **avant** écriture — sans quoi le repli
d'affichage deviendrait faux et des requêtes identiques apparaîtraient
séparées. La clé de forme est une donnée dérivée : la recalculer ne modifie pas
l'acte. Chaîne brute, horodatage, filtres, sources et résultats gelés restent
exactement ceux de l'export.

Le cache HTTP n'est pas inclus dans l'export — c'est un cache, pas une donnée.
Il se reconstitue tout seul.

**`Tout effacer…`** vide le graphe — événements, articles, arêtes, listes de
références — pour repartir de zéro. C'est une restauration d'un graphe vide :
même chemin d'écriture, même atomicité, même confirmation chiffrée. Deux choses
survivent, délibérément : les clés API (c'est de la configuration, pas du
travail) et le cache HTTP (un article déjà résolu ne sera pas redemandé au
réseau si vous le recroisez). L'effacement étant irréversible, exportez d'abord
si le graphe courant a la moindre valeur — le dialogue le rappelle.

## Limites connues

- **Fusion de deux graphes** — non implémentée, par choix. Voir ci-dessus.
- **Arête de proximité sémantique** — réservée dans le modèle, non implémentée.
  Elle produirait des voisinages plausibles et non vérifiables ; c'est le lien le
  plus faible et il est traité comme tel.
- **Pertinence des résultats** — elle est celle de Semantic Scholar. Une requête
  sur un nom propre peut ramener du bruit. L'extension n'y touche pas : elle gèle
  ce que la source a renvoyé, y compris quand c'est mauvais.

## Développement

```bash
npm test           # 58 tests : identité, couverture, version, nommage, restauration
npm run lint       # web-ext lint (0 erreur)
npm run dev        # recompilation à la volée
npm run package    # → web-ext-artifacts/
```

```
src/
  core/       identité, modèle, recouvrement, couverture — pur, testé, sans I/O
  data/       clients API, file d'appels, IndexedDB
  background/ pipeline d'ingestion, routeur de messages (event page MV3)
  ui/         panneau et rendu du graphe
  export/     les quatre familles du §6
```

Le code embarque ses dépendances : Manifest V3 interdit le chargement de script
distant, tout est bundlé par esbuild. Firefox n'implémente pas
`background.service_worker` en MV3 — l'extension utilise une *event page*, ce qui
donne accès à IndexedDB et aux modules ES sans détour.

Le rendu s'appuie sur **sigma.js + graphology**, retenus sur mesure plutôt que
par préférence :

| Candidat | brut | gzip | rendu |
|---|---|---|---|
| cytoscape.js | 433 Ko | 138 Ko | canvas 2D |
| **sigma + graphology + forceatlas2** | **166 Ko** | **40 Ko** | **WebGL** |
| d3-force + selection + zoom | 61 Ko | 20 Ko | moteur à écrire |

graphology fournit en prime l'export GEXF.

Le voile de couverture est dessiné hors WebGL, sur un canvas 2D superposé, à pas
constant en espace écran : une trame qui grossirait avec le zoom se lirait comme
une texture d'objet, alors qu'elle doit se lire comme une absence de donnée.

## Licence

GPL-3.0 license

## Note

Ce projet est construit par intérêt pour la construction elle-même, en
connaissance de l'existence d'équivalents partiels. Il n'est pas évalué au
nombre d'installations.
