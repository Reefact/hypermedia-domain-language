# Métadonnées

## Le rôle de `_metadata`

Dans Hypermedia Domain Language (HDL), une réponse transporte deux natures d'information : les **données métier** (la ressource elle-même) et les informations sur **l'échange** qui vient de se produire — qui, quand, quelle version. Ces dernières sont regroupées dans un objet `_metadata`, placé à la **racine** de la réponse.

`_metadata` est un **dictionnaire extensible** : chaque API décide des clés qu'elle expose. C'est la **même enveloppe sur les réponses de succès et d'erreur** : même membre `_metadata`, même rôle, lu de la même façon par le client (voir la spécification de gestion des erreurs HDL).

> **En clair —** `_metadata` parle de *la réponse* — sa traçabilité, son horodatage, sa version, ou toute autre information sur l'**échange** — pas du *contenu métier*. Un champ qui décrit la ressource n'a rien à y faire ; un champ qui décrit l'appel y a sa place.

## Pourquoi le préfixe `_` ?

Comme `_links`, `_metadata` est un **membre structurel réservé** de HDL, distinct des données métier. Le préfixe `_` :

- **isole** ces informations de service du vocabulaire métier : une ressource peut porter ses propres champs `metadata` ou `links` sans jamais entrer en collision avec les membres réservés `_metadata` et `_links` ;
- **signale** un emplacement standard, identique d'une réponse à l'autre, qu'un client peut lire sans rien connaître du domaine.

## Structure générale

**Exemple — une ressource avec ses métadonnées**

```json
{
  "id": "1",
  "title": "Article 1",
  "_links": {
    "self": { "href": "http://example.com/articles/1" }
  },
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-05-25T14:30:00Z",
    "apiVersion": "1.0.0"
  }
}
```

Ici le serveur a choisi d'exposer un identifiant de corrélation, un horodatage de réponse et une version d'API. Rien de plus qu'un choix : HDL ne définit, ne réserve ni n'attend aucune clé de `_metadata` — le format fixe l'enveloppe, pas son contenu.

## Extensibilité

Une API reste libre d'ajouter ses propres clés, **à condition qu'elles décrivent l'échange** : le nœud ayant répondu, un identifiant de trace distribuée, un identifiant de corrélation amont… À l'inverse, ce qui décrit le *contenu* — la pagination d'une collection, par exemple — n'a pas sa place ici : cela relève des données ou de `_links`, et dispose de ses propres mécanismes. Deux recommandations :

- **Des clés stables et nommées.** Une clé de `_metadata` devient un point d'observabilité durable — tableaux de bord, filtres, corrélation entre systèmes. La renommer souvent casse l'outillage en aval.
- **Uniquement du *client-safe*.** Tout ce qui entre dans `_metadata` est **envoyé au client**. On n'y met donc que des informations sûres à exposer : jamais de secret, jamais une donnée interne réservée au support — ces dernières restent côté serveur.

## Récapitulatif

HDL ne définit qu'**un seul** élément ici : l'objet `_metadata` lui-même. Il ne normalise **aucune clé** — leur choix appartient entièrement à l'API.

| Aspect | Règle |
|---|---|
| Emplacement | racine de la réponse |
| Type | objet (dictionnaire de clés) |
| Préfixe `_` | membre structurel réservé, isolé du métier |
| Obligation | optionnel — y compris son contenu |
| Portée | réponses de succès **et** d'erreur (même enveloppe) |
| Contenu | libre ; clés **stables** et **client-safe** recommandées |
