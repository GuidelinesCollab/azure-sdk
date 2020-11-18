---
title: "TypeScript Guidelines: API Design"
keywords: guidelines typescript
permalink: typescript_design.html
folder: typescript
sidebar: general_sidebar
---

## Introduction

The TypeScript guidelines are for the benefit of client library designers targeting service applications written in TypeScript for the benefit of both TypeScript and JavaScript.

### Design principles

The Azure SDK should be designed to enhance the productivity of developers connecting to Azure services. Other qualities (such as completeness, extensibility, and performance) are important but secondary. Productivity is achieved by adhering to the principles described below:

#### Idiomatic

* The SDK should follow the general design guidelines and conventions for the target language. It should feel natural to a developer in the target language.
* We embrace the ecosystem with its strengths and its flaws.
* We work with the ecosystem to improve it for all developers.

#### Consistent

* Client libraries should be consistent within the language, consistent with the service and consistent between all target languages. In cases of conflict, consistency within the language is the highest priority and consistency between all target languages is the lowest priority.
* Service-agnostic concepts such as logging, HTTP communication, and error handling should be consistent. The developer should not have to relearn service-agnostic concepts as they move between client libraries.
* Consistency of terminology between the client library and the service is a good thing that aids in diagnosability.
* All differences between the service and client library must have a good (articulated) reason for existing, rooted in idiomatic usage rather than whim.
* The Azure SDK for each target language feels like a single product developed by a single team.
* There should be feature parity across target languages. This is more important than feature parity with the service.

#### Approachable

* We are experts in the supported technologies so our customers, the developers, don't have to be.
* Developers should find great documentation (hero tutorial, how to articles, samples, and API documentation) that makes it easy to be successful with the Azure service.
* Getting off the ground should be easy through the use of predictable defaults that implement best practices. Think about progressive concept disclosure.
* The SDK should be easily acquired through the most normal mechanisms in the target language and ecosystem.
* Developers can be overwhelmed when learning new service concepts. The core use cases should be discoverable.

#### Diagnosable

* The developer should be able to understand what is going on.
* It should be discoverable when and under what circumstances a network call is made.
* Defaults are discoverable and their intent is clear.
* Logging, tracing, and exception handling are fundamental and should be thoughtful.
* Error messages should be concise, correlated with the service, actionable, and human readable. Ideally, the error message should lead the consumer to a useful action that they can take.
* Integrating with the preferred debugger for the target language should be easy.

#### Dependable

* Breaking changes are more harmful to a user's experience than most new features and improvements are beneficial.
* Incompatibilities should never be introduced deliberately without thorough review and very strong justification.
* Do not rely on dependencies that can force our hand on compatibility.

### General Guidelines

{% include requirement/MUST id="ts-follow-general-guidelines" %} follow the [General Azure SDK Guidelines].

{% include requirement/MUST id="general-implementing-httppipeline" %} use the HTTP pipeline component within `@azure/core-http` package for communicating to service REST endpoints.

TODO: Add discussion of policies and if/when to add them, and how to do this in JS in an appendix.

{% include requirement/MUST id="ts-repository-location" %} locate all source code in the [azure/azure-sdk-for-js] GitHub repository.

{% include requirement/MUST id="ts-engineering-systems" %} follow Azure SDK engineering systems guidelines for working in the [azure/azure-sdk-for-js] GitHub repository.

{% include requirement/MUST id="ts-azure-scope" %} follow these guidelines if you publish your client library under the `@azure` scope in npm.

{% include requirement/SHOULD id="ts-azure-scope-for-others" %} follow these guidelines even if you're not publishing an Azure library under the `@azure` scope.

### HTTP and Non-HTTP Services

Currently, this document describes guidelines for client libraries exposing HTTP/REST services. It may be expanded in the future to cover other, non-REST, services.

{% include requirement/MUST id="general-other-protocols-consult-on-policies" %} reach out to the [Architecture Board] for guidance on non-HTTP Services.

### Terminology

AMD Module
: A module format often used in the browser, for example as implemented by [RequireJS].

[CommonJS][cjs] (CJS)
: The module format of Node.js (`require`, `module.exports`).

[ECMAScript Module][esm] (ES Module or ESM)
: The standard import/export syntax defined in ECMAScript 6.

## Azure SDK API Design

The API surface of your client library must have the most thought as it is the primary interaction that the consumer has with your service.

### The Service Client {#ts-apisurface-serviceclient}

Your API surface will consist of one or more _service clients_ that the consumer will instantiate to connect to your service, plus a set of supporting types. The basic shape of JavaScript service clients is shown in the following example:

{% include requirement/MUST id="ts-apisurface-serviceclientnaming" %} name service client types with the _Client_ suffix.

TODO: Use a real Track 2 service example?

```javascript
export class ServiceClient {
  // client constructors have overloads for handling different
  // authentication schemes.
  constructor(connectionString: string, options?: ServiceClientOptions);
  constructor(host: string, credential: TokenCredential, options?: ServiceClientOptions);
  constructor(...) { }

  // Service methods. Options take at least an abortSignal.
  async createItem(options?: CreateItemOptions): CreateItemResponse;
  async deleteItem(options?: DeleteItemOptions): DeleteItemResponse;

  // Simple paginated API
  listItems(): PagedAsyncIterableIterator<Item, ItemPage> { }

  // Clients for sub-resources
  getItemClient(itemName: string) { }
}
```

