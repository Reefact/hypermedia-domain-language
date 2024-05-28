# Gestion des Erreurs

Hypermedia Domain Language (HDL) fournit une structure flexible et détaillée pour la gestion des erreurs survenues lors de l'exécution de requêtes HTTP. Cette documentation décrit le format des erreurs dans HDL.

Dans les cas les plus ismples, le format HDL propose une structure minimale pour les erreurs, avec une seule propriété obligatoire, `message`, qui contient une description claire et consise de l'erreur.

Le fichier ci-dessous donne une exemple minimal de retour d'erreur pouvant survenir lors d'un cas d'utilisation métier:

**Exemple de fichier de retour d'erreur minimale**

```
HTTP/1.1 422 Unprocessable Entity
```
```json
{
  "message": "Payment refused."
}
```
Un exemple un peu plus complexe est la gestion d'erreur lors de la validation d'une requête. On voit dans cet exemple que l'on peut ajouter à l'erreur principal le détail de / des erreur(s) qui l'ont provoquée.

**Exemple d'erreur suite à une erreur BAD_REQUEST**

```
HTTP/1.1 400 Bad Request
```
```json
{
    "message": "The command to create the article is invalid.", 
    "errors": [
    {
      "message": "Title is required.",
      "source": {
        "pointer": "/title"
      }
    }
    ]
}
```

Le format HDL permet également de gérer des cas d'erreurs plus complexes en ajoutant des propriétés supplémentaires, fournissant ainsi un niveau de détail très précis si nécessaire. Cela  pemret aux développeurs de fournir des informations complètes sur les erreurs, facilitant ainsi le débogage et l'intégration.

#### Structure Générale des Erreurs

#### 1. httpResponse (optionnel)

L'objet `httpResponse` contient les informations liées à l'état HTTP de la réponse. Cet objet est optionnel car ces informations sont déjà fournies dans les en-têtes HTTP de la réponse. Il permet cependant d'offrir aux applications l'ensemble des informations dans un seul objet.

- **status** (obligatoire) : Le code de statut HTTP de l'erreur.
- **title** (optionel) : Un titre court décrivant l'erreur, correspondant au statut HTTP.
- **message** (optionnel) : Une description standardisée du statut HTTP, offrant un contexte supplémentaire sur la nature de l'erreur.

**Exemple de propriété `httpResponse`**
```json
  "httpResponse": {
    "status": 422,
    "title": "Unprocessable Entity",
    "message": "The request was well-formed but was unable to be followed due to business errors."
  }
```

#### 2. message (obligatoire)

Le champ `message` est un texte obligatoire qui fournit une explication globale et compréhensible de l'erreur rencontrée. Ce message est destiné à donner une indication claire et concise sur la cause de l'erreur, adaptée aux utilisateurs finaux ou aux développeurs. Par exemple, "Invalid command." ou "Payment refused."

**Exemple de propriété `message`**
```json
{
  "message": "Payment refused."
}
```

#### 3. code (optionnel)

Le champ `code` est un identifiant unique associé au message global de l'erreur. Ce code, une chaîne de caractères, peut être utilisé pour référencer de manière unique une erreur spécifique, facilitant ainsi le suivi et la documentation des erreurs.

**Exemples de propriété `code`**

Un exemple dans lequel les développeurs ont choisis d'utiliser un code sous forme de chaîne de caractères:
```json
{
  "message": "Payment refused.",
  "code": "PAYMENT_REFUSED"
}
```
Un exemple dans lequel les développeurs ont choisis d'utiliser un code sous format de GUID:
```json
{
  "message": "Payment refused.",
  "code": "f2b26bf1-19c9-4ead-94d3-e8d97846dd33"
}
```

#### 4. errors (optionnel)

Le tableau `errors` contient des informations détaillées sur des erreurs spécifiques survenues lors de la requête. Chaque entrée du tableau `errors` est un objet décrivant une erreur particulière.

```json
    "errors": [
      {
        "code": "4f46694f-52e2-4af3-b7cb-def91b9598ca",
        "message": "Title is too long.",
        "source": {
          "pointer": "/title"
        },
        "_links": {
            "about": "https://example.com/api/doc/err/4f46694f-52e2-4af3-b7cb-def91b9598ca.htm"
        }
      },
      {
        "message": "Author is required.",
        "source": {
          "pointer": "/authorId"
        }
      }
    ]
```

