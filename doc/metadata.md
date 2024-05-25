### Métadonnées

## Introduction

Dans Hypermedia Domain Language (HDL), les métadonnées globales fournissent des informations contextuelles importantes sur la réponse API. Ces métadonnées sont incluses dans un objet `_metadata`. 

L'objet `_metadata` est un simple dictionnaire extensible, ce qui permet à chaque API de décider quelles métadonnées spécifiques elle souhaite inclure.

## Structure Générale

**Exemple de HDL avec Métadonnées**

```json
{
  "id": "1",
  "title": "Article 1",
  "_links": {
    "self": {
      "href": "http://example.com/articles/1"
    }
  },
  "_metadata": {
    "transactionId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-05-25T14:30:00Z",
    "apiVersion": "1.0.0"
  }
}
```
Dans cet exemple le serveur à choisi de fournir a client les informations suivantes:

- **transactionId** : Identifiant unique pour la requête/réponse, représenté par un GUID. Cet identifiant peut-etre utile pour le traçage et le débogage des requêtes.
- **timestamp** : Horodatage de la réponse qui indique l'heure à laquelle la réponse a été générée.
- **apiVersion** : Version de l'API qui répond à la requête. Cela peut permettre de suivre les versions de l'API et de s'assurer que les clients utilisent la bonne version.

## Conclusion

L'inclusion de métadonnées globales dans HDL via l'objet `_metadata` améliore la transparence, la traçabilité et la gestion des versions des API. En fournissant un cadre extensible, HDL permet aux développeurs d'API de personnaliser et d'enrichir les réponses avec des informations contextuelles supplémentaires, tout en maintenant une structure claire et lisible.
