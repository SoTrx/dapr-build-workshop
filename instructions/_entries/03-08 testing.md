---
sectionid: lab2-testing
sectionclass: h2
title: (Advanced) Test with Dapr
parent-id: lab-2
---

## Préambule

Cette partie du laboratoire va s'intéresser à la phase de test du cycle de développement d'une application. Par conséquent elle concernera un public plus restreint, celui ayant déjà une expérience dans le domaine. Ce sujet est donc entièrement **facultatif** est peut être passé en faveur du prochain laboratoire.

Pour commencer, un certain vocabulaire sera utilisé dans cette partie:

- Une **application** est une réponse partielle ou totale à un problème métier. Les différentes sous-parties du problème sont des **besoins**. L'ensemble des besoins auquel répond une application est appelé **couverture fonctionelle**.
- Un **service** est une partie d'une application distribuée orientée (micro)services. Chaque service répond à une partie plus ou moins importante d'un ou plusieurs besoin(s).
- Un **module/package** est ici défini comme une partie du code d'un service étant séparé logiquement du reste de l'implémentation via une encapsulation (classe, package, injection...)

### Que tester

Avec toutes les parties précédentes, nous avons pu voir que Dapr est un outil qui vient se greffer facilement sur une application en cours de développement.
Il reste cependant une question en suspend.

**Comment tester une application utilisant Dapr ?**

En effet, en déléguant une partie de la couverture fonctionelle d'une application à Dapr, une partie de notre routine applicative est forcément effectuée par un sidecar. Mais alors comment tester ces routines ?

> **Question**: Citez les grands types de tests effectués lors du développement d'un service

{% collapsible %}

Pour illustrer la réponse, prenons l'exemple d'une calculatrice distribuée factice ne pouvant que additionner ou multiplier:

- _Plus_ effectue une addition
- _Mult_ effectue une multiplication
- _Lexer_ prend une chaîne de caractères représentant un calcul, et le transforme dans une forme que l'application peut utiliser. "2+3" devenant conceptuellement "Appel à _Plus_ avec les opérandes '2' et '3'"

![Calculator app](/media/lab2/testing/calculator-app.png)

Parmi les types de tests que nous pourrons effectuer sur cette application nous trouverons:

- **Les tests unitaires** : Testant des modules d'un service en isolation les uns des autres.

  - Ex: La méthode add(3,6) du service "Plus" doit renvoyer "9"

- **Les tests d'intégration** : Testant l'interaction de 2..n services d'une application

  - Ex: "Lexer" doit appeler "Plus" ou "Mult" en fonction de l'opérateur et interpréter le résultat

- **Les tests fonctionnels**: Testant la réponse d'une application à un besoin en particulier.

  - Ex: L'application "Calculatrice" doit renvoyer une résultat correct en respectant la priorité des opérateurs
  - 3+6\*3 =?= 21

- **Les test de bout en bout (end2end)** : Simulant l'intéraction d'un utilisateur avec l'application, ce sont les plus couteux des 4 types évoqués, et ils peuvent parfois êtres manuels.
  - Ex: L'utilisateur doit pouvoir effectuer des opérations sur la calculatrice

