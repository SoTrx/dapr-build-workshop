---
sectionid: lab2-testing
sectionclass: h2
title: (Advanced) Test with Dapr
parent-id: lab-2
---
### Preamble

This section of the lab will focus on the testing phase of the application development cycle. Therefore, it will target a more specific audience—those with prior experience in the field. This topic is entirely optional and can be skipped in favor of the next lab.

To begin, specific vocabulary will be used in this section:

- An **application** is a partial or complete solution to a business problem. The various sub-parts of the problem are **requirements**. The set of requirements fulfilled by an application is called **functional coverage**.
- A **service** is a part of a distributed (micro)services-oriented application. Each service addresses a more or less significant part of one or more needs.
- A **module/package** is defined here as a part of the code of a service that is logically separated from the rest of the implementation through encapsulation (class, package, injection...).

### What to test

With the preceding sections, we have seen that Dapr is a tool that easily integrates into a developing application. However, there is still a lingering question.

**How to test an application using Dapr?**

Indeed, by delegating a portion of the functional coverage of an application to Dapr, a portion of our application routine is inevitably performed by a sidecar. But then, how do we test these routines?

> **Question**: List the major types of tests performed during the development of a service.

{% collapsible %}

To illustrate the answer, let's take the example of a fictitious distributed calculator capable only of addition or multiplication:

- _Plus_ performs an addition
- _Mult_ performs a multiplication
- _Lexer_ takes a string representing a calculation and transforms it into a form that the application can use. For example, "2+3" conceptually becomes "Call to _Plus_ with operands '2' and '3'"

![Calculator app](/media/lab2/testing/calculator-app.png)

Among the types of tests that we can perform on this application, we find:

- **Unit tests**: Testing modules of a service in isolation from each other.

  - Example: The add(3,6) method of the "Plus" service should return "9"

- **Integration tests**: Testing the interaction of 2..n services in an application.

  - Example: "Lexer" should call "Plus" or "Mult" depending on the operator and interpret the result.

- **Functional tests**: Testing the response of an application to a specific need.

  - Example: The "Calculator" application should return a correct result while respecting the priority of operators. 3+6\*3 =?= 21

- **End-to-end (e2e) tests**: Simulating the interaction of a user with the application. These are the most expensive of the four types mentioned and can sometimes be manual.

  - Example: The user should be able to perform operations on the calculator.

