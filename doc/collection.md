# Collections

## Introduction

Une collection HDL regroupe des ressources : un tableau d'éléments, accompagné de ses liens et de ses métadonnées de vue.

## Collection-ressource ou collection imbriquée

Deux choses très différentes s'appellent « collection ». **Cette page ne décrit que la première.**

- **Collection-ressource** — adressée en propre (`GET /articles`). Elle a un `self` ; c'est une **vue requêtable** d'un ensemble potentiellement vaste. Pagination, tri, filtre, recherche, navigation et actions **lui appartiennent**. C'est l'objet de cette page.
- **Collection imbriquée** — un tableau *à l'intérieur* d'une ressource (`team.members`, `commande.lignes`). C'est une **propriété multivaluée** du parent, livrée avec lui : un tableau nu sous son nom métier, **sans** `_pagination`, `_sort`, `_filter`, ni liens de navigation. Elle relève de [Ressources et sous-ressources](resources.md).

> **À retenir —** pagination, tri et filtre n'existent **que** sur une collection-ressource. Être tenté de paginer un tableau imbriqué est le signe qu'il ne devait pas être imbriqué, mais exposé par un lien (voir la règle *inline ou lien* dans [Ressources et sous-ressources](resources.md)).

## Structure

Une collection-ressource est un objet englobant qui réunit :

- le **tableau des éléments**, sous une clé **nommée par le domaine** (`articles`, `commandes`…) — comme toute donnée et toute sous-ressource en HDL ;
- `_links` — son `self`, ses liens de navigation, ses actions ;
- `_pagination` — la tranche renvoyée (champs optionnels selon les capacités, voir [Pagination](pagination.md)) ;
- selon la vue, `_sort`, `_filter`, `_search` — les critères appliqués.

## Une collection est une ressource

Une collection a un `self` : c'est donc une **ressource**, au même titre qu'une ressource individuelle. Et comme ressource, elle porte ses propres **actions** dans `_links` (`method: "POST"`) — par exemple ajouter un élément.

Toutes les collections ne sont pas modifiables, cependant : l'action d'ajout n'apparaît **que si** la collection l'est. C'est une propriété de la collection, pas une règle du format.

> **À ne pas confondre — ressource et vue.** `/articles` est la **ressource** : stable, c'est là qu'on crée. `/articles?page=2&sort=…&filter=…` est une **vue** de cette ressource — une tranche. Le `self` d'une réponse pointe la vue courante ; l'action de création, elle, vise la ressource de base.

## Navigation

La navigation entre tranches utilise les relations `first`, `prev`, `self`, `next`, `last`, dans `_links`. `prev` est absent sur la première tranche, `next` sur la dernière ; `first` et `last` ne sont présents que si la collection sait atteindre ses extrémités. Chaque lien **conserve les critères de vue appliqués** (tri, filtre, recherche) et reste **opaque** au client, qui le suit sans le construire. HDL est agnostique de la stratégie de pagination — page, curseur, token (voir [Pagination](pagination.md)).

## Deux familles de métadonnées

Une réponse de collection peut porter deux familles de membres réservés, à ne pas mélanger :

- `_metadata` décrit l'**échange** — qui, quand, quelle version (voir [Métadonnées](metadata.md)) ;
- `_pagination`, `_sort`, `_filter`, `_search` décrivent la **vue** — quelle tranche de la collection est renvoyée, et selon quels critères.

## Exemple complet

```json
{
  "articles": [
    {
      "title": "Article 1",
      "created": "2025-03-12",
      "_links": { "self": { "href": "http://example.com/articles/1" } }
    },
    {
      "title": "Article 2",
      "created": "2025-01-08",
      "_links": { "self": { "href": "http://example.com/articles/2" } }
    }
  ],
  "_links": {
    "self":          { "href": "http://example.com/articles?page=3&sort=title,-created" },
    "creer":         { "href": "http://example.com/articles:creer", "method": "POST" },
    "first":         { "href": "http://example.com/articles?page=1&sort=title,-created" },
    "prev":          { "href": "http://example.com/articles?page=2&sort=title,-created" },
    "next":          { "href": "http://example.com/articles?page=4&sort=title,-created" },
    "last":          { "href": "http://example.com/articles?page=100&sort=title,-created" }
  },
  "_pagination": {
    "totalItems": 1000,
    "pageSize": 10,
    "currentPage": 3,
    "totalPages": 100
  },
  "_sort": [
    { "property": "title",   "order": "asc" },
    { "property": "created", "order": "desc" }
  ]
}
```

Le `self` pointe la **vue** (page 3, triée) ; `creer` vise la **ressource de base** (`/articles`), pas la vue. Les éléments sont sous `articles`, nommés par le domaine ; chacun, portant un `self`, est lui-même une sous-ressource.

## Récapitulatif

| Membre | Rôle |
|---|---|
| `<domaine>` (ex. `articles`) | le tableau des éléments, nommé par le domaine |
| `_links` | `self`, navigation (`first`/`prev`/`next`/`last`), actions |
| `_pagination` | la tranche renvoyée (optionnel, selon les capacités) |
| `_sort` / `_filter` / `_search` | les critères de vue appliqués |

Une collection a un `self` : c'est une ressource, et elle porte ses actions (ajout…) quand elle est modifiable. La ressource (`/articles`) et ses vues (`/articles?…`) se distinguent par les paramètres de requête ; `_metadata` décrit l'échange, les membres de vue décrivent la tranche.

Voir : [Pagination](pagination.md), [Tri](sorting.md), [Filtrage](filtering.md), [Recherche](search.md).
