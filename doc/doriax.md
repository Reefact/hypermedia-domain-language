# DORIAX

## 1. Présentation

### 1.1 Qu'est-ce que DORIAX ?

DORIAX (**Domain-Oriented Representation for Inter-Application eXchanges**) est une approche pragmatique basée sur l'architecture REST, conçue pour mieux refléter les concepts métier dans les API. Contrairement à REST classique, souvent limité à des opérations CRUD (Create, Read, Update, Delete), DORIAX introduit une distinction claire entre **ressources** et **services**, et permet l'exposition explicite d'**actions métier** sous forme d'endpoints dédiés.

L'objectif principal de DORIAX est de rendre les API plus naturelles à utiliser, en évitant de tordre REST pour répondre aux besoins métier. Il s'appuie sur les principes fondamentaux de REST tout en intégrant des concepts issus du DDD (Domain-Driven Design) et du CQRS (Command Query Responsibility Segregation), sans pour autant les imposer.

DORIAX assume cependant un écart vis-à-vis de REST « pur » au sens de Fielding : exposer des verbes métier dans l'URL relève d'un hybride pragmatique entre REST et RPC-over-HTTP. DORIAX conserve ce qui a une vraie valeur opérationnelle dans REST — en particulier la seule distinction de verbe qui change quelque chose pour l'infrastructure : **lecture sûre (`GET`) contre effet (`POST`)** — sans s'imposer la pureté de l'interface uniforme ou de l'hypermédia.

### 1.2 Pourquoi DORIAX ?

REST est devenu le standard de facto pour la conception d'API web, mais son implémentation traditionnelle repose principalement sur une logique CRUD, où chaque entité est exposée sous forme de ressource manipulable via `GET`, `POST`, `PUT` et `DELETE`. Si cette approche fonctionne bien pour des systèmes simples, elle montre vite ses limites dans les applications métier plus riches.

**REST impose une approche CRUD inadaptée aux actions métier**

Une API REST classique force souvent à modifier une ressource via `PUT` ou `PATCH`, même lorsque le changement d'état résulte d'une action métier spécifique. Par exemple, une action comme confirmer une commande (`confirm-order`) ne se résume pas à modifier un champ `status`. D'autres mises à jour peuvent être nécessaires, comme la mise à jour de la date de confirmation, la réservation de stock ou l'envoi d'une notification.

Avec un modèle CRUD strict, l'appelant doit connaître et gérer ces modifications en passant manuellement tous les champs affectés (`PATCH /orders/42` avec un body `{ "status": "confirmed", "confirmedAt": "2025-02-11T10:00:00Z", "stockReserved": true }`). Cela introduit un **problème majeur** : une partie de la logique métier se retrouve dans l'appelant, alors qu'elle devrait être entièrement gérée côté backend.

DORIAX résout ce problème en exposant des actions métier explicites (`POST /orders/42:confirm`), ce qui permet au backend de prendre en charge tous les effets métier associés, sans que le client ait à les connaître.

**REST ne distingue pas clairement Ressources et Services**

Dans une API REST classique, tout est souvent exposé sous forme de ressources, même lorsqu'il s'agit d'un processus métier. Cette confusion mène à des conceptions incohérentes où des services métier sont représentés comme des entités factices (`POST /commands` ou `POST /actions`), ce qui brouille la lisibilité de l'API.

DORIAX clarifie cette distinction :
- Une **ressource** représente un élément du domaine, potentiellement composé de plusieurs entités sous-jacentes.
- Un **service** exécute une logique métier sans persister d'état. Il est accessible via `GET` (lecture seule) ou `POST` (avec effet).

**REST gère mal les workflows complexes**

Dans REST standard, plusieurs chemins peuvent mener au même état final, mais chacun peut avoir des effets de bord différents. Cette flexibilité conduit à des API où la modification d'un statut (`PATCH`) ou l'exécution d'une action (`POST /commands`) ne décrivent pas explicitement ce qui se passe côté métier.

Avec DORIAX, chaque action ayant un impact sur l'état métier est clairement identifiée (`POST /teams/42/members:onboard`). Il n'y a pas d'ambiguïté sur quelle opération est effectuée et quels effets secondaires sont pris en charge par le backend. Cela évite aux clients de devoir comprendre la logique métier interne et simplifie la gestion des workflows.

## 2. Ressources, Services et Actions

DORIAX introduit une séparation claire entre **ressources**, **services**, et **actions** afin d'éviter les ambiguïtés présentes dans de nombreuses implémentations REST classiques. Cette distinction permet d'exposer une API qui reflète mieux le métier et qui évite de masquer les processus métier derrière des structures artificielles.

### 2.1 Ressources

Une **ressource** DORIAX représente un élément structuré du domaine, qui peut être interrogé et manipulé via des opérations spécifiques. Une ressource peut correspondre à une entité métier, à un agrégat ou à une vue métier enrichie.

**Exemples de ressources :**
- `GET /users/42` → Récupère l'utilisateur avec son profil, ses préférences, etc.
- `GET /orders/123` → Récupère une commande avec tous ses détails.
- `GET /products/789` → Récupère un produit et ses informations associées.

Une ressource peut être une agrégation de plusieurs entités sous-jacentes, mais elle reste une unité cohérente du point de vue de l'API.

Dans DORIAX, les ressources doivent être cohérentes avec le domaine métier et ne pas être réduites à de simples objets CRUD. Par exemple, au lieu d'exposer une ressource `User` avec des mutations génériques (`PATCH /users/42`), on peut exposer une sous-ressource métier dédiée comme `UserSecurity` :
- `GET /users/42/security` → Récupère les paramètres de sécurité de l'utilisateur.
- `POST /users/42/security:change-password` → Déclenche l'action métier correspondante (modification de la sécurité).

Cela permet de garder une structure claire et d'éviter que l'appelant ait à manipuler des détails internes de la ressource.

### 2.2 Services

Un **service** DORIAX représente un processus métier sans état, qui exécute une logique métier mais qui ne correspond pas à une ressource persistée. Contrairement à une ressource, un service ne possède pas d'identifiant unique et son résultat dépend uniquement des paramètres fournis à l'appel.

