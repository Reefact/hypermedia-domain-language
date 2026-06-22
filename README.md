# Hypermedia Domain Language

## Introduction

Hypermedia Domain Language (HDL) est un format JSON pour représenter des ressources hypermedia, **orienté domaine** : les ressources se lisent avec le vocabulaire métier, sans enveloppe technique, afin de rester compréhensibles autant par les développeurs que par les experts du domaine. HDL décrit une **représentation exposée au client**, pas l'architecture interne du système.

HDL s'appuie sur **HAL** et en reprend la simplicité et le caractère incrémental — on enrichit l'organisation REST habituelle en ressources et sous-ressources, **graduellement** et de manière **additive** : sans modifier les données métier existantes, pour des clients tolérants aux membres inconnus. Il s'en écarte délibérément sur un point — l'embarquement des sous-ressources — au profit d'un affichage plus proche du **métier** que de la technique (voir [Ressources et sous-ressources](doc/resources.md)). C'est pourquoi HDL porte son **propre type de média**, `application/hdl+json`.

## Pourquoi HDL ?

HATEOAS est un concept puissant — il remonte aux années 2000 — mais peu adopté dans les API REST : sa mise en œuvre demande une conception plus réfléchie, et l'outillage pour l'intégrer aux API existantes reste limité.

HDL en propose une version simple, extensible et **orientée domaine**, que l'on adopte progressivement : on conserve une représentation de ressources proche du REST traditionnel, et on ajoute les capacités hypermedia là où elles apportent de la valeur. L'objectif n'est pas d'empiler des fonctionnalités, mais de rendre l'hypermedia **lisible par ceux qui connaissent le métier**.

## Caractéristiques principales

HDL formalise trois choses — les ressources, les collections de ressources et les erreurs — dans un format aussi clair et proche du métier que possible.

### Formalisation des ressources

Une ressource REST « traditionnelle », nommée avec le vocabulaire du domaine :

```json
{
  "numero": "CMD-2024-5821",
  "statut": "en-attente-de-paiement",
  "montantTotal": { "valeur": 47.90, "devise": "EUR" },
  "lignes": [
    { "produit": "Café moulu 250g", "quantite": 2 }
  ]
}
```

HDL l'enrichit de manière **additive** — liens et actions sur la ressource, sous-ressource référencée, métadonnées d'échange :

```json
{
  "numero": "CMD-2024-5821",
  "statut": "en-attente-de-paiement",
  "montantTotal": { "valeur": 47.90, "devise": "EUR" },
  "lignes": [
    { "produit": "Café moulu 250g", "quantite": 2, "_links": { "self": { "href": "/commandes/CMD-2024-5821/lignes/1" } } }
  ],
  "client": {
    "nom": "Camille Durand",
    "_links": { "self": { "href": "/clients/9" } }
  },
  "_links": {
    "self":    { "href": "/commandes/CMD-2024-5821" },
    "payer":   { "href": "/commandes/CMD-2024-5821:payer", "method": "POST" },
    "annuler": { "href": "/commandes/CMD-2024-5821:annuler", "method": "POST" }
  },
  "_metadata": { "apiVersion": "2.1.0" }
}
```

`montantTotal` reste une **donnée**. `lignes` et `client` portent un `self` : ce sont des **ressources navigables**, dont la représentation ici peut être partielle — on rejoint le reste à leur URI. Les actions portent leur `method`. Il n'y a **pas de compartiment `_embedded`** : une sous-ressource est imbriquée sous son nom métier, et son `self` suffit à la distinguer d'une donnée. Côté identité : la commande porte un identifiant **métier** (`numero`), tandis qu'une ligne n'a **aucun identifiant** — seulement son `self`. L'identité navigable passe par `self`, pas par un identifiant technique ajouté pour reconstruire l'URI côté client.

Voir : [Ressources et sous-ressources](doc/resources.md), [Hyperliens](doc/hyperlinks.md), [Métadonnées](doc/metadata.md).

### Collections

HDL définit un format pour les collections de ressources : un objet englobant qui réunit le **tableau des éléments, nommé par le domaine**, les liens (`_links`), et les membres réservés décrivant la tranche renvoyée (`_pagination`, et selon la vue `_sort` / `_filter` / `_search`). Une collection a un `self` : c'est une ressource, qui porte ses actions (ajout…) quand elle est modifiable.

```json
{
  "articles": [
    {
      "title": "How to paint a bikeshed",
      "_links": {
        "self": { "href": "/articles/1" }
      }
    }
  ],
  "_links": {
    "self":  { "href": "/articles?page=1" },
    "next":  { "href": "/articles?page=2" },
    "creer": { "href": "/articles:creer", "method": "POST" }
  },
  "_pagination": {
    "currentPage": 1,
    "pageSize": 20,
    "totalItems": 42,
    "totalPages": 3
  }
}
```

Voir : [Collections](doc/collection.md).

### Erreurs

HDL définit également un format pour les retours en erreur, fondé sur [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (Problem Details for HTTP APIs) :

```json
{
  "type": "http://example.com/docs/errors/payment-refused",
  "title": "Payment refused.",
  "detail": "The card was refused by the issuing bank.",
  "code": "PAYMENT_REFUSED"
}
```

Voir : [Erreurs](doc/errors.md).

## Comparaison avec d'autres formats

HDL est un format hypermedia **inspiré de HAL**, mais **orienté domaine**. Le situer dans le paysage hypermedia revient donc surtout à assumer ses partis pris — et ses renoncements.

**HAL** est le plus proche : même minimalisme, mêmes attributs de lien (`title`, `type`, `deprecation`…). HDL y ajoute la méthode HTTP sur les liens-actions (`method`) — ce que HAL traite, lui, via son extension **HAL-FORMS**, avec des formulaires à champs typés — et l'enveloppe `_metadata`. Surtout, HDL **diverge** sur l'embarquement : pas de `_embedded`, les sous-ressources sont imbriquées sous leur nom métier. HDL n'est donc **pas** un sur-ensemble strict de HAL.

**JSON:API** et **Siren** sont plus complets et plus normalisés (enveloppes `data`/`attributes`/`relationships`, classes, actions à champs typés), au prix d'un JSON plus technique à lire. HDL fait le choix inverse : la lisibilité métier d'abord.

Ce que HDL ne fait **pas**, et qu'il assume : pas d'affordances à champs typés (HAL-FORMS et Siren vont plus loin), pas de vocabulaire sémantique (Hydra, JSON-LD), pas de compatibilité HAL stricte. C'est le prix de la simplicité et de l'orientation domaine.

## Exemple complet

Une ressource HDL illustrant les liens-actions, les templates URI et les dépréciations. La sous-ressource `author` porte un `self` — une ressource navigable d'un autre agrégat, dont la représentation ici est partielle. L'identité navigable passe par `self`, pas par un identifiant technique ajouté pour reconstruire l'URI :

```json
{
  "title": "How to paint a bikeshed",
  "body": "The shortest article. Ever.",
  "author": {
    "name": "John Doe",
    "_links": {
      "self": { "href": "http://example.com/authors/9" }
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

## Documentation du format HDL

* [Ressources et sous-ressources](doc/resources.md)
* [Hyperliens](doc/hyperlinks.md)
* [Collections](doc/collection.md)
  * [Pagination](doc/pagination.md)
  * [Tri](doc/sorting.md)
  * [Filtrage](doc/filtering.md)
  * [Recherche](doc/search.md)
* [Métadonnées](doc/metadata.md)
* [Erreurs](doc/errors.md)
