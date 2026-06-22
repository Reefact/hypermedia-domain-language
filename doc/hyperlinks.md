# Hyperliens

## Principe

Dans [Hypermedia Domain Language](../README.md) (HDL), les liens d'une ressource sont regroupés dans `_links`, un **membre réservé** (préfixe `_`, voir [Métadonnées](metadata.md)). HDL reprend ici le mécanisme de **HAL** : le même conteneur `_links`, le même objet lien minimaliste. Il y ajoute un seul attribut — `method`, pour les liens-actions — et s'écarte de HAL sur un point traité ailleurs : l'absence de `_embedded` (voir [Ressources et sous-ressources](resources.md)).

Un lien associe une **relation** (la clé) à une **cible** (un objet lien portant son `href`). Le client **suit** les liens fournis par le serveur ; il ne fabrique jamais d'URL lui-même. C'est le cœur de HATEOAS, et la raison d'être de tout ce qui suit : une URL est **opaque** au client, qui lit `href` et s'y rend sans en interpréter la structure.

```json
{
  "_links": {
    "self":  { "href": "/commandes/CMD-2024-5821" },
    "payer": { "href": "/commandes/CMD-2024-5821:payer", "method": "POST" }
  }
}
```

`_links` apparaît à la racine d'une ressource, mais aussi sur chacune de ses **sous-ressources** et sur chaque **élément d'une collection** — partout où il y a une identité (`self`) à exposer ou des liens à offrir.

> HDL s'appuie sur **HAL** (*JSON Hypertext Application Language*, `draft-kelly-json-hal`) pour la forme de ses liens, et en conserve les attributs. La divergence porte sur l'imbrication des sous-ressources, pas sur l'objet lien lui-même.

## Ce que `_links` contient

`_links` n'est pas un répertoire des ressources liées : il porte ce qui relève de la **navigation** et de l'**action** sur la ressource courante —

- **`self`** — son identité (voir plus bas) ;
- **la navigation** d'une collection : `first` / `prev` / `next` / `last` (voir [Pagination](pagination.md)) ;
- **les liens de composition** : `sort` / `filter` / `search`, *templated* (voir [Tri](sorting.md), [Filtrage](filtering.md), [Recherche](search.md)) ;
- **les actions** : un verbe métier portant sa `method` — `payer`, `annuler`, `change-payment-method`.

Une **ressource liée**, elle, ne s'y range pas comme une entrée parmi d'autres. HDL la montre **imbriquée sous son nom métier**, marquée d'un `self` : un `author`, un `client` sont des objets du domaine, pas des lignes du compartiment des liens.

```json
{
  "title": "How to paint a bikeshed",
  "author": {
    "name": "John Doe",
    "_links": { "self": { "href": "/authors/9" } }
  },
  "_links": { "self": { "href": "/articles/1" } }
}
```

> **C'est la divergence avec HAL.** HAL est *graphe-de-ressources-first* : tout l'adressable vit dans `_links`. HDL est *objet-du-domaine-first* : une ressource liée est un champ nommé par le métier, et `self` n'en est que le marqueur d'adressabilité. Ranger `author` dans `_links` reproduirait le réflexe que HDL écarte.

Une ressource liée n'apparaît **comme un lien** dans `_links` que lorsqu'elle est **référencée sans être imbriquée** — un pointeur vers un autre agrégat qu'on n'inline pas, ou l'entrée vers une collection :

```json
{
  "nom": "Camille Durand",
  "_links": {
    "self":      { "href": "/clients/9" },
    "commandes": { "href": "/commandes?filter[client]=9" }
  }
}
```

`commandes` n'inline rien : c'est l'**entrée** vers une collection-ressource, légitimement portée par un lien. Le choix — imbriquer sous le nom métier ou pointer par un lien — relève de [Ressources et sous-ressources](resources.md) (« inline ou lien ») ; cette page décrit la **forme** des liens, pas ce qu'on inline.

## L'objet lien

Un objet lien décrit une cible. Son seul membre obligatoire est `href` ; tous les autres sont optionnels.

| Attribut | Type | Rôle |
|---|---|---|
| `href` | string | **obligatoire** — l'URI cible, ou un gabarit d'URI si `templated` |
| `templated` | booléen | `href` est un gabarit RFC 6570 à étendre (défaut : `false`) |
| `method` | string | *extension HDL* — méthode HTTP d'un lien-action ; absent ⇒ `GET` |
| `type` | string | indice du type de média attendu à la cible |
| `hreflang` | string | langue de la ressource cible (`fr`, `en`…) |
| `title` | string | libellé humain, pour l'IHM ou la documentation |
| `deprecation` | string (URL) | marque le lien déprécié ; pointe la doc de migration |
| `name` | string | clé secondaire distinguant plusieurs liens d'une même relation |

