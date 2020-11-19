---
title: "Python Guidelines: API Design"
keywords: guidelines python
permalink: python_design.html
folder: python
sidebar: general_sidebar
---

## Azure SDK API Design

The API surface of your client library must have the most thought as it is the primary interaction that the consumer has with your service.

{% include requirement/MUST id="python-feature-support" %} support 100% of the features provided by the Azure service the client library represents. Gaps in functionality cause confusion and frustration among developers.

## The Service Client

Your API surface will consist of one or more _service clients_ that the consumer will instantiate to connect to your service, plus a set of supporting types.

{% include requirement/MUST id="python-client-naming" %} name service client types with a **Client** suffix.

```python
# Yes
class CosmosClient(object) ... 

# No
class CosmosProxy(object) ... 

# No
class CosmosUrl(object) ... 

```

{% include requirement/MUST id="python-client-namespace" %} expose the service clients the user is more likely to interact with from the root namespace of your package.

TODO: Code sample - what does it look like to create a client instance and use it to call the service?  Where can I see a Track 2 implementation to copy off of?

### Constructors and factory methods

{% include requirement/MUST id="python-client-constructor-form" %} provide a constructor that takes binding parameters (for example, the name of, or a URL pointing to the service instance), a `credentials` parameter, a `transport` parameter, and `**kwargs` for passing settings through to individual HTTP pipeline policies.

Only the minimal information needed to connect and interact with the service should be required. All additional information should be optional.

The constructor **must not** take a connection string.

{% include requirement/MUST id="python-client-connection-string" %} use a separate factory method `ExampleServiceClient.from_connection_string` to create a client from a connection string (if the client supports connection strings).

The method **should** parse the connection string and pass the values to the constructor.  Provide a `from_connection_string` factory method only if the Azure portal exposes a connection string for your service.

#### Authentication

{% include requirement/MUST id="python-auth-credential-azure-core" %} use the credentials classes in `azure-core` whenever possible.

{% include requirement/MAY  id="python-auth-service-credentials" %} add additional credential types if required by the service. Contact @adparch for guidance if you believe you have need to do so.

{% include requirement/MUST id="python-auth-service-support" %} support all authentication methods that the service supports.

#### Client Configuration

TODO: If this is kwargs, would it be worth saying which kwargs are required?  How is the service version specified?  How are retries?  In other languages, this is baked into inherited types ... what should the API designer know here to make sure they get it right?

##### REST API method versioning

{% include requirement/MUST id="python-versioning-latest-service-api" %} use the latest service protocol version when making requests.

{% include requirement/MUST id="python-versioning-select-service-api" %} allow a client application to specify an earlier version of the service protocol.

#### Async support

The `asyncio` library has been available since Python 3.4, and the `async`/`await` keywords were introduced in Python 3.5. Despite such availability, most Python developers aren't familiar with or comfortable using libraries that only provide asynchronous methods.

{% include requirement/MUST id="python-client-sync-async" %} provide both sync and async versions of your APIs

{% include requirement/MUST id="python-client-async-keywords" %} use the `async`/`await` keywords (requires Python 3.5+). Don't use the [yield from coroutine or asyncio.coroutine](https://docs.python.org/3.4/library/asyncio-task.html) syntax.

{% include requirement/MUST id="python-client-separate-sync-async" %} provide two separate client classes for synchronous and asynchronous operations.  Don't combine async and sync operations in the same class.

```python
# Yes
# In module azure.example
class ExampleClient(object):
    def some_service_operation(self, name, size) ...

# In module azure.example.aio
class ExampleClient:
    # Same method name as sync, different client
    async def some_service_operation(self, name, size) ... 

# No
# In module azure.example
class ExampleClient(object):
    def some_service_operation(self, name, size) ...

class AsyncExampleClient: # No async/async pre/postfix.
    async def some_service_operation(self, name, size) ...

# No
# In module azure.example
class ExampleClient(object): # Don't mix'n match with different method names
    def some_service_operation(self, name, size) ...
    async def some_service_operation_async(self, name, size) ...

```

{% include requirement/MUST id="python-client-same-name-sync-async" %} use the same client name for sync and async packages

Example:

|Sync/async|Namespace|Package name|Client name|
|-|-|-|-|
|Sync|`azure.sampleservice`|`azure-sampleservice`|`azure.sampleservice.SampleServiceClient`|
|Async|`azure.sampleservice.aio`|`azure-sampleservice-aio`|`azure.sampleservice.aio.SampleServiceClient`|

