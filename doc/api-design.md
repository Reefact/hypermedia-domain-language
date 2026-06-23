# Concevoir une API pour un domaine riche

> Document d'introduction à DORIAX. Il explique *quand* et *pourquoi* l'employer, et comment il cohabite avec REST « plat » sur une même API. La grammaire elle-même — format des URL, choix des verbes, réponses HTTP, séparation commande/requête — est spécifiée dans [doriax.md](doriax.md). On lit le *pourquoi* ici ; on applique le *comment* là.

## Le point de départ

Concevoir une API, c'est exposer un domaine. Or un domaine n'a pas une densité métier uniforme. Certaines parties sont **riches** : une écriture y fait plus que renseigner un champ — elle déclenche une transition d'état, des effets que le serveur orchestre, et le métier a un *mot* pour ce qui se passe (*confirmer*, *dissoudre*, *clôturer*). D'autres sont **plates** : de la donnée de référence qu'on crée, lit et met à jour, sans règle ni cycle de vie.

Cette différence n'est pas cosmétique — elle commande la forme que doit prendre l'API. C'est tout l'objet de ce document.

## Sur un domaine riche, REST « plat » plie mal

REST, dans son usage courant, est **CRUD** : chaque entité est une ressource qu'on manipule via `GET`/`POST`/`PUT`/`PATCH`/`DELETE`. Pour des données simples, c'est parfait. Sur un domaine riche, trois limites apparaissent.

**La logique fuit vers l'appelant.** Confirmer une commande n'est pas « mettre `status` à `confirmed` » : c'est aussi réserver le stock, dater la confirmation, notifier. En CRUD strict, le client doit connaître et envoyer tous ces champs (`PATCH /orders/42` avec `{ "status": "confirmed", "confirmedAt": "...", "stockReserved": true }`) — une part de la logique métier se retrouve chez lui, alors qu'elle devrait être entièrement côté serveur. Une action nommée l'encapsule : `POST /orders/42:confirm`, et le backend orchestre tout.

**Ressources et processus se confondent.** Faute de place pour un *processus* métier, on invente des ressources factices (`POST /commands`, `POST /actions`) qui brouillent l'API.

**Les workflows deviennent ambigus.** Plusieurs `PATCH` peuvent mener au même état avec des effets de bord différents ; rien dans l'URL ne dit *ce qui se passe* côté métier.

Le fond du problème : REST fait porter l'intention par la **méthode HTTP**, qui n'a pas assez de mots pour un domaine riche. DORIAX la fait porter par l'**URL**.

## La réponse : DORIAX

DORIAX (**Domain-Oriented Representation for Inter-Application eXchanges**) garde de REST son **modèle de ressources** — l'adressage hiérarchique (`/teams/{id}/members/{mid}`) et la seule loi de verbe qui change quelque chose pour l'infrastructure, **lecture sûre (`GET`) contre effet (`POST`)** — et lui ajoute, à la manière du **RPC**, la **méthode métier nommée dans l'URL** : `POST /orders/{id}:confirm`, `POST /teams/{id}:disband`. C'est un hybride REST / RPC-over-HTTP assumé : adressage et lecture à la REST, opérations nommées à la RPC.

Sa règle cardinale : **toute écriture porte un nom métier**, jamais déduite de la seule méthode HTTP. DORIAX distingue trois choses — des **ressources** (éléments du domaine), des **services** (processus métier sans état) et des **actions** (les intentions métier, le `:{verbe}`). La grammaire complète — format des URL, les deux verbes, verbes exclus, nommage, réponses — est spécifiée dans [doriax.md](doriax.md). DORIAX s'y marie avec **HDL**, le profil hypermédia de ses réponses, qui découple le client des URL.

## Mais un domaine riche n'est pas riche *partout*

Voilà le point qui décide du reste. Le même domaine — le même *bounded context* — contient presque toujours, à côté de ses objets riches, de la **donnée de référence** plate : un préparateur, une zone d'entrepôt, un type d'emballage. Pour elle, il n'y a aucune intention à nommer ; on l'*ajoute*, on la *modifie*, on la *supprime*.

Lui imposer DORIAX ne gagnerait rien : un `:update` générique là où un `PATCH` suffit n'est que de la **cérémonie**, et habiller du plat en riche est une forme de mensonge. Donc on ne force pas — cette donnée-là reste servie par **REST plat**, à côté de DORIAX, dans la même API. DORIAX prend le relais là où le métier a quelque chose à dire, et laisse la place ailleurs.

