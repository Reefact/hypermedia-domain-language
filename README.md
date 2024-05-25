# Hypermedia Domain Language (HDL)

## Introduction

Hypermedia Domain Language (HDL) est un format JSON conçu pour représenter des ressources hypermedia dans les API RESTful. Inspiré par des formats existants comme HAL et JSON:API, HDL se concentre sur la clarté, la flexibilité et une représentation fidèle des objets et des termes métiers. Cela facilite la compréhension et l'utilisation des API pour tous les utilisateurs, y compris les développeurs et les experts métiers.

HDL met en œuvre les principes HATEOAS (Hypermedia as the Engine of Application State), permettant aux clients de naviguer dynamiquement à travers les API en suivant des liens hypermedia et des affordances clairement définis. Ce format est spécialement conçu pour offrir une interaction intuitive et évolutive avec les ressources, tout en maintenant une structure simple et lisible.

## Caractéristiques Principales de HDL

### Représentation des Objets et Termes Métiers

HDL se distingue par sa capacité à utiliser des termes et des concepts métiers pour nommer les affordances et les ressources. Cela facilite la communication et l'alignement avec l'ubiquitous language, unifiant ainsi les développeurs et les experts métiers autour d'une terminologie commune. En représentant les objets métiers de manière claire et compréhensible, HDL améliore la lisibilité des API, permettant à tous les utilisateurs de comprendre et d'interagir efficacement avec les ressources disponibles. Cette approche réduit la courbe d'apprentissage et améliore la collaboration entre les équipes techniques et non techniques.

### Flexibilité et Richesse des Interactions

HDL offre une grande flexibilité grâce à l'utilisation de templates URI, permettant la génération dynamique des URLs en fonction des paramètres. Cela permet de gérer efficacement les interactions avec les API, en offrant des options variées sans complexifier la structure. Les affordances dans HDL vont au-delà de la simple navigation, en décrivant également les actions disponibles sur les ressources, incluant les méthodes HTTP et les types de médias. Cette description riche des actions permet une interaction plus complète et intuitive avec les API.

En outre, HDL gère efficacement la dépréciation des liens hypermedia. Les liens peuvent inclure une propriété `deprecation` pour indiquer les affordances obsolètes et fournir des alternatives, assurant une transition en douceur sans casser la compatibilité. Cette gestion proactive des dépréciations aide à maintenir l'intégrité et la robustesse de l'API au fil du temps.

## Comparaison avec d'Autres Formats

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
        "href": "http://example.com/authors/9",
        "title": "Author Details"
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
      }
    ],
    "download-v2": {
      "href": "http://example.com/articles/1/download?lang={lang}&type={type}",
      "templated": true,
      "title": "Download Article v2"
    }
  }
}
```

HDL est conçu pour évoluer avec les besoins des utilisateurs et des développeurs, en intégrant les meilleures pratiques des formats existants tout en apportant une simplicité et une clarté unique. Ce format offre une base solide pour créer des API RESTful robustes et conviviales, orientées vers les termes métiers et faciles à utiliser pour tous.