{% include requirement/MUST id="ts-apisurface-serviceclientnamespace" %} place service client types that the consumer is most likely to interact as a top-level export from your library.  That is, the service client type should be something that can be imported directly by the consumer.

{% include requirement/MUST id="ts-apisurface-supportallfeatures" %} support 100% of the features provided by the Azure service the client library represents.

Gaps in functionality cause confusion and frustration among developers. A feature may be omitted if there isn't support on the platform. For example, a library that depends on local file system access may not work in a browser.

#### Client constructors and factories

{% include requirement/MUST id="ts-apisurface-serviceclientconstructor" %} allow the consumer to construct a service client with the minimal information needed to connect and authenticate to the service.

{% include requirement/SHOULD id="ts-use-constructor-overloads" %} provide overloaded constructors for all client construction scenarios.

Don't use static methods to construct a client unless an overload would be ambiguous.  Prefix the static method with `from` if you require a static constructor.

##### Authentication

Azure services use different kinds of authentication schemes to allow clients to access the service.  Conceptually, there are two entities responsible in this process: a credential and an authentication policy.  Credentials provide confidential authentication data.  Authentication policies use the data provided by a credential to authenticate requests to the service.

{% include requirement/MUST id="ts-apisurface-supportcancellation" %} support all authentication techniques that the service supports.

{% include requirement/MUST id="ts-apisurface-check-cancel-on-io-calls" %} use credential and authentication policy implementations from the Azure Core library where available.

{% include requirement/MUST id="general-apisurface-no-leaking-implementation" %} provide credential types that can be used to fetch all data needed to authenticate a request to the service. Credential types should be non-blocking and atomic.  Use credential types from the `@azure/core-auth` library where possible.

{% include requirement/MUST id="general-apisurface-auth-in-constructors" %} provide service client constructors or factories that accept any supported authentication credentials.

Client libraries may support connection strings __ONLY IF__ the service provides a connection string to users via the portal or other tooling. Connection strings are easily integrated into an application by copy/paste from the portal.  However, connection strings don't allow the credentials to be rotated within a running process.

{% include requirement/MUSTNOT id="general-apisurface-no-connection-strings" %} support constructing a service client with a connection string unless such connection string is available within tooling (for copy/paste operations).

##### Client Configuration (Options) {#ts-options}

The guidelines in this section apply to options passed in options bags to clients, whether methods or constructors. When referring to option names, this means the key of the object users must use to specify that option when passing it into a method or constructor.

{% include requirement/MUST id="ts-naming-options" %} name the type of the options bag as `<class name>Options` and `<method name>Options` for constructors and methods respectively.

{% include requirement/MUST id="ts-options-abortSignal" %} name abort signal options `abortSignal`.

{% include requirement/MUST id="ts-options-suffix-durations" %} suffix durations with `In<Unit>`. Unit should be `ms` for milliseconds, and otherwise the name of the unit. Examples include `timeoutInMs` and `delayInSeconds`.

#### Service Methods

##### Async Methods

{% include requirement/MUST id="ts-use-promises" %} use built-in promises for asynchronous operations. You may provide overloads that take a callback. Don't import a polyfill or library to implement promises.

Promises were added to JavaScript ES2015. ES2016 and later added `async` functions to make working with promises easier. Promises are broadly supported in JavaScript runtimes, including all currently supported versions of Node.

{% include requirement/SHOULD id="ts-use-async-functions" %} use `async` functions for implementing asynchronous library APIs.

If you need to support ES5 and are concerned with library size, use `async` when combining asynchronous code with control flow constructs.  Use promises for simpler code flows.  `async` adds code bloat (especially when targeting ES5) when transpiled.

##### Service Method Naming

