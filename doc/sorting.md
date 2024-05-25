# Tri

## Introduction

La gestion du tri est essentielle pour permettre aux utilisateurs de contrôler l'ordre dans lequel les données sont présentées. En utilisant les fonctionnalités de tri, les résultats peuvent être affichés dans l'ordre souhaité, que ce soit par ordre alphabétique, chronologique ou selon d'autres critères. HDL inclut une structure flexible de tri qui permet aux utilisateurs de spécifier les critères de tri de manière simple et claire.

## Concepts Généraux

### Liens de Tri

HDL utilise des liens hypermedia pour permettre aux clients de spécifier des critères de tri. Ces liens incluent des paramètres de requête pour définir les critères. Les paramètres de tri sont ajoutés à l'URL sous la forme `sort=`, suivis des propriétés à trier, séparées par des virgules. Un signe `-` est utilisé pour indiquer un ordre descendant (DESC), tandis qu'un ordre ascendant (ASC) est implicite.

**Exemple de Liens de Tri**

```json
{
  "self": { "href": "http://example.com/articles?page=1&sort=property1,-property2" },
  "first": { "href": "http://example.com/articles?page=1&sort=property1,-property2" },
  "last": { "href": "http://example.com/articles?page=100&sort=property1,-property2" },
  "next": { "href": "http://example.com/articles?page=2&sort=property1,-property2" }
}
```

### Métadonnées de Tri

Les métadonnées de tri fournissent des informations sur les critères de tri actuellement appliqués à la collection. Cela inclut les propriétés sur lesquelles les données sont triées et l'ordre (ascendant ou descendant) de chaque critère. Ces informations peuvent être utilisées par le client (par exemple, le front-end) pour afficher correctement le tri à l'utilisateur, tandis que les liens sont utilisés pour la navigation.

**Exemple de Métadonnées de Tri**

```json
{
  "sort": [
    {
      "property": "property1",
      "order": "asc"
    },
    {
      "property": "property2",
      "order": "desc"
    }
  ]
}
```

## Mise en Application Complète

Voici un exemple complet de la gestion du tri dans HDL, incluant les liens de tri et les métadonnées associées.

**Exemple Complet**

```json
{
  "items": [
    {
      "id": "1",
      "title": "Article 1",
      "description": "Content of article 1.",
      "_links": {
        "self": {
          "href": "http://example.com/articles/1"
        }
      }
    },
    {
      "id": "2",
      "title": "Article 2",
      "description": "Content of article 2.",
      "_links": {
        "self": {
          "href": "http://example.com/articles/2"
        }
      }
    }
    // Autres articles...
  ],
  "_links": {
    "self": {
      "href": "http://example.com/articles?page=1&sort=title,-created"
    },
    "first": {
      "href": "http://example.com/articles?page=1&sort=title,-created"
    },
    "last": {
      "href": "http://example.com/articles?page=100&sort=title,-created"
    },
    "next": {
      "href": "http://example.com/articles?page=2&sort=title,-created"
    }
  },
  "pagination": {
    "totalItems": 1000,
    "pageSize": 10,
    "currentPage": 1,
    "totalPages": 100,
    "span": 5
  },
  "sort": [
    {
      "property": "title",
      "order": "asc"
    },
    {
      "property": "created",
      "order": "desc"
    }
  ]
}
```

## Conclusion

L'intégration du tri dans HDL améliore la gestion des collections volumineuses en fournissant une navigation contextuelle et intuitive. En utilisant des liens hypermedia et des métadonnées spécifiques, HDL offre une solution flexible et claire pour gérer efficacement les collections de données dans une API RESTful.

**Flexibilité et Clarté** :
   - Les clients peuvent facilement naviguer dans la collection tout en conservant les critères de tri en suivant les liens hypermedia fournis.
   - Les métadonnées de tri fournissent un contexte clair sur les critères appliqués à la collection actuelle et peuvent être utilisées par le client pour afficher correctement le tri à l'utilisateur.

**Simplicité d'Intégration** :
   - La structure proposée est simple à intégrer et à utiliser, en maintenant la compatibilité avec le reste du format HDL tout en ajoutant des fonctionnalités précieuses.