##### 4.1. message (obligatoire)

Un message détaillé décrivant la nature spécifique de l'erreur, similaire à la propriété `message` globale mais spécifique à cette sous-erreur.

##### 4.2. code (optionnel)

Un identifiant unique pour chaque erreur spécifique, similaire à la propriété `code` globale mais spécifique à cette sous-erreur.

##### 4.3. source (optionnel) 

Un objet décrivant la source de l'erreur.

  - **pointer** : Référence à une partie spécifique du document JSON, utilisant la syntaxe [JSON Pointer (RFC 6901)](https://datatracker.ietf.org/doc/html/rfc6901). Par exemple, `/data/attributes/title` pour indiquer un problème avec le champ `title`.
  - **parameter** : Indique un paramètre de requête problématique. Par exemple, `sort` pour une erreur liée au tri.
  - **header** : Indique un en-tête de requête problématique. Par exemple, `Authorization` pour un problème avec l'en-tête d'autorisation.

**Exemple d'utilisation de la propriété `source`**

Ici la valeur de la proriété de tri n'est pas valide lors de l'appel `HTTP/GET http://example.com/articles`:

```json
  "errors": [
    {
      "message": "Invalid sort parameter value.",
      "source": {
        "parameter": "sort"
      }
    }
  ],
  "_links": {
    "origin": {
      "href": "http://example.com/articles",
      "method": "GET"
    }
  }
```

Cet exemple montre comment indiquer un en-tête de requête:

```json
  "errors": [
    {
      "message": "Report-Id header is missing. User cannot download the report.",
      "source": {
        "header": "Report-Id"
      }
    }
  ]
```


##### 4.4. _links (optionnel)

Contient des liens hypermedia spécifiques à cette erreur.

  - **about** : Un lien vers une documentation détaillée sur l'erreur spécifique, permettant aux développeurs de comprendre et de corriger plus facilement le problème.

```json
    "errors": [
      {
        "code": "4f46694f-52e2-4af3-b7cb-def91b9598ca",
        "message": "Title is too long.",
        "source": {
          "pointer": "/title"
        },
        "_links": {
            "about": "https://example.com/api/doc/err/4f46694f-52e2-4af3-b7cb-def91b9598ca.htm"
        }
      }
    ]
```

Dans cet exemple l'url `https://example.com/api/doc/err/4f46694f-52e2-4af3-b7cb-def91b9598ca.htm` peut donner des informations sur le format attendu pour un titre d'article.

#### 5. _links (optionnel)

L'objet `_links` contient des liens hypermedia fournissant des informations supplémentaires ou des actions possibles.

- **origin** (optionnel) : Un lien vers l'URL de la requête ayant provoqué l'erreur, incluant la méthode HTTP utilisée. Cela permet de contextualiser l'erreur en montrant exactement quelle action a conduit à cette situation. Par exemple, `href: "http://example.com/articles/42:buy"` avec `method: "POST"`.
- **about** (optionnel) : Un lien vers une description générale de l'erreur de haut niveau, utilisant le code unique. Cela fournit une documentation supplémentaire sur l'erreur globale, aidant à comprendre la nature et la résolution de l'erreur.

```json
{
  "message": "Payment refused.",
  "code": "c01a4e2b-7f3d-4d3b-9101-f5a1e3b2e2b1",
  "_links": {
    "origin": {
      "href": "http://example.com/articles/42:buy",
      "method": "POST"
    },
    "about": {
      "href": "http://example.com/docs/errors/c01a4e2b-7f3d-4d3b-9101-f5a1e3b2e2b1"
    }
  }
}
```

#### 6. _metadata (optionnel)

L'objet `_metadata` est un tableau optionnel dont le contenu dépend du serveur. Il fournit des métadonnées supplémentaires sur la réponse d'erreur, telles que des identifiants de transaction, des horodatages, ou des versions de l'API, bien que le contenu exact dépende des besoins spécifiques de l'application et du serveur.

```json
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440001",
    "timestamp": "2024-05-25T14:35:00Z",
    "apiVersion": "1.2.3"
  }
```
## Quelques exemples d'erreurs

### Exemple de Réponse d'Erreur pour un Code 400 (Bad Request)

**Requête**

```http
POST http://example.com/articles
Content-Type: application/json
{
  "message": "Too short"
}
```

**Réponse**

```json
{
  "httpResponse": {
    "status": 400,
    "title": "Bad Request",
    "message": "The server could not understand the request due to invalid syntax."
  },
  "message": "Invalid command.",
  "code": "d02b5e1c-8f4d-4c7e-8102-f6b1e4c3e3c2",
  "errors": [
    {
      "code": "123e4567-e89b-12d3-a456-426614174001",
      "message": "Title is required.",
      "source": {
        "pointer": "/title"
      },
      "_links": {
        "about": {
          "href": "http://example.com/docs/errors/123e4567-e89b-12d3-a456-426614174001"
        }
      }
    },
    {
      "code": "123e4567-e89b-12d3-a456-426614174002",
      "message": "Description must be at least 10 characters long.",
      "source": {
        "pointer": "/info/description"
      },
      "_links": {
        "about": {
          "href": "http://example.com/docs/errors/123e4567-e89b-12d3-a456-426614174002"
        }
      }
    }
  ],
  "_links": {
    "origin": {
      "href": "http://example.com/articles",
      "method": "POST"
    },
    "about": {
      "href": "http://example.com/docs/errors/d02b5e1c-8f4d-4c7e-8102-f6b1e4c3e3c2"
    }
  },
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440002",
    "timestamp": "2024-05-25T14:40:00Z"
  }
}
```

## Exemple de Réponse d'Erreur pour un Code 403 (Forbidden)

**Requête**

```http
GET http://example.com/reports/42:download
Authorization: Bearer valid_token
```

**Réponse**

```json
{
  "httpResponse": {
    "status": 403,
    "title": "Forbidden",
    "message": "The server understood the request, but it refuses to authorize it."
  },
  "message": "Report download denied.",
  "code": "c9a5f1e2-4f9d-45ab-b5e3-2b9f72b8f1d7",
  "errors": [
    {
      "message": "User is not authorized to download this report.",
      "_links": {
        "about": {
          "href": "http://example.com/docs/errors/c9a5f1e2-4f9d-45ab-b5e3-2b9f72b8f1d7"
        }
      }
    }
  ],
  "_links": {
    "origin": {
      "href": "http://example.com/reports/42:download",
      "method": "GET"
    },
    "about": {
      "href": "http://example.com/docs/errors/c9a5f1e2-4f9d-45ab-b5e3-2b9f72b8f1d7"
    }
  },
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440003",
    "timestamp": "2024-05-25T15:00:00Z"
  }
}
```

### Exemple de Réponse d'Erreur pour un Code 422 (Unprocessable Entity)

**Requête**

```http
POST http://example.com/articles/42:buy
Content-Type: application/json
{
  "paymentMethod": "invalid-method"
}
```

**Réponse**

```json
{
  "httpResponse": {
    "status": 422,
    "title": "Unprocessable Entity",
    "message": "The request was well-formed but was unable to be followed due to business errors."
  },
  "message": "Payment refused.",
  "code": "c01a4e2b-7f3d-4d3b-9101-f5a1e3b2e2b1",
  "errors": [
    {
      "message": "The card was refused by the bank.",
      "_links": {
        "about": {
          "href": "http://example.com/docs/errors/123e4567-e89b-12d3-a456-426614174000"
        }
      }
    }
  ],
  "_links": {
    "origin": {
      "href": "http://example.com/articles/42:buy",
      "method": "POST"
    },
    "about": {
      "href": "http://example.com/docs/errors/c01a4e2b-7f3d-4d3b-9101-f5a1e3b2e2b1"
    }
  },
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440001",
    "timestamp": "2024-05-25T14:35:00Z"
  }
}
```

### Conclusion

Le format des erreurs dans HDL permet une description détaillée et claire des erreurs survenues, facilitant le débogage et la résolution des problèmes. En utilisant JSON Pointer pour spécifier les sources d'erreurs spécifiques et en fournissant des liens hypermedia pertinents, HDL assure une bonne traçabilité et une documentation complète des erreurs. Les éléments optionnels permettent une flexibilité d'utilisation tout en assurant la clarté et la précision des informations fournies.
