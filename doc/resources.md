# Ressources et sous-ressources

## Principe

HDL représente une ressource comme l'**objet du domaine** qu'elle modélise : ses propriétés portent des noms métier, et ses sous-ressources sont imbriquées **sous leur nom métier**, là où on les attend. Là où HAL range tout ce qui est adressable dans un compartiment `_embedded`, HDL ne le fait pas : la structure suit la forme de l'objet, pas celle du protocole.

> **En clair —** une ressource HDL se lit comme l'objet qu'elle décrit. Une équipe *a* des membres : on les trouve sous `members`, pas dans un compartiment séparé.

## Donnée ou sous-ressource : le critère d'adressabilité

Au sein d'une ressource, un objet imbriqué est soit de la **donnée**, soit une **sous-ressource**. Le critère qui tranche est l'**adressabilité** : l'objet a-t-il une identité propre — un `self`, une URI — sur laquelle on peut revenir et, éventuellement, agir ?

- **Attribut simple** (`numeroCommande`, `statut`) → propriété.
- **Value object ou entité interne sans identité propre** (`montantTotal { valeur, devise }`) → propriété imbriquée. Pas d'URI, donc pas une ressource.
- **Objet adressable** (porte un `self`) → sous-ressource, imbriquée sous son nom métier.

> **À ne pas confondre —** le critère n'est *pas* la frontière d'agrégat du DDD tactique, ni « donnée intrinsèque vs donnée référencée ». C'est l'adressabilité, et elle se lit d'un coup d'œil : **un `self`, ou pas de `self`**.

## `self` : le marqueur de projection

La présence d'un `self` porte deux informations à la fois :

- un objet **sans** `self` est une **donnée complète** et autoritative : il n'y a rien d'autre à aller chercher ;
- un objet **avec** `self` est une **projection** d'une ressource adressable : il en expose la part utile au contexte, et son `self` signale qu'il existe davantage à cette URI — d'autres propriétés, des actions.

Un seul signal porte donc ce que HAL répartissait entre `_embedded` (« ceci est une ressource ») et le caractère potentiellement partiel d'une représentation embarquée (« ceci peut être incomplet »).

> **En clair —** inutile de marquer qu'une représentation est partielle : son `self` le dit. Pas de `self`, c'est complet ; un `self`, va voir si tu veux le reste.

## Sous-ressources et actions

Une sous-entité peut être exposée comme une **sous-ressource adressable**, avec son propre `self` et ses propres actions — par exemple une ligne modifiable à `/commandes/CMD-2024-5821/lignes/2`. L'action s'adresse alors à la sous-ressource, pas à la racine.

Ce n'est pas une projection du modèle tactique. En DDD tactique, on passe par la racine d'agrégat ; ici, l'**URI** adresse la sous-ressource, tandis que côté serveur l'implémentation charge l'agrégat, applique la modification via sa racine et le persiste — les invariants restent à la racine. L'API expose des ressources et des actions **parlantes du domaine**, en suivant la sémantique REST des ressources, sans imposer le goulot de la racine dans les URIs ni faire fuiter les constructions tactiques dans le contrat. C'est une API **orientée domaine**, pas une transposition du DDD tactique.

## Exemple

Une équipe et ses membres. Chaque membre est inliné sous `members` : ses attributs propres à l'appartenance (`role`, `joinedAt`) sont de la donnée ; son `self` en fait une sous-ressource (l'appartenance, `/teams/team-42/members/1`), qui porte aussi ses actions ; et le développeur, lui, est un **autre agrégat**, simplement **référencé** par un lien.

```json
{
  "name": "Core Platform",
  "members": [
    {
      "role": "Lead Developer",
      "joinedAt": "2025-01-15",
      "_links": {
        "self": { "href": "/teams/team-42/members/1" },
        "developer": { "href": "/developers/dev-123" },
        "promouvoir": { "href": "/teams/team-42/members/1:promouvoir", "method": "POST" }
      }
    },
    {
      "role": "Developer",
      "joinedAt": "2025-03-01",
      "_links": {
        "self": { "href": "/teams/team-42/members/2" },
        "developer": { "href": "/developers/dev-456" }
      }
    }
  ],
  "_links": {
    "self": { "href": "/teams/team-42" }
  }
}
```