**Note**: There are _many_ other types of tests (acceptance, performance, mutation, static, A/B, scripts...) that address more specific requirements of certain applications. The field of testing is constantly evolving, and over time, some types of tests have become obsolete with languages inherently solving problems such as memory leaks (garbage collector, scoped mallocs...) or [_deadlocks/livelocks_](https://en.wikipedia.org/wiki/Deadlock) ([Rust's borrow checker](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)).

{% endcollapsible %}

> **Question**: Among the types of tests from the previous question, which ones should use Dapr?

{% collapsible %}
Whenever communication between services is included in the tests, Dapr should be used. Indeed, tests including communication between services try to reproduce a real-life situation. If Dapr is used as a communication medium in the application's life, it should also be used in the tests. End-to-end and functional tests are therefore relevant.

On the contrary, unit tests should not have external dependencies.

However, providing a categorical answer for integration tests is more challenging. If integration includes communication between services, it may sometimes be judicious to decide whether or not to inject the sidecar to test communication failure.

{% endcollapsible %}

### How to test

For this lab, we will propose three methodologies for testing an application with Dapr.

#### Method 1: Mandatory Middleware

The first method is the simplest. A feature of Dapr is its ability to function independently of an orchestrator. In this perspective, it is then possible to consider Dapr as a prerequisite for **integration tests**.
The CI/CD pipeline would be initiated by installing Dapr in the CI environment (or possibly in a test Kubernetes cluster like KinD).

While this method has the advantage of not requiring a particular test structure, it may, in some cases, constrain the use of Dapr in **unit tests** as well (e.g., injecting an SDK into a class, subscribing to an event in the constructor), which may not be desirable.

#### Method 2: Localhost Interface

Another method is to simply replace calls to sidecars with mocks. The default port for Dapr is **3500**. Before running a battery of tests, it is possible to start a server (like [APItest](https://apitest.dev/) in Go) on this port and mock the sidecar responses.

This method has the advantage of giving the developer control over sidecar responses, allowing for a broader exploration of edge cases. It also does not structure the development of the application and can be applied in all cases. The main drawbacks of this method are the need to know the sidecar response format and the requirement for the test server to be carefully managed to avoid uncontrolled growth.

#### Method 3: Dependency Injection

The last method is the most complex to implement but also gives the developer the most control. The principle of this method is to consider Dapr as an injectable class/module of a [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) container.

Dependency injection is a programming method that determines at runtime the chain of dependencies between the objects/classes of a program. Thus, these dependencies are reconfigurable based on the runtime environment of the application, allowing for greater flexibility when this dependency chain is complex.

Applied to the testing domain, this method allows testing classes in isolation that are composed of [composite classes](). An example could be as follows:

### State Store Example

We want to persist the state of our application. To achieve this, we will use a state storage module utilizing Dapr. A naive implementation would be:

```typescript
export class Store {
  public static readonly STATE_KEY = "KEY";

  private readonly dapr = new DaprClient();

  constructor(
    /** Dapr component name */
    private readonly storeName: string
  ) {}

  async getState(): Promise<IRecordingState> {
    // Retrieve the state
    const state = await this.dapr.state.get(this.storeName, Store.STATE_KEY);
    // Processing...
    return state;
  }

  async setState(state: IRecordingState) {...}
}
```

However, by including the initialization of Dapr in the class itself, we encounter the issues of the first method. Indeed, the simple instantiation of this class will require Dapr to run in the background, including during unit tests.

The idea is to decouple the application-specific code from the SDK call. Instead of instantiating the Dapr SDK directly, we consider the SDK as a possible implementation of the **IStoreProxy** interface. This interface will then be used in the application code, the state store.

![Example state store](/media/lab2/testing/example-state-store.png)

```typescript
/** A storage backend must be able to... */
export interface IStoreProxy {
  /** Store information */
  save(storeName: string, [{ key: string, value: any }]): Promise<void>;
  /** Retrieve information */
  get(storeName: string, key: string): Promise<any>;
}
```

By using this additional abstraction layer, it is possible to define a class that can optionally use Dapr or any other implementation satisfying the **IStoreProxy** interface.

```typescript
@injectable()
export class Store<T extends IStoreProxy> {
  public static readonly STATE_KEY = "KEY";

  constructor(
    /** storeProxy can be an instance of the Dapr client or a Mock */
    @inject(TYPES.StoreProxy) private readonly storeProxy: T,
    /** Dapr component name */
    private readonly storeName: string
  ) {}

  async getState(): Promise<IRecordingState> {
    // Retrieve the state via the proxy
    const state = await this.storeProxy.get(this.storeName, Store.STATE_KEY);
    // Processing...
    return state;
  }

  async setState(state: IRecordingState) {...}
}
```

In production, the value of the proxy can be set to that of the Dapr SDK client.

```typescript
export const container = new Container();
// The proxy is set to the value of the Dapr JS SDK client
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

While in **unit tests**, a mock will be used, eliminating the requirement to start Dapr.

```typescript
describe("State store", () => {
  /** Note: these tests are for example purposes; they have no practical interest */
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
  // We replace the implementation of Dapr with a pseudo-implementation before the tests
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

Having the advantage of allowing complete control over the relationship between Dapr and the rest of the service, this method is entirely structuring; it requires the application to use dependency injection.

## Conclusion

Testing with Dapr in a microservices configuration, where each service performs a very simple task, is relatively easy.

In cases where services evolve and their responsibilities increase, the developer will have to choose a testing implementation based on the desired level of control granularity.
