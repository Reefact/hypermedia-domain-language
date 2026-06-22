# Recherche dans une collection de ressources

## Introduction

La recherche plein-texte permet au client de retrouver des éléments d'une collection à partir d'un terme libre. Le terme se transmet en paramètre de requête `search={query}`. Contrairement au tri et au filtrage, la recherche n'expose **aucun vocabulaire** : c'est un champ unique, opaque.

## Liens : composer et naviguer

`_links` porte deux usages distincts :

- **composer** une recherche — un lien **templated** que le client remplit ;
- **naviguer** les résultats — `first` / `prev` / `self` / `next` / `last`, qui conservent le terme **déjà appliqué**.

```json
{
  "_links": {
    "search": { "href": "http://example.com/articles{?search}", "templated": true },
    "first":  { "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F" },
    "prev":   { "href": "http://example.com/articles?page=2&search=Where%20is%20Bryan%20%3F" },
    "self":   { "href": "http://example.com/articles?page=3&search=Where%20is%20Bryan%20%3F" },
    "next":   { "href": "http://example.com/articles?page=4&search=Where%20is%20Bryan%20%3F" },
    "last":   { "href": "http://example.com/articles?page=100&search=Where%20is%20Bryan%20%3F" }
  }
}
```

> La **présence** du lien `search` (templated) signale que la collection est cherchable ; son absence, qu'elle ne l'est pas. Il n'y a rien d'autre à découvrir : la recherche n'a pas de champs.

## L'état appliqué : `_search`

`_search` expose le terme **actuellement appliqué**, pour que le client l'affiche sans analyser d'URL. C'est un **membre réservé** (préfixe `_`).

```json
{
  "_search": { "query": "Where is Bryan ?" }
}
```

## Exemple complet

```json
{
  "articles": [
    {
      "title": "Article 1",
      "description": "Content of article 1.",
      "_links": { "self": { "href": "http://example.com/articles/1" } }
    },
    {
      "title": "Article 2",
      "description": "Content of article 2.",
      "_links": { "self": { "href": "http://example.com/articles/2" } }
    }
  ],
  "_links": {
    "search": { "href": "http://example.com/articles{?search}", "templated": true },
    "first":  { "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F" },
    "prev":   { "href": "http://example.com/articles?page=2&search=Where%20is%20Bryan%20%3F" },
    "self":   { "href": "http://example.com/articles?page=3&search=Where%20is%20Bryan%20%3F" },
    "next":   { "href": "http://example.com/articles?page=4&search=Where%20is%20Bryan%20%3F" },
    "last":   { "href": "http://example.com/articles?page=100&search=Where%20is%20Bryan%20%3F" }
  },
  "_pagination": {
    "totalItems": 1000,
    "pageSize": 10,
    "currentPage": 3,
    "totalPages": 100
  },
  "_search": { "query": "Where is Bryan ?" }
}
```

Voir : [Collections](collection.md).