Un service ne doit pas être utilisé par défaut, il ne sert que lorsqu'il n'existe aucune ressource métier évidente à laquelle rattacher l'opération. Cela suit la même logique que les services de domaine en DDD : ils doivent être utilisés avec parcimonie.

Un service peut lire, modifier ou créer des ressources, mais il ne doit pas être traité comme une ressource persistée que l'on pourrait récupérer avec un `GET /service/{id}`. Il déclenche une action métier globale qui concerne plusieurs entités, sans être lui-même une entité stockée.

Un service est invoqué par une action (`/service:{action}`, voir §3.1). Le verbe HTTP suit la nature de l'effet : `GET` si le service ne fait que lire ou calculer (sûr, sans effet de bord), `POST` s'il produit un effet.

**Exemples de services :**
- `GET /forex:convert?from=EUR&to=USD&amount=100` → Obtient la conversion d'un montant entre deux devises au taux courant (lecture seule, sans effet de bord).
- `POST /auth:reset-password` → Déclenche l'envoi d'un email de réinitialisation.
- `POST /billing:process-invoices` → Déclenche un traitement global sur plusieurs factures (au lieu d'un `POST /invoices/123:generate` qui concernerait une ressource unique).

### 2.3 Actions

Une **action** exprime une intention métier explicite, appliquée à une ressource ou à un service. Pour qu'elle ne se confonde jamais avec une ressource, DORIAX lui réserve un séparateur dédié dans l'URL : le `/` introduit toujours un nom — une ressource, une sous-ressource ou une collection — que l'on parcourt ; le `:` introduit toujours un verbe métier que l'on invoque.

Sans ce signal, certaines URL sont irrémédiablement ambiguës, car un même mot peut être à la fois un nom et un verbe. Prenons `version` : `/documents/42/version` désigne-t-il *la version* du document — une sous-ressource que l'on consulte — ou l'action de *versionner* le document, c'est-à-dire en créer une nouvelle ? La grammaire de l'URL ne permet pas de trancher. Avec la convention DORIAX, l'ambiguïté disparaît : `/documents/42/version` est sans conteste une sous-ressource, tandis que `/documents/42:version` invoque l'action de versionner.

Cette levée d'ambiguïté profite autant aux humains qu'aux machines. Faute de signal syntaxique, les outils qui analysent une API (générateurs, linters, passerelles, documentation automatique) ne peuvent que *deviner* la nature du dernier segment d'une URL — par heuristiques sur le vocabulaire, voire par des modèles entraînés à classer `/{dernier-segment}` en « ressource » ou « action ». Une inférence reste une inférence : elle échoue précisément sur les cas comme `version` et dépend du nommage. DORIAX rend cette information déterministe — le séparateur porte le sens, plus rien à deviner.

Cette convention est celle retenue par Google pour ses *custom methods* ([AIP-136](https://google.aip.dev/136)), de la forme `POST /v1/users/123:undelete`, à l'échelle de l'ensemble des API Google Cloud. Elle est également conforme à la RFC 3986, qui autorise le `:` comme `pchar` à l'intérieur d'un segment de chemin.

DORIAX reprend d'AIP-136 **à la fois la syntaxe du `:` et sa règle de choix du verbe** : une *custom method* est un `POST`, ou un `GET` lorsqu'elle se borne à lire. DORIAX adopte la même discipline — une action (`:{action}`) est un `GET` si elle est sûre, un `POST` sinon (voir §3.1) — et s'engage à en respecter la sémantique HTTP : sûreté du `GET`, neutralité du `POST`. Une fois l'intention nommée dans l'URL, faire porter au verbe une nuance supplémentaire (`PATCH`, `DELETE` métier…) n'ajoute rien que le suffixe ne dise déjà mieux, tout en détournant la sémantique de ces verbes ; DORIAX y renonce donc délibérément.

#### Actions sur les ressources

Les actions permettent d'effectuer des changements métier sur une ressource sans exposer directement sa structure interne. Contrairement à un `PATCH` ou `PUT` générique où l'appelant doit connaître les champs à modifier, une action encapsule la logique côté serveur.

**Exemple : confirmer une commande**

REST classique :
```http
PATCH /orders/42
{
    "status": "confirmed",
    "confirmedAt": "2025-02-11T10:00:00Z",
    "stockReserved": true
}
```
Ici, l'appelant doit savoir quels champs modifier, ce qui introduit du couplage avec la logique métier interne.

DORIAX :
```http
POST /orders/42:confirm
```
L'action `confirm` encapsule toute la logique métier associée (modification du statut, mise à jour des stocks, notifications, etc.), sans que l'appelant ait à en connaître les détails. Le verbe est `POST` car l'action produit un effet ; l'intention, elle, est déjà portée par le suffixe `:confirm` (voir le choix du verbe en §3.1).

#### Actions sur les services

Les services sont eux aussi invoqués par des actions, sous la forme d'un `GET` (lecture seule) ou d'un `POST` (avec effet).

**Exemple : réinitialisation de mot de passe**
- `POST /auth:reset-password` → Déclenche l'envoi d'un email de réinitialisation.
- `POST /auth:change-password` → Change le mot de passe après vérification du token.
- `POST /billing:process-invoices` → Lance le traitement d'un lot de factures.

**Note — même intention, cadrages différents.** Le changement de mot de passe illustre comment la **cible** varie alors que le verbe reste le même. Côté ressource, un utilisateur authentifié modifie ses propres paramètres : `POST /users/{id}/security:change-password` (action sur une sous-ressource existante). Côté service, dans un flux de réinitialisation où aucun contexte de ressource n'existe (l'utilisateur n'est pas connecté, il agit via un token), l'opération devient une action de service : `POST /auth:change-password`. Le verbe est identique — l'action a un effet, donc `POST` — et c'est la cible (sous-ressource vs service), non la méthode, qui distingue les deux cadrages. La même intention métier se projette sur deux URL différentes selon qu'une ressource cible existe ou non.

**NOTE IMPORTANTE :** _Les exemples de services présentés ici sont volontairement naïfs et génériques pour illustrer le concept. Dans une implémentation réelle, il est essentiel de questionner la nécessité d'un service avant de l'adopter. Dans de nombreux cas, il est possible de rattacher une opération métier à une ressource existante, plutôt que d'introduire un service distinct. Comme en DDD (Domain-Driven Design), les services doivent être utilisés en dernier recours, uniquement lorsqu'une entité métier ou une action sur une ressource ne peut raisonnablement les couvrir._

### 2.4 Synthèse

| **Concept**  | **Définition** | **Exemple** |
|-------------|---------------|------------|
| **Ressource** | Élément structuré du domaine, identifiable par un ID | `GET /orders/42` |
| **Service** | Processus métier sans état, distinct des ressources | `POST /billing:process-invoices` |
| **Action** | Opération métier explicite, appliquée à une ressource ou un service | `POST /orders/42:confirm` |

DORIAX clarifie ces distinctions pour éviter les dérives observées dans certaines implémentations REST classiques. Cette approche permet d'obtenir des API plus cohérentes et plus lisibles tout en restant alignées avec les principes REST.

## 3. Convention de conception d'une API DORIAX

DORIAX suit les principes fondamentaux de REST tout en renforçant l'orientation métier. Cette section définit les conventions utilisées pour structurer une API DORIAX de manière claire et cohérente.

### 3.1 Conventions générales

#### Rappel

DORIAX structure ses endpoints en suivant une logique métier claire et cohérente.

1. **Les ressources et leurs collections**
   - Une **collection de ressources** regroupe plusieurs ressources du même type et permet d'interagir avec elles.
   - Une **ressource** représente une entité métier identifiable.
   - Une **sous-ressource** est un élément directement dépendant d'une ressource principale.
   - Les **actions métier** peuvent être appliquées à une ressource ou à une collection.

2. **Les services**
   - Un **service** exécute une logique métier sans être une ressource persistée.
   - Il ne possède pas d'identifiant et ne peut pas être manipulé comme une ressource.
   - Les fonctionnalités des services ne peuvent être invoquées que via une **action métier**.

#### Format des URL

**Ressources et sous-ressources**
- `/resources` → Collection de ressources.
- `/resources/{id}` → Ressource spécifique.
- `/resources:{action}` → Action appliquée à une collection de ressources.
- `/resources/{id}:{action}` → Action appliquée à une ressource spécifique.
- `/resources/{id}/sub-resources` → Collection de sous-ressources d'une ressource.
- `/resources/{id}/sub-resources/{sub-id}` → Une sous-ressource spécifique.
- `/resources/{id}/sub-resources:{action}` → Une action sur la collection de sous-ressources.
- `/resources/{id}/sub-resources/{sub-id}:{action}` → Une action sur une sous-ressource spécifique.

**Services**
- `/service:{action}` → Exécution d'une action sur un service.

#### Choix du verbe HTTP

DORIAX n'utilise que **deux verbes HTTP** : `GET` pour tout ce qui lit, `POST` pour tout ce qui écrit. Le choix ne dépend que de cette distinction.

> **Règle.** Une opération **sûre** (lecture ou calcul, aucun effet de bord) est un **`GET`** — qu'il s'agisse d'une ressource (`GET /teams/{id}`), d'une collection (`GET /teams`) ou d'une action sûre (`GET /forex:convert`). Toute opération qui **écrit** — création, transition d'état, mutation, **suppression** ou opération de service — est un **`POST`** et porte toujours un nom d'action (`:{action}`). DORIAX **n'emploie ni `PUT` ni `DELETE`** (voir « Verbes délibérément exclus » ci-dessous).

Le suffixe `:{action}` porte l'**intention métier** ; le verbe HTTP ne porte plus que la **nature de l'effet** — sûr (`GET`) ou non (`POST`). DORIAX s'engage à respecter la sémantique HTTP correspondante (sûreté telle que définie par la RFC 9110, §9.2).

**Les deux verbes DORIAX**

| Verbe | L'opération… | Propriété HTTP | Exemples |
|-------|--------------|----------------|----------|
| **GET** | …lit ou calcule, sans aucun effet de bord : ressource, collection, ou action sûre `:{action}`. | *Sûr* (RFC 9110 §9.2.1) | `GET /teams`, `GET /teams/{id}`, `GET /forex:convert?from=EUR&to=USD&amount=100` |
| **POST** | …produit un effet quelconque : crée, déclenche une transition d'état, modifie, supprime (suppression métier), ou exécute une opération de service. Toujours nommée par `:{action}`. | Ni sûr ni idempotent (RFC 9110 §9.3.3) | `POST /teams:initiate`, `POST /orders/{id}:confirm`, `POST /users/{id}:activate`, `POST /teams/{id}:disband`, `POST /teams/{id}/members/{mid}:fire`, `POST /auth:reset-password` |

> **Précisions utiles :**
> - **`GET` doit rester sûr** — aucun effet de bord. C'est précisément pourquoi `reset-password` est `POST` (il envoie un email) et `forex:convert` est `GET`.
> - **`POST` ⇒ effet, à une seule exception près.** L'unique entorse à la règle « `POST` ⇒ effet » est le `POST` employé comme *transport* d'un `GET` qui exige un body (voir la note plus bas) : ce `POST` ne produit aucun effet et **reste sûr**. L'entorse est purement technique — le verbe pallie une limite d'outillage — et l'opération demeure sémantiquement une lecture. La table ci-dessus décrit le cas nominal ; ce cas en est l'exception documentée.
> - **`POST` ne sur-promet rien** : il signifie « traite cette représentation selon la sémantique de la cible ». Ni sûreté, ni idempotence, ni sémantique de body imposée. C'est exactement ce qui en fait le verbe universel des écritures — et la raison pour laquelle DORIAX n'emploie **aucun autre verbe d'écriture** : la sémantique propre de `PATCH` (patch document), `PUT` (remplacement) ou `DELETE` (suppression) serait détournée ou redondante, l'intention étant déjà nommée par le suffixe.
> - **Idempotence des actions `POST`** : la plupart des transitions d'état (`:confirm`, `:activate`, `:demote`) — et la quasi-totalité des suppressions métier (`:disband`, `:fire`) — sont idempotentes **côté métier** : rejouées, elles ne changent rien. Mais c'est une propriété **du domaine**, pas du verbe : `POST` n'est pas idempotent au sens HTTP, et aucun intermédiaire (proxy, client à retry) ne peut s'y fier sans connaître le métier. Pour rendre une action rejouable sans risque sur un retry réseau, DORIAX s'appuie donc sur une **clé d'idempotence** portée par la requête (en-tête `Idempotency-Key`, de fait répandu et en cours de standardisation à l'IETF — `draft-ietf-httpapi-idempotency-key-header`), **et non sur le verbe**. C'est la contrepartie assumée de l'abandon de `DELETE`, qui offrait, lui, une idempotence native au niveau du protocole (voir « Verbes délibérément exclus »).
> - **Toute écriture est nommée** : DORIAX n'utilise ni le `POST /collection` nu, ni `PUT`, ni `DELETE`. Création, modification et suppression prennent toutes la forme `POST /…:{action}` (voir convention ci-dessous).

> **Note sur les `GET` avec body (`POST` utilisé à la place de `GET`) :** _Dans certains cas complexes, un appel `GET` peut nécessiter un payload. Cependant, certaines API REST empêchent de passer un body dans une requête `GET`. Dans ces situations, DORIAX privilégie `POST` pour ces requêtes tout en maintenant une logique métier explicite, afin de favoriser une API au vocabulaire métier explicite plutôt que technique. Ce `POST` est alors une **lecture déguisée** : il doit rester **sûr** (aucun effet de bord) et constitue la **seule exception** à la règle « `POST` ⇒ effet » énoncée plus haut._

#### Verbes délibérément exclus — `PUT` et `DELETE`

DORIAX **n'emploie ni `PUT` ni `DELETE`** : toute écriture — création, modification, suppression — prend la forme d'une action nommée `POST /…:{action}`. Deux ressorts l'établissent.

**La cohérence de la grammaire le commande.** DORIAX pose que *toute création porte un nom métier* (détaillé plus bas), parce que `POST` est polymorphe — sur une collection, il pourrait créer, importer, cloner — et parce que le domaine a un mot pour chaque intention. La modification et la suppression relèvent de la même règle ; les nommer maintient la grammaire homogène, là où nommer la seule création tout en laissant la méthode HTTP porter le reste introduirait une asymétrie arbitraire. La règle se généralise donc : *toute écriture porte un nom métier*, et il ne reste que `GET` et `POST /…:{action}`.

**Le langage du domaine le conforte.** En DDD, toute écriture passe par les invariants d'un agrégat : elle porte une intention métier, que l'URL nomme. Le domaine fournit ainsi, pour chaque écriture, le verbe que l'action réclame — `:disband`, `:fire`, `:replace` — et la cohérence de la grammaire en fait la règle générale.

**L'intention prime sur la forme de la ressource.** DORIAX nomme l'action d'après l'intention métier que porte l'opération. « Retirer un membre d'une équipe » est une action métier — `POST /teams/{id}/members/{mid}:fire` — parce que l'intention déclenche une transition à orchestrer : révocation d'accès, réaffectation des tâches, notification, audit, invariante « pas le dernier owner ». La suppression de la ligne en base reste un *effet de bord parmi d'autres*, un détail d'implémentation. De même, un remplacement (l'ancien `PUT /preferences`) devient une action nommée (`:replace`, ou mieux un verbe du domaine).

**Le coût, assumé.** La contrepartie est double :
- **Idempotence protocolaire.** `PUT` et `DELETE` sont idempotents *par le protocole* ; un proxy ou un client à retry peut les rejouer sans risque, sans rien savoir du métier. Avec `POST`, cette garantie remonte au niveau applicatif : DORIAX la porte par la clé d'idempotence (`Idempotency-Key`, déjà requise). L'idempotence d'un `:fire` rejoué reste réelle, mais comme propriété métier garantie par la clé — au niveau du domaine plutôt que du verbe.
- **Interop déclarative.** L'outillage déclaratif (Terraform et apparentés) et les passerelles s'appuient sur la sémantique des méthodes pour distinguer lecture et mutation. En traitant *toute* écriture comme une action `POST`, DORIAX réduit cette lisibilité, sur l'ensemble de l'API et plus seulement sur ses opérations spécifiques.

**Divergence assumée vis-à-vis d'AIP-136.** DORIAX emprunte à AIP-136 la syntaxe du `:` et la règle binaire `GET`/`POST` des *custom methods*, mais s'écarte de sa recommandation centrale : AIP-136 demande de **privilégier les méthodes standard** (Get/List/Create/Update/Delete, donc `GET`/`POST`/`PATCH`/`DELETE`) et de réserver les *custom methods* aux cas qu'elles n'expriment pas — au point d'interdire de réutiliser un verbe standard dans le nom d'une *custom method*. DORIAX **inverse** cette hiérarchie : l'action nommée devient la règle et les méthodes standard d'écriture disparaissent. La citation d'AIP-136 (§2.3) vaut donc pour la *syntaxe*, non pour ce choix-ci — une divergence revendiquée, au même titre que l'écart plus large vis-à-vis de REST « pur ».

#### Nommage des actions

Le séparateur `:` rend déterministe la **nature** du segment — nom ou verbe (voir §2.3) ; encore faut-il que le **verbe** lui-même soit nommé de façon cohérente, sans quoi deux équipes écriront trois mots différents pour la même intention, et le déterminisme s'arrête au `:`. Les contraintes de **forme** ci-dessous sont mécaniquement vérifiables (linter, passerelle) ; le **choix du bon verbe** relève du langage du domaine (*Ubiquitous Language*) et ne se contrôle qu'à la relecture.

**Forme (vérifiable automatiquement)**
- **`kebab-case`**, sans exception : `:reset-password`, `:process-invoices`.
- **Verbe à l'impératif** : `:confirm`, jamais `:confirmation` (nom) ni `:confirmed` (état).
- **Pas de préfixe redondant avec la cible** : `POST /orders/{id}:confirm`, jamais `:confirm-order` — la ressource est déjà dans le chemin.

**Sens (relève du domaine)**
- **Priorité au terme métier.** C'est la projection sur l'URL de la convention de code DORIAX (vocabulaire fonctionnel, pas technique) : `:disband` plutôt que `:delete`, `:onboard` plutôt que `:add-member`, `:initiate` plutôt que `:new`. Le verbe doit venir, autant que possible, du langage du *bounded context*.
- **Repli générique toléré, jamais par défaut.** Lorsque le domaine n'offre honnêtement aucun terme spécifique, un verbe générique (`:update`, `:close`…) est acceptable — c'est un dernier recours assumé, pas un choix de facilité. La question à se poser d'abord reste toujours : *le métier a-t-il un mot pour ça ?*
- **Harmoniser les verbes récurrents, sans liste fermée.** Pour les opérations transverses qui reviennent partout (créer, annuler, archiver…), il est recommandé qu'une équipe tienne un petit lexique de référence, afin d'éviter que `:cancel`, `:annul` et `:void` cohabitent pour une même intention. Ce lexique **guide**, il n'enferme pas : chaque contexte reste libre d'étendre avec son propre vocabulaire métier, qui prime toujours.

> **Note — renommer un verbe, et le rôle de HDL.** _Faire porter l'intention par l'URL crée un couplage : corriger une coquille ou faire évoluer le terme métier change le chemin, et casse donc les clients qui l'ont codé en dur. C'est une rupture de contrat d'API ordinaire, qui se traite par les pratiques habituelles de non-régression : exposer l'ancien chemin en alias, le déprécier, le versionner. Surtout, c'est précisément ce que **HDL** neutralise : un client qui suit les liens hypermédia fournis dans les réponses, au lieu de construire les URL lui-même, absorbe ce changement de manière transparente._

#### Convention DORIAX — toute écriture porte un nom métier

Là où REST exprime la création par un `POST` sur la collection (`POST /teams`), DORIAX impose un verbe métier explicite (`POST /teams:initiate`). Et ce qui vaut pour la création vaut pour **toute** écriture : modification et suppression sont elles aussi des actions nommées (`POST /…:{action}`), jamais des `PUT`/`DELETE` nus (voir « Verbes délibérément exclus » ci-dessus). Dans une URL DORIAX, le dernier segment est toujours **soit un nom (une ressource), soit un verbe métier (une action)** — jamais une écriture implicite.

**Pourquoi nommer plutôt que déduire de la méthode ?** Parce que `POST` est *polymorphe* : sur une collection, il pourrait signifier « créer à vide », « importer », « cloner »… son intention n'est pas déductible de la méthode. La nommer lève l'ambiguïté. Cette règle a deux vertus :
- elle **nomme** l'intention plutôt que de la déduire de la méthode HTTP ;
- elle **absorbe sans rupture** les cas où plusieurs façons de faire coexistent. C'est la projection sur l'URL du pattern des *constructeurs nommés* utilisé côté code (`Team.Initiate()`, `Team.ImportFrom(...)`, `Team.CloneOf(...)`) :
  - `POST /teams:initiate` → création vierge ;
  - `POST /teams:import` → depuis un annuaire (AD/LDAP) ;
  - `POST /teams:clone` → par duplication d'une équipe existante.

Le `POST /teams` nu de REST ne pourrait pas distinguer ces trois chemins, et conduirait à l'asymétrie peu lisible où la création d'origine est anonyme (`POST /teams`) mais où l'on ajoute plus tard `POST /teams:import` à côté. Nommer dès le départ évite cette dette.

### 3.2 Exemples concrets

Cette section illustre les conventions de DORIAX avec des exemples concrets.

**Exemples pour les ressources et leurs actions**
- `GET /teams` → Obtient la liste des équipes.
- `GET /teams/{id}` → Obtient une équipe spécifique.
- `POST /teams:initiate` → Crée une équipe vierge (terme du domaine).
- `POST /teams:import` → Crée une équipe à partir d'un annuaire (AD/LDAP).
- `POST /teams:clone` → Crée une équipe par duplication d'une équipe existante.
- `GET /users/{id}/security` → Obtient les paramètres de sécurité d'un utilisateur.
- `POST /users/{id}/security:reset` → Réinitialise les paramètres de sécurité.
- `POST /orders/{id}:confirm` → Confirme une commande.
- `POST /users/{id}:activate` → Active un utilisateur.
- `POST /products/{id}:discount` → Applique une réduction sur un produit.
- `POST /teams/{id}/members:onboard` → Ajoute un membre à une équipe avec onboarding.
- `POST /teams/{id}/members/{mid}:fire` → Retire un membre de l'équipe (avec la logique métier associée).
- `POST /teams/{id}:disband` → Dissout une équipe (terme du domaine).
- `GET /user-activity?range=last-30-days` → Statistiques d'activité (ressource **calculée** : un nom, donc `/`, en lecture seule — un service, lui, s'invoquerait avec `:`).

> _Note : `:fire` et `:disband` sont des **actions métier** — retrait d'un membre avec ses effets de bord, dissolution d'une équipe — donc `POST /…:{action}`. Le réflexe « supprimer une ligne, donc `DELETE` » raisonne sur la forme de la ressource, pas sur l'intention : la suppression en base n'est qu'un des effets à orchestrer (notification, cascade, transition d'état, invariante « pas le dernier owner »). C'est l'intention qui nomme l'action ; le `DELETE` technique n'est qu'un détail d'implémentation (voir « Verbes délibérément exclus », §3.1)._

**Exemples pour les services**
- `POST /billing:process-invoices` → Lance un traitement global sur plusieurs factures.

### 3.3 Gestion des réponses HTTP

DORIAX suit les standards HTTP pour structurer ses réponses en apportant des précisions métier adaptées. Cette section fournit une vue des différentes réponses HTTP DORIAX, en regroupant les cas d'usage courants.

**Format de représentation.** _Tout corps de réponse d'une API DORIAX (ressource, collection, et autres contenus de succès) suit le format **HDL** (Hypermedia Domain Language), spécifié séparément (voir [HDL](README.md)). HDL est un profil hypermédia bâti sur HAL : il en reprend la structure (`_links`, `self`, `href`) et l'enrichit (méthode HTTP portée par le lien, titres, dépréciations, templates d'URI). Les exemples de payload de cette section n'en utilisent qu'un sous-ensemble — les `_links` de base — et restent donc du HDL valide. La représentation des erreurs suit également HDL (voir [Erreurs](doc/errors.md))._

| Opération | Codes HTTP possibles | Contenu du payload |
|-----------|---------------------|-------------------|
| **GET** (lecture d'une ressource ou action sûre `:{action}`, ou `POST` remplaçant un `GET` nécessitant un body) | `200 OK`, `403 Forbidden`, `404 Not Found` | Payload contenant la ressource demandée (format HDL) ou une erreur |
| **POST** — l'action **crée une ressource** (sur une collection `POST /coll:{action}`, ou via un service qui crée) | `201 Created`, `400 Bad Request`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity` | `Location` dans l'en-tête HTTP, et optionnellement un payload (HDL) contenant les liens vers la ressource créée |
| **POST** — l'action **produit un autre effet** (transition d'état, mutation, **suppression métier**, opération de service ; `:{action}`) | `200 OK`, `202 Accepted`, `204 No Content`, `400 Bad Request`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity` | Aucun payload en cas de `204`, un payload (HDL) en cas de `200`, ou un lien de suivi en cas de `202` (traitement asynchrone) |
| **Requête asynchrone (`202 Accepted`)** | `202 Accepted`, `400 Bad Request`, `403 Forbidden` | `Location` dans l'en-tête HTTP pour suivre le traitement, et optionnellement un payload (HDL) avec des liens HATEOAS |
| **Erreur métier (`422 Unprocessable Entity`)** | `422 Unprocessable Entity` | Payload d'erreur en format HDL contenant les détails de l'erreur |

👉 **Cette table couvre les principaux cas d'usage des réponses HTTP DORIAX, mais des ajustements peuvent être nécessaires selon les contextes métier et technique spécifiques. Les autres statuts HTTP standards restent applicables selon les besoins.**

**Note — répartition des statuts.** _Chaque cause de rejet a son statut, et la répartition est **exclusive** : `400` pour une requête invalide (mal formée ou champ invalide), `401` pour un appelant non authentifié, `403` pour un appelant authentifié mais non autorisé, `404` pour une ressource cible absente, `409` pour un conflit avec l'état courant (concurrence, doublon, version obsolète), `422` pour une **violation de règle métier**. `422` ne sert qu'à cela : aucun autre rejet ne l'emprunte, et une règle métier ne se signale jamais autrement que par `422`._

**Note — codes transverses.** _Trois codes s'appliquent à **toute** opération et ne sont donc pas répétés ligne à ligne : `400 Bad Request` (requête invalide du fait du client : syntaxe malformée ou champ invalide — y compris une query string malformée sur un `GET`), `401 Unauthorized` (appelant **non authentifié**) et `403 Forbidden` (authentifié mais **non autorisé**). La distinction `401`/`403` est normative : ne pas renvoyer `403` à un appelant simplement non authentifié._

**Note — `409` vs `422`.** _`409 Conflict` signale un conflit avec l'**état courant** de la ressource — accès concurrent, doublon, version obsolète : l'opération échoue parce que l'état a changé, et un nouvel essai après relecture peut réussir. `422 Unprocessable Entity` signale une **règle métier violée** sur une requête par ailleurs bien formée : rejouer à l'identique ne sert à rien tant que la règle s'applique. Le départage ne tient pas à la consultation de l'état — une règle qui s'appuie sur l'état courant (p. ex. « pas le dernier owner ») reste un `422`, jamais un `409` — mais à la **nature** du rejet : collision d'état → `409`, transgression d'une règle → `422`._

**Note — `422 Unprocessable Entity`.** _Ce code est utilisé par DORIAX pour signaler une erreur métier lorsque l'action demandée est logiquement invalide dans le contexte métier, même si la requête est techniquement bien formée._

**Note — validation et `400` (et non `422`).** _DORIAX classe la **validation du contenu de la requête** — champ manquant, format invalide, valeur hors bornes — en `400`, et réserve `422` aux seules **violations de règle métier**. Le partage : un rejet **décidable sur la seule requête** relève de `400` ; un rejet qui n'a de sens qu'au regard d'une **règle du domaine** relève de `422`. `422` garde ainsi un sens unique et stable — « le domaine a refusé » —, jamais « un champ est mal rempli ». La précision du cas reste portée par le corps HDL (`code`, `errors[].source.pointer`), de sorte que `400` couvre sans ambiguïté la requête illisible **comme** le champ invalide. C'est un parti pris **assumé**, et l'un des deux usages répandus : **Spring** renvoie `400` par défaut à l'échec de validation d'un corps de requête (exception `MethodArgumentNotValidException`), là où **Ruby on Rails** (validations ActiveRecord, statut `:unprocessable_entity`) et **Laravel** (Form Requests) renvoient `422`. DORIAX suit la convention de Spring, pour la cohérence interne décrite ci-dessus._

👉 **Conventions observées** : Spring renvoie `400` (`MethodArgumentNotValidException`) ; [Rails](https://guides.rubyonrails.org/active_record_validations.html) et [Laravel](https://laravel.com/docs/validation) renvoient `422`.

**Note — concurrence optimiste (`ETag` / `If-Match` / `412`).** _Le `409` « version obsolète » évoqué ci-dessus suppose un mécanisme pour détecter le conflit. DORIAX recommande le standard HTTP : le serveur expose un `ETag` sur le `GET`, le client le renvoie dans `If-Match` sur l'écriture suivante ; si la ressource a changé entre-temps, le serveur répond `412 Precondition Failed` (ou `428 Precondition Required` s'il exige l'en-tête). C'est, côté lecture/écriture, le pendant de la clé d'idempotence vue côté rejouabilité (§3.1)._

**Note — corps porteur du résultat (écriture à résultat non re-lisible).** _Les lignes du tableau supposent qu'un `201`/`200` ne transporte que des liens vers une ressource par ailleurs consultable. Une classe étroite d'écritures y fait exception : celles dont le corps porte un **résultat qu'aucune lecture ultérieure ne réexpose** — secret affiché une seule fois, token à usage unique, artefact généré non interrogeable (p. ex. `POST /api-keys:create` renvoyant la clé en clair). Pour elles, le corps n'est pas un confort mais **l'unique résultat de l'opération**, et `return=minimal` n'y est pas le défaut. Voir §3.4, « Exception : l'écriture dont le corps est le résultat »._

#### Exemples détaillés

1. **Création réussie :**
   ```http
   HTTP/1.1 201 Created
   Content-Type: application/hdl+json

   {
     "_links": {
       "self": { "href": "/teams/123" }
     }
   }
   ```

2. **Action exécutée sans contenu de retour :**
   ```http
   HTTP/1.1 204 No Content
   ```

3. **Requête acceptée pour un traitement asynchrone :**
   ```http
   HTTP/1.1 202 Accepted
   Content-Type: application/hdl+json

   {
     "_links": {
       "status": { "href": "/jobs/42/status" }
     }
   }
   ```

4. **Erreur métier avec `422 Unprocessable Entity` :**

   _Contexte : Un appel `POST /teams/42/members/22:demote` tente de rétrograder un membre **propriétaire** (`owner`) en **membre simple**. Cependant, il est le **seul propriétaire** restant de l'équipe, ce qui rend cette action invalide._

   ```http
   HTTP/1.1 422 Unprocessable Entity
   Content-Type: application/problem+json

   {
     "type": "http://example.com/docs/errors/last-owner-restriction",
     "title": "Last owner restriction.",
     "status": 422,
     "detail": "Cannot demote the last owner of the team.",
     "instance": "/teams/42/members/22:demote",
     "code": "LAST_OWNER_RESTRICTION"
   }
   ```

**Note — format d'erreur.** _Les réponses d'erreur DORIAX suivent le **format d'erreur HDL**, un sur-ensemble de la **RFC 9457** (« Problem Details for HTTP APIs », qui a remplacé la RFC 7807 en juillet 2023) servi en `application/problem+json` : il en conserve les champs `type`, `title`, `status`, `detail`, `instance` et les complète d'extensions — code applicatif stable (`code`), erreurs multiples (`errors[]`), liens de reprise (`_links`), métadonnées d'échange (`_metadata`). L'exemple ci-dessus n'en montre qu'une forme simple ; voir [Erreurs](doc/errors.md) pour la spécification complète._

👉 **Référence sur les codes HTTP** : la sémantique normative est définie par la **RFC 9110 §15** ; [MDN HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) en donne un panorama plus accessible.

### 3.4 Séparation commande / requête

DORIAX sépare nettement les opérations qui **modifient** l'état du système (les commandes) de celles qui le **lisent** (les requêtes). Cette discipline est la **séparation commande/requête (CQS)** appliquée au niveau de l'API : une opération change l'état *ou* renvoie une donnée, elle ne fait pas les deux à la fois — à une réserve près, bornée et nommée plus bas, pour le résultat qu'une écriture est seule à produire et qu'aucune lecture ne reconstitue.

**CQS (et non CQRS).** _DORIAX applique la séparation commande/requête (CQS) au seul niveau des opérations : une règle de conception, sans infrastructure dédiée ni second modèle. CQRS (Command Query Responsibility Segregation) va plus loin — deux **modèles** distincts, un côté écriture validé par le domaine, un côté lecture en projections, éventuellement sur un autre store et en cohérence différée — et DORIAX s'en tient en deçà. CQS **prépare le terrain** pour un CQRS ultérieur tout en restant une étape autonome. La distinction est structurelle autant que terminologique : elle conditionne la garantie de relecture décrite plus bas._

Cette séparation est une **conséquence naturelle de la grammaire DORIAX**, pas un ajout : une action à effet est un `POST` (la commande), une lecture est un `GET` (la requête). Les règles ci-dessous en explicitent les conséquences sur les réponses.

#### Règle de réponse des écritures

> **Par défaut**, une écriture (`POST /…:{action}`) **ne renvoie pas la représentation complète et mise à jour** de la ressource. Elle renvoie un **statut HTTP** confirmant l'opération, accompagné le cas échéant de **liens hypermédia** ou d'un **identifiant de suivi** : le `Location` d'un `201 Created`, le lien de statut d'un `202 Accepted`. Le client effectue ensuite un `GET` s'il a besoin de l'état à jour — ou demande la représentation directement via `Prefer: return=representation` (voir plus bas).

« Pas de représentation complète » n'interdit pas un corps de réponse léger — les `_links` d'un `201`, un lien de suivi d'un `202` (cohérent avec §3.3) ; cela interdit seulement de **renvoyer l'entité mise à jour** comme le ferait un CRUD classique.

Avantages :
- éviter des retours de données inutiles après une modification ;
- laisser le client récupérer exactement ce dont il a besoin, via un `GET` adapté ;
- clarifier les responsabilités en distinguant commandes (actions) et requêtes (lecture).

#### Le défaut DORIAX : le minimum utile, et le coût qu'il implique

Le comportement **par défaut** d'une écriture DORIAX est de renvoyer le **minimum utile** : de quoi confirmer l'opération et retrouver son résultat — un statut, le `Location` d'un `201`, des liens — **jamais la représentation complète** de la ressource. « Minimum » ne veut pas dire « corps vide » : un `201 Created` accompagné de son `Location` et de ses `_links` (voir §3.3) *est* une réponse minimale ; c'est l'**entité mise à jour** que l'on ne renvoie pas, pas le moindre octet.

Ce choix de défaut découle de CQS (une commande ne renvoie pas de donnée), mais il se justifie aussi en soi : le défaut doit être le cas **le moins coûteux pour le plus grand nombre**. Renvoyer systématiquement l'entité ferait payer à tous les clients une sérialisation que beaucoup ne lisent pas, et les inciterait à **dépendre** de cette représentation — recréant le couplage action↔représentation que DORIAX cherche à éviter. Le défaut « minimum utile » pousse au contraire le client à lire, via un `GET` adapté, exactement ce dont il a besoin.

**Le coût, en contrepartie.** Forcer un `GET` de relecture après chaque écriture, c'est **deux allers-retours réseau** là où un retour de représentation en aurait fait un seul. Pour une transition d'état où le client ne veut que le nouvel état, c'est un coût net. DORIAX l'accepte comme contrepartie du découplage, mais le rend **négociable** : le client peut demander à l'écriture de lui renvoyer directement l'entité.

**La négociation, via l'en-tête `Prefer` (RFC 7240).** Cet en-tête définit une paire de valeurs qui permet au client de basculer par rapport au défaut du serveur :

| En-tête de requête | Demande au serveur de… | Usage en DORIAX |
|--------------------|------------------------|-----------------|
| `Prefer: return=minimal` | …ne renvoyer que le minimum (statut, liens). | C'est **déjà le défaut DORIAX** : inutile à envoyer, sauf pour l'expliciter. |
| `Prefer: return=representation` | …renvoyer la représentation complète et à jour de la ressource. | La **dérogation** : à envoyer quand on veut éviter le `GET` de relecture (cas sensibles à la latence). |

`Prefer` est une **préférence**, pas un ordre : le serveur peut l'ignorer. S'il l'honore, il **devrait** le signaler en réponse avec `Preference-Applied: return=representation`. Et comme `Prefer` est facultatif, DORIAX **doit documenter son défaut** — c'est fait ici : `return=minimal`.

Par exemple, après `POST /teams/42/members:onboard`, on récupère en général les seules informations utiles via `GET /teams/42/members/33` plutôt que de recharger toute la liste — sauf si l'application a réellement besoin de l'entité, auquel cas elle l'obtient directement avec `Prefer: return=representation` sur l'écriture, ou la liste complète via un `GET` explicite.

#### Exception : l'écriture dont le corps *est* le résultat

Le défaut « minimum utile » et la relecture par `GET` reposent sur une prémisse tacite : **le résultat d'une écriture est reconstructible par une lecture ultérieure**. Une classe étroite d'écritures la met en défaut — celles qui produisent, *au moment de l'acte*, une valeur qu'aucune lecture ne réexpose :

- un **secret affiché une seule fois** : la clé en clair renvoyée par `POST /api-keys:create`, qui n'est plus jamais ré-exposée ensuite (seule son empreinte est conservée côté serveur) ;
- un **token à usage unique** ou un code de récupération ;
- un **artefact généré** non persisté sous une forme interrogeable.

Pour cette classe, le corps de réponse n'est pas un confort qui éviterait un `GET` : il est l'**unique résultat de l'opération**. Trois conséquences, qui sont autant de **renversements** du défaut :

- **`return=minimal` n'y est pas le défaut.** La réponse porte le matériel produit ; un client qui la perd n'a **aucun recours** — aucun `GET` ne le lui rendra. C'est le seul cas où une commande renvoie légitimement une donnée substantielle, et la réserve annoncée en tête de section.
- **La relecture ne s'applique pas.** Le pattern « écris puis relis » (ci-dessous) suppose un résultat re-lisible ; ici, par construction, il n'y en a pas.
- **Le corps fait partie du résultat idempotent.** Comme rien ne réexpose la valeur, un rejeu de la même requête (même `Idempotency-Key`, §3.1) doit la **restituer à l'identique** : pour cette classe — et pour elle seule — la réponse est mémorisée et rejouée *verbatim*. Le détail du mécanisme relève de la spécification d'idempotence ; DORIAX se borne à poser que, pour ces écritures, le corps est constitutif de l'issue.

**Cadrage.** DORIAX traite ce cas comme une **exception nommée et bornée** à la règle d'écriture, et non en le requalifiant en service (§2.2) : ouvrir la voie « service » à toute écriture désireuse de renvoyer un corps réintroduirait le couplage action↔représentation que la §3.4 écarte. La question à se poser d'abord reste : *cette valeur est-elle réellement non reconstructible par une lecture ?* Si une lecture peut la rendre, l'écriture suit le défaut — minimal, puis `GET`.

#### Relecture et cohérence (read-after-write)

La règle « écris, puis relis » suppose deux choses. D'abord que le résultat *soit* re-lisible : la classe décrite à l'exception ci-dessus y échappe par construction. Ensuite que le `GET` qui suit une écriture renvoie bien l'état que cette écriture vient de produire. **C'est vrai tant que le modèle de lecture est immédiatement cohérent** — ce qui est le cas en CQS pur, où lecture et écriture partagent le même modèle. Le jour où une API évoluerait vers un vrai CQRS avec lecture en cohérence différée, ce `GET` pourrait renvoyer un état antérieur : le pattern de relecture immédiate cesserait alors d'être sûr et devrait être traité (jeton de version, attente de propagation, ou lien de suivi). DORIAX reste en CQS et ne contracte donc pas cette difficulté ; il la signale pour que le passage éventuel à CQRS soit fait en connaissance de cause.

#### Et HDL

C'est ici aussi que [HDL](../README.md) referme le sujet proprement. Plutôt que de laisser le client *deviner* quelle URL relire après une écriture, la réponse lui fournit les **liens** vers ce qu'il peut consulter ou faire ensuite : la relecture devient un lien donné par le serveur, pas une URL reconstruite côté client. La séparation commande/requête et HDL répondent ainsi au même manque.
