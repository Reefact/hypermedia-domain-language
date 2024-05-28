# Filtrage de collection de ressources

## Introduction

La gestion du filtrage est essentielle pour permettre aux utilisateurs de restreindre les résultats selon certains critères. En utilisant les fonctionnalités de filtrage, les résultats peuvent être affinés pour répondre aux besoins spécifiques des utilisateurs. HDL inclut une structure flexible de filtrage qui permet aux utilisateurs de spécifier les critères de filtrage de manière simple et claire.

## Concepts Généraux

### Liens de Filtrage

HDL utilise des liens hypermedia pour permettre aux clients de spécifier des critères de filtrage. Ces liens incluent des paramètres de requête pour définir les critères. Les paramètres de filtrage sont ajoutés à l'URL sous la forme `filter[#property]=#value`. Pour plusieurs valeurs d'un même critère, les valeurs sont séparées par des virgules.

**Exemple de Liens de Filtrage**

```json
{
  "self": {
    "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction"
  },
  "first": {
    "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction"
  },
  "last": {
    "href": "http://example.com/articles?page=100&filter[author]=John,Jane&filter[genre]=Fiction"
  },
  "next": {
    "href": "http://example.com/articles?page=2&filter[author]=John,Jane&filter[genre]=Fiction"
  }
}
```

### Métadonnées de Filtrage

Les métadonnées de filtrage fournissent des informations sur les critères de filtrage actuellement appliqués à la collection. Cela inclut les propriétés filtrées et les valeurs appliquées. Ces informations peuvent être utilisées par le client (par exemple, le front-end) pour afficher correctement les filtres appliqués à l'utilisateur, tandis que les liens sont utilisés pour la navigation.

**Exemple de Métadonnées de Filtrage**

```json
{
  "filter": [
    {
      "property": "author",
      "values": ["John", "Jane"]
    },
    {
      "property": "genre",
      "values": ["Fiction"]
    }
  ]
}
```

## Mise en Application Complète

Voici un exemple complet de la gestion du filtrage dans HDL, incluant les liens de filtrage et les métadonnées associées.

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
      "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction"
    },
    "first": {
      "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction"
    },
    "last": {
      "href": "http://example.com/articles?page=1&filter[author]=John,Jane&filter[genre]=Fiction"
    }
  },
  "pagination": {
    "totalItems": 10,
    "pageSize": 10,
    "currentPage": 1,
    "totalPages": 1
  },
  "filter": [
    {
      "property": "author",
      "values": ["John", "Jane"]
    },
    {
      "property": "genre",
      "values": ["Fiction"]
    }
  ]
}
```

## Conclusion

L'intégration du filtrage multiples dans HDL améliore la gestion des collections volumineuses en fournissant une navigation contextuelle et intuitive. En utilisant des liens hypermedia et des métadonnées spécifiques, HDL offre une solution flexible et claire pour gérer efficacement les collections de données dans une API RESTful. 

**Flexibilité et Clarté** :
   - Les clients peuvent facilement appliquer des critères de filtrage multiples en suivant les liens hypermedia fournis.
   - Les métadonnées de filtrage fournissent un contexte clair sur les critères appliqués à la collection actuelle.

**Simplicité d'Intégration** :
   - La structure proposée est simple à intégrer et à utiliser, en maintenant la compatibilité avec le reste du format HDL tout en ajoutant des fonctionnalités précieuses.