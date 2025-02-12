# Hypermedia Domain Language

## Introduction

Hypermedia Domain Language (HDL) est un format JSON conçu pour représenter des ressources hypermedia dans les API [DORIAX](doc/do-rest.md). Inspiré par des formats existants comme HAL et JSON:API, HDL se concentre sur la clarté, la flexibilité et une représentation fidèle des objets et des termes métiers. Cela facilite la compréhension et l'utilisation des API pour tous les utilisateurs, y compris les développeurs et les experts métiers.

Là où d'autres formats imposent une restructuration complète des contrats API, le Hypermedia Domain Language tire parti de l'organisation traditionnelle REST en ressources et sous-ressources. HDL permet d'enrichir ces structures de manière graduelle, selon les fonctionnalités désirées, offrant une flexibilité accrue sans compromettre la clarté du contrat initial. Cette approche facilite l'intégration d'HATEOAS en augmentant les capacités des API tout en restant fidèle aux principes de simplicité et d'évolutivité inhérents à REST.

Le Hypermedia Domain Language propose une approche intégrée des principes HATEOAS pour les API RESTful, en enrichissant les structures JSON traditionnelles sans les compliquer. HDL facilite une navigation dynamique et intuitive, permettant des interactions plus claires et plus directes à travers les applications. C’est une option pratique pour ceux qui cherchent à adopter HATEOAS tout en maintenant la simplicité et l’efficacité de leurs interfaces API.

## Pourquoi HDL ?

Bien que particulièrement puissant et remontant aux années 2000, le concept HATEOAS n'a pas été largement adopté dans l'implémentation des API REST pour plusieurs raisons. Premièrement, sa mise en œuvre peut s'avérer complexe et peut demander une conception initiale plus réfléchie, ce qui dissuade souvent les développeurs habitués à des approches plus simples. De plus, le manque de support des outils et des bibliothèques pour faciliter l'intégration de HATEOAS dans les API existantes constitue un autre obstacle notable.

Dans ce contexte, HDL se présente comme une version simplifiée et extensible qui permet aux développeurs d'adopter progressivement les principes de HATEOAS. Cette approche offre une transition en douceur vers des architectures basées sur l'hypermedia, tout en conservant une structure plus traditionnelle pour la représentation des ressources. HDL vise ainsi à réduire la courbe d'apprentissage et à encourager une intégration plus large de HATEOAS dans le développement des API modernes.

## Caractéristiques Principales de HDL

HDL se propose de formaliser les ressources, les collection de ressources, ainsi que les erreurs dans un format le plus clair possible et le plus proche du métier possible.

### Formalisation des ressources

Voici un exemple de ressource REST écrite de façon "traditionnelle":

```json
{
  "property1": "value1",
  "property2": "value2",
  "property3": "value3",
  "subResource": {
    "subProperty1": "subValue1"
  }
}
```

HDL permet d'étendre cette ressource sans casser son contrat:

```json
{
  "property1": "value1",
  "property2": "value2",
  "property3": "value3",
  "subResource": {
    "subProperty1": "subValue1",
    "_links": {
      "self": {
        "href": "http://example.com/resources/656/subResources/uy"
      }
    }
  },
  "_links": {
    "self": { "href": "http://example.com/resources/656" }
  },
  "_metadata": {
    "hreflang": "fr-FR"
  }
}
```

Voir: [Formalisation des ressources](doc/resources.md), [Hyperliens](doc/hyperlinks.md)

### Collection

HDL défini un format pour les collection de ressource.

```json
{
  "items": [
    // the resource collection
  ],
  "pagination": {
    // pagination data
  },
  "sort": {
    // sorting data
  },
  "filter": {
    // filtering data
  },
  "search": {
    // search data
  }

}
```

Voir: [Collections](doc/collection.md)

### Erreurs

HDL défini également un format pour les retours en erreur:

```json
{
  "message": "Payment refused."
}
```

Voir: [Erreurs](doc/errors.md)

## Comparaison avec d'Autres Formats

HDL s'inspire largement d'autres implémentation de HATEOAS tou en essayant de conserver son objetif principal: un format simple et extensible orienté domaine.

### JSON:API