> **À ne pas confondre —** `member` et `developer` sont deux ressources distinctes. `member` est l'**appartenance** : elle n'existe que dans le contexte de l'équipe, et porte `role`, `joinedAt`, ses actions. `developer` est la **personne** : un agrégat autonome, seulement référencé. `role` et `joinedAt` appartiennent à l'appartenance, jamais au développeur.

## Exposer un « plusieurs » : inline ou lien

Une ressource a souvent *plusieurs* de quelque chose — une équipe a des membres, un client a des commandes. HDL offre deux façons de l'exposer, et le choix n'est **pas** une question de taille.

**Inline — un tableau nu sous son nom métier.** Quand le « plusieurs » est un **composant borné, toujours livré avec la ressource** : les membres d'une équipe *sont* l'équipe. On les rend en entier, sous leur nom métier, **sans** pagination ni tri — c'est une propriété multivaluée, pas une vue requêtable. C'est le cas de `members` dans l'exemple ci-dessus.

**Lien — vers une collection-ressource.** Quand ce sont des **entités autonomes**, qui vivent et s'interrogent indépendamment de la ressource : les commandes ne sont pas *dans* le client ; ce sont des ressources de plein droit qu'on **filtre** par client. La ressource expose alors une **entrée** vers cette collection, éventuellement déjà filtrée :

```json
{
  "nom": "Camille Durand",
  "_links": {
    "self":      { "href": "/clients/9" },
    "commandes": { "href": "/commandes?filter[client]=9" }
  }
}
```

> **Le critère n'est pas « est-ce gros ? » mais « est-ce que ça vit indépendamment ? ».** La taille n'est qu'un symptôme : un composant borné reste inline même à cinquante éléments ; des entités autonomes passent par un lien même à trois. La question « et si la sous-collection devient énorme ? » ne se pose donc jamais — si c'est énorme, c'est que ce n'était pas un composant, mais une collection-ressource à lier.

> **Et la pagination, le tri, le filtre ?** Ils n'appartiennent qu'à la **collection-ressource**, au bout du lien (voir [Collections](collection.md)). Un tableau inline n'est jamais paginé : s'il a besoin de l'être, c'est qu'il devait être un lien.

**Parallèle avec le DDD.** Ce partage **ressemble** aux frontières d'agrégat — composant interne (inline) contre autre agrégat (lien) — et ce n'est pas un hasard : un agrégat est défini autour de ce qui vit et charge ensemble, soit ce qu'on livre ensemble dans une représentation. Mais le critère du *format* reste l'**usage** — « toujours voulu avec le parent, et borné » contre « adressé et interrogé indépendamment » —, pas une projection du modèle tactique. Une équipe qui ne fait pas de DDD applique la même règle sans rien connaître des agrégats.

## Identité : `self`, pas `id` technique

Puisque `self` porte déjà l'identité d'une ressource, HDL n'expose pas d'identifiant technique de substitution en doublon. Un identifiant de compteur (`"id": "team-42"`) **duplique** ce que dit déjà `self` et **invite le client à fabriquer des URIs** (`/teams/{id}`) — un couplage contraire à HATEOAS. On n'expose un identifiant que s'il est **métier** (un numéro de commande, un ISBN), porteur de sens pour le domaine ; jamais une clé de substitution.

## Divergence avec HAL : pas de `_embedded`

L'absence de `_embedded` est une **divergence assumée** avec HAL, pas un oubli.