{% include requirement/MUST id="ts-apisurface-standardized-verbs" %} standardize verb prefixes within a set of client libraries for a service (see [approved verbs](#ts-approved-verbs)).

The service speaks about specific operations in a cross-language manner within outbound materials (such as documentation, blogs, and public speaking).  The service can't be consistent across languages if the same operation is referred to by different verbs in different languages.

{% include requirement/SHOULD id="ts-approved-verbs" %} use one of the approved verbs in the below table when referring to service operations.

|Verb|Parameters|Returns|Comments|
|-|-|-|-|
|`create\<Noun>`|key, item|Created item|Create new item. Fails if item already exists.|
|`upsert\<Noun>`|key, item|Updated or created item|Create new item, or update existing item. Verb is primarily used in database-like services |
|`set\<Noun>`|key, item|Updated or created item|Create new item, or update existing item. Verb is primarily used for dictionary-like properties of a service |
|`update\<Noun>`|key, partial item|Updated item|Fails if item doesn't exist. |
|`replace\<Noun>`|key, item|Replace existing item|Completely replaces an existing item. Fails if the item doesn't exist. |
|`append\<Noun>`|item|Appended item|Add item to a collection. Item will be added last. |
|`add\<Noun>`|index, item|Added item|Add item to a collection. Item will be added at the given index. |
|`get\<Noun>`|key|Item|Will return null if item doesn't exist |
|`list\<Noun>s`||`PagedAsyncIterableIterator<TItem, TPage>`|Return list of items. Returns empty list if no items exist |
|`\<noun>Exists`|key|`bool`|Return true if the item exists. |
|`delete\<Noun>`|key|None|Delete an existing item. Will succeed even if item didn't exist.|
|`remove\<Noun>`|key|None or removed item|Remove item from a collection.|

{% include requirement/MUSTNOT id="ts-naming-drop-noun" %} include the `Noun` when the operation is operating on the resource itself,  For example, if you have an `ItemClient` with a delete method, it should be called `delete` rather than `deleteItem`. The noun is implicitly `this`.

{% include requirement/MUST id="ts-naming-subclients" %} prefix methods that create or vend subclients with `get` and suffix with `client`.  For example, `container.getBlobClient()`.

{% include requirement/MUST id="ts-naming-options" %} suffix options bag parameters names with `Options`, and prefix with the name of the operation. For example, if an operation is called createItem, its options type must be called `CreateItemOptions`.

<a name="ts-example-naming"></a>

The following are good examples of names for operations in a TypeScript client library:

```javascript
containerClient.listBlobs();
containerClient.delete();
```

The following are bad examples:

```javascript
containerClient.deleteContainer(); // don't include noun for direct manipulation
containerClient.newBlob(); // use create instead of new
containerClient.createOrUpdate(); // use upsert
containerClient.createBlobClient(); // should be `getBlobClient`.
```

##### Service Method Input and Output Types

###### Model Types

TODO

###### Enumerations

TODO

###### Iterators

TODO: Move to a Common Type Usage section?

{% include requirement/MUST id="ts-use-iterators" %} use [Iterators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators) and [Async Iterators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of) for sequences and streams of all sorts.

Both iterators and async iterators are built into JavaScript and easy to consume. Other streaming interfaces (such as node streams) may be used where appropriate as long as they're idiomatic.

###### Interfaces and Class Types

{% include requirement/SHOULD id="ts-use-interface-parameters" %} prefer interface types to class types. JavaScript is fundamentally a duck-typed language, and so alternative classes that implement the same interface should be allowed. Declare parameters as interface types over class types whenever possible. Overloads with specific class types are fine but there should be an overload present with the generic interface.

<a name="ts-example-iterators"></a>

```javascript
// bad synchronous example
function listItems() {
  return {
    nextItem() { /*...*/ }
  }
}

// better synchronous example
function* listItems() {
  /* ... */
}

// bad asynchronous examples
function listItems() {
  return Rx.Observable.of(/* ... */)
}

function listItems(callback) {
  // fetch items
  for (const item of items) {
    callback (item)
    }
}

// better asynchronous example
async function* listItems() {
  for (const item of items) {
    yield item;
  }
}
```
~

###### Service Method Parameters

{% include requirement/SHOULD id="ts-use-overloads-over-unions" %} prefer overloads over unions when either:

1. Users want to see documentation (for example, signature help) tailored specifically to the parameters they're passing in, or
2. Multiple parameters are correlated.

Unions may be used if neither of these conditions are met. Let's say we have an API that takes either two numbers or two strings but not both. In this case, the parameters are correlated. If we implemented the types using unions like the following code:

```javascript
function foo(a: string | number, b: string | number): void {}
```

We have mistakenly allowed invalid arguments; that is `foo(number, string)` and `foo(string, number)`. Overloads naturally express this correlation:

```javascript
function foo(a: string, b: string): void;
function foo(a: number, b: number): void;
function foo(a: string | number, b: string | number): void {}
```

The overload approach also lets us attach documentation to each overload individually.

```javascript
// bad example
class ExampleClient {
  constructor (connectionString: string, options: ExampleClientOptions);
  constructor (url: string, options: ExampleClientOptions);
  constructor (urlOrCS: string, options: ExampleClientOptions) {
    // have to dig into the first parameter to see whether its
    // a url or a connection string. Not ideal.
  }
}

// better example
class ExampleClient {
  constructor (url: string, options: ExampleClientOptions) {

  }

  static fromConnectionString(connectionString: string, options: ExampleClientOptions) {

  }
}
```

######### Parameter validation {#general-parameter-validation}

The service client will have several methods that perform requests on the service.  _Service parameters_ are directly passed across the wire to an Azure service.  _Client parameters_ are not passed directly to the service, but used within the client library to fulfill the request.  Examples of client parameters include values that are used to construct a URI, or a file that needs to be uploaded to storage.

{% include requirement/MUST id="general-parameter-validation-client" %} validate client parameters.

{% include requirement/MUSTNOT id="general-parameter-validation-service" %} validate service parameters.  This includes null checks, empty strings, and other common validating conditions. Let the service validate any request parameters.

{% include requirement/MUST id="general-parameter-validation-errors" %} validate the developer experience when the service parameters are invalid to ensure appropriate error messages are generated by the service.  If the developer experience is compromised due to service-side error messages, work with the service team to correct prior to release.

{% include requirement/MUST %} accept an `AbortSignalLike` parameter on all asynchronous calls. This type is provided by `@azure/abort-controller`.

###### Service Method Return Types

TODO: Why not use the same client example as the other guidelines, so it's clear how one API looks across multiple services?

Using names that are commonly used as reserved words can cause confusion and will cause consistency issues between languages.

<a name="ts-example-return-types"></a>

An example:

```javascript
// Service operation method on a service client
  public async getProperties(
    options: ContainerGetPropertiesOptions = {}
  ): Promise<Models.ContainerGetPropertiesResponse> {
    // ...
  }

// Response type, in this case for a service which returns the
// relevant info in headers. Note how the headers are represented
// in first-class properties with intellisense etc.
export type ContainerGetPropertiesResponse = ContainerGetPropertiesHeaders & {
  /**
   * The underlying HTTP response.
   */
  _response: msRest.HttpResponse & {
      /**
       * The parsed HTTP response headers.
       */
      parsedHeaders: ContainerGetPropertiesHeaders;
    };
};

export interface ContainerGetPropertiesHeaders {
  // ...
  /**
   * @member {PublicAccessType} [blobPublicAccess] Indicated whether data in
   * the container may be accessed publicly and the level of access. Possible
   * values include: 'container', 'blob'
   */
  blobPublicAccess?: PublicAccessType;
  /**
   * @member {boolean} [hasImmutabilityPolicy] Indicates whether the container
   * has an immutability policy set on it.
   */
  hasImmutabilityPolicy?: boolean;
}
```

##### Methods Returning Lists (Pagination) {#ts-pagination}

Most developers will want to process a list one item at a time. Higher-level APIs (for example, async iterators) are preferred in the majority of use cases.  Finer-grained control over handling paginated result sets is sometimes required (for example, to handle over-quota or throttling).

{% include requirement/MUST id="ts-pagination-provide-list" %} provide a `list` method that returns a `PagedAsyncIterableIterator` from the module `@azure/core-paging`.

{% include requirement/MUST id="ts-pagination-provide-bypage-settings" %} provide page-related settings to the `byPage()` iterator and not the per-item iterator.

{% include requirement/MUST id="ts-pagination-take-continuationToken" %} take a `continuationToken` option in the `byPage()` method. You must rename other parameters that perform a similar function (for example, `nextMarker`).  If your page type has a continuation token, it must be named `continuationToken`.

{% include requirement/MUST id="ts-pagination-take-maxpagesize" %} take a `maxPageSize` option in the `byPage()` method.

An example of a paginating client:
<a name="ts-example-pagination"></a>

```javascript
// usage
const client = new ServiceClient()
for await (const item of client.listItems()) {
    console.log(item);
}

for await (const page of client.listItems().byPage({ maxPageSize: 50 })) {
    console.log(page);
}

// implementation
interface Item {
    name: string;
}

interface Page {
    continuationToken: string;
    items: Item[];
}

class ServiceClient {
    /* ... */
    listItems(): PagedAsyncIterableIterator<Item, Page> {
        async function* pages () { /* ... */ }
        async function* items () {
            for (const page of pages()) {
                for (const item of page.items) {
                    yield item;
                }
            }
        }

        const itemIter = items();

        return {
            next() {
                return itemIter.next();
                /* ... */
            },
            byPage() {
                return pages();
            },
            [Symbol.asyncIterator]() { return this }
        }
    }
}
```

{% include requirement/MUST id="general-pagination-distinct-types" %} use different types for entities returned from a `list` endpoint and a `get` endpoint if the returned entities have a different shape.  If both entities are the same form, use the same type.

{% include note.html content="Services should return the same shape for entities from a <code>list</code> endpoint vs. a <code>get</code> endpoint unless there's a good reason for the difference.  Using the same type for both operations will make the API surface in the client library simpler." %}

{% include requirement/MUSTNOT id="general-pagination-no-item-iterators" %} expose an iterator over individual items if it causes additional service requests.  Some services charge on a per-request basis. One `GET` per item is often too expensive when the data isn't used.

{% include requirement/MUSTNOT id="general-pagination-support-toArray" %} expose an API to get a paginated collection into an array. Services may return many pages, which can lead to memory exhaustion in the application.

##### Methods Invoking Long Running Operations {#ts-lro}

TODO: Summarize LROs

TODO: Describe what LRO API looks like in JS/TS, with code example

{% include requirement/MUST id="ts-lro-prefix-methods" %} prefix method names which return a poller with  `begin`.

TODO: Describe JS-specific way of creating a poller from a serialized poller, with code example

TODO: Say how we expose progress reporting for LROs in JS/TS

{% include draft.html content="Long-running operations will use the <code>@azure/core-lro</code> package, which is an abstration that provides the above requirements" %}

### Exceptions and Service Errors

{% include requirement/MUST id="ts-error-handling" %} use ECMAScript built-in error types for validation failures when appropriate. Specifically,

* Use `TypeError` for errors relating to passing in an incorrect type, such as an Object when a string is expected.
* Use `RangeError` for errors relating to values that are outside an allowable range, such as passing 0 for a number that must be greater than 0.
* Use `Error` for all other validation failures.

{% include requirement/SHOULD id="ts-error-handling-coercion" %} coerce incorrect types into an appropriate type, if possible. JavaScript users expect some amount of fuzziness with parameters as the standard library will coerce types if possible. TypeScript users should get pedantic types as they've opted in to types and expect errors.

{% include requirement/MUST id="ts-errors-documentation" %} document the errors that are produced by each method (with the exception of commonly thrown errors that are generally not documented in the target language).

{% include requirement/SHOULD id="ts-error-use-name" %} check the name property inside catch clauses rather than using `instanceof`.

TODO: How do JS exceptions/errors map to HTTP error codes?  Include a code sample.

### Modern & Idiomatic TypeScript {#ts-modern-typescript}

TODO: Move to an idiomatic or appendix section?

{% include requirement/MUST id="ts-use-typescript" %} implement your library in TypeScript.

{% include requirement/MUST id="ts-ship-type-declarations" %} include type declarations for your library.

TypeScript static types provide significant benefit for both the library authors and consumers.  TypeScript also compiles modern JavaScript language features for use with older runtimes.

#### tsconfig.json {#ts-tsconfig.json}

Your `tsconfig.json` should look similar to the following example:
<a name="ts-figure-tsconfig-json"></a>

```javascript
{
  "compilerOptions": {
    "declaration": true,
    "module": "es6",
    "moduleResolution": "node",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "outDir": "./dist-esm",
    "target": "es6",
    "sourceMap": true,
    "declarationMap": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "importHelpers": true
  },
  "include": ["./src/**/*"],
  "exclude": ["node_modules"]
}
```

{% include requirement/MUST id="ts-config-exclude" %} have at least "node_modules" in the `exclude` array. TypeScript shouldn't needlessly type check your dependencies.

{% include requirement/MUSTNOT id="ts-config-lib" %} use the `compilerOptions.lib` field. Built in typescript libraries (for example, `esnext.asynciterable`) should be included via reference directives. See also [Microsoft/TypeScript#27416](https://github.com/Microsoft/TypeScript/issues/27416).

{% include requirement/MUST id="ts-config-strict" %} set `compilerOptions.strict` to true. The `strict` flag is a best practice for developers as it provides the best TypeScript experience. The `strict` flag also ensures that your type definitions are maximally pedantic.

{% include requirement/MUST id="ts-config-esModuleInterop" %} set `compilerOptions.esModuleInterop` to true.

{% include requirement/MUST id="ts-config-allowSyntheticDefaultImports" %} set `compilerOptions.allowSyntheticDefaultImports` to true.

{% include requirement/MUST id="ts-config-target" %} set `compilerOptions.target`, but it can be any valid value so long as the final source distributions are compatible with the runtimes your library targets. See also [#ts-source-distros].

{% include requirement/MUST id="ts-config-forceConsistentCasingInFileNames" %} set `compilerOptions.forceConsistentCasingInFileNames` to true. `forceConsistentCasingInFileNames` forces TypeScript to treat files as case sensitive, and ensures you don't get surprised by build failures when moving between platforms.

{% include requirement/MUST id="ts-config-module" %} set `compilerOptions.module` to `es6`. Use a bundler such as [Rollup](https://rollupjs.org/guide/en/) or [Webpack](https://webpack.js.org/) to produce the CommonJS and UMD builds.

{% include requirement/MUST id="ts-config-moduleResolution" %} set `compilerOptions.moduleResolution` to "node" if your library targets Node. Otherwise, it should be absent.

{% include requirement/MUST id="ts-config-declaration" %} set `compilerOptions.declaration` to true. The `--declaration` option tells TypeScript to emit a `d.ts` file that contains the public surface area of your library. TypeScript and editors use this file to provide intellisense and type checking capabilities. Ensure you reference this type declaration file from the `types` field of your package.json.

{% include requirement/MUSTNOT id="ts-config-no-experimentalDecorators" %} set `compilerOptions.experimentalDecorators` to `true`. The experimentalDecorators flag adds support for "v1 decorators" to TypeScript. Unfortunately the standards process has moved on to an incompatible second version that is not yet implemented by TypeScript. Taking a dependency on decorators now means signing up your users for breaking changes later.

{% include requirement/MUST id="ts-config-sourceMap" %} set `compilerOptions.sourceMap` and `compilerOptions.declarationMap` to true. Shipping source maps in your package ensures clients can easily debug into your library code. `sourceMap` maps your emitted JS source to the declaration file and `declarationMap` maps the declaration file back to the TypeScript source that generated it. Be sure to include your original TypeScript sources in the package.

{% include requirement/MUST id="ts-config-importHelpers" %} set `compilerOptions.importHelpers` to true. Using external helpers keeps your package size down. Without this flag, TypeScript will add a helper block to each file that needs it. The file size savings using this option can be huge when using `async` functions (as an example) in a number of different files.

#### TypeScript Coding Guidelines {#ts-coding-guidelines}

{% include requirement/SHOULDNOT id="ts-no-namespaces" %} use TypeScript namespaces. Namespaces either use the `namespace` keyword explicitly, or the `module` keyword with a module name (for example, `module Microsoft.ApplicationInsights { ... }`). Use top-level imports/exports with ECMAScript modules instead. Namespaces make your code less compatible with standard ECMAScript and create significant friction with the TypeScript community.

{% include requirement/SHOULDNOT id="ts-no-const-enums" %} use `const enum`. `Const enum` requires global understanding of your program to compile properly. As a result, `const enum` can't be used with Babel 7, which otherwise supports TypeScript. Avoiding `const enum` will make sure your code can be compiled by any tool. Use regular enums instead.

## Azure SDK Library Design

### Namespaces, NPM Scopes, and Distribution Tags {#ts-namespace}

### Packaging {#ts-npm-package}

{% include requirement/MUST id="ts-npm-package-ownership" %} have npm package ownership set to either the Azure or Microsoft organizations.

#### Package Layout {#ts-package-file-layout}

Use the following canonical file structure for your npm package:
<a name="ts-figure-package-layout"></a>

```
azure-library
├─ README.md
├─ LICENSE
├─ dist
│  ├─ index.js
│  ├─ index.js.map
│  └─ ... *.js
│
├─ dist-esm
│  └─ lib
│    ├─ index.js
│    ├─ index.js.map
│    └─ ... *.js
│
├─ types
│  └─ service.d.ts
│
└─ package.json
```

{% include requirement/MUST id="ts-file-layout-conventions" %} follow these conventions where applicable.

{% include requirement/MUSTNOT id="ts-no-tsconfig" %} include a `tsconfig.json` file in your package. While generally useful to include, our `tsconfig.json` files are heavily tied to our monorepo structure and so won't work properly when read from inside an individual package.

{% include requirement/MAY id="ts-can-have-other-files" %} include other files.

{% include requirement/MUSTNOT id="ts-no-npmignore" %} use `.npmignore` files to control which files are included in the package. All files must be added to the package explicitly using the `package.json` files key.

#### The `package.json` file {#ts-package-json}

The following sections describe the package.json file that must be included with every npm package. A compliant `package.json` file looks like the following:
<a name="ts-figure-package-json"></a>

```javascript
{
  "name": "@azure/package",
  "description": "A pithy but accurate description",
  "keywords": [
    "azure",
    "cloud",
    "..."
  ],
  "version": "1.0.0",
  "author": "Microsoft Corporation",
  "main": "./dist/index.js",
  "module": "./dist-esm/index.js",
  "browser": {
    "./dist-esm/src/index.js": "./browser/index.js"
  },
  "types": "./dist-esm/index.d.ts",
  "engine": {
    "node": ">=6.0.0"
  },
  "scripts": {
    "build": "...",
    "test": "...",
    "prepack": "npm install && npm run build"
  },
  "files": [
    "dist",
    "dist-esm"
  ],
  "devDependencies": { /* ... */ },,
  "dependencies": { /* ... */ },
  "repository": "github:Azure/azure-sdk",
  "homepage": "https://github.com/Azure/azure-sdk-for-js/tree/master/sdk/servicebus/service-bus",
  "bugs": {
    "url": "https://github.com/Azure/azure-sdk-for-js/issues"
  },
  "license": "MIT",
  "sideEffects": false
}
```

{% include requirement/MUST id="ts-package-json-name" %} set `name` to `@azure/<name>`, where `<name>` is the name of the service. Package names are kebab-case: all lowercase with words joined by dashes.

{% include requirement/MUST id="ts-package-json-homepage" %} set `homepage` to a URL pointing to your library's readme inside the git repo. Since the repository link goes to the monorepo, this link exists to serve as an easier way to reach the actual package's source.

{% include requirement/MUST id="ts-package-json-bugs" %} set `bugs` to an object with a `url` key pointing to your library's issue tracker: https://github.com/Azure/azure-sdk-for-js/issues.

{% include requirement/MUST id="ts-package-json-repo" %} set `repository` to the JS SDK monorepo - `github:Azure/azure-sdk-for-js`. Use of the `github:user/repo` short-hand is recommended.

{% include requirement/MUST id="ts-package-json-description" %} set `description` to a useful but terse description of your library. The description is used and shown when searching for packages on [npmjs.org](https://npmjs.org).

{% include requirement/MUST id="ts-package-json-keywords" %} set `keywords` to an array that includes at least the entries "Azure" and "cloud". It must also contain at least the name of your service. It should contain other entries relevant to your SDK.

{% include requirement/MUST id="ts-package-json-author" %} set `author` to `"Microsoft Corporation"`.

{% include requirement/MUST id="ts-package-json-sideeffects" %} set `sideEffects` to `false`. Side effecting libraries must be explicitly approved during design review. The `sideEffects` field is used by [Webpack](https://webpack.js.org) and potentially other tools as an indicator of how aggressively the package can be optimized.

Side effects are modifications to the runtime environment of the program. For example, including a polyfill library is a `sideEffect`. It mutates the global environment. Side effects make it harder for tools to optimize your build and should be avoided.

{% include requirement/MUST id="ts-package-json-main-is-cjs" %} set `main` to point to either a CommonJS or a UMD module. Main is the entry point of your application for Node users.

{% include requirement/MUSTNOT id="ts-package-json-main-is-not-es6" %} set `main` to include any ES6+ syntax.

{% include requirement/MUST id="ts-package-json-module" %} set `module` to the ES6 module entrypoint of your application.

Tools such as [Webpack](https://webpack.js.org) use this key to discover the static module graph of your application for optimization purposes.

{% include requirement/MUST id="ts-package-json-browser" %} include a file map in the `browser` object if your library supports the browser.  The file map must include the `main` entry, and map it to the corresponding (unminified) browser code.

For example, the following JSON snippet demonstrates the minimum requirements:

```json
{
    "main": "./dist/index.js",
    "browser": {
        "./dist/index.js": "./dist/browser/index.js"
    }
}
```

{% include requirement/MUST id="ts-package-json-engine-is-present" %} set `engine` to the versions of Node your library supports. See [#ts-supported-node-versions] for Node support requirements.

{% include requirement/MUST id="ts-package-json-required-scripts" %} set `scripts` to an object with the following scripts:

- `"build"`: generates the main export of the application.
- `"test"`: runs your package's functional test suite for inner-loop development. Additional test tasks (for example, continuous integration tests) are allowed but `test` must be how developers test your package during development.

{% include requirement/MUSTNOT id="ts-package-json-required-scripts-for-development" %} depend on shell scripts to build or test the package.  Shell scripts need to be platform-specific.  Include a `script` for any task required during development of your package.

{% include requirement/MUST id="ts-package-json-files-required" %} set `files` to an array containing paths of your package contents. Setting this field prevents extraneous files from ending up in your package by being explicit about which files you ship to npm.

{% include requirement/MUST id="ts-package-json-types" %} set `types` to point to the TypeScript type declarations for your library's public surface area, usually `"./types/index.d.ts"`.

{% include requirement/MUST id="ts-package-json-license" %} set `license` to "MIT".

#### Distributions {#ts-source-distros}

Modern npm packages often ship multiple source distributions targeting different usage scenarios. Packages must include a CJS or UMD build, an ESM build, and original soure files. Packages may include other source distributions as necessary for their particular usage scenarios. The main downside of including additional source distributions is the increased package size (larger packages mean CIs take longer). However, performance, compatibility, and developer experience goals are often more important.

{% include requirement/MUST id="ts-include-original-source" %} include the source code in your source map files' `sourcesContent` by using the TypeScript compiler option `inlineSources`.

The source code in your package helps developers debug your package. _Go-to-definition_ is a quick way to confirm how to use a function. Seeing useful names and readable source code in call stacks helps with debugging. We can aggressively optimize the build artifacts since users won't need to puzzle through the mangled code.

{% include requirement/MUST id="ts-include-cjs" %} include a CommonJS (CJS) build in your package if you intend to support Node.

{% include requirement/SHOULD id="ts-use-umd" %} distribute your package as a UMD module if you intend to support browsers.

A UMD module is recommended even if your library isn't intended for the browser.  The overhead of UMD over CJS is slight and it will make an eventual move to the Web platform easier later.

When building the CommonJS module, the library name must be under the `Azure` namespace.  Refer to the [namespace guidelines] to determine the correct namespace.

{% include requirement/MUST id="ts-flatten-umd" %} flatten the CommonJS or UMD module.  [Rollup](https://rollupjs.org) is recommended for producing a flattened module.

The process of packing multiple modules into a single file is known as _flattening_. It's used to significantly reduce the load time for the library.  Flattening can make a measurable impact on cold start times for services such as Azure Functions. While performance-sensitive developers will likely package their applications themselves, faster start-up is still important especially during development.

{% include requirement/MUST id="ts-include-esm" %} include an ECMAScript Module (ESM) build in your package.

{% include requirement/MUSTNOT id="ts-include-esm-not-flattened" %} flatten the ESM build.

An ESM distribution is consumed by tools such as [Webpack](https://webpack.js.org) that optimize the module graph. It should be "transpiled" to support the runtime versions you're targeting. Versions of Webpack before Webpack 4.0 produce better optimized bundles if the ESM build is flattened. However, flattening doesn't play so well with tree-shaking. The latest versions of Webpack do a better job when using an unflattened ESM build.


{% include requirement/SHOULD id="ts-browser-umd" %} provide a browser build in UMD format for your library.

{% include requirement/MUST id="ts-browser-umd-global-naming" %} name the UMD global according to the [namespace guidelines].

{% include requirement/MUST id="ts-browser-minify-umd" %} provide both minified and non-minified versions, both with source mapping.

{% include requirement/MUST id="ts-browser-location" %} lace browser builds in a top level `browser` folder. The name of the file should be the service name. Append `.min` to the name of minified files. For example, Storage Blob should have `storage-blob.min.js` and `storage-blob.js` under `./browser`.

#### Modules {#ts-modules}

{% include requirement/MUST id="ts-modules-only-named" %} have named exports at the top level

{% include requirement/MUSTNOT id="ts-modules-no-default" %} have a default export at the top level

Azure packages authored using TypeScript export standard ES6 modules. As Node doesn't support ES6 modules natively, authoring ES6 modules for consumption in Node has a bit of friction. Most notably, a commonJS package can only import a single value.

### Platform Support {#ts-platform-support}

{% include requirement/MUST id="ts-node-support" %} support [all LTS versions of Node](https://github.com/nodejs/Release#release-schedule) and newer versions up to and including the latest release. At time of writing, this means Node 8.x through Node 12.x.

{% include requirement/MUST id="ts-browser-support" %} support the following browsers and versions:

* Apple Safari: latest two versions
* Google Chrome: latest two versions
* Microsoft Edge: all supported versions
* Mozilla FireFox: latest two versions

Use [caniuse.com](https://caniuse.com) to determine whether you can use a given platform feature in the runtime versions you support. Syntax support is provided by TypeScript.

{% include requirement/SHOULDNOT id="ts-no-ie11-support" %} support IE11. If you have a business justification for IE11 support, contact the [Architecture Board].

{% include requirement/MUST id="ts-support-ts" %} compile without errors on all versions of TypeScript greater than 3.1.

While consumers are fast at adopting new versions of TypeScript, version 3.1 is used by Angular 7, which is still commonly used.  Supporting older versions of TypeScript can be a challenge. There are two general approaches:

1. Don't use new features.
2. Use [`typesVersions`](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-1.html#version-selection-with-typesversions), which might require manual effort to produce typings compatible with older versions based on the new typings.

{% include requirement/MUST id="ts-register-dropped-platforms" %} get approval from the [Architecture Board] to drop support for any platform (except IE11 and Node 6) even if support isn't required.


{% include requirement/MUST id="ts-azure-scope" %} publish your library to the `@azure` npm scope.

{% include requirement/MUST id="ts-namespace-serviceclient" %} pick a package name that allows the consumer to tie the namespace to the service being used.  As a default, use the compressed service name at the end of the namespace.  The namespace does **NOT** change when the branding of the product changes. Avoid the use of marketing names that may change.

{% include requirement/MUST id="ts-npm-dist-tag-beta" %} tag beta packages with the npm distribution tag `next`. If there is no generally available release of this package, it should also be tagged `latest`.

{% include requirement/MUST id="ts-npm-dist-tag-next" %} tag generally available npm packages `latest`. Generally available packages may also be tagged `next` if they include the changes from the most recent beta.

{% include requirement/MUST id="ts-namespace-in-browsers" %} use one of the appropriate namespaces (see below) for browser globals when producing UMD builds.

In cases where namespaces are supported, the namespace should be named `Azure.<group>.<service>`. All consumer-facing APIs that are commonly used should exist within this namespace.  Here:

- `<group>` is the group for the service (see the list above)
- `<service>` is the service name represented as a single word

{% include requirement/MUST id="ts-namespace-startswith" %} start the namespace with `Azure`.

{% include requirement/MUST id="ts-namespace-camelcase" %} use camel-casing for elements of the namespace.

A compressed service name is the service name without spaces.  It may further be shortened if the shortened version is well known in the community.  For example, "Azure Media Analytics" would have a compressed service name of "MediaAnalytics", and "Azure Service Bus" would become "ServiceBus".

{% include requirement/MUST id="ts-namespace-names" %} use the following list as the group of services (if the target language supports namespaces):

{% include tables/data_namespaces.md %}

{% include requirement/MUST id="ts-namespace-split-management-api" %} place the management (Azure Resource Manager) API in the "management" group.  Use the grouping `Azure.Management.<group>.<servicename>` for the namespace.  Since more services require control plane APIs than data plane APIs, other namespaces may be used explicitly for control plane only.  Data plane usage is by exception only.  Additional namespaces that can be used for control plane libraries include:

{% include tables/mgmt_namespaces.md %}

Many `management` APIs don't have a data plane.  It's reasonable to place the management library in the `Azure.Management` namespace.  For example, use `Azure.Management.CostAnalysis`, not `Azure.Management.Management.CostAnalysis`.

{% include requirement/MUSTNOT id="ts-namespace-avoid-ambiguity" %} choose similar names for clients that do different things.

{% include requirement/MUST id="ts-namespace-register" %} register the chosen namespace with the [Architecture Board].  Open an issue to request the namespace.  See [the registered namespace list](registered_namespaces.html) for a list of the currently registered namespaces.

These namespace examples meet the guidelines:

- `Azure.Data.Cosmos`
- `Azure.Identity.ActiveDirectory`
- `Azure.IoT.DeviceProvisioning`
- `Azure.Storage.Blob`
- `Azure.Messaging.NotificationHubs` (the client library for Notification Hubs)
- `Azure.Management.Messaging.NotificationHubs` (the management client for Notification Hubs)

These examples that don't meet the guidelines:

- `Microsoft.Azure.CosmosDB` (not in the Azure namespace, no grouping)
- `Azure.MixedReality.Kinect` (invalid group)
- `Azure.IoT.IoTHub.DeviceProvisioning` (too many levels)

Contact the [Architecture Board] for advice if the appropriate group isn't obvious.  If you feel your service requires a new group, open a "Design Guidelines Change" request.

{% include refs.md %}
{% include_relative refs.md %}