{% include requirement/MUST id="python-client-namespace-sync" %} use the same namespace for the synchronous client as the synchronous version of the package with `.aio` appended.

{% include requirement/SHOULD id="python-client-separate-async-pkg" %} ship a separate package for async support if the async version requires additional dependencies.

{% include requirement/MUST id="python-client-same-pkg-name-sync-async" %} use the same name for the asynchronous version of the package as the synchronous version of the package with `-aio` appended.

{% include requirement/MUST id="python-client-async-http-stack" %} use [`aiohttp`](https://aiohttp.readthedocs.io/en/stable/) as the default HTTP stack for async operations. Use `azure.core.pipeline.transport.AioHttpTransport` as the default `transport` type for the async client.

### Service Methods

#### Service Method Naming

{% include requirement/SHOULD id="python-client-service-verbs" %} prefer the usage one of the preferred verbs for method names.

|Verb|Parameters|Returns|Comments|
|-|-|-|-|
|`create_\<noun>`|key, item, `[allow_overwrite=True]`|Created item|Create new item. Fails if item already exists.|
|`upsert_\<noun>`|key, item|item|Create new item, or update existing item. Verb is primarily used in database-like services |
|`set_\<noun>`|key, item|item|Create new item, or update existing item. Verb is primarily used for dictionary-like properties of a service |
|`update_\<noun>`|key, partial item|item|Fails if item doesn't exist. |
|`replace_\<noun>`|key, item|item|Completely replaces an existing item. Fails if the item doesn't exist. |
|`append_\<noun>`|item|item|Add item to a collection. Item will be added last. |
|`add_\<noun>`|index, item|item|Add item to a collection. Item will be added at the given index. |
|`get_\<noun>`|key|item|Raises an exception if item doesn't exist |
|`list_\<noun>`||`azure.core.Pageable[Item]`|Return an iterable of items. Returns iterable with no items if no items exist (doesn't return `None` or throw)|
|`\<noun>\_exists`|key|`bool`|Return `True` if the item exists. Must raise an exception if the method failed to determine if the item exists (for example, the service returned an HTTP 503 response)|
|`delete_\<noun>`|key|`None`|Delete an existing item. Must succeed even if item didn't exist.|
|`remove_\<noun>`|key|removed item or `None`|Remove a reference to an item from a collection. This method doesn't delete the actual item, only the reference.|

{% include requirement/MUST id="python-client-standardize-verbs" %} standardize verb prefixes outside the list of preferred verbs for a given service across language SDKs. If a verb is called `download` in one language, we should avoid naming it `fetch` in another.

#### Service Method Input and Outputs

##### Model Types

{% include requirement/MUST id="python-models-repr" %} implement `__repr__` for model types. The representation **must** include the type name and any key properties (that is, properties that help identify the model instance).

{% include requirement/MUST id="python-models-repr-length" %} truncate the output of `__repr__` after 1024 characters.

##### Service Method Parameters

{% include requirement/MUST id="python-client-service-args" %} support the common arguments for service operations:

|Name|Description|Applies to|Notes|
|-|-|-|-|
|`timeout`|Timeout in seconds|All service methods|
|`headers`|Custom headers to include in the service request|All requests|Headers are added to all requests made (directly or indirectly) by the method.|
|`continuation_token`|Opaque token indicating the first page to retrieve. Retrieved from a previous `Paged` return value.|`list` operations.|
|`client_request_id`|Caller specified identification of the request.|Service operations for services that allow the client to send a client-generated correlation ID.|Examples of this include `x-ms-client-request-id` headers.|The client library **must** use this value if provided, or generate a unique value for each request when not specified.|
|`response_hook`|`callable` that is called with (response, headers) for each operation.|All service methods|

{% include requirement/MUST id="python-client-splat-args" %} accept a Mapping (`dict`-like) object in the same shape as a serialized model object for parameters.

```python
# Yes:
class Model(object):

    def __init__(self, name, size):
        self.name = name
        self.size = size

def do_something(model: "Model"):
    ...

do_something(Model(name='a', size=17)) # Works
do_something({'name': 'a', 'size', '17'}) # Does the same thing...
```

{% include requirement/MUST id="python-client-flatten-args" %} use "flattened" named arguments for `update_` methods. **May** additionally take the whole model instance as a named parameter. If the caller passes both a model instance and individual key=value parameters, the explicit key=value parameters override whatever was specified in the model instance.

```python
class Model(object):

    def __init__(self, name, size, description):
        self.name = name
        self.size = size
        self.description = description

class Client(object):

    def update_model(self, name=None, size=None, model=None): ...

model = Model(name='hello', size=4711, description='This is a description...')

client.update_model(model=model, size=4712) # Will send a request to the service to update the model's size to 4712
model.description = 'Updated'
model.size = -1
# Will send a request to the service to update the model's size to 4713 and description to 'Updated'
client.update_model(name='hello', size=4713, model=model)  
```

###### Parameter validation

The service client will have several methods that send requests to the service. **Service parameters** are directly passed across the wire to an Azure service. **Client parameters** aren't passed directly to the service, but used within the client library to fulfill the request. Parameters that are used to construct a URI, or a file to be uploaded are examples of client parameters.

{% include requirement/MUST id="python-params-client-validation" %} validate client parameters. Validation is especially important for parameters used to build up the URL since a malformed URL means that the client library will end up calling an incorrect endpoint.

```python
# No:
def get_thing(name: "str") -> "str":
    url = f'https://<host>/things/{name}'
    return requests.get(url).json()

try:
    thing = get_thing('') # Ooops - we will end up calling '/things/' which usually lists 'things'. We wanted a specific 'thing'.
except ValueError:
    print('We called with some invalid parameters. We should fix that.')

# Yes:
def get_thing(name: "str") -> "str":
    if not name:
        raise ValueError('name must be a non-empty string')
    url = f'https://<host>/things/{name}'
    return requests.get(url).json()

try:
    thing = get_thing('')
except ValueError:
    print('We called with some invalid parameters. We should fix that.')
```

{% include requirement/MUSTNOT id="python-params-service-validation" %} validate service parameters. Don't do null checks, empty string checks, or other common validating conditions on service parameters. Let the service validate all request parameters.

{% include requirement/MUST id="python-params-devex" %} validate the developer experience when the service parameters are invalid to ensure appropriate error messages are generated by the service. Work with the service team if the developer experience is compromised because of service-side error messages.


##### Service Method Return Types

Requests to the service fall into two basic groups - methods that make a single logical request, or a deterministic sequence of requests. An example of a single logical request is a request that may be retried inside the operation. An example of a deterministic sequence of requests is a paged operation.

The logical entity is a protocol neutral representation of a response. For HTTP, the logical entity may combine data from headers, body, and the status line. For example, you may wish to expose an `ETag` header as a property on the logical entity.

{% include requirement/MUST id="python-response-logical-entity" %} optimize for returning the logical entity for a given request. The logical entity MUST represent the information needed in the 99%+ case.

#### Methods Returning Collections (Pagination)

{% include requirement/MUST id="python-response-paged-protocol" %} return a value that implements the [Paged protocol](#python-core-protocol-paged) for operations that return collections. The [Paged protocol](#python-core-protocol-paged) allows the user to iterate through all items in a returned collection, and also provides a method that gives access to individual pages.

#### Methods Invoking Long Running Operations

{% include requirement/MUST id="python-lro-prefix" %} prefix methods with `begin_` for long running operations. Long running operations *must* return a [Poller](#python-core-protocol-lro-poller) object.

{% include requirement/MUST id="python-lro-poller" %} return a value that implements the [Poller protocol](#python-core-protocol-lro-poller) for long running operations.

### Client Design Considerations

#### Hierarchical Clients

Many services have resources with nested child (or sub) resources. For example, Azure Storage provides an account that contains zero or more containers, which in turn contains zero or more blobs.

{% include requirement/MUST id="python-client-hierarchy" %} create a client type corresponding to each level in the hierarchy except for leaf resource types. You **may** omit creating a client type for leaf node resources.

{% include requirement/MUST id="python-client-hier-creation" %} make it possible to directly create clients for each level in the hierarchy.  The constructor can be called directly or via the parent.

```python
class ChildClient:
    # Yes:
    __init__(self, parent, name, credentials, **kwargs) ...

class ChildClient:
    # Yes:
    __init__(self, url, credentials, **kwargs) ...
```

{% include requirement/MUST id="python-client-hier-vend" %} provide a `get_<child>_client(self, name, **kwargs)` method to retrieve a client for the named child. The method must not make a network call to verify the existence of the child.

{% include requirement/MUST id="python-client-hier-create" %} provide method `create_<child>(...)` that creates a child resource. The method **should** return a client for the newly created child resource.

{% include requirement/SHOULD id="python-client-hier-delete" %} provide method `delete_<child>(...)` that deletes a child resource.

### Exceptions

{% include requirement/MUST id="python-errors-exceptions" %} raise an exception if a method fails to perform its intended functionality. Don't return `None` or a `boolean` to indicate errors.

```python
# Yes
try:
    resource = client.create_resource(name)
except azure.core.errors.ResourceExistsException:
    print('Failed - we need to fix this!')

# No
resource = client.create_resource(name):
if not resource:
    print('Failed - we need to fix this!')
```

{% include requirement/MUSTNOT id="python-errors-normal-responses" %} throw an exception for "normal responses".

Consider an `exists` method. The method **must** distinguish between the service returned a client error 404/NotFound and a failure to even make a request:

```python
# Yes
try:
    exists = client.resource_exists(name):
    if not exists:
        print("The resource doesn't exist...")
except azure.core.errors.ServiceRequestError:
    print("We don't know if the resource exists - so it was appropriate to throw an exception!")

# No
try:
    client.resource_exists(name)
except azure.core.errors.ResourceNotFoundException:
    print("The resource doesn't exist... but that shouldn't be an exceptional case for an 'exists' method")
```

{% include requirement/SHOULDNOT id="python-errors-new-exceptions" %} create a new exception type unless the developer can remediate the error by doing something different.  Specialized exception types should be based on existing exception types present in the `azure-core` package.

{% include requirement/MUST id="python-errors-on-http-request-failed" %} produce an error when an HTTP request fails with an unsuccessful HTTP status code (as defined by the service).

{% include requirement/MUST id="python-errors-include-request-response" %} include the HTTP response (status code and headers) and originating request (URL, query parameters, and headers) in the exception.

For higher-level methods that use multiple HTTP requests, either the last exception or an aggregate exception of all failures should be produced.

{% include requirement/MUST id="python-errors-rich-info" %} include any service-specific error information in the exception.  Service-specific error information must be available in service-specific properties or fields.

{% include requirement/MUST id="python-errors-documentation" %} document the errors that are produced by each method. Don't document commonly thrown errors that wouldn't normally be documented in Python.

{% include requirement/MUSTNOT id="python-errors-use-standard-exceptions" %} create new exception types when a [built-in exception type](https://docs.python.org/3/library/exceptions.html) will suffice.

{% include requirement/MUST id="python-errors-use-chaining" %} allow exception chaining to include the original source of the error.

```python
# Yes:
try:
    # do something
except:
    raise MyOwnErrorWithNoContext()

# No:
success = True
try:
    # do something
except:
    success = False
if not success:
    raise MyOwnErrorWithNoContext()

# No:
success = True
try:
    # do something
except:
    raise MyOwnErrorWithNoContext() from None
```

## Azure SDK Library Design

### Namespaces

{% include requirement/MUST id="python-namespaces-prefix" %} implement your library as a subpackage in the `azure` namespace.

{% include requirement/MUST id="python-namespaces-naming" %} pick a package name that allows the consumer to tie the namespace to the service being used. As a default, use the compressed service name at the end of the namespace. The namespace does NOT change when the branding of the product changes. Avoid the use of marketing names that may change.

A compressed service name is the service name without spaces. It may further be shortened if the shortened version is well known in the community. For example, “Azure Media Analytics” would have a compressed service name of `mediaanalytics`, and “Azure Service Bus” would become `servicebus`.  Separate words using an underscore if necessary. If used, `mediaanalytics` would become `media_analytics`

{% include requirement/MAY id="python-namespaces-grouping" %} include a group name segment in your namespace (for example, `azure.<group>.<servicename>`) if your service or family of services have common behavior (for example, shared authentication types). 

If you want to use a group name segment, use one of the following groups:

{% include tables/data_namespaces.md %} 

{% include requirement/MUST id="python-namespaces-mgmt" %} place management (Azure Resource Manager) APIs in the `mgmt` group. Use the grouping `azure.mgmt.<servicename>` for the namespace. Since more services require control plane APIs than data plane APIs, other namespaces may be used explicitly for control plane only.

{% include requirement/MUST id="python-namespaces-register" %} register the chosen namespace with the [Architecture Board].  Open an issue to request the namespace.  See [the registered namespace list](registered_namespaces.html) for a list of the currently registered namespaces.

{% include requirement/MUST id="python-namespaces-async" %} use an `.aio` suffix added to the namespace of the sync client for async clients.

Example:

```python
# Yes:
from azure.exampleservice.aio import ExampleServiceClient

# No: Wrong namespace, wrong client name...
from azure.exampleservice import AsyncExampleServiceClient
```

### Packaging

{% include requirement/MUST id="python-packaging-name" %} name your package after the namespace of your main client class.

{% include requirement/MUST id="python-packaging-name-allowed-chars" %} use all lowercase in your package name with a dash (-) as a separator.

{% include requirement/MUSTNOT id="python-packaging-name-disallowed-chars" %} use underscore (_) or period (.) in your package name. If your namespace includes underscores, replace them with dash (-) in the distribution package name.

{% include requirement/MUST id="python-packaging-follow-repo-rules" %} follow the specific package guidance from the [azure-sdk-packaging wiki](https://github.com/Azure/azure-sdk-for-python/wiki/Azure-packaging)

{% include requirement/MUST id="python-packaging-follow-python-rules" %} follow the [namespace package recommendations for Python 3.x](https://docs.python.org/3/reference/import.html#namespace-packages) for packages that only need to target 3.x.

{% include requirement/MUST id="python-packaging-nspkg" %} depend on `azure-nspkg` for Python 2.x.

{% include requirement/MUST id="python-packaging-init" %} include `__init__.py` for the namespace(s) in sdists

#### Binary extensions

{% include requirement/MUST id="python-native-approval" %} be approved by the [Architecture Board].

{% include requirement/MUST id="python-native-plat-support" %} support Windows, Linux (manylinux - see [PEP513](https://www.python.org/dev/peps/pep-0513/), [PEP571](https://www.python.org/dev/peps/pep-0571/)), and MacOS.  Support the earliest possible manylinux to maximize your reach.

{% include requirement/MUST id="python-native-arch-support" %} support both x86 and x64 architectures.

{% include requirement/MUST id="python-native-charset-support" %} support unicode and ASCII versions of CPython 2.7.

#### Service-specific common library code

There are occasions when common code needs to be shared between several client libraries.  For example, a set of cooperating client libraries may wish to share a set of exceptions or models.

{% include requirement/MUST id="python-commonlib-approval" %} gain [Architecture Board] approval prior to implementing a common library.

{% include requirement/MUST id="python-commonlib-minimize-code" %} minimize the code within a common library.  Code within the common library is available to the consumer of the client library and shared by multiple client libraries within the same namespace.

{% include requirement/MUST id="python-commonlib-namespace" %} store the common library in the same namespace as the associated client libraries.

A common library will only be approved if:

* The consumer of the non-shared library will consume the objects within the common library directly, AND
* The information will be shared between multiple client libraries.

Let's take two examples:

1. Implementing two Cognitive Services client libraries, we find a model is required that is produced by one Cognitive Services client library and consumed by another Coginitive Services client library, or the same model is produced by two client libraries.  The consumer is required to do the passing of the model in their code, or may need to compare the model produced by one client library vs. that produced by another client library.  This is a good candidate for choosing a common library.

2. Two Cognitive Services client libraries throw an `ObjectNotFound` exception to indicate that an object was not detected in an image.  The user might trap the exception, but otherwise will not operate on the exception.  There is no linkage between the `ObjectNotFound` exception in each client library.  This is not a good candidate for creation of a common library (although you may wish to place this exception in a common library if one exists for the namespace already).  Instead, produce two different exceptions - one in each client library.

### Dependencies

{% include requirement/MUST id="python-dependencies-approved-list" %} only pick external dependencies from the following list of well known packages for shared functionality:

{% include_relative approved_dependencies.md %}

{% include requirement/MUSTNOT id="python-dependencies-external" %} use external dependencies outside the list of well known dependencies. To get a new dependency added, contact the [Architecture Board].

{% include requirement/MUSTNOT id="python-dependencies-vendor" %} vendor dependencies unless approved by the [Architecture Board].

When you vendor a dependency in Python, you include the source from another package as if it was part of your package.

{% include requirement/MUSTNOT id="python-dependencies-pin-version" %} pin a specific version of a dependency unless that is the only way to work around a bug in said dependencies versioning scheme.

Applications are expected to pin exact dependencies. Libraries aren't. A library should use a [compatible release](https://www.python.org/dev/peps/pep-0440/#compatible-release) identifier for the dependency.

### Versioning

{% include requirement/MUST id="python-versioning-semver" %} use [semantic versioning](https://semver.org) for your package.

{% include requirement/MUST id="python-versioning-beta" %} use the `bN` pre-release segment for [beta releases](https://www.python.org/dev/peps/pep-0440/#pre-releases).

Don't use pre-release segments other than the ones defined in [PEP440](https://www.python.org/dev/peps/pep-0440) (`aN`, `bN`, `rcN`). Build tools, publication tools, and index servers may not sort the versions correctly.

{% include requirement/MUST id="python-versioning-changes" %} change the version number if *anything* changes in the library.

{% include requirement/MUST id="python-versioning-patch" %} increment the patch version if only bug fixes are added to the package.

{% include requirement/MUST id="python-verioning-minor" %} increment the minor version if any new functionality is added to the package.

{% include requirement/MUST id="python-versioning-apiversion" %} increment the minor version if the default REST API version is changed, even if there's no public API change to the library.

{% include requirement/MUSTNOT id="python-versioning-api-major" %} increment the major version for a new REST API version unless it requires breaking API changes in the python library itself.

{% include requirement/MUST id="python-versioning-major" %} increment the major version if there are breaking changes in the package. Breaking changes require prior approval from the [Architecture Board].

#### Version Numbers

{% include requirement/MUST id="python-versioning-major" %} select a version number greater than the highest version number of any other released Track 1 package for the service in any other scope or language.

The bar to make a breaking change is extremely high for GA client libraries.  We may create a new package with a different name to avoid diamond dependency issues.

### Logging

{% include requirement/MUST id="python-logging-usage" %} use Pythons standard [logging module](https://docs.python.org/3/library/logging.html).

{% include requirement/MUST id="python-logging-nameed-logger" %} provide a named logger for your library.

The logger for your package **must** use the name of the module. The library may provide additional child loggers. If child loggers are provided, document them.

For example:

- Package name: `azure-someservice`
- Module name: `azure.someservice`
- Logger name: `azure.someservice`
- Child logger: `azure.someservice.achild`

These naming rules allow the consumer to enable logging for all Azure libraries, a specific client library, or a subset of a client library.

{% include requirement/MUST id="python-logging-error" %} use the `ERROR` logging level for failures where it's unlikely the application will recover (for example, out of memory).

{% include requirement/MUST id="python-logging-warn" %} use the `WARNING` logging level when a function fails to perform its intended task. The function should also raise an exception.

Don't include occurrences of self-healing events (for example, when a request will be automatically retried).

{% include requirement/MUST id="python-logging-info" %} use the `INFO` logging level when a function operates normally.

{% include requirement/MUST id="python-logging-debug" %} use the `DEBUG` logging level for detailed trouble shooting scenarios.

The `DEBUG` logging level is intended for developers or system administrators to diagnose specific failures.

{% include requirement/MUSTNOT id="python-logging-sensitive-info" %} send sensitive information in log levels other than `DEBUG`.  For example, redact or remove account keys when logging headers.

{% include requirement/MUST id="python-logging-request" %} log the request line, response line, and headers for an outgoing request as an `INFO` message.

{% include requirement/MUST id="python-logging-cancellation" %} log an `INFO` message, if a service call is canceled.

{% include requirement/MUST id="python-logging-exceptions" %} log exceptions thrown as a `WARNING` level message. If the log level set to `DEBUG`, append stack trace information to the message.

You can determine the logging level for a given logger by calling [`logging.Logger.isEnabledFor`](https://docs.python.org/3/library/logging.html#logging.Logger.isEnabledFor).

## Distributed tracing

> **DRAFT** section

{% include requirement/MUST id="python-tracing-span-per-method" %} create a new trace span for each library method invocation. The easiest way to do so is by adding the distributed tracing decorator from `azure.core.tracing`.

{% include requirement/MUST id="python-tracing-span-name" %} use `<package name>/<method name>` as the name of the span.

{% include requirement/MUST id="python-tracing-span-per-call" %} create a new span for each outgoing network call. If using the HTTP pipeline, the new span is created for you.

{% include requirement/MUST id="python-tracing-propagate" %} propagate tracing context on each outgoing service request.




{% include refs.md %}
{% include_relative refs.md %}