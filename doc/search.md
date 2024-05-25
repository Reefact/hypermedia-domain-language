# Recherche dans une collection de ressources

## Introduction

La gestion de la recherche dans une collection de ressources est essentielle pour permettre aux utilisateurs de trouver des résultats spécifiques en fonction de termes de recherche textuels. En utilisant les fonctionnalités de recherche, les résultats peuvent être affinés pour répondre aux besoins spécifiques des utilisateurs. HDL inclut une structure flexible de recherche qui permet aux utilisateurs de spécifier des critères de recherche de manière simple et claire.

## Concepts Généraux

### Liens de Recherche

HDL utilise des liens hypermedia pour permettre aux clients de spécifier des critères de recherche textuelle. Ces liens incluent des paramètres de requête pour définir les termes de recherche. Les paramètres de recherche sont ajoutés à l'URL sous la forme `search={query}`.

**Exemple de Liens de Recherche**

```json
{
  "self": {
    "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F"
  },
  "first": {
    "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F"
  },
  "last": {
    "href": "http://example.com/articles?page=100&search=Where%20is%20Bryan%20%3F"
  },
  "next": {
    "href": "http://example.com/articles?page=2&search=Where%20is%20Bryan%20%3F"
  }
}
```

### Métadonnées de Recherche

Les métadonnées de recherche fournissent des informations sur les termes de recherche actuellement appliqués à la collection. Cela inclut le terme de recherche. Ces informations peuvent être utilisées par le client (par exemple, le front-end) pour afficher correctement les termes de recherche appliqués à l'utilisateur, tandis que les liens sont utilisés pour la navigation.

**Exemple de Métadonnées de Recherche**

```json
{
  "search": {
    "query": "Where is Bryan ?"
  }
}
```

## Mise en Application Complète

Voici un exemple complet de la gestion de la recherche dans HDL, incluant les liens de recherche et les métadonnées associées.

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
      "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F"
    },
    "first": {
      "href": "http://example.com/articles?page=1&search=Where%20is%20Bryan%20%3F"
    },
    "last": {
      "href": "http://example.com/articles?page=100&search=Where%20is%20Bryan%20%3F"
    },
    "next": {
      "href": "http://example.com/articles?page=2&search=Where%20is%20Bryan%20%3F"
    }
  },
  "pagination": {
    "totalItems": 1000,
    "pageSize": 10,
    "currentPage": 1,
    "totalPages": 100
  },
  "search": {
    "query": "Where is Bryan ?"
  }
}
```

## Conclusion

L'intégration de la recherche dans HDL améliore la gestion des collections volumineuses en fournissant une navigation contextuelle et intuitive. En utilisant des liens hypermedia et des métadonnées spécifiques, HDL offre une solution flexible et claire pour gérer efficacement les collections de données dans une API RESTful.

**Flexibilité et Clarté** :
   - Les clients peuvent facilement appliquer des critères de recherche en suivant les liens hypermedia fournis.
   - Les métadonnées de recherche fournissent un contexte clair sur les termes de recherche appliqués à la collection actuelle.

**Simplicité d'Intégration** :
   - La structure proposée est simple à intégrer et à utiliser, en maintenant la compatibilité avec le reste du format HDL tout en ajoutant des fonctionnalités précieuses.