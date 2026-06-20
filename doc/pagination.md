# Pagination

## Principe

Une collection-ressource renvoie ses éléments par **tranches**. En HDL, la pagination repose sur deux mécanismes complémentaires :

- la **navigation par liens** (`first`, `prev`, `next`, `last`) — le **contrat** : le client suit les liens, il ne fabrique jamais d'URL ;
- l'objet **`_pagination`** — une **aide d'affichage** optionnelle : totaux, numéro de tranche courante, taille de tranche, lorsqu'ils sont disponibles.

HDL est **agnostique de la stratégie de pagination du serveur**. Le client suit `next.href` sans savoir si l'URL contient `?page=3` ou `?after=<curseur>` : le serveur choisit son mécanisme, le client n'en voit rien.

> **En clair —** ce que le client fait, c'est **suivre des liens**. Comment le serveur récupère la tranche derrière ces liens ne le regarde pas.

## La navigation : le contrat

Les liens de navigation vivent dans `_links` :

- `next` / `prev` — la tranche suivante / précédente. **Toujours** le moyen de parcourir, quelle que soit la stratégie. `prev` est absent sur la première tranche, `next` sur la dernière.
- `first` / `last` — les extrémités. Présents **lorsque la collection peut les atteindre** : `last` suppose que le serveur connaît la fin (un total, ou une borne de dernière tranche).

Le client suit ces liens tels quels. Une URL de page (`?page=3`) ou de curseur (`?after=xyz`) lui est **opaque** : il ne lit que `href`.

## `_pagination` : l'aide d'affichage

`_pagination` est un **membre réservé optionnel** qui décrit la tranche, pour que le client l'affiche sans rien calculer. Ses champs sont **tous optionnels** et dépendent de ce que la collection sait fournir :

- `pageSize` — la taille d'une tranche ;
- `totalItems` — le nombre total d'éléments, **si le serveur le connaît** ;
- `totalPages` — le nombre de tranches, dérivé de `totalItems` et `pageSize` ;
- `currentPage` — le numéro de la tranche courante, **si la collection est numérotée**.

Une collection inclut **ce qu'elle peut**, rien de plus. Ces champs ne sont pas le contrat : la navigation l'est. Un client peut ignorer `_pagination` entièrement et se contenter de `next`/`prev`.

## Capacités, pas stratégie

Ce qu'une collection expose ne dépend pas de *comment* le serveur pagine, mais de **deux capacités** qu'elle a ou non :

- **le total** — le serveur connaît (et accepte de payer) le compte des éléments → `totalItems` / `totalPages`, et un `last` ;
- **le saut** — le client peut atteindre une tranche arbitraire → un pager numéroté, le numéro de tranche étant un paramètre de vue réglable comme les autres critères.

Une collection peut avoir les deux (pager « 1 2 3 … 47 »), une seule, ou aucune (parcours « charger plus », `next` seul).

> **Le numéro de page est un marqueur de capacité, pas de stratégie.** Afficher « page 1, 2, 3… » n'impose **pas** l'offset côté serveur : il faut seulement un **total** et de quoi **sauter** à une tranche. Un serveur peut servir un pager numéroté par keyset (bornes de tranche pré-calculées) sans aucun `OFFSET`. Le client n'y voit que des liens et des numéros.

## Les trois stratégies serveur, et leur place dans HDL

Côté serveur, trois familles — invisibles au client :

| Stratégie | URL typique | Force | Limite | En HDL |
|---|---|---|---|---|
| **Offset / page** | `?page=2&pageSize=10` | saut et total faciles | lent sur gros offsets, instable sous écritures concurrentes | `first`/`prev`/`next`/`last`, `_pagination` complet |
| **Keyset / curseur** | `?after=<curseur>` | stable, rapide à l'échelle | pas de saut ni de total faciles | `next`/`prev`, `_pagination` minimal (ni `last`, ni total) |
| **Token de continuation** | `?pageToken=…` | comme le keyset | comme le keyset | comme le keyset |

La stratégie ne change **que** deux choses : **quelles capacités** la collection expose (total, saut), et **ce que contiennent les `href`**. Le contrat — suivre les liens — reste identique.

## Exemples

**Pagination par page — pager numéroté complet** (capacités : total + saut)

```json
{
  "articles": [
    { "title": "Article 21", "_links": { "self": { "href": "http://example.com/articles/21" } } }
  ],
  "_links": {
    "self":  { "href": "http://example.com/articles?page=3" },
    "first": { "href": "http://example.com/articles?page=1" },
    "prev":  { "href": "http://example.com/articles?page=2" },
    "next":  { "href": "http://example.com/articles?page=4" },
    "last":  { "href": "http://example.com/articles?page=100" }
  },
  "_pagination": {
    "pageSize": 10,
    "totalItems": 1000,
    "totalPages": 100,
    "currentPage": 3
  }
}
```

**Pagination par curseur — parcours « charger plus »** (aucune des deux capacités)

```json
{
  "articles": [
    { "title": "Article 21", "_links": { "self": { "href": "http://example.com/articles/21" } } }
  ],
  "_links": {
    "self": { "href": "http://example.com/articles?after=eyJpZCI6MjB9" },
    "next": { "href": "http://example.com/articles?after=eyJpZCI6MzB9" }
  },
  "_pagination": {
    "pageSize": 10
  }
}
```

Le second n'a ni `first`/`last`, ni totaux, ni `currentPage` : la collection ne sait ni compter ni sauter. Le client affiche « charger plus » et suit `next`. **Même contrat, capacités différentes.**

## Récapitulatif

- La **navigation par liens** (`next`/`prev`, et `first`/`last` si la collection sait sauter) est le **contrat** : le client suit, il ne fabrique pas.
- **`_pagination`** est une **aide d'affichage optionnelle** ; ses champs (`pageSize`, `totalItems`, `totalPages`, `currentPage`) sont présents **selon les capacités** de la collection.
- HDL est **agnostique de la stratégie serveur** (offset, curseur, token) : elle ne détermine que les capacités exposées et le contenu des `href`, jamais le contrat.

Voir : [Collections](collection.md).