JSON:API est un format complet et standardisé pour les API RESTful, avec des conventions strictes pour les structures de données, les filtres, la pagination et les relations. Cependant, cette standardisation technique peut complexifier la lecture du JSON, rendant parfois difficile la compréhension rapide des données et des relations. HDL se distingue par sa simplicité et son orientation métier, offrant une structure plus lisible et intuitive, particulièrement adaptée pour représenter les objets métiers de manière compréhensible pour tous.

### HAL (Hypertext Application Language)

HAL est un format simple pour les API hypermedia, mettant en avant les liens et les ressources. Bien que HAL soit conçu pour être minimaliste, son approche technique peut parfois rendre la lecture des JSON moins accessible pour ceux qui ne sont pas familiers avec les conventions hypermedia. HDL enrichit cette approche en ajoutant des titres descriptifs, des méthodes HTTP et une gestion explicite des dépréciations, tout en maintenant une simplicité similaire, mais avec une meilleure clarté orientée métier.

### Siren

Siren est un format hypermedia riche qui permet des interactions complexes avec les ressources. Cependant, cette richesse peut également introduire une complexité technique qui rend la lecture du JSON plus difficile. HDL adopte une approche plus orientée vers la clarté des termes métiers, en mettant l'accent sur la lisibilité et la flexibilité pour tous les utilisateurs, tout en offrant des affordances et des templates URI pour des interactions dynamiques. Cela simplifie l'interaction avec l'API tout en conservant les avantages des interactions complexes.

## Exemples de HDL

Voici un exemple de ressource HDL montrant la structure de base et l'utilisation des templates URI et des dépréciations :

```json
{
  "id": "1",
  "title": "How to paint a bikeshed with JSON:API",
  "body": "The shortest article. Ever.",
  "author": {
    "id": "9",
    "name": "John Doe",
    "_links": {
      "self": {
        "href": "http://example.com/authors/9"
      }
    }
  },
  "_links": {
    "self": {
      "href": "http://example.com/articles/1",
      "title": "Article Details"
    },
    "buy": {
      "href": "http://example.com/articles/1:buy",
      "method": "POST",
      "deprecation": "http://example.com/docs/deprecations/buy",
      "title": "Buy Article (Deprecated)"
    },
    "purchase": {
      "href": "http://example.com/articles/1:purchase",
      "method": "POST",
      "title": "Purchase Article"
    },
    "download": [
      {
        "href": "http://example.com/articles/1/download?lang=en&type=pdf",
        "type": "application/pdf",
        "lang": "en",
        "title": "Download PDF (EN)",
        "deprecation": "http://example.com/docs/deprecations/download"
      },
      {
        "href": "http://example.com/articles/1/download?lang=en&type=md",
        "type": "text/markdown",
        "lang": "en",
        "title": "Download Markdown (EN)",
        "deprecation": "http://example.com/docs/deprecations/download"
      },
      {
        "href": "http://example.com/articles/1/download?lang=fr&type=pdf",
        "type": "application/pdf",
        "lang": "fr",
        "title": "Download PDF (FR)",
        "deprecation": "http://example.com/docs/deprecations/download"
      },
      {
        "href": "http://example.com/articles/1/download?lang=fr&type=md",
        "type": "text/markdown",
        "lang": "fr",
        "title": "Download Markdown (FR)",
        "deprecation": "http://example.com/docs/deprecations/download"
      },
      {
        "href": "http://example.com/articles/1/download?lang={lang}&type={type}",
        "templated": true,
        "title": "Download Article"
      }
    ]
  }
}
```

HDL est conçu pour évoluer avec les besoins des utilisateurs et des développeurs, en intégrant les meilleures pratiques des formats existants tout en apportant une simplicité et une clarté unique. Ce format offre une base solide pour créer des API RESTful robustes et conviviales, orientées vers les termes métiers et faciles à utiliser pour tous.

## Documentation du format HDL

* [Collections](doc/collection.md)
  * [Pagination](doc/pagination.md)
  * [Tri](doc/sorting.md)
  * [Filtrage](doc/filtering.md)
  * [Recherche](doc/search.md)
* [Métadonnées](doc/metadata.md)
* [Erreurs](doc/errors.md)
