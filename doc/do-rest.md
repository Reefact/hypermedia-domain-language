# DO-REST

## 1. Présentation

### 1.1 Qu’est-ce que DO-REST ?

DO-REST (**Domain-Oriented REST**) est une approche pragmatique de l’architecture REST, conçue pour mieux refléter les concepts métier dans les API. Contrairement à REST classique, souvent limité à des opérations CRUD (Create, Read, Update, Delete), DO-REST introduit une distinction claire entre **ressources** et **services**, et permet l’exposition explicite d’**actions métier** sous forme d’endpoints dédiés.  

L’objectif principal de DO-REST est de rendre les API plus naturelles à utiliser, en évitant de tordre REST pour répondre aux besoins métier. Il s’appuie sur les principes fondamentaux de REST tout en intégrant des concepts issus du DDD (Domain-Driven Design) et du CQRS (Command Query Responsibility Segregation), sans pour autant les imposer.  

###1.2. Pourquoi DO-REST ?

REST est devenu le standard de facto pour la conception d’API web, mais son implémentation traditionnelle repose principalement sur une logique CRUD, où chaque entité est exposée sous forme de ressource manipulable via `GET`, `POST`, `PUT` et `DELETE`. Si cette approche fonctionne bien pour des systèmes simples, elle montre vite ses limites dans les applications métier plus riches.  

**REST impose une approche CRUD inadaptée aux actions métier**

Une API REST classique force souvent à modifier une ressource via `PUT` ou `PATCH`, même lorsque le changement d’état résulte d’une action métier spécifique. Par exemple, une action comme confirmer une commande (`confirmOrder`) ne se résume pas à modifier un champ `status`. D’autres mises à jour peuvent être nécessaires, comme la mise à jour de la date de confirmation, la réservation de stock ou l’envoi d’une notification.  

Avec un modèle CRUD strict, l’appelant doit connaître et gérer ces modifications en passant manuellement tous les champs affectés (`PATCH /orders/42` avec un body `{ "status": "confirmed", "confirmedAt": "2025-02-11T10:00:00Z", "stockReserved": true }`). Cela introduit un **problème majeur** : une partie de la logique métier se retrouve dans l’appelant, alors qu’elle devrait être entièrement gérée côté backend.  

DO-REST résout ce problème en exposant des actions métier explicites (`POST /orders/42/confirm`), ce qui permet au backend de prendre en charge tous les effets métier associés, sans que le client ait à les connaître.  

**REST ne distingue pas clairement Ressources et Services**

Dans une API REST classique, tout est souvent exposé sous forme de ressources, même lorsqu’il s’agit d’un processus métier. Cette confusion mène à des conceptions incohérentes où des services métier sont représentés comme des entités factices (`POST /commands` ou `POST /actions`), ce qui brouille la lisibilité de l’API.  

DO-REST clarifie cette distinction :  
- Une **ressource** représente un élément du domaine, potentiellement composé de plusieurs entités sous-jacentes.  
- Un **service** exécute une logique métier sans persister d’état. Il est accessible uniquement via `POST` ou `GET`.  

**REST gère mal les workflows complexes**

Dans REST standard, plusieurs chemins peuvent mener au même état final, mais chacun peut avoir des effets de bord différents. Cette flexibilité conduit à des API où la modification d’un statut (`PATCH`) ou l’exécution d’une action (`POST /commands`) ne décrivent pas explicitement ce qui se passe côté métier.  

Avec DO-REST, chaque action ayant un impact sur l’état métier est clairement identifiée (`POST /teams/42/members/onboard`). Il n’y a pas d’ambiguïté sur quelle opération est effectuée et quels effets secondaires sont pris en charge par le backend. Cela évite aux clients de devoir comprendre la logique métier interne et simplifie la gestion des workflows.

Voici la version mise à jour de la **section 2 - Ressources, Services et Actions** en intégrant tes retours.  

## 2. Ressources, Services et Actions

DO-REST introduit une séparation claire entre **ressources**, **services**, et **actions** afin d'éviter les ambiguïtés présentes dans de nombreuses implémentations REST classiques. Cette distinction permet d’exposer une API qui reflète mieux le métier et qui évite de masquer les processus métier derrière des structures artificielles.

### 2.1 Ressources

Une **ressource** en DO-REST représente un élément structuré du domaine, qui peut être interrogé et manipulé via des opérations spécifiques. Une ressource peut correspondre à une entité métier, à un agrégat ou à une vue métier enrichie.  

**Exemples de ressources :**  
- `GET /users/42` → Récupère l’utilisateur avec son profil, ses préférences, etc.  
- `GET /orders/123` → Récupère une commande avec tous ses détails.  
- `GET /products/789` → Récupère un produit et ses informations associées.  