`_embedded` rend un service précis : permettre à un client **générique** — un navigateur d'API HAL, un SDK auto-généré, un outil de documentation agnostique du domaine — de parser, suivre et mettre en cache les ressources de façon **uniforme**, via un compartiment dédié. Lorsque les consommateurs d'une API sont des clients qui **connaissent le domaine** (typiquement, les fronts d'une même organisation), ce client générique n'existe pas : `_embedded` et les liens jumeaux qu'il impose deviennent une complexité payée pour une capacité que personne n'exploite. L'abolir est alors **cohérent avec la cible** de HDL, pas un raccourci.

S'y ajoute une duplication que HDL évite : en HAL, chaque ressource embarquée apparaît **deux fois** — en lien dans `_links.<rel>` et en entier dans `_embedded.<rel>`. HDL n'a pas ce doublon.

Le prix à payer est réel, et doit être assumé :

- **HDL n'est pas un sur-ensemble de HAL.** Un client HAL strict cherche les sous-ressources dans `_embedded` ; en HDL, il verra une sous-ressource comme une donnée opaque. C'est une divergence — contrairement au format d'erreur HDL, qui reste, lui, un sur-ensemble de RFC 9457.
- **Le client générique parcourt l'arbre.** Pour collecter les ressources, il ne lit plus un compartiment dédié : il descend la structure en repérant les `_links.self`. Le signal se déplace de « est-ce sous `_embedded` ? » vers « cet objet a-t-il un `self` ? ».

Au fond, les deux formats ne diffèrent pas par la validité mais par la **priorité** : HAL est *graphe-de-ressources-first* — tout l'adressable vit dans `_links`/`_embedded`, au service du client générique ; HDL est *objet-du-domaine-first* — la forme naturelle de l'objet prime, l'adressabilité n'étant qu'un sous-signal porté par `self`, au service du lecteur métier.

Parce que HDL s'écarte ainsi du modèle de HAL, il porte son **propre type de média** : `application/hdl+json`, et non `application/hal+json`. Un client HAL parserait le JSON sans erreur, mais lirait les sous-ressources comme de l'**état opaque**, pas comme des ressources. Le type de média annonce donc la bonne grille de lecture : en HDL, c'est le `self` qui marque une sous-ressource, jamais un compartiment `_embedded`.

Ce choix d'imbrication existe d'ailleurs « dans la nature », mais **non assumé**. L'API IIS Administration de Microsoft sert du `application/hal+json` tout en imbriquant ses ressources liées — un pool d'applications au sein d'un site web — comme des propriétés d'état portant leur propre `self`, sans jamais recourir à `_embedded` ni nommer l'écart. HDL fait le même choix d'affichage, plus proche du métier que du protocole ; il le **nomme et le justifie**, là où d'autres le pratiquent en silence.

## Pas de CURIEs

HAL définit les **CURIEs** (`_links.curies`) : des préfixes qui **namespacent** les relations personnalisées et leur associent une **URI de documentation** déréférençable (`acme:widget` → `https://docs.example.com/rels/widget`). HDL n'en a pas l'usage.

Les relations de liens, en HDL, sont des **noms du domaine** — `commandes`, `developer`, `promouvoir`, `change-payment-method`. Pour un client qui connaît le domaine, ces noms sont déjà porteurs de sens : ni le namespacing ni la documentation déréférençable des CURIEs n'apportent quoi que ce soit. C'est, comme `_embedded`, une commodité pour le client **générique** — celui qui ignore le domaine — que HDL ne cible pas.

Les relations standard (`self`, `next`, `prev`, `first`, `last`) conservent leur sens IANA ; les autres sont des termes métier. Aucune des deux familles n'appelle de préfixe.

## Récapitulatif

| Cas | Représentation | Marqueur |
|---|---|---|
| Attribut simple | propriété | — |
| Value object / entité interne sans identité | propriété imbriquée | pas de `self` |
| Sous-ressource adressable | objet sous son nom métier | `_links.self` |
| Ressource d'un autre agrégat, référencée | lien | `_links.<rel>` |
| Ressource d'un autre agrégat, inlinée (perf) | objet sous son nom métier | `_links.self` (projection) |

Règle unique : **pas de `self`, c'est une donnée complète ; un `self`, c'est une sous-ressource — la projection d'une ressource adressable.** Aucun compartiment `_embedded`.

Pour un *plusieurs* : **tableau inline** sous son nom métier si c'est un composant borné de la ressource ; **lien** vers une collection-ressource (filtrée au besoin) si ce sont des entités autonomes. Jamais de pagination sur un tableau inline.