Reste à savoir *où* passe la ligne. C'est l'objet de la section suivante.

## La frontière : la ressource

**En une phrase.** L'unité de choix n'est ni l'API, ni le bounded context : c'est la **ressource**. Une ressource a des règles métier à protéger (c'est un *agrégat*) → **toutes** ses écritures sont **DORIAX** ; ou elle n'en a aucune (c'est une *donnée de référence*) → **toutes** ses écritures sont du **CRUD**, en REST plat. Une ressource est donc **uniformément** l'un ou l'autre, jamais moitié-moitié. Et un même contexte héberge normalement des ressources des deux sortes.

### Deux notions de DDD, en clair

_Cette section s'appuie sur deux termes du Domain-Driven Design. Si vous les maîtrisez, passez directement à la suivante ; sinon, voici le strict nécessaire._

- **Invariant** — une *règle métier qui doit toujours rester vraie* pour que les données aient un sens : « une équipe a toujours au moins un propriétaire », « le total d'une commande égale la somme de ses lignes ». Ce n'est **pas** la validation d'un champ (« l'email est bien formé ») — ça, c'est de la syntaxe. Un invariant porte sur la **cohérence métier** de l'état.
- **Agrégat** — un groupe de données traité comme **un seul bloc**, dont une *frontière* garantit que les invariants restent vrais après chaque changement. On ne modifie jamais son intérieur directement : on passe par l'agrégat, qui contrôle ses règles à chaque écriture — comme un portier qui vérifie chaque entrée. **Dans DORIAX, une ressource correspond à un agrégat.**

Une **donnée de référence** (ou entité de support), à l'inverse, n'a aucun invariant : des champs qu'on renseigne, sans règle qui les lie ni cycle de vie. Un préparateur, une zone d'entrepôt, un type d'emballage.

### La règle : la ressource a-t-elle des invariants ?

> **Pour chaque ressource, une seule question : a-t-elle des règles métier à protéger (est-ce un agrégat) ?**
>
> - **Oui** → c'est un agrégat. **Toutes** ses écritures sont **DORIAX** : `POST /ressource/{id}:{verbe}`.
> - **Non** → c'est une donnée de référence. **Toutes** ses écritures sont du **CRUD**, en REST plat : `POST /ressource`, `PATCH /ressource/{id}`, `DELETE /ressource/{id}`.

**Le signal lisible : le langage du domaine a-t-il un *verbe* ?** On ne « mesure » pas les invariants à l'œil ; on les repère à leur trace dans l'*Ubiquitous Language*. Une ressource avec des règles a un cycle de vie, donc le métier a des **verbes** pour la faire évoluer — *confirmer* une commande, *dissoudre* une équipe, *clôturer* une préparation. Une donnée de référence n'a pas de verbe : on l'*ajoute*, on la *modifie*, on la *supprime*. Le verbe est la projection observable de l'invariant ; c'est aussi la projection de la règle « toute écriture porte un nom métier — *quand ce mot existe* » (voir [doriax.md](doriax.md), §3.1).

**Pourquoi une ressource ne peut pas être moitié-moitié.** C'est le point qui rend la règle étanche, et il découle directement de la définition de l'agrégat : *un invariant appartient à l'agrégat, pas à une écriture.* Il est gardé à la frontière, et cette frontière contrôle **toute** mutation. On ne peut donc pas imaginer une ressource qui ferait un `PATCH` « libre, sans règle » sur certains champs et un `POST` « avec règle » sur d'autres : s'il existe une seule règle à protéger, *tous* les champs sont derrière le portier — aucun `PATCH` ne le contourne. Et s'il n'y a aucune règle, il n'y a aucun verbe métier à exposer — juste du CRUD. Les deux moitiés ne coexistent pas sur une même ressource. La ressource est l'unité, et elle est uniforme.

> **Ce que le test ne regarde pas.** _Ni le **type technique** (« c'est un `User`, donc… ») ni ce que l'écriture fait **en base**. Une suppression métier (`:disband`, `:fire`) reste DORIAX même si elle efface une ligne, parce que l'effacement n'est qu'un effet parmi d'autres — notification, cascade, audit, invariante « pas le dernier owner » (voir [doriax.md](doriax.md), §3.1, « Verbes délibérément exclus »). Ce qui décide est l'existence d'invariants sur la ressource, pas la mécanique de persistance._

### Pourquoi les deux régimes cohabitent dans un même bounded context

Un bounded context est une **frontière de langage** : à l'intérieur, chaque terme a un sens unique et le modèle est cohérent. C'est ce que le BC garantit — et **rien de plus**. Il ne garantit pas que toutes les ressources qui vivent dedans soient des agrégats.

Or elles ne le sont pas. Un modèle de domaine, c'est presque toujours quelques **agrégats** — les objets autour desquels le contexte existe, riches en règles — entourés de **données de référence** plates qu'on se contente d'administrer. Les premiers relèvent de DORIAX, les secondes de REST plat, **dans le même contexte, sans contradiction**. Cohérence de langage et présence d'invariants sont deux propriétés indépendantes : le bounded context assure la première, jamais la seconde. Un contexte entièrement riche, ou entièrement plat, est l'exception ; le panachage est la texture normale d'un domaine réaliste.

Cette hétérogénéité est **tactique** et interne au modèle : les invariants ne se répartissent pas également sur les concepts, ils se concentrent sur quelques-uns, qui deviennent des agrégats. La règle de conception d'agrégat — *garder les agrégats petits, n'enfermer dans la frontière de cohérence que l'état réellement lié par un invariant, et référencer le reste par identité* — produit donc mécaniquement ce partage plat / riche **à l'intérieur d'un seul BC**. Il ne s'agit pas d'une classification stratégique de sous-domaines (Core / Supporting / Generic), qui opère au grain du BC et explique les écarts *entre* contextes, non *dans* un contexte.

### Exemple — le contexte « préparation de commande »

**Le préparateur** est une donnée de référence : nom, matricule, zone d'affectation. Aucune règle ne lie ces champs, aucun cycle de vie — on *ajoute* un préparateur, on ne l'*intronise* pas. Ressource plate → REST plat :

```http
POST   /preparateurs
PATCH  /preparateurs/{id}
DELETE /preparateurs/{id}
```

**La préparation** est un agrégat : un cycle de vie, des règles (« tout doit être scanné avant la clôture »), des transitions qui émettent des événements. Le langage a un verbe pour chaque écriture → DORIAX :

```http
POST /preparations/{id}:start            # démarrer
POST /preparations/{id}:scan-item        # scanner un article
POST /preparations/{id}:report-shortage  # signaler une rupture
POST /preparations/{id}:complete         # clôturer (vérifie « tout scanné »)
```

Même API, même bounded context, même langage — deux ressources, **chacune uniforme**. Le préparateur est une donnée de support qu'on administre ; la préparation est l'agrégat qu'on orchestre.

> **Le cas du `User` « moitié CRUD, moitié riche ».** _On croit parfois voir une ressource mixte : un `User` qu'on édite en CRUD mais qui aurait aussi un `:change-password`. La lecture juste est qu'il y a **deux ressources** : le profil `User` (plat) et la sous-ressource `UserSecurity` (un agrégat, avec ses règles de sécurité), exposée séparément — `GET /users/{id}/security`, `POST /users/{id}/security:change-password` (voir [doriax.md](doriax.md), §2.1). Chacune est uniforme. **Une sous-ressource compte comme une ressource distincte** : c'est elle, et non le `User` entier, qui est l'agrégat. Ce n'est pas une entorse à la règle, c'est la règle appliquée au bon grain — l'agrégat, pas la table._

### Sain ou symptôme ?

Avec ce cadrage, le test de cohérence devient net.

> **La couture entre les deux régimes passe-t-elle *entre* deux ressources, chacune uniforme ?**
>
> - **Oui** — `/preparateurs` plat, `/preparations` riche : **sain**. Deux ressources, deux natures, chacune entière dans son régime.
> - **Non** — un `PATCH` « libre » qui cohabite avec un `:confirm` **sur la même ressource** : **symptôme**. C'est exactement la ressource moitié-moitié qui ne peut pas exister : soit la ressource a des règles, et le `PATCH` libre les contourne (bug) ; soit elle n'en a pas, et le `:confirm` n'a rien à orchestrer (cérémonie). Le modèle n'a pas tranché ce qu'est la ressource.

Le problème, quand il survient, n'est jamais que CRUD et DORIAX cohabitent *dans le contexte* — c'est qu'ils cohabitent *sur une même ressource*. Le remède n'est pas de choisir un régime global : c'est de décider si cette ressource est un agrégat ou non, et le cas échéant d'en extraire une sous-ressource, comme `UserSecurity`.

### Le régime peut évoluer — mais c'est la ressource entière qui bascule

L'étiquette se pose sur la **ressource**, et elle peut changer dans le temps. Mais quand elle change, c'est **toute la ressource** qui est reclassée — pas une opération isolée.

Reprenons le préparateur. Si, demain, le *créer* acquiert des règles — vérifier une habilitation CACES, provisionner un compte SSO, l'affecter à une zone soumise à des contraintes —, alors le préparateur **devient un agrégat**. Un verbe apparaît (*habiliter*), et à partir de là **toutes** ses écritures passent en DORIAX, y compris celles qui semblaient anodines :

```http
POST /preparateurs:enroll       # création devenue riche
POST /preparateurs/{id}:update  # correction du nom : repli générique, PAS un PATCH plat
```

> **`:update` (DORIAX) ou `PATCH` (REST plat) ?** _La nuance qui prête le plus à confusion. Sur un **agrégat**, une écriture sans verbe métier précis n'est pas un `PATCH` plat : c'est le repli générique DORIAX `:update` — « toléré, jamais par défaut » (voir [doriax.md](doriax.md), §3.1) —, parce qu'elle franchit quand même la frontière de l'agrégat et ses règles. Le `PATCH` plat est réservé aux **données de référence**, qui n'ont pas de frontière à franchir. Donc le jour où le préparateur devient un agrégat, sa correction de nom **n'est plus** `PATCH /preparateurs/{id}` mais `POST /preparateurs/{id}:update`. C'est cohérent avec « une ressource est uniforme » : on ne laisse pas survivre un `PATCH` plat sur un agrégat._

La leçon : une ressource n'a pas « des opérations qui migrent une à une ». Elle est plate, puis — le jour où une règle apparaît — elle bascule en bloc. C'est une vraie migration (on introduit un modèle de domaine, des ports, des mappers là où il n'y avait que du CRUD direct), mais elle est **localisée à une ressource**, pas systémique.

### Deux régimes ≠ deux bounded contexts (ni deux APIs)

Dernier piège, miroir du premier : **ne scindez pas un bounded context au prétexte qu'il porte deux régimes.** La frontière d'un bounded context est une **rupture de langage** — un endroit où un même mot change de sens —, jamais une différence de régime entre deux ressources.

*Préparateur* et *préparation* parlent le même langage : un seul contexte, rien à séparer, même si l'un est plat et l'autre riche. Une différence de régime n'est pas le signe de deux contextes déguisés ; c'est un contexte qui, comme presque tous, contient un ou plusieurs agrégats et leurs données de référence. Le découpage en contextes se décide sur le **langage**, pas sur le régime.

De la même façon, deux régimes dans un BC **n'imposent pas deux APIs** : le défaut est **une API, deux zones de régime**. La décision de scinder en plusieurs déployables relève de critères orthogonaux au régime — autonomie de déploiement, cycle de vie propre, équipe distincte, ou donnée réellement transverse partagée par plusieurs contextes (qui relève alors d'un autre BC, donc d'un découpage par langage, non par régime).

### En résumé

| Niveau | Question | Réponse |
|--------|----------|---------|
| **Ressource** (= agrégat ?) | A-t-elle des invariants à protéger ? | Oui → agrégat → **toutes** ses écritures en DORIAX (`POST /…:{verbe}`) · Non → donnée de référence → **toutes** en CRUD (`POST`/`PATCH`/`DELETE`) |
| **Signal pratique** | Le domaine a-t-il un *verbe* pour ses écritures ? | C'est la trace observable de l'invariant : verbe → agrégat ; pas de verbe → référence |
| **Bounded context** | Un seul régime pour tout le contexte ? | Non — un contexte mêle normalement des agrégats (DORIAX) et des données de référence (CRUD) |
| **Frontière de contexte** | Qu'est-ce qui justifie de scinder ? | Une **rupture de langage**, jamais une différence de régime |

Le fil conducteur : **classer la ressource, pas l'opération.** Une ressource avec des règles est un agrégat, uniformément DORIAX ; une ressource sans règles est une donnée de référence, uniformément CRUD. L'API, le contexte, le type technique ne sont pas le bon niveau de décision — la ressource l'est.

## Où aller ensuite

- **[doriax.md](doriax.md)** — la spécification de la grammaire : format des URL, les deux verbes `GET`/`POST`, verbes délibérément exclus (`PUT`/`DELETE`), nommage des actions, gestion des réponses HTTP, séparation commande/requête.
- **HDL** — le profil hypermédia des réponses DORIAX (liens, affordances, format d'erreur), spécifié séparément.
