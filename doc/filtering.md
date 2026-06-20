# Filtrage de collection de ressources

## Introduction

Le filtrage restreint les éléments d'une collection selon des critères. Un critère se transmet sous la forme `filter[propriété]=valeur`.

## Sémantique

HDL s'en tient à l'**égalité**, avec une combinaison **explicite** :

- plusieurs valeurs d'un même critère, séparées par des virgules, se combinent en **OU** — `filter[author]=John,Jane` sélectionne les articles dont l'auteur est John *ou* Jane ;
- plusieurs critères distincts se combinent en **ET** — `filter[author]=John,Jane&filter[genre]=Fiction` exige les deux.

HDL ne définit **pas** d'opérateurs (`≥`, `≠`, plages), pas de OU entre critères, pas de négation. Un besoin de requêtage plus riche relève d'un langage dédié (RSQL, OData) et sort du périmètre de HDL.

## Liens : composer et naviguer

`_links` porte deux usages distincts :

- **composer** un filtre — un lien **templated** que le client remplit ;
- **naviguer** les résultats — `first` / `prev` / `self` / `next` / `last`, qui conservent les filtres **déjà appliqués**.

```json
{
  "_links": {
    "filter": { "href": "http://example.com/articles{?filter}", "templated": true },
    "first":  { "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction" },
    "prev":   { "href": "http://example.com/articles?page=2&filter[author]=John,Jane&filter[genre]=Fiction" },
    "self":   { "href": "http://example.com/articles?page=3&filter[author]=John,Jane&filter[genre]=Fiction" },
    "next":   { "href": "http://example.com/articles?page=4&filter[author]=John,Jane&filter[genre]=Fiction" },
    "last":   { "href": "http://example.com/articles?page=24&filter[author]=John,Jane&filter[genre]=Fiction" }
  }
}
```

> Le client compose un filtre via le lien `filter` (templated). HDL **n'expose pas** la liste des champs filtrables ni leurs types : composer un filtre suppose la connaissance du domaine, partagée avec le client. La forme par champ `filter[propriété]=valeur` est conventionnelle (un gabarit RFC 6570 ne l'exprime pas nativement) ; le lien `filter` signale surtout que le filtrage est disponible.

## L'état appliqué : `_filter`

`_filter` expose les critères **actuellement appliqués**, sous forme parsée, pour que le client les affiche sans analyser d'URL. C'est un **membre réservé** (préfixe `_`). Les liens servent à naviguer, `_filter` sert à afficher.

```json
{
  "_filter": [
    { "property": "author", "values": ["John", "Jane"] },
    { "property": "genre",  "values": ["Fiction"] }
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
      "author": "John",
      "genre": "Fiction",
      "_links": { "self": { "href": "http://example.com/articles/1" } }
    },
    {
      "title": "Article 2",
      "description": "Content of article 2.",
      "author": "Jane",
      "genre": "Fiction",
      "_links": { "self": { "href": "http://example.com/articles/2" } }
    }
  ],
  "_links": {
    "filter": { "href": "http://example.com/articles{?filter}", "templated": true },
    "first":  { "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction" },
    "prev":   { "href": "http://example.com/articles?page=2&filter[author]=John,Jane&filter[genre]=Fiction" },
    "self":   { "href": "http://example.com/articles?page=3&filter[author]=John,Jane&filter[genre]=Fiction" },
    "next":   { "href": "http://example.com/articles?page=4&filter[author]=John,Jane&filter[genre]=Fiction" },
    "last":   { "href": "http://example.com/articles?page=24&filter[author]=John,Jane&filter[genre]=Fiction" }
  },
  "_pagination": {
    "totalItems": 240,
    "pageSize": 10,
    "currentPage": 3,
    "totalPages": 24
  },
  "_filter": [
    { "property": "author", "values": ["John", "Jane"] },
    { "property": "genre",  "values": ["Fiction"] }
  ]
}
```

Voir : [Collections](collection.md).