Le plus souvent, un lien se réduit à `href` :

```json
{ "self": { "href": "/articles/1" } }
```

Les attributs `type`, `hreflang`, `title`, `deprecation` et `name` sont **hérités de HAL** et conservent leur sens. Seul `method` est **propre à HDL**.

## Relations : standard et domaine

La clé d'un lien est sa **relation** — ce que la cible représente par rapport à la ressource courante. HDL en distingue deux familles.

**Relations standard (IANA).** `self`, `next`, `prev`, `first`, `last` conservent le sens du registre IANA des relations de lien (RFC 8288). `self` désigne la ressource elle-même ; les quatre autres servent la navigation d'une collection (voir [Pagination](pagination.md)).

**Relations du domaine.** Tout le reste est un **terme métier** : une référence portée par un lien (`commandes`, `developer`) ou un verbe d'action (`payer`, `change-payment-method`). Lorsqu'elles portent plusieurs mots, ces relations sont en `kebab-case`, à l'image des actions.

**Pas de CURIEs.** Une **CURIE** (*Compact URI*) est un mécanisme de HAL pour les relations personnalisées. L'idée : pour qu'une relation maison soit comprise d'un client générique, on l'identifie par une **URI** pointant vers sa documentation ; comme l'écrire en entier à chaque clé serait lourd, on déclare un **préfixe** une fois, puis on l'emploie en abrégé.

```json
"_links": {
  "curies": [{ "name": "ex", "href": "https://example.com/rels/{rel}", "templated": true }],
  "ex:invoice": { "href": "/invoices/42" }
}
```

La clé `ex:invoice` se « déplie » alors en `https://example.com/rels/invoice`, l'URL où un client générique lirait le sens de cette relation. HDL n'en a pas l'usage : ses relations sont des **noms du domaine** (`commandes`, `change-payment-method`), déjà parlants pour un client qui connaît le métier. Ni le namespace ni la documentation déréférençable n'y ajoutent rien — c'est, comme `_embedded`, une commodité pour le client **générique** que HDL ne cible pas. Voir aussi [Ressources et sous-ressources](resources.md).

Une relation peut être présente une seule fois, ou répétée — voir « Liens multiples » plus bas.

## `self` : l'identité navigable

`self` est la relation qui porte l'**identité** d'une ressource : son URI canonique. Sa présence sur un objet imbriqué en fait une **sous-ressource navigable** ; son absence, une donnée complète. C'est le pivot de la représentation HDL, traité en détail dans [Ressources et sous-ressources](resources.md).

## Liens-ressources et liens-actions : `method`

Un lien est de deux natures, et `method` les sépare :

- **un lien-ressource** se *lit* — on déréférence `href` en `GET`. C'est le cas par défaut : **un lien sans `method` est un `GET`**.
- **un lien-action** s'*exécute* — il déclenche une opération. Sa méthode est alors **explicite** et non-`GET`, typiquement `POST`.

```json
{
  "_links": {
    "self":  { "href": "/commandes/CMD-2024-5821" },
    "payer": { "href": "/commandes/CMD-2024-5821:payer", "method": "POST" }
  }
}
```

> **À ne pas confondre —** une URL ne dit pas, à elle seule, s'il faut la *lire* ou l'*exécuter*. `{ "href": "/commandes/…:payer" }` sans `method` serait un `GET` ; c'est `method: "POST"` qui en fait une action. Dès qu'un lien produit un effet, sa méthode est explicite.

Pour le client, la règle est donc **déterministe** et ne laisse rien à deviner : il lit `method` ; présent, il l'emploie ; absent, c'est un `GET`. L'absence n'est pas une inconnue mais la **valeur par défaut**, choisie parce que la plupart des liens — `self`, navigation, références — sont des lectures, `method` ne marquant que les actions à effet.

`method` est une **extension HDL** : un objet lien HAL ne porte pas de méthode — HAL délègue les affordances à son extension **HAL-FORMS**, et d'autres formats (JSON Hyper-Schema) portent aussi la méthode sur le lien. HDL fait le choix d'un attribut simple, suffisant pour qu'un lien-action soit autoporteur.

Une action **sûre** — une lecture exprimée comme un verbe — reste mécaniquement un `GET` : son lien n'a donc **pas** de `method`. Seules les actions à **effet** en portent une. Le suffixe `:{verbe}` de l'URI est le signal pour le lecteur humain ; `method` est le signal pour la machine.