Une ressource peut être une agrégation de plusieurs entités sous-jacentes, mais elle reste une unité cohérente du point de vue de l’API.  

Dans DO-REST, les ressources doivent être cohérentes avec le domaine métier et ne pas être réduites à de simples objets CRUD. Par exemple, au lieu d’exposer une ressource `User` avec des mutations génériques (`PATCH /users/42`), on peut exposer une sous-ressource métier dédiée comme `UserSecurity` :  
- `GET /users/42/security` → Récupère les paramètres de sécurité de l’utilisateur.  
- `POST /users/42/security/change-password` → Déclenche l’action métier correspondante.  

Cela permet de garder une structure claire et d’éviter que l’appelant ait à manipuler des détails internes de la ressource.

### 2.2 Services

Un **service** en DO-REST représente un processus métier sans état, qui exécute une logique métier mais qui ne correspond pas à une ressource persistée. Contrairement à une ressource, un service ne possède pas d'identifiant unique et son résultat dépend uniquement des paramètres fournis à l’appel.  

Un service ne doit pas être utilisé par défaut, il ne sert que lorsqu’il n’existe aucune ressource métier évidente à laquelle rattacher l’opération. Cela suit la même logique que les services de domaine en DDD : ils doivent être utilisés avec parcimonie.

Un service peut lire, modifier ou créer des ressources, mais il ne doit pas être traité comme une ressource persistée que l’on peut récupérer avec un GET /service/{id}. Il déclenche une action métier globale qui concerne plusieurs entités, sans être lui-même une entité stockée.

**Exemples de services :**  
- `POST /auth/reset-password` → Déclenche l’envoi d’un email de réinitialisation.  
- `POST /billing/process-invoices` → Déclenche un traitement global sur plusieurs factures (au lieu d’un `POST /invoices/123/generate` qui concernerait une ressource unique).  
- `GET /billing/tax-rules?country=FR` → Retourne des règles fiscales calculées dynamiquement (un service purement en lecture).  

### 2.3 Actions

Les **actions** sont un élément clé de DO-REST. Elles permettent d’exprimer des intentions métier sans détourner les méthodes REST classiques (`PATCH`, `PUT`). Une action peut être appliquée à une ressource ou à un service, selon le contexte.  

#### Actions sur les ressources

Les actions permettent d’effectuer des changements métier sur une ressource sans exposer directement sa structure interne. Contrairement à un `PATCH` ou `PUT` où l’appelant doit connaître les champs à modifier, une action encapsule la logique côté serveur.  

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
Ici, l’appelant doit savoir quels champs modifier, ce qui introduit du couplage avec la logique métier interne.  

DO-REST :  
```http
POST /orders/42/confirm
```
L’action `confirm` encapsule toute la logique métier associée (modification du statut, mise à jour des stocks, notifications, etc.), sans que l’appelant ait à en connaître les détails.

#### Actions sur les services

Les services utilisent eux aussi des actions, qui prennent la forme d’un `GET` ou d'un `POST`.  

**Exemple : réinitialisation de mot de passe**  
- `POST /auth/reset-password` → Déclenche l’envoi d’un email de réinitialisation.
- `POST /auth/change-password` → Change directement le mot de passe après vérification du token.
- `GET /auth/session-status` → Vérifie si une session est active sans la modifier.
- `GET /billing/tax-rules?country=FR` → Récupère les règles fiscales d’un pays sans créer d’état persistant.

**NOTE IMPORTANTE :** _Les exemples de services présentés ici sont volontairement naïfs et génériques pour illustrer le concept. Dans une implémentation réelle, il est essentiel de questionner la nécessité d’un service avant de l’adopter. Dans de nombreux cas, il est possible de rattacher une opération métier à une ressource existante, plutôt que d’introduire un service distinct. Comme en DDD (Domain-Driven Design), les services doivent être utilisés en dernier recours, uniquement lorsqu’une entité métier ou une action sur une ressource ne peut raisonnablement les couvrir._

### 2.4 Synthèse

| **Concept**  | **Définition** | **Exemple** |
|-------------|---------------|------------|
| **Ressource** | Élément structuré du domaine, identifiable par un ID | `GET /orders/42` |
| **Service** | Processus métier sans état, distinct des ressources | `POST /billing/process-invoices` |
| **Action** | Opération métier explicite, appliquée à une ressource ou un service | `POST /orders/42/confirm` |

DO-REST clarifie ces distinctions pour éviter les dérives observées dans certaines implémentations REST classiques. Cette approche permet d’obtenir des API plus cohérentes et plus lisibles tout en restant alignées avec les principes REST.

# 3. Convention de conception d’une API DO-REST

