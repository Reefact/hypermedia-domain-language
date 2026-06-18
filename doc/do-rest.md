# DORIAX

## 1. Présentation

### 1.1 Qu'est-ce que DORIAX ?

DORIAX (**Domain-Oriented Representation for Inter-Application eXchanges**) est une approche pragmatique basée sur l'architecture REST, conçue pour mieux refléter les concepts métier dans les API. Contrairement à REST classique, souvent limité à des opérations CRUD (Create, Read, Update, Delete), DORIAX introduit une distinction claire entre **ressources** et **services**, et permet l'exposition explicite d'**actions métier** sous forme d'endpoints dédiés.

L'objectif principal de DORIAX est de rendre les API plus naturelles à utiliser, en évitant de tordre REST pour répondre aux besoins métier. Il s'appuie sur les principes fondamentaux de REST tout en intégrant des concepts issus du DDD (Domain-Driven Design) et du CQRS (Command Query Responsibility Segregation), sans pour autant les imposer.

DORIAX assume cependant un écart vis-à-vis de REST « pur » au sens de Fielding : exposer des verbes métier dans l'URL relève d'un hybride pragmatique entre REST et RPC-over-HTTP. DORIAX conserve ce qui a une vraie valeur opérationnelle dans REST — en particulier la sémantique HTTP des verbes (sûreté, idempotence) — sans s'imposer la pureté de l'interface uniforme ou de l'hypermédia.

### 1.2 Pourquoi DORIAX ?

REST est devenu le standard de facto pour la conception d'API web, mais son implémentation traditionnelle repose principalement sur une logique CRUD, où chaque entité est exposée sous forme de ressource manipulable via `GET`, `POST`, `PUT` et `DELETE`. Si cette approche fonctionne bien pour des systèmes simples, elle montre vite ses limites dans les applications métier plus riches.

**REST impose une approche CRUD inadaptée aux actions métier**

Une API REST classique force souvent à modifier une ressource via `PUT` ou `PATCH`, même lorsque le changement d'état résulte d'une action métier spécifique. Par exemple, une action comme confirmer une commande (`confirmOrder`) ne se résume pas à modifier un champ `status`. D'autres mises à jour peuvent être nécessaires, comme la mise à jour de la date de confirmation, la réservation de stock ou l'envoi d'une notification.

Avec un modèle CRUD strict, l'appelant doit connaître et gérer ces modifications en passant manuellement tous les champs affectés (`PATCH /orders/42` avec un body `{ "status": "confirmed", "confirmedAt": "2025-02-11T10:00:00Z", "stockReserved": true }`). Cela introduit un **problème majeur** : une partie de la logique métier se retrouve dans l'appelant, alors qu'elle devrait être entièrement gérée côté backend.

DORIAX résout ce problème en exposant des actions métier explicites (`PATCH /orders/42:confirm`), ce qui permet au backend de prendre en charge tous les effets métier associés, sans que le client ait à les connaître.

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
- `PATCH /users/42/security:change-password` → Déclenche l'action métier correspondante (modification en place de la sécurité).

Cela permet de garder une structure claire et d'éviter que l'appelant ait à manipuler des détails internes de la ressource.

### 2.2 Services

Un **service** DORIAX représente un processus métier sans état, qui exécute une logique métier mais qui ne correspond pas à une ressource persistée. Contrairement à une ressource, un service ne possède pas d'identifiant unique et son résultat dépend uniquement des paramètres fournis à l'appel.

Un service ne doit pas être utilisé par défaut, il ne sert que lorsqu'il n'existe aucune ressource métier évidente à laquelle rattacher l'opération. Cela suit la même logique que les services de domaine en DDD : ils doivent être utilisés avec parcimonie.

Un service peut lire, modifier ou créer des ressources, mais il ne doit pas être traité comme une ressource persistée que l'on pourrait récupérer avec un `GET /service/{id}`. Il déclenche une action métier globale qui concerne plusieurs entités, sans être lui-même une entité stockée.

Un service est invoqué par une action (`/service:{action}`, voir §3.1). Le verbe HTTP suit la nature de l'effet : `GET` si le service ne fait que lire ou calculer (sûr, sans effet de bord), `POST` s'il produit un effet.

