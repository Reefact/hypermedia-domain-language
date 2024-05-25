# Pagination

## Introduction

La pagination est une fonctionnalité essentielle pour gérer efficacement les collections volumineuses dans une API. En utilisant la pagination, les résultats sont divisés en plusieurs pages, ce qui améliore la performance et la lisibilité des réponses. 

HDL inclut une structure de pagination flexible qui permet aux utilisateurs de naviguer facilement à travers les pages des collections de données.

## Concepts Généraux

### Structure des éléments de la collection

Les éléments de la collection doivent se trouver dans un objet `Items`. Chaque élément représente une ressource HDL.

```json
{
  "items": [
    {
      "id": "1",
      "title": "Article 1",
      "description": "Content of article 1.",
      "_links": {
        "self": { "href": "http://example.com/articles/1" }
      }
    },
    {
      "id": "2",
      "title": "Article 2",
      "description": "Content of article 2.",
      "_links": {
        "self": { "href": "http://example.com/articles/2" }
      }
    }
    // Autres articles...
  ]
}
```

### Liens de Pagination

HDL utilise des liens hypermedia pour permettre la navigation entre les différentes pages d'une collection. Ces liens incluent :
- `self` : Lien vers la page actuelle.
- `first` : Lien vers la première page.
- `last` : Lien vers la dernière page.
- `prev` : Lien vers la page précédente.
- `next` : Lien vers la page suivante.
- `pages` : Liens vers les pages environnantes, déterminées par la valeur de la métadonnée fournie par la propriété de pagination `span` (voir ci-après).

### Métadonnées de Pagination

Les métadonnées de pagination fournissent des informations supplémentaires sur la collection paginée, telles que :
- `totalItems` : Le nombre total d'éléments dans la collection.
- `pageSize` : Le nombre d'éléments par page.
- `currentPage` : Le numéro de la page actuelle.
- `totalPages` : Le nombre total de pages.
- `span` : Le nombre de liens de pages à inclure autour de la page actuelle. Si `span` n'est pas précisé, les liens de pages environnantes n'apparaissent pas.

## Détails des Éléments de Pagination

### Liens de Pagination

Les liens de pagination sont des hyperliens standard qui utilisent les relations de pagination prédéfinis par HDL. Par exemple :

```json
{
  "self": {
    "method": "GET",
    "href": "http://example.com/articles?page=2",
    "title": "Current Page"
  },
  "first": { "href": "http://example.com/articles?page=1" },
  "prev": { "href": "http://example.com/articles?page=1" },
  "next": { "href": "http://example.com/articles?page=3" },
  "last": { "href": "http://example.com/articles?page=10" }
}
```

Si un lien de pagination n'est pas navigable, il ne doit pas apparaître. Par exemple:

- si l'utilisateur est sur la page 1 et qu'il y a 10 pages au total, seuls `self`, `first`, `next`, et `last` doivent apparaître
- si la collection ne contient qu'une seule page, seuls `self`, `first`, et `last` doivent apparaître

### Métadonnées de Pagination

Les métadonnées fournissent un contexte clair sur la taille de la collection et la position actuelle dans la pagination. Par exemple :

```json
{
  "pagination": {
    "totalItems": 100,
    "pageSize": 10,
    "currentPage": 2,
    "totalPages": 10,
    "span": 5
  }
}
```

### Liens de Pages avec Span

Les liens de pages environnantes (span) sont déterminés par la valeur de la métadonnée `span`. Par exemple, si `currentPage` est 50 et `span` est 3, les liens incluront les pages 47 à 49 et 51 à 53 :

```json
{
  "pages": [
    {
      "name": "47",
      "href": "http://example.com/articles?page=47"
    },
    {
      "name": "48",
      "href": "http://example.com/articles?page=48"
    },
    {
      "name": "49",
      "href": "http://example.com/articles?page=49"
    },
    {
      "name": "51",
      "href": "http://example.com/articles?page=51"
    },
    {
      "name": "52",
      "href": "http://example.com/articles?page=52"
    },
    {
      "name": "53",
      "href": "http://example.com/articles?page=53"
    }
  ]
}
```


De même que précédemment, si un lien de pagination n'est pas navigable, il ne doit pas apparaître. Par exemple:

- si l'utilisateur est sur la page 1, qu'il y a 10 pages au total et que le span est a 3, seuls `self`, `first`, `next`, `last`, ainsi que les `pages` 2, 3, et 4 doivent apparaître

## Mise en Application Complète

Voici un exemple complet de la pagination dans HDL, incluant les liens de pagination, les métadonnées et les liens de pages environnantes.

### Exemple Complet

```json
{
  "items": [
    {
      "id": "1",
      "title": "Article 1",
      "description": "Content of article 1.",
      "_links": {
        "self": { "href": "http://example.com/articles/1" }
      }
    },
    {
      "id": "2",
      "title": "Article 2",
      "description": "Content of article 2.",
      "_links": {
        "self": { "href": "http://example.com/articles/2" }
      }
    }
    // Autres articles...
  ],
  "_links": {
    "self": { "href": "http://example.com/articles?page=50" },
    "first": { "href": "http://example.com/articles?page=1" },
    "prev": { "href": "http://example.com/articles?page=49" },
    "next": { "href": "http://example.com/articles?page=51" },
    "last": { "href": "http://example.com/articles?page=100" },
    "pages": [
      {
        "name": "47",
        "href": "http://example.com/articles?page=47"
      },
      {
        "name": "48",
        "href": "http://example.com/articles?page=48"
      },
      {
        "name": "49",
        "href": "http://example.com/articles?page=49"
      },
      {
        "name": "51",
        "href": "http://example.com/articles?page=51"
      },
      {
        "name": "52",
        "href": "http://example.com/articles?page=52"
      },
      {
        "name": "53",
        "href": "http://example.com/articles?page=53"
      }
    ]
  },
  "pagination": {
    "totalItems": 1000,
    "pageSize": 10,
    "currentPage": 50,
    "totalPages": 100,
    "span": 3
  }
}
```

Note: Si `span` n'est pas précisé les hyperliens `pages` ne seront pas fournis.

### Conclusion

L'intégration de la pagination dans HDL améliore la gestion des collections volumineuses en fournissant une navigation plus contextuelle et intuitive. En utilisant des liens hypermedia pour naviguer entre les pages et des métadonnées pour donner un contexte clair sur la pagination, HDL offre une solution complète et flexible pour gérer efficacement les collections de données dans une API RESTful.