**Note**: Il existe _beaucoup_ d'autres types de tests (acceptation, performance, mutation, static, A/B, scripts...) qui répondent à des exigences plus particulières de certaines applications.
Le domaine des tests évolue contamment, et au cours du temps, certains types de tests sont devenus obsolètes avec des langages répondant par leur conception même à des problèmes comme la fuite mémoire (garbage collector, scoped mallocs...) ou les [_deadlocks/livelocks_](https://en.wikipedia.org/wiki/Deadlock) ([borrow checker de Rust](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)).

{% endcollapsible %}

> **Question**: Parmi les types de tests de la question précédente, quels sont ceux qui devraient utiliser Dapr ?

{% collapsible %}
Dès lors que la communication entre services est incluse dans les tests, Dapr devrait être utilisé. En effet, les tests incluant la communication entre services essaient de reproduire une situation réelle. Si dans la vie de l'application, Dapr est utilisé comme médium de communication, il devrait aussi être utilisé dans les tests.
Les tests end2end et fonctionnels sont donc concernés.

Les tests unitaires au contraire ne devraient pas avoir de dépendance externe.

Il est cependant plus difficile d'avoir une réponse catégorique pour les tests d'intégration. Si l'intégration inclue la communication entre services, il peut parfois être judicieux de se laisser la liberté ou non d'injecter le sidecar afin de tester un échec de communication.

{% endcollapsible %}

### Comment tester

Pour ce laboratoire, nous allons proposer trois méthodologies pour tester une application avec Dapr.

#### Méthode 1: Le middleware obligatoire

La première méthode est la plus simple. Une fonctionnalité de Dapr est de pouvoir fonctionner indépendament d'un orchestrateur.
Dans cette optique, il est alors possible de simplement considérer Dapr comme un pré-requis pour les tests **d'intégration**.
Le pipeline de CI serait alors initié par l'installation de Dapr dans l'environnement de CI (ou éventuellement dans un cluster Kubernetes de test de type KinD).

Cette méthode, si elle a l'avantage de se pas demander de structure de test particulière, peut cependant contraindre dans certains cas (injection d'un SDK dans une classe, subscription à un evènement dans le constructeur...) à également devoir utiliser Dapr dans les test **unitaires**, ce qui n'est pas forcément souhaitable.

#### Méthode 2: L'interface localhost

Une autre méthode est de simplement remplacer les appels aux sidecars par des _mocks_. Le port par défaut de Dapr est **3500**. Avant d'effectuer une batterie de tests, il est alors possible de démarrer un serveur (type [APItest](https://apitest.dev/) en Go) sur ce port et de _mocker_ les réponses du sidecar.

Cette méthode a l'avantange de donner au développeur le contrôle des réponses du sidecar, permettant une exploration plus vaste des cas limites. Elle ne structure pas non plus le développement de l'application et peut être appliquée dans tous les cas.
Les inconvénients principaux de cette méthode est qu'il faut connaître le format de réponse du sidecar, et que le serveur de test nécessite une attention particulière pour ne pas grandir de manière incontrolée.

#### Méthode 3: Injection de dépendances