**Exemples de services :**
- `GET /forex:convert?from=EUR&to=USD&amount=100` → Obtient la conversion d'un montant entre deux devises au taux courant (lecture seule, sans effet de bord).
- `POST /auth:reset-password` → Déclenche l'envoi d'un email de réinitialisation.
- `POST /billing:process-invoices` → Déclenche un traitement global sur plusieurs factures (au lieu d'un `PATCH /invoices/123:generate` qui concernerait une ressource unique).

### 2.3 Actions

Une **action** exprime une intention métier explicite, appliquée à une ressource ou à un service. Pour qu'elle ne se confonde jamais avec une ressource, DORIAX lui réserve un séparateur dédié dans l'URL : le `/` introduit toujours un nom — une ressource, une sous-ressource ou une collection — que l'on parcourt ; le `:` introduit toujours un verbe métier que l'on invoque.

Sans ce signal, certaines URL sont irrémédiablement ambiguës, car un même mot peut être à la fois un nom et un verbe. Prenons `version` : `/documents/42/version` désigne-t-il *la version* du document — une sous-ressource que l'on consulte — ou l'action de *versionner* le document, c'est-à-dire en créer une nouvelle ? La grammaire de l'URL ne permet pas de trancher. Avec la convention DORIAX, l'ambiguïté disparaît : `/documents/42/version` est sans conteste une sous-ressource, tandis que `/documents/42:version` invoque l'action de versionner.

Cette levée d'ambiguïté profite autant aux humains qu'aux machines. Faute de signal syntaxique, les outils qui analysent une API (générateurs, linters, passerelles, documentation automatique) ne peuvent que *deviner* la nature du dernier segment d'une URL — par heuristiques sur le vocabulaire, voire par des modèles entraînés à classer `/{dernier-segment}` en « ressource » ou « action ». Une inférence reste une inférence : elle échoue précisément sur les cas comme `version` et dépend du nommage. DORIAX rend cette information déterministe — le séparateur porte le sens, plus rien à deviner.

Cette convention n'est pas isolée : c'est celle retenue par Google pour ses *custom methods* (AIP-136), de la forme `POST /v1/users/123:undelete`, appliquée à l'échelle de l'ensemble des API Google Cloud. Elle est également conforme à la RFC 3986, qui autorise le `:` comme `pchar` à l'intérieur d'un segment de chemin.

DORIAX emprunte à AIP-136 cette **syntaxe** du `:` ; il ne reprend en revanche pas sa règle de choix du verbe. Là où les *custom methods* de Google s'en tiennent à `POST` (ou `GET`), DORIAX fait porter au verbe HTTP la nature de l'effet (voir §3.1) et s'engage à en respecter la sémantique (sûreté, idempotence).

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
PATCH /orders/42:confirm
```
L'action `confirm` encapsule toute la logique métier associée (modification du statut, mise à jour des stocks, notifications, etc.), sans que l'appelant ait à en connaître les détails. Le verbe est `PATCH` car l'action modifie la commande en place (voir le choix du verbe en §3.1).

#### Actions sur les services

Les services sont eux aussi invoqués par des actions, sous la forme d'un `GET` (lecture seule) ou d'un `POST` (avec effet).

**Exemple : réinitialisation de mot de passe**
- `POST /auth:reset-password` → Déclenche l'envoi d'un email de réinitialisation.
- `POST /auth:change-password` → Change le mot de passe après vérification du token.
- `POST /billing:process-invoices` → Lance le traitement d'un lot de factures.

**Note — même intention, cadrages différents.** Le changement de mot de passe illustre comment le cadrage dicte le verbe. Côté ressource, un utilisateur authentifié modifie ses propres paramètres : `PATCH /users/{id}/security:change-password` (modification en place d'une ressource existante). Côté service, dans un flux de réinitialisation où aucun contexte de ressource n'existe (l'utilisateur n'est pas connecté, il agit via un token), l'opération devient une action de service : `POST /auth:change-password`. La même intention métier se projette différemment selon qu'une ressource cible existe ou non.

**NOTE IMPORTANTE :** _Les exemples de services présentés ici sont volontairement naïfs et génériques pour illustrer le concept. Dans une implémentation réelle, il est essentiel de questionner la nécessité d'un service avant de l'adopter. Dans de nombreux cas, il est possible de rattacher une opération métier à une ressource existante, plutôt que d'introduire un service distinct. Comme en DDD (Domain-Driven Design), les services doivent être utilisés en dernier recours, uniquement lorsqu'une entité métier ou une action sur une ressource ne peut raisonnablement les couvrir._

### 2.4 Synthèse

| **Concept**  | **Définition** | **Exemple** |
|-------------|---------------|------------|
| **Ressource** | Élément structuré du domaine, identifiable par un ID | `GET /orders/42` |
| **Service** | Processus métier sans état, distinct des ressources | `POST /billing:process-invoices` |
| **Action** | Opération métier explicite, appliquée à une ressource ou un service | `PATCH /orders/42:confirm` |

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

#### Choix du verbe HTTP (par l'effet, pas par la cible)

Le suffixe `:{action}` porte l'**intention métier** ; le verbe HTTP porte la **nature de l'effet**, et DORIAX s'engage à en respecter la sémantique HTTP — la sûreté et l'idempotence telles que définies par la RFC 9110, §9.2. La cible (service, ressource ou collection) ne *décide* pas du verbe : elle ne fait que se corréler avec l'effet le plus courant.

| Verbe | L'action… | Propriété | Exemples |
|-------|-----------|-----------|----------|
| **GET** | …ne fait que lire ou calculer, sans aucun effet de bord. | *Sûr* | `GET /user-activity?range=last-30-days`, `GET /forex:convert?from=EUR&to=USD&amount=100` |
| **POST** | …crée une ressource ou une sous-ressource, ou déclenche un effet sur un service. | Non idempotent | `POST /teams:initiate`, `POST /teams/{id}/members:onboard`, `POST /auth:reset-password` |
| **PATCH** | …modifie en place une ressource existante ; les paramètres éventuels passent dans le body. | — | `PATCH /orders/{id}:confirm`, `PATCH /users/{id}:activate` |
| **PUT** | …remplace entièrement une ressource ou sous-ressource. | *Idempotent* | `PUT /users/{id}/preferences` |
| **DELETE** | …supprime une ressource ou sous-ressource. | *Idempotent* | `DELETE /teams/{id}:disband`, `DELETE /teams/{id}/members/{mid}:fire` |

> **Précisions utiles :**
> - **`GET` doit rester sûr** — aucun effet de bord. C'est précisément pourquoi `reset-password` est `POST` (il envoie un email) et `user-activity` est `GET`.
> - **`DELETE` doit être idempotent** — un second appel sur une ressource déjà supprimée ne doit pas échouer (il renvoie `204`, jamais `404`).
> - **`PATCH` n'est ni sûr ni idempotent** au sens de la RFC 5789. DORIAX ne s'impose donc pas l'idempotence sur `PATCH`, même si la plupart des transitions d'état (`:confirm`, `:activate`, `:demote`) le sont en pratique. Le body d'un `PATCH` DORIAX n'est pas un *patch document* au sens de la RFC 5789 : c'est une commande métier qui porte les paramètres de l'action.
> - **`POST` ne sur-promet rien** : il signifie « traite cette représentation selon la sémantique de la ressource ». C'est ce qui en fait le verbe par défaut pour toute opération non couverte par les autres.
> - **Service → `GET` ou `POST`** : `GET` s'il ne fait que lire, `POST` s'il produit un effet — `POST` n'étant retenu pour une lecture que lorsqu'un body est nécessaire (voir ci-dessous).

> **Note sur les `GET` avec body (`POST` utilisé à la place de `GET`) :** _Dans certains cas complexes, un appel `GET` peut nécessiter un payload. Cependant, certaines API REST empêchent de passer un body dans une requête `GET`. Dans ces situations, DORIAX privilégie `POST` pour ces requêtes tout en maintenant une logique métier explicite, afin de favoriser une API au vocabulaire métier explicite plutôt que technique._

#### Convention DORIAX — toute création porte un nom métier

Là où REST exprime la création par un `POST` sur la collection (`POST /teams`), DORIAX impose un verbe métier explicite (`POST /teams:initiate`). C'est une divergence assumée vis-à-vis de REST : DORIAX préfère un vocabulaire métier explicite à une convention implicite. Dans une URL DORIAX, le dernier segment est toujours **soit un nom (une ressource), soit un verbe métier (une action)** — jamais une création implicite.

Cette règle a deux vertus :
- elle **nomme** l'intention plutôt que de la déduire de la méthode HTTP ;
- elle **absorbe sans rupture** les cas où plusieurs façons de créer coexistent. C'est la projection sur l'URL du pattern des *constructeurs nommés* utilisé côté code (`Team.Initiate()`, `Team.ImportFrom(...)`, `Team.CloneOf(...)`) :
  - `POST /teams:initiate` → création vierge ;
  - `POST /teams:import` → depuis un annuaire (AD/LDAP) ;
  - `POST /teams:clone` → par duplication d'une équipe existante.

Le `POST /teams` nu de REST ne pourrait pas distinguer ces trois chemins, et conduirait à l'asymétrie peu lisible où la création d'origine est anonyme (`POST /teams`) mais où l'on ajoute plus tard `POST /teams:import` à côté. Nommer dès le départ évite cette dette.

### 3.2 Exemples concrets

Cette section illustre les conventions de DORIAX avec des exemples concrets.

**Exemples pour les ressources**
- `GET /teams` → Obtient la liste des équipes.
- `GET /teams/{id}` → Obtient une équipe spécifique.
- `POST /teams:initiate` → Crée une équipe vierge (terme du domaine).
- `POST /teams:import` → Crée une équipe à partir d'un annuaire (AD/LDAP).
- `POST /teams:clone` → Crée une équipe par duplication d'une équipe existante.
- `GET /users/{id}/security` → Obtient les paramètres de sécurité d'un utilisateur.
- `PATCH /users/{id}/security:reset` → Réinitialise les paramètres de sécurité.
- `PATCH /orders/{id}:confirm` → Confirme une commande.
- `PATCH /users/{id}:activate` → Active un utilisateur.
- `PATCH /products/{id}:discount` → Applique une réduction sur un produit.
- `POST /teams/{id}/members:onboard` → Ajoute un membre à une équipe avec onboarding.
- `DELETE /teams/{id}/members/{mid}:fire` → Retire un membre de l'équipe.
- `DELETE /teams/{id}:disband` → Dissout une équipe (terme du domaine).

**Exemples pour les services**
- `POST /billing:process-invoices` → Lance un traitement global sur plusieurs factures.
- `GET /user-activity?range=last-30-days` → Retourne des statistiques sur l'activité utilisateur (ressource calculée : un nom, donc `/`, en lecture seule).

### 3.3 Gestion des réponses HTTP

DORIAX suit les standards HTTP pour structurer ses réponses en apportant des précisions métier adaptées. Cette section fournit une vue des différentes réponses HTTP DORIAX, en regroupant les cas d'usage courants.

| Action | Codes HTTP possibles | Contenu du payload |
|--------|---------------------|-------------------|
| **GET (récupération de données, ou POST remplaçant un GET nécessitant un body)** | `200 OK`, `403 Forbidden`, `404 Not Found` | Payload contenant la ressource demandée (format HAL ou autre format supporté) ou une erreur |
| **POST (création de ressource)** | `201 Created`, `400 Bad Request`, `403 Forbidden`, `409 Conflict`, `422 Unprocessable Entity` | `Location` dans l'en-tête HTTP, et optionnellement un payload (HAL ou autre) contenant les liens vers la ressource créée |
| **PATCH (modification d'une ressource)** | `200 OK`, `204 No Content`, `400 Bad Request`, `403 Forbidden`, `409 Conflict`, `422 Unprocessable Entity` | Aucun payload en cas de `204`, ou un payload (HAL ou autre) en cas de `200` si des informations complémentaires sont nécessaires |
| **DELETE (suppression d'une ressource)** | `200 OK`, `204 No Content`, `403 Forbidden`, `409 Conflict`, `422 Unprocessable Entity` | `200 OK` si la ressource existait et a été supprimée (payload optionnel), `204 No Content` si la ressource était déjà inexistante ou s'il n'y a rien à retourner. **DELETE est idempotent : l'absence de la ressource ne produit pas de `404`.** |
| **POST (action métier sur un service)** | `200 OK`, `202 Accepted`, `204 No Content`, `400 Bad Request`, `403 Forbidden`, `409 Conflict`, `422 Unprocessable Entity` | Aucun payload en cas de `204`, un payload (HAL ou autre) en cas de `200`, ou un lien de suivi en cas de `202` (requête asynchrone) |
| **Requête asynchrone (`202 Accepted`)** | `202 Accepted`, `400 Bad Request`, `403 Forbidden` | `Location` dans l'en-tête HTTP pour suivre le traitement, et optionnellement un payload (HAL ou autre) avec des liens HATEOAS |
| **Erreur métier (`422 Unprocessable Entity`)** | `422 Unprocessable Entity` | Payload d'erreur en format HAL ou autre format supporté contenant les détails de l'erreur |

👉 **Cette table couvre les principaux cas d'usage des réponses HTTP DORIAX, mais des ajustements peuvent être nécessaires selon les contextes métier et technique spécifiques. Les autres statuts HTTP standards restent applicables selon les besoins.**

**Note — `409` vs `422`.** _`409 Conflict` signale un conflit avec l'**état courant** de la ressource (accès concurrent, doublon, version obsolète). `422 Unprocessable Entity` signale une **règle métier violée** alors que la requête est syntaxiquement bien formée. Certains cas sont à la frontière des deux ; le choix relève alors d'une convention d'équipe, à documenter une fois pour toutes._

**Note — `422 Unprocessable Entity`.** _Ce code est utilisé par DORIAX pour signaler une erreur métier lorsque l'action demandée est logiquement invalide dans le contexte métier, même si la requête est techniquement bien formée._

#### Exemples détaillés

1. **Création réussie :**
   ```http
   HTTP/1.1 201 Created
   Content-Type: application/json

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
   Content-Type: application/json

   {
     "_links": {
       "status": { "href": "/jobs/42/status" }
     }
   }
   ```

4. **Erreur métier avec `422 Unprocessable Entity` :**

   _Contexte : Un appel `PATCH /teams/42/members/22:demote` tente de rétrograder un membre **propriétaire** (`owner`) en **membre simple**. Cependant, il est le **seul propriétaire** restant de l'équipe, ce qui rend cette action invalide._

   ```http
   HTTP/1.1 422 Unprocessable Entity
   Content-Type: application/json

   {
     "errors": [
       {
         "code": "LAST_OWNER_RESTRICTION",
         "message": "Cannot demote the last owner of the team.",
         "_links": {
           "origin": { "href": "/teams/42/members/22:demote", "method": "PATCH" },
           "about": { "href": "http://example.com/docs/errors/last-owner-restriction" }
         }
       }
     ]
   }
   ```

**Note — format d'erreur et standard IETF.** _Le format présenté ici (`{ "errors": [ … ] }`) est un format maison enrichi de liens hypermédia. Le standard IETF pour les erreurs HTTP est la **RFC 9457** (« Problem Details for HTTP APIs », qui a remplacé la RFC 7807 en juillet 2023), avec le média type `application/problem+json` et les champs `type`, `title`, `status`, `detail`, `instance`. DORIAX reste compatible avec ce standard ; le format retenu ici en est une variation délibérée. À défaut d'une bonne raison de diverger, s'aligner sur la RFC 9457 facilite l'interopérabilité et l'outillage._

👉 **Référence complète sur les codes HTTP** : [MDN HTTP Response Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status).

### 3.4 Note sur l'approche CQRS de DORIAX

DORIAX applique naturellement une séparation entre les opérations de lecture et d'écriture, ce qui rejoint le principe de Command Query Responsibility Segregation (CQRS). Cependant, cette approche ne doit pas être perçue comme une contrainte lourde nécessitant une infrastructure complexe avec plusieurs bases de données.

Dans DORIAX, CQRS se traduit par une simple distinction logique entre les actions qui modifient l'état du système et celles qui récupèrent des informations. Une écriture (`POST`, `PATCH`, `DELETE`) ne renvoie pas la représentation complète et mise à jour de la ressource ; elle se limite à un statut HTTP confirmant l'opération, accompagné le cas échéant de liens hypermédia ou d'un identifiant de suivi (par exemple le `Location` d'un `201` ou le lien de statut d'un `202`). L'application effectue ensuite un appel `GET` si elle a besoin d'obtenir les nouvelles données.

Cette séparation présente plusieurs avantages :

- Éviter des retours de données inutiles après une modification.
- Garantir une récupération optimisée avec un `GET` adapté au besoin réel.
- Clarifier les responsabilités métier en distinguant les commandes (actions) des requêtes (lecture).

Par exemple, après l'ajout d'un membre à une équipe via `POST /teams/42/members:onboard`, il est préférable de récupérer uniquement ses informations essentielles via `GET /teams/42/members/33` (avec un éventuel paramètre pour obtenir une représentation allégée), plutôt que de recharger toute la liste des membres. Mais peut-être l'application nécessite-t-elle la récupération complète de la liste ?

En résumé, l'objectif de CQRS en DORIAX est donc de fluidifier les interactions tout en maintenant une structure cohérente et performante, sans imposer de complexité excessive.
