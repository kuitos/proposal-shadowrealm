# Realms Explainer

<!-- vscode-markdown-toc -->
* [Introduction](#Introduction)
* [API (TypeScript Format)](#APITypeScriptFormat)
* [Motivations](#Motivations)
* [Clarifications](#Clarifications)
	* [Terminology](#Terminology)
	* [The Realm's Global Object](#TheRealmsGlobalObject)
	* [Evaluation](#Evaluation)
	* [Module graph](#Modulegraph)
	* [Compartments](#Compartments)
* [Use Cases](#UseCases)
	* [Third Party Scripts](#ThirdPartyScripts)
	* [Code Testing](#CodeTesting)
		* [Running tests in a Realm](#RunningtestsinaRealm)
		* [Test FWs + Tooling to run tests in a realm](#TestFWsToolingtoruntestsinarealm)
	* [Codebase segmentation](#Codebasesegmentation)
	* [Template libraries](#Templatelibraries)
	* [DOM Virtualization](#DOMVirtualization)
		* [DOM Virtualization: AMP WorkerDOM Challenge](#DOMVirtualization:AMPWorkerDOMChallenge)
* [More Examples](#MoreExamples)
* [Modules](#Modules)
* [General Goals and Values for Realms](#GeneralGoalsandValuesforRealms)
	* [Integrity](#Integrity)
		* [Security vs Integrity](#SecurityvsIntegrity)
	* [Virtualized Environment](#VirtualizedEnvironment)
		* [DOM mocking](#DOMmocking)
* [ Alternatives](#Alternatives)
	* [Status Quo](#StatusQuo)
	* [Iframes](#Iframes)
	* [Why not separate processes?](#Whynotseparateprocesses)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

## <a name='Introduction'></a>Introduction

Realms are a distinct global environment, with its own global object containing its own intrinsics and built-ins (standard objects that are not bound to global variables, like the initial value of Object.prototype).

The Realms API allow execution of script within an isolated [global environment record](https://tc39.es/ecma262/#sec-global-environment-records). Just like the the Global Environment Record, each new realm will provide the [bindings for built-in globals](https://tc39.es/ecma262/#table-7), properties of the [global object](https://tc39.es/ecma262/#sec-global-object), and for all top-level declarations that occur within the Realm's Script.

The Realms API does not create - or rely on - a new executing thread. New realms will not behave like different [Agents](https://tc39.es/ecma262/#sec-agents). Although, the Realms API offers a way to import modules asynchronously, just like the `import()` expression, following the same design patterns. It also offers a way to execute code synchronously, through regular evaluation built-ins.

Any code executed within this realm may introduce changes to global variables or built-ins, but limited to the realm global Record.

## <a name='APITypeScriptFormat'></a>API (TypeScript Format)

```ts
declare class Realm {
    constructor();
    readonly globalThis: typeof globalThis;
    import(specifier: string): Promise<Namespace>;
}
```

## <a name='Motivations'></a>Motivations

Why do developers need realms?

It's quite common for an applications to contain programs from multiple sources, whether from different teams, vendors, package managers, etc. These programs must currently contend for the global shared resources, specifically, the shared global object, and the side effect of executing those programs are often hard to observe, causing conflicts between the different programs, and potentially affecting the integrity of the app itself.

Various examples where Realms can be used to avoid this:

  * Web-based IDEs or any kind of 3rd party code execution uses same origin evaluation.
  * DOM Virtualization (e.g.: AMP)
  * Test frameworks and reporters(in-browser tests, but also in node using `vm`).
  * testing/mocking (e.g.: jsdom)
  * Most plugin mechanism for the web (e.g., spreadsheet functions).
  * Sandboxing (e.g.: Oasis Project)
  * Server side rendering (to avoid collision and data leakage)
  * in-browser code editors
  * in-browser transpilation

Note: the majority of the examples above will require synchronous operations to be supported, which makes it almost impossible to use Workers & co., or any other isolation mechanism in browsers and nodejs today.

## <a name='Clarifications'></a>Clarifications

### <a name='Terminology'></a>Terminology

In the Web Platform, both `Realm` and `Global Object` are usually associated to Window, Worker, and Worklets semantics. They are also associated to their detachable nature.

This proposal is limited to the semantics specified by ECMA-262 with no extra requirements from the web counterparts.

### <a name='TheRealmsGlobalObject'></a>The Realm's Global Object

- The Realm's [Global Object](https://tc39.es/ecma262/#sec-ordinary-object) is an [Ordinary Object](https://tc39.es/ecma262/#sec-ordinary-object).
- A Realm Object (and its global object) are not detachable
- A Realm Object has a lifeline to its incubator Realm

![](assets/detachable-realms.png)

### <a name='Evaluation'></a>Evaluation

The Realm API does not introduce a new way to evaluate code, it reuses the existing evaluation mechanisms:

- If you disable CSP `unsafe-eval`, you cannot evaluate code synchronously.
- If you disable CSP `default-src`, you cannot use `Realm.prototype.import()`.

### <a name='Modulegraph'></a>Module graph

Every Realm object has its own module graph.

```js
const realm = new Realm();

// imports code that executes within its own environment.
const { doSomething } = await realm.import('./file.js');

doSomething();
```

### <a name='Compartments'></a>Compartments

- A new [compartment](https://github.com/tc39/proposal-compartments) provides a new Realm constructor
- A realm object created from a compartment is subject to the compartment's virtualization mechanism

```js
const compartment = new Compartment(options);
const VirtualizedRealm = compartment.globalThis.Realm;
const realm = new VirtualizedRealm();
const { doSomething } = await realm.import('./file.js');
```

## <a name='UseCases'></a>Use Cases

- Third Party Scripts
- Code Testing
- Codebase segmentation
- Template libraries
- DOM Virtualization

### <a name='ThirdPartyScripts'></a>Third Party Scripts

- We want a quick and simple execution of third party scripts.
- Not one, not two, not three, but many scripts.
- We don't need a new host/agent, but just integrity of the expected Intrinsics.
- Non-blocking async loading and execution via `realm.import`.
- No immediate access to application globals (e.g. `window`) is also convenient.
- No immediate access to unforgeables (e.g. `window.top`, `window.location`) is great.

```js
import { api } from 'pluginFramework';
const realm = new Realm();

realm.globalThis.api = api;

await realm.import('./plugin1.js');
```

### <a name='CodeTesting'></a>Code Testing

While threading is good for testing, the layering from Realms is also great.

Test frameworks can use Realms to inject code and also control the order the injections if necessary.

#### <a name='RunningtestsinaRealm'></a>Running tests in a Realm

```js
import { test } from 'testFramework';
const realm = new Realm();

realm.globalThis.test = test;
await realm.import('./main-spec.js');

test.report();
```

#### <a name='TestFWsToolingtoruntestsinarealm'></a>Test FWs + Tooling to run tests in a realm

```js
const realm = new Realm();
const [ framework, { tap } ] = await Promise.all([
 realm.import('testFramework'),
 realm.import('reporters')
]);

framework.use(tap);
await realm.import('./main-spec.js');
```

### <a name='Codebasesegmentation'></a>Codebase segmentation

- A big codebase tend to evolve slowly.
- Old code vs new code is a constant struggle.
- Modifying code to resolve a conflict (e.g.: global variables) is non-trivial.
- With a lightweight mechanism to preserve the integrity of the intrinsics you could isolate libraries, or logical pieces of the codebase.

### <a name='Templatelibraries'></a>Template libraries

Code generation should not be subject to pollution (global, prototype, etc.). E.g.: Lodash's `_.template()` uses `Function(...)` to create a compiled template function, instead it could use a Realm to avoid leaking global variables. The same Realm could be reused multiple times.

```js
var compiled = _.template(
'<% _.forEach(users, function(user) { %><li><%- user %></li><% }); %>'
);

compiled({ users: ['user1', 'user2'] });
```

### <a name='DOMVirtualization'></a>DOM Virtualization

- We still want things to still interact with the DOM
- We don't want to spend any excessive amount of resources
- We want to emulate the DOM as best as possible
  - Requiring other libraries to change to meet our requirements is difficult

```js
import virtualDocument from 'virtual-document';

const realm = new Realm();

realm.globalThis.document = virtualDocument;

await realm.import('./publisher-amin.js');
```

#### <a name='DOMVirtualization:AMPWorkerDOMChallenge'></a>DOM Virtualization: AMP WorkerDOM Challenge

Problem: Element.getBoundingClientRect() doesn't work over async comm channels (i.e. [worker-dom](https://github.com/ampproject/worker-dom)).

![AMP WorkerDOM Challenge diagram](assets/amp-workerdom-challenge.png)

## <a name='MoreExamples'></a>More Examples

See some other examples [here](EXAMPLES.md).

## <a name='Modules'></a>Modules

In principle, the Realm proposal does not provide the controls for the module graphs. Every new Realm initializes its own module graph, while any invocation to `Realm.prototype.import()` method, or by using `import()` when evaluating code inside the realm, will populate this module graph. This is analogous to same-domain iframes, and VM in nodejs.

However, the [Compartments]() proposal plans to provide the low level hooks to control the module graph per Realm. This is one of the intersection semantics between the two proposals. Although, the biggest challenge when sharing modules across realms is the hazard of the identity discontinuity. For example, when interacting with a module evaluated in another Realm:

```js
import { x } from '/path/to/foo.js';
const d = new Date();
x(d);
```

If `x` function attempt to check `arguments[0] instanceof Date`, it yields `false` since the date object was created from a constructor from another realm.

There are some precedents on how to solve the identity discontinuity issues by using a "near membrane" via proxies. For now, providing the Realms as building block seems sufficient.

There is one important thing to keep in mind when it comes to sharing module graphs. The ESM linkage is not asynchronous. This dictates that in order to share modules between realms, those realms should share the same process, otherwise the bindings between those modules cannot work according to the language. This is another reason to support our claim that Realms should be running within the same process.

## <a name='GeneralGoalsandValuesforRealms'></a>General Goals and Values for Realms

### <a name='Integrity'></a>Integrity

We believe that realms can be a good complement to integrity mechanisms by providing ways to evaluate code who access different object graphs (different global objects) while maintaining the integrity of the outer realm. A concrete example of this is the Google's AMP current mechanism:

* Google News App creates multiples sub-apps that can be presented to the user.
* Each sub-app runs in a cross-domain iframe (communicating with the main app via post-message).
* Each vendor (one per app) can attempt to enhance their sub-app that display their content by executing their code in a realm that provide access to a well defined set of APIs to preserve the integrity of the sub-app.

There are many examples like this for the web: Google Sheets, Figma's plugins, or Salesforce's Locker Service for Web Components.

#### <a name='SecurityvsIntegrity'></a>Security vs Integrity

There are also other more exotic cases in which measuring of time ("security") is not a concern, especially in IOT where many devices might not have process boundaries at all. Or examples where security is not a concern, e.g.: test runners like jest (from facebook) that relies on nodejs, JSDOM and VM contexts to execute individual tests while sharing a segment of the object graph to achieve the desired performance budget. No doubts that this type of tools are extremely popular these days, e.g.: JSDOM has 10M installs per week according to NPM's registry.

### <a name='VirtualizedEnvironment'></a>Virtualized Environment

The usage of different realms allow customized access to the global environment. To start, The global object could be immediately frozen.

```js
let realm = new Realm();

Object.freeze(realm.globalThis);
```

In web browsers, this is currently not possible. The way to get manage new realms would be through iframes, but they also share a window proxy object.

```js
let iframe = document.createElement('iframe');
document.body.appendChild(iframe);
const rGlobal = iframe.contentWindow; // same as iframe.contentWindow.globalThis

Object.freeze(rGlobal); // TypeError, cannot freeze window proxy
```

The same iframe approach won't also have a direct access to import modules dynamically.

```js
realm.import('./file.js');

// instead of (roughly)
iframe.contentWindow.eval("import('./file.js')");
```

#### <a name='DOMmocking'></a>DOM mocking

The Realms API allows a much smarter approach for DOM mocking, where the globalThis can be setup in userland. This also takes advantage of the Realm constructor being subclassable:

```js
class FakeWindow extends Realm {
  constructor(...args) {
    super(...args);
    let globalThis = this.globalThis;

    globalThis.document = new FakeDocument(...);
    globalThis.alert = new Proxy(fakeAlert, { ... });
    ...
  }
}
```

This code allows a customized set of properties to each new Realm and avoid issues on handling immutable accessors/properties from the Window proxy. e.g.: `window.top`, `window.location`, etc..

## <a name='Alternatives'></a> Alternatives

### <a name='StatusQuo'></a>Status Quo

Using VM module in nodejs, and same-domain iframes in browsers. Although, VM modules in node is a very good approximation to this proposal, iframes are problematic. 

### <a name='Iframes'></a>Iframes

Developers can technically already create a new Realm by creating new same-domain iframe, but there are few impediments to use this as a reliable mechanism:

* the global object of the iframe is a window proxy, which implements a bizarre behavior, including its unforgeable proto chain.
* there are multiple ~~unforgeable~~ unvirtualizable objects due to the DOM semantics, this makes it almost impossible to eliminate certain capabilities while downgrading the window to a brand new global without DOM.
* global `top` reference is ~~unforgeable~~ not virtualizable and leaks a reference to another global object. The only way to null out this behavior is to detach the iframe, which imposes other problems, the more relevant is dynamic `import()` calls, __which cannot work in detached realms__.

### <a name='Whynotseparateprocesses'></a>Why not separate processes?

This is another alternative, creating a Realm that runs in a separate process, while allowing users to define and create their own protocol of communication between these processes. This alternative was discarded for two main reasons:

1. There are existing mechanism to achieve this today in both browsers, and nodejs. E.g.: cross domain iframes, workers, etc. They seem to be good enough when asynchronous communication is sufficient to implement the feature.
2. Asynchronous communication is a deal-breaker for many use-cases, and sometimes it just added complexity for cases where a same-process Realm is sufficient.