La dernière méthode est la plus complexe à mettre en place mais aussi celle qui donne le plus de contrôle au développeur. Le principe de cette méthode est de considérer Dapr comme une classe/module injectable d'un conteneur d'[injection de dépendances](https://fr.wikipedia.org/wiki/Injection_de_d%C3%A9pendances).

L'injection de dépendances est une méthode de programmation qui consiste à déterminer à l'exécution la chaîne de dépendances entre les objets/classes d'un programme.
Ainsi, ces dépendances sont reconfigurables en fonction de l'environnement d'exécution de l'application, permettant une meilleure flexibilité quand cette chaîne de dépendance est complexe.

Appliquée au domaine des tests, cette méthode permet de tester en isolation des [classes composées]().

Un exemple pourrait être le suivant :

Nous voulons sauvegarder l'état de notre application. Pour cela, nous allons un module de stockage d'état, utilisant Dapr.
Une implémentation naïve serait :

```ts
export class Store {
  public static readonly STATE_KEY = "KEY";

  private readonly dapr = new DaprClient()

  constructor(
    /** Nom du composant Dapr */
    private readonly storeName: string
  ) {}

  async getState(): Promise<IRecordingState> {
    // Récupérer l'état
    const state = await this.dapr.state.get(this.storeName, ExtStore.STATE_KEY)
    // Traitements...
    return state
  }

  async setState(state: IRecordingState) {...}

}
```

Cependant, en incluant l'initialisation de Dapr dans la classe elle-même, nous retombons sur les problèmes de la première méthode. En effet, la simple instanciation de cette classe demandera à ce que Dapr fonctionne en arrière-plan, y compris lors des tests unitaires.

L'idée est alors de découpler le code propre à l'application de l'appel au SDK. Au lieu d'instancier le SDK Dapr directement, nous considérons le SDK comme une implémentation possible réalisant une interface **IStoreProxy**. Cette interface sera ensuite utilisée dans le code de l'application, le state store.

![Example state store](/media/lab2/testing/example-state-store.png)

```ts
/** Un backend de stockage doit pouvoir... */
export interface IStoreProxy {
  /** Stocker une information */
  save(storeName: string, [{ key: string, value: any }]): Promise<void>;
  /** Récupérer une information */
  get(storeName: string, key: string): Promise<any>;
}
```

En utilisant cette couche d'abstraction supplémentaire, il est alors possible de définir une classe qui pourra au choix utiliser Dapr ou n'importe quelle autre implémentation satisfaisant l'interface **IStoreProxy**.

```ts
@injectable()
export class Store<T extends IStoreProxy> {
  public static readonly STATE_KEY = "KEY";

  constructor(
    /** storeProxy peut être une instance du client Dapr ou un Mock */
    @inject(TYPES.StoreProxy) private readonly storeProxy: T,
    /** Nom du composant Dapr */
    private readonly storeName: string
  ) {}

  async getState(): Promise<IRecordingState> {
    // Récupérer l'état via le proxy
    const state = await this.storeProxy.get(this.storeName, ExtStore.STATE_KEY)
    // Traitements...
    return state
  }

  async setState(state: IRecordingState) {...}

}
```

En production, la valeur du proxy pourra être fixée à celle du client Dapr

```ts
export const container = new Container();
// Le proxy est fixé à la valeur du client du SDK JS de DAPR
container.bind(TYPES.StoreProxy).toConstantValue(new DaprClient().state);
container
  .bind(TYPES.StateStore)
  .toConstantValue(
    new ExternalStore(
      container.get<IStoreProxy>(TYPES.StoreProxy),
      process.env.STORE_NAME
    )
  );
```

Tandis que dans les tests **unitaires**, un mock sera utilisé, supprimant le pré-requis de démarrer Dapr.

```ts
describe("State store", () => {
  /** Note : ces tests sont là pour l'exemple, ils n'ont pas d'intérêt pratique*/
  describe("GET", () => {
    it("Store empty", async () => {
      const ss = getDynamicExternalStore();
      const state = await ss.getState();
      expect(state).toBeUndefined();
    });
    it("Store non-empty", async () => {
      const mockData: IRecordingState = {
        recordsIds: ["0"],
      };
      const ss = getDynamicExternalStore(mockData);
      const state = await ss.getState();
      expect(state).toEqual(mockData);
    });
  });
});

function getDynamicExternalStore(startValue?: IRecordingState) {
  let state = startValue;
  // On remplace l'implémentation de Dapr par une pseudo-implémentation avant les tests
  const mockProxy: IStoreProxy = {
    get<T>(storeName: string, key: string): Promise<T> {
      return Promise.resolve(state as unknown as T);
    },
    save<T>(
      storeName: string,
      keyVal: readonly [{ key: any; value: any }]
    ): Promise<void> {
      state = keyVal[0].value;
      return Promise.resolve(undefined);
    },
  };
  return new ExternalStore(mockProxy, "");
}
```

Ayant l'avantage de permettre un contrôle total sur la relation entre Dapr et le reste du service, cette méthode est cepedant entièrement structurante, elle impose que l'application utilise l'injection de dépendances.

## Conclusion

Il est relativement facile de tester avec Dapr dans une configuration micro-services, où chaque service effectue une tâche très simple.

Dans le cas où les services évoluent et leurs responsabilités augmentent, le développeur devra choisir une implémentation de tests en fonction de la granularité de contrôle voulu.