DO-REST suit les principes fondamentaux de REST tout en renforçant l’orientation métier. Cette section définit les conventions utilisées pour structurer une API DO-REST de manière claire et cohérente.

## 3.1 Conventions générales

### Rappel

DO-REST structure ses endpoints en suivant une logique métier claire et cohérente.  

1. **Les ressources et leurs collections**
   - Une **collection de ressources** regroupe plusieurs ressources du même type et permet d’interagir avec elles.
   - Une **ressource** représente une entité métier identifiable.   
   - Une **sous-ressource** est un élément directement dépendant d’une ressource principale.  
   - Les **actions métier** peuvent être appliquées à une ressource ou à une collection.  

2. **Les services**  
   - Un **service** exécute une logique métier sans être une ressource persistée.  
   - Il ne possède pas d’identifiant et ne peut pas être manipulé comme une ressource.  
   - Les fonctionnalités des services ne peuvent être invoquées que via une **action métier**.

### Format des URLs

**Ressources et sous-ressources**  
- `/resources` → Collection de ressources.  
- `/resources/{id}` → Ressource spécifique.  
- `/resources/{action}` → Action appliquée à une collection de ressources.  
- `/resources/{id}/{action}` → Action appliquée à une ressource spécifique.  
- `/resources/{id}/sub-resources` → Collection de sous-ressources d’une ressource.  
- `/resources/{id}/sub-resources/{sub-id}` → Une sous-ressource spécifique.  
- `/resources/{id}/sub-resources/{action}` → Une action sur la collection de sous-ressources.  
- `/resources/{id}/sub-resources/{sub-id}/{action}` → Une action sur une sous-ressource spécifique.  

**Services**
- `/service/{action}` → Exécution d’une action sur un service.  

### Usage des verbes HTTP  

| Verbe | Usage en DO-REST |
|-------|----------------|
| **GET** | Récupérer une collection de ressources, une ressource spécifique ou le résultat d’un traitement sans effet de bord. |
| **POST** | Exécuter une action de service ou créer une ressource/sous-ressource via une action métier explicite. Peut aussi être utilisé pour des requêtes GET avec un body lorsque cela est justifié pour favoriser une API explicite métier plutôt qu'explicite technique. |
| **PUT** | Remplacer entièrement une ressource ou sous-ressource. |
| **PATCH** | Exécuter une action qui va modifier une ressource ou une sous-ressource. |
| **DELETE** | Exécuter une action métier qui entraîne la suppression d’une ressource ou sous-ressource. |

**Note sur les `GET` avec body (`POST` utilisé à la place de `GET`) :** _Dans certains cas complexes, un appel `GET` peut nécessiter un payload. Cependant, certaines API REST empêchent de passer un body dans une requête `GET`. Dans ces situations, DO-REST privilégie `POST` pour ces requêtes tout en maintenant une logique métier explicite._

## 3.2 Exemples concrets

Cette section illustre les conventions de DO-REST avec des exemples concrets.

**Exemples pour les ressources**  
- `GET /teams` → Obtient la liste des équipes.  
- `GET /teams/{id}` → Obtient une équipe spécifique.  
- `POST /teams/initiate` → Créer une nouvelle équipe dans le système en utilisant le terme du domaine.  
- `GET /users/{id}/security` → Obtient les paramètres de sécurité d’un utilisateur.  
- `PATCH /users/{id}/security/reset` → Réinitialise les paramètres de sécurité.  
- `PATCH /orders/{id}/confirm` → Confirme une commande.  
- `PATCH /users/{id}/activate` → Active un utilisateur.  
- `PATCH /products/{id}/discount` → Applique une réduction sur un produit.  
- `POST /teams/{id}/members/onboard` → Ajoute un membre à une équipe avec onboarding.  
- `DELETE /teams/{id}/disband` → Supprime une équipe du système en utilisant le terme du domaine.  

**Exemples pour les services**  
- `POST /billing/process-invoices` → Lance un traitement global sur plusieurs factures.  
- `GET /user-activity?range=last-30-days` → Retourne des statistiques sur l’activité utilisateur.  

## 3.3 Gestion des réponses HTTP

DO-REST suit les standards HTTP pour structurer ses réponses.  

**Exemples :**  
1. Création réussie :  
   ```http
   HTTP/1.1 201 Created
   Location: /teams/123
   ```
2. Action exécutée sans contenu de retour :  
   ```http
   HTTP/1.1 204 No Content
   ```
3. Erreur de validation :  
   ```http
   HTTP/1.1 400 Bad Request
   Content-Type: application/json
   
   {
     "error": "Invalid team name",
     "field": "name"
   }
   ```

👉 **Référence complète sur les codes HTTP** : [MDN HTTP Response Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status).
