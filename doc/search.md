# Recherche plain-text

## Introduction

La recherche plein-texte permet au client de retrouver des éléments d'une collection à partir d'un terme libre. Le terme se transmet en paramètre de requête `search={query}`. Contrairement au tri et au filtrage, la recherche n'expose **aucun vocabulaire** : c'est un champ unique, opaque. Le client envoie une chaîne ; le serveur décide ce que « correspond » veut dire — tokenisation, racinisation, pertinence, tolérance aux fautes relèvent de son implémentation, jamais du contrat.

## Recherche, filtrage, tri : trois mécanismes distincts

Sur une même collection, trois critères de vue coexistent sans se confondre :

- la **recherche** (`search=`) confronte un terme libre à des champs **choisis par le serveur**, avec une notion de **pertinence** ; elle est non structurée et opaque ;
- le **filtrage** (`filter[propriété]=`) restreint sur des **champs nommés**, par **égalité** ; il est structuré, déterministe, et suppose la connaissance du domaine (voir [Filtrage](filtering.md)) ;
- le **tri** (`sort=`) ordonne le résultat (voir [Tri](sorting.md)).

Les trois **se composent** : une requête peut chercher, filtrer, trier et paginer à la fois. « Les chaussures de running de la marque Speedline, les moins chères d'abord » se traduit par un `search`, un `filter[marque]` et un `sort=prix` sur la même collection.

Pour la recherche, HDL ne définit **ni recherche par champ** — `search[nom]=` n'existe pas, c'est le rôle du filtre —, **ni opérateurs**, **ni syntaxe booléenne**, **ni exposition du score de pertinence** dans le contrat. Un besoin de requêtage structuré relève du **filtrage**, ou d'un langage dédié (RSQL, OData) hors périmètre de HDL. La recherche, elle, reste un seul terme.

> **Recherche et pertinence.** En l'absence de tri explicite, un résultat de recherche est typiquement ordonné par **pertinence**, déterminée par le serveur. Un `sort=` explicite reprend la main sur cet ordre. Le client n'a pas à connaître l'algorithme : il lit l'ordre tel qu'il vient et l'affiche.

## Liens : composer et naviguer

`_links` porte deux usages distincts :

- **composer** une recherche — un lien **templated** que le client remplit ;
- **naviguer** les résultats — `first` / `prev` / `self` / `next` / `last`, qui conservent le terme **déjà appliqué** (et tout autre critère de vue actif).

```json
{
  "_links": {
    "search": { "href": "http://example.com/produits{?search}", "templated": true },
    "first":  { "href": "http://example.com/produits?page=1&search=chaussures%20running" },
    "prev":   { "href": "http://example.com/produits?page=2&search=chaussures%20running" },
    "self":   { "href": "http://example.com/produits?page=3&search=chaussures%20running" },
    "next":   { "href": "http://example.com/produits?page=4&search=chaussures%20running" },
    "last":   { "href": "http://example.com/produits?page=12&search=chaussures%20running" }
  }
}
```

> La **présence** du lien `search` (templated) signale que la collection est cherchable ; son absence, qu'elle ne l'est pas. Il n'y a rien d'autre à découvrir : la recherche n'a pas de champs.

## L'état appliqué : `_search`

`_search` expose le terme **actuellement appliqué**, pour que le client l'affiche sans analyser d'URL — remplir la barre de recherche, proposer de l'effacer. C'est un **membre réservé** (préfixe `_`). Là où `_sort` et `_filter` sont des **tableaux** de critères, `_search` est un **objet** : il n'y a qu'un terme, pas une liste.

```json
{
  "_search": { "query": "chaussures running" }
}
```

## Exemple — recherche dans un catalogue produit

Une recherche simple sur un catalogue, sans autre critère. `prix` est un value object (pas de `self`) : c'est de la donnée. Chaque produit, porteur d'un `self`, est une sous-ressource.

```json
{
  "produits": [
    {
      "nom": "Chaussure de running Aero 7",
      "prix": { "valeur": 89.90, "devise": "EUR" },
      "marque": "Speedline",
      "_links": { "self": { "href": "http://example.com/produits/4187" } }
    },
    {
      "nom": "Chaussure de running Trail Pro",
      "prix": { "valeur": 129.00, "devise": "EUR" },
      "marque": "Montak",
      "_links": { "self": { "href": "http://example.com/produits/5023" } }
    }
  ],
  "_links": {
    "search": { "href": "http://example.com/produits{?search}", "templated": true },
    "self":   { "href": "http://example.com/produits?page=1&search=chaussures%20running" },
    "next":   { "href": "http://example.com/produits?page=2&search=chaussures%20running" },
    "last":   { "href": "http://example.com/produits?page=12&search=chaussures%20running" }
  },
  "_pagination": {
    "totalItems": 113,
    "pageSize": 10,
    "currentPage": 1,
    "totalPages": 12
  },
  "_search": { "query": "chaussures running" }
}
```

## Exemple — recherche, filtre et tri combinés

La même collection, cette fois cherchée *et* filtrée par marque *et* triée par prix croissant. Les trois critères vivent dans l'URL ; chaque lien de navigation les **conserve tous**, et chacun est exposé sous son membre réservé pour l'affichage.

```json
{
  "produits": [
    {
      "nom": "Chaussure de running Aero 7",
      "prix": { "valeur": 89.90, "devise": "EUR" },
      "marque": "Speedline",
      "_links": { "self": { "href": "http://example.com/produits/4187" } }
    },
    {
      "nom": "Chaussure de running Aero 7 GTX",
      "prix": { "valeur": 119.90, "devise": "EUR" },
      "marque": "Speedline",
      "_links": { "self": { "href": "http://example.com/produits/4192" } }
    }
  ],
  "_links": {
    "search": { "href": "http://example.com/produits{?search}", "templated": true },
    "filter": { "href": "http://example.com/produits{?filter}", "templated": true },
    "sort":   { "href": "http://example.com/produits{?sort}",   "templated": true },
    "self":   { "href": "http://example.com/produits?page=1&search=chaussures%20running&filter[marque]=Speedline&sort=prix" },
    "next":   { "href": "http://example.com/produits?page=2&search=chaussures%20running&filter[marque]=Speedline&sort=prix" },
    "last":   { "href": "http://example.com/produits?page=3&search=chaussures%20running&filter[marque]=Speedline&sort=prix" }
  },
  "_pagination": {
    "totalItems": 24,
    "pageSize": 10,
    "currentPage": 1,
    "totalPages": 3
  },
  "_search": { "query": "chaussures running" },
  "_filter": [
    { "property": "marque", "values": ["Speedline"] }
  ],
  "_sort": [
    { "property": "prix", "order": "asc" }
  ]
}
```

Voir : [Collections](collection.md), [Filtrage](filtering.md), [Tri](sorting.md).
