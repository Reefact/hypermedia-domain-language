# Tri

## Introduction

Le tri permet au client de contrôler l'ordre des éléments d'une collection. Le critère se transmet en paramètre de requête `sort=`, suivi des propriétés séparées par des virgules ; un préfixe `-` indique l'ordre descendant, l'ascendant étant implicite (`sort=title,-created` : par titre croissant, puis par date décroissante).

## Liens : composer et naviguer

`_links` porte deux usages distincts :

- **composer** un tri — un lien **templated** que le client remplit lui-même, sans fabriquer l'URL à la main ;
- **naviguer** la collection — `first` / `prev` / `self` / `next` / `last`, qui conservent le tri **déjà appliqué** pour paginer sans le perdre.

```json
{
  "_links": {
    "sort":  { "href": "http://example.com/articles{?sort}", "templated": true },
    "first": { "href": "http://example.com/articles?page=1&sort=title,-created" },
    "prev":  { "href": "http://example.com/articles?page=2&sort=title,-created" },
    "self":  { "href": "http://example.com/articles?page=3&sort=title,-created" },
    "next":  { "href": "http://example.com/articles?page=4&sort=title,-created" },
    "last":  { "href": "http://example.com/articles?page=100&sort=title,-created" }
  }
}
```

> Le client compose un tri via le lien `sort` (templated). HDL **n'expose pas** la liste des champs triables : connaître les champs relève du domaine, partagé avec le client.

## L'état appliqué : `_sort`

`_sort` expose le tri **actuellement appliqué**, sous forme parsée, pour que le client (un front, par exemple) l'affiche sans analyser d'URL. C'est un **membre réservé** (préfixe `_`), au même titre que `_links` et `_metadata`. Les liens servent à naviguer, `_sort` sert à afficher : deux rôles, pas une redite.

```json
{
  "_sort": [
    { "property": "title",   "order": "asc" },
    { "property": "created", "order": "desc" }
  ]
}
```

## Exemple complet

```json
{
  "articles": [
    {
      "title": "Article 1",
      "description": "Content of article 1.",
      "created": "2025-03-12",
      "_links": { "self": { "href": "http://example.com/articles/1" } }
    },
    {
      "title": "Article 2",
      "description": "Content of article 2.",
      "created": "2025-01-08",
      "_links": { "self": { "href": "http://example.com/articles/2" } }
    }
  ],
  "_links": {
    "sort":  { "href": "http://example.com/articles{?sort}", "templated": true },
    "first": { "href": "http://example.com/articles?page=1&sort=title,-created" },
    "prev":  { "href": "http://example.com/articles?page=2&sort=title,-created" },
    "self":  { "href": "http://example.com/articles?page=3&sort=title,-created" },
    "next":  { "href": "http://example.com/articles?page=4&sort=title,-created" },
    "last":  { "href": "http://example.com/articles?page=100&sort=title,-created" }
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

Voir : [Collections](collection.md).