## Templates d'URI : `templated`

Quand `templated` vaut `true`, `href` n'est pas une URL prête à l'emploi mais un **gabarit d'URI** (RFC 6570) que le client **étend** en y injectant des valeurs avant de l'utiliser.

```json
{ "sort": { "href": "http://example.com/articles{?sort}", "templated": true } }
```

C'est le mécanisme des **liens de composition** : `sort`, `filter`, `search` exposent un gabarit que le client remplit pour bâtir sa requête, sans fabriquer l'URL à la main (voir [Tri](sorting.md), [Filtrage](filtering.md), [Recherche](search.md)).

> Un gabarit RFC 6570 n'exprime pas toute forme de paramètre. `{?sort}` et `{?search}` sont natifs ; la forme par champ du filtrage (`filter[propriété]=valeur`) ne l'est pas — le lien `filter` signale alors surtout que le filtrage est **disponible**, la forme exacte relevant de la connaissance du domaine (voir [Filtrage](filtering.md)).

## Liens multiples : un tableau

Une relation peut désigner **plusieurs** cibles. Sa valeur est alors un **tableau** d'objets lien, que le client départage par leurs attributs (`type`, `hreflang`, ou la clé secondaire `name`). Une même ressource offre ainsi son téléchargement en plusieurs formats et langues :

```json
{
  "_links": {
    "self": { "href": "/articles/1" },
    "download": [
      { "href": "/articles/1/download?type=pdf&lang=fr", "type": "application/pdf", "hreflang": "fr", "title": "PDF (FR)" },
      { "href": "/articles/1/download?type=pdf&lang=en", "type": "application/pdf", "hreflang": "en", "title": "PDF (EN)" },
      { "href": "/articles/1/download?type=md&lang=fr",  "type": "text/markdown",   "hreflang": "fr", "title": "Markdown (FR)" }
    ]
  }
}
```

> Une relation porte **soit** un objet lien unique, **soit** un tableau d'objets lien. Le client traite les deux cas : une cible isolée, ou une liste à filtrer sur les attributs.

## Dépréciation : `deprecation`

`deprecation` marque un lien en voie de retrait. Sa valeur est une **URL** déréférençable qui documente l'échéance et la marche à suivre. Le lien reste utilisable tant qu'il est présent, mais le client — ou le développeur qui lit la réponse — est averti qu'il disparaîtra.

Lorsqu'un lien est remplacé, HDL n'ajoute rien de spécial : on **expose la relève à côté** du lien déprécié, et la migration se lit dans l'URL de `deprecation`. Ici `buy` est déprécié, `purchase` prend le relais :

```json
{
  "_links": {
    "self": { "href": "/articles/1" },
    "buy": {
      "href": "/articles/1:buy",
      "method": "POST",
      "deprecation": "https://example.com/docs/deprecations/buy",
      "title": "Buy Article (Deprecated)"
    },
    "purchase": {
      "href": "/articles/1:purchase",
      "method": "POST"
    }
  }
}
```

## `_links` dans une réponse d'erreur

Le même objet `_links`, le même parseur, servent aussi sur une réponse d'erreur : il y porte les **transitions de reprise** que le serveur propose après l'échec (retenter, corriger, ouvrir un ticket). Les liens-actions y obéissent aux mêmes règles — `method` explicite si non-`GET`. Le détail est dans [Erreurs](errors.md).

## Récapitulatif

| Attribut | Obligation | Origine | Rôle |
|---|---|---|---|
| `href` | obligatoire | HAL | URI cible, ou gabarit si `templated` |
| `templated` | optionnel | HAL | `href` est un gabarit RFC 6570 |
| `method` | optionnel | **HDL** | méthode d'un lien-action ; absent ⇒ `GET` |
| `type` | optionnel | HAL | type de média attendu à la cible |
| `hreflang` | optionnel | HAL | langue de la cible |
| `title` | optionnel | HAL | libellé humain |
| `deprecation` | optionnel | HAL | URL ; marque le lien déprécié |
| `name` | optionnel | HAL | clé secondaire pour une relation répétée |

Règles : un lien **sans `method`** est un `GET`, un lien-**action** porte une méthode explicite ; les relations sont **standard (IANA)** ou **du domaine** (`kebab-case`), sans CURIEs ; une relation porte **un objet lien** ou **un tableau** d'objets lien. Le client **suit** les liens — il ne construit pas d'URL.

Voir : [Ressources et sous-ressources](resources.md), [Collections](collection.md), [Métadonnées](metadata.md), [Erreurs](errors.md).
