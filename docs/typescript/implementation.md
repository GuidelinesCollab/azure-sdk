---
title: "TypeScript Guidelines: Implementation"
keywords: guidelines typescript
permalink: typescript_implementation.html
folder: typescript
sidebar: general_sidebar
---

Once you've worked through an acceptable API design, you can start implementing the service clients.

{% include requirement/SHOULD id="ts-should-use-template" %} use the [TypeScript client library template].

## Configuration {#ts-configuration}

When configuring your client library, particular care must be taken to ensure that the consumer of your client library can properly configure the connectivity to your Azure service both globally (along with other client libraries the consumer is using) and specifically with your client library.

{% include requirement/MUST id="ts-configuration-use-core-lib" %} use the `@azure/core-configuration` package.  The `@azure/core-configuration` package implements the general guidelines for configuration; specifically:

* Retrieve any relevant settings from the environment.
* Retrieve any relevant global settings from the consumers code.

### Client configuration

{% include requirement/MUST id="ts-configuration-use-global-config" %} use relevant global configuration settings either by default or when explicitly requested to by the user, for example by passing in a configuration object to a client constructor.

{% include requirement/MUST id="ts-configuration-allow-different-configs" %} allow different clients of the same type to use different configurations.

{% include requirement/MUST id="ts-configuration-allow-optout" %} allow consumers of your service clients to opt out of all global configuration settings at once.

{% include requirement/MUST id="ts-configuration-allow-overrides" %} allow all global configuration settings to be overridden by client-provided options. The names of these options should align with any user-facing global configuration keys.

{% include requirement/MUSTNOT id="ts-configuration-no-config-changes-after-construction" %} change behavior based on configuration changes that occur after the client is constructed. Hierarchies of clients inherit parent client configuration unless explicitly changed or overridden. Exceptions to this requirement are as follows:

1. Log level, which must take effect immediately across the Azure SDK.
2. Tracing on/off, which must take effect immediately across the Azure SDK.

### Service-specific environment variables

{% include requirement/MUST id="ts-configuration-env-prefix" %} prefix Azure-specific environment variables with `AZURE_`.

{% include requirement/MAY id="ts-configuration-service-envs" %} use client library-specific environment variables for portal-configured settings which are provided as parameters to your client library. This generally includes credentials and connection details. For example, Service Bus could support the following environment variables:

* `AZURE_SERVICEBUS_CONNECTION_STRING`
* `AZURE_SERVICEBUS_NAMESPACE`
* `AZURE_SERVICEBUS_ISSUER`
* `AZURE_SERVICEBUS_ACCESS_KEY`

Storage could support:

* `AZURE_STORAGE_ACCOUNT`
* `AZURE_STORAGE_ACCESS_KEY`
* `AZURE_STORAGE_DNS_SUFFIX`
* `AZURE_STORAGE_CONNECTION_STRING`

{% include requirement/MUST id="ts-configuration-approval-for-envs" %} get approval from the [Architecture Board] for every new environment variable.

{% include requirement/MUST id="ts-configuration-env-syntax" %} use this syntax for environment variables specific to a particular Azure service:

* `AZURE_<ServiceName>_<ConfigurationKey>`

where _ServiceName_ is the canonical shortname without spaces, and _ConfigurationKey_ refers to an unnested configuration key for that client library.

{% include requirement/MUSTNOT id="ts-configuration-posix-envs" %} use non-alpha-numeric characters in your environment variable names with the exception of underscore. This ensures broad interoperability.

## Logging {#general-logging}

Client libraries must support robust logging mechanisms so that the consumer can adequately diagnose issues with the method calls and quickly determine whether the issue is in the consumer code, client library code, or service.

{% include requirement/MUST id="ts-logging-use-debug-module" %} use the `debug` module to log to stderr or the browser console.

{% include requirement/MUST id="general-logging-console" %} make it easy for a consumer to enable logging output to the console. The specific steps required to enable logging to the console must be documented.

{% include requirement/MUST id="ts-logging-prefix-channel-names" %} prefix channel names with `azure:<service-name>`.

{% include requirement/MUST id="ts-logging-channels" %} create log channels for the following log levels with the following channel name suffixes:

* Error: `:error`
* Warning: `:warning`
* Info: `:info`
* Verbpse: `:verbose`

{% include requirement/MAY id="ts-logging-additional-channels" %} have additional log channels, for example, to log from separate components. However, these channels MUST still provide the three log levels from above for each subchannel.

{% include requirement/MUST id="ts-logging-top-level-exports" %} expose all log channels as top-level exports of your package, allowing the consumer to configure how the logging happens and integrate with 3rd-party loggers.

{% include requirement/MUST id="general-logging-levels-error" %} use the Error channel for failures that the application is unlikely to recover from (out of memory, etc.).

{% include requirement/MUST id="general-logging-levels-warning" %} use the Warning channel when a function fails to perform its intended task. This generally means that the function will raise an exception.  Do not include occurrences of self-healing events (for example, when a request will be automatically retried).

{% include requirement/MUST id="general-logging-levels-informational" %} use the Info channel when a function operates normally.

{% include requirement/MUST id="general-logging-levels-verbose" %} use the Verbose channel for detailed troubleshooting scenarios. This is primarily intended for developers or system administrators to diagnose specific failures.

{% include requirement/MUSTNOT id="general-logging-no-sensitive-info" %} send sensitive information in channels other than Verbose. For example, remove account keys when logging headers.

{% include requirement/MUST id="general-logging-requests-in-info" %} log request line, response line, and headers on the Info channel.

{% include requirement/MUST id="general-logging-info-if-cancelled" %} log to the Info channel if a service call is cancelled.

{% include requirement/MUST id="general-logging-error-if-exceptions" %} log exceptions thrown to the Warning channel. Additionally, send stack trace information to the Verbose channel.

## Distributed tracing {#general-distributed-tracing}

Distributed tracing mechanisms allow the consumer to trace their code from frontend to backend. The distributed tracing library creates spans - units of unique work.  Each span is in a parent-child relationship.  As you go deeper into the hierarchy of code, you create more spans.  These spans can then be exported to a suitable receiver as needed.  To keep track of the spans, a _distributed tracing context_ (called a context in the remainder of this section) is passed into each successive layer.  For more information on this topic, visit the [OpenTelemetry] topic on tracing.

{% include draft.html content="DRAFT GUIDELINES" %}

{% include requirement/MUST id="general-tracing-support-opentelemetry" %} support [OpenTelemetry] for distributed tracing.

{% include requirement/MUST id="general-tracing-parent-span" %} take an option named `parentSpanId` for all asynchronous operations.

{% include requirement/MUST id="general-tracing-pass-context" %} pass the context to the backend service through the appropriate headers (`traceparent`, `tracestate`, etc.) to support [Azure Monitor].  This is generally done with the HTTP pipeline.

{% include requirement/MUST id="general-tracing-create-span-on-entry" %} create a new span for each method that user code calls.  New spans must be children of the context that was passed in.  If no context was passed in, a new root span must be created.

{% include requirement/MUST id="general-tracing-create-span-on-rest" %} create a new span (which must be a child of the per-method span) for each REST call that the client library makes.  This is generally done with the HTTP pipeline.

Some of these requirements will be handled by the HTTP pipeline.  However, as a client library writer, you must handle the incoming context appropriately.  JavaScript doesn't have primitives similar to a local context.  As such, we must manually plumb parent span IDs into the library.

## Dependencies {#ts-dependencies}

Dependencies bring in many considerations that are often easily avoided by avoiding the dependency:

**Versioning**: Many programming languages don't allow a consumer to load multiple versions of the same package. For example, if we have a client library that requires v3 of package `Foo` and the consumer wants to use v5 of package `Foo`, then the consumer can't build their application. Client libraries shouldn't have dependencies by default.

**Size**: Consumer applications need to deploy as fast as possible into the cloud. Remove additional code (like dependencies) to improve deployment performance.

**Licensing**: You must be conscious of the licensing restrictions of a dependency and often provide proper attribution and notices when using them.

**Compatibility**: You don't control the dependency. It may choose to evolve in a direction that is incompatible with your original use.

**Security**: If a vulnerability is discovered in a dependency, it may be difficult or time consuming to get the vulnerability corrected.

{% include requirement/MUST id="ts-dependencies-azure-core" %} depend on the Azure Core library for functionality that is common across all client libraries.  This library includes APIs for HTTP connectivity, global configuration, and credential handling.

{% include requirement/MUSTNOT id="ts-dependencies-no-other-packages" %} depend any other packages within the client library distribution package. Dependencies are thoroughly vetted through architecture review.  Build dependencies, by contrast, are acceptable and commonly used.

{% include requirement/SHOULD id="ts-dependencies-consider-vendoring" %} consider copying or linking required code into the client library to avoid taking a dependency on another package. Don't violate the license agreements. Consider the maintenance that will be required when duplicating code. ["A little copying is better than a little dependency"](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=9m28s) (YouTube).

{% include requirement/MUSTNOT id="general-no-concrete-logging" %} depend on concrete logging, dependency injection, or configuration technologies (except as implemented in the Azure Core library).

{% include requirement/SHOULDNOT id="ts-dependencies-no-tiny-libraries" %} take dependencies on tiny libraries as the cost of many small libraries adds up over time. Larger dependencies are subject to approval.

{% include requirement/SHOULDNOT id="ts-dependencies-no-polyfills" %} depend directly on polyfills or other libraries that modify global scope. If developers using older runtimes need to polyfill some capability, the package install and usage instructions (in the `README`) should indicate this dependency.

The following table lists the well-known and already-blessed dependencies outside of Azure Core that may be used in production (non-test) code.
<a name="ts-known-deps"></a>

{% include_relative approved_dependencies.md %}

## Service-specific common library code

There are occasions when common code needs to be shared between several client libraries.  For example, a set of cooperating client libraries may wish to share a set of exceptions or models.

{% include requirement/MUST id="general-implementing-common-library-usage" %} gain [Architecture Board] approval prior to implementing a common library.

{% include requirement/MUST id="general-implementing-minimal-common-content" %} minimize the code within a common library.  Code within the common library is available to the consumer of the client library and shared by multiple client libraries within the same namespace.

{% include requirement/MUST id="general-implementing-common-namespace" %} store the common library in the same namespace as the associated client libraries.

A common library will only be approved if:

* The consumer of the non-shared library will consume the objects within the common library directly, AND
* The information will be shared between multiple client libraries.

Let's take two examples:

1. Implementing two Cognitive Services client libraries, we find a model is required that is produced by one Cognitive Services client library and consumed by another Coginitive Services client library, or the same model is produced by two client libraries.  The consumer is required to do the passing of the model in their code, or may need to compare the model produced by one client library vs. that produced by another client library.  This is a good candidate for choosing a common library.

2. Two Cognitive Services client libraries throw an `ObjectNotFound` exception to indicate that an object was not detected in an image.  The user might trap the exception, but otherwise will not operate on the exception.  There is no linkage between the `ObjectNotFound` exception in each client library.  This is not a good candidate for creation of a common library (although you may wish to place this exception in a common library if one exists for the namespace already).  Instead, produce two different exceptions - one in each client library.

## Testing TypeScript libraries

{% include requirement/SHOULD id="ts-use-mocha-karma" %} use [Mocha](https://mochajs.org/) and [Karma](http://karma-runner.github.io/4.0/index.html) as these tools support build pipelines and work in browsers and node.

## Versioning {#ts-versioning}

Consistent versioning allows consumers to determine what to expect from a new version of the library.  However, versioning rules tend to be very idiomatic to the language.  The engineering system release guidelines require the use of _MAJOR_._MINOR_._PATCH_ format for the version.

{% include requirement/MUST id="general-versioning-bump" %} change the version number of the client library when **ANYTHING** changes in the client library.

{% include requirement/MUST id="general-versioning-patch" %} increment the patch version when fixing a bug.

{% include requirement/MUSTNOT id="general-versioning-no-features-in-patch" %} include new features in a patch release.

{% include requirement/MUST id="general-versioning-adding-features" %} increment the major or minor version when adding support for a service API version, or add a backwards-compatible feature.

{% include requirement/MUSTNOT id="general-versioning-no-breaking-changes" %} make breaking changes.  If a breaking change is absolutely required, then you **MUST** engage with the [Architecture Board] prior to making the change.  If a breaking change is approved, increment the major version.

{% include requirement/SHOULD id="general-versioning-major-bump" %} increment the major version when making large feature changes.

{% include requirement/MUST id="general-versioning-serviceapi-support" %} provide the ability to call a specific supported version of the service API.

A particular (major.minor) version of a library can choose what service APIs it supports.  We recommend the support window be no less than two service versions (if available) and no less than what is specified in the [Fixed Lifecycle Policy for Microsoft business, developer, and desktop systems](https://support.microsoft.com/help/14085).

{% include requirement/MUST id="ts-versioning-semver" %} version with [semver](https://semver.org/). Deprecated features and flags must offer an alternate stable or beta path for developers.

{% include requirement/MUSTNOT id="ts-versioning-no-ga-prerelease" %} have a pre-release version or any additional build metadata for GA packages.

{% include requirement/MUST id="ts-versioning-beta" %} give beta packages a pre-release version of the format `1.0.0-beta.X` where X is an integer. Pre-release package versions shouldn't have additional build metadata.

{% include requirement/MUSTNOT id="ts-versioning-no-version-0" %} use a major version of 0, even for beta packages.

{% include requirement/MUST id="general-versioning-bump" %} select a version number greater than the highest version number of any other released Track 1 package for the service in any other npm scope or language.

Semantic versioning is more of a lofty ideal than a practical specification for some libraries. Also, [one person's bug might be another person's key feature](https://xkcd.com/1172/). Package authors are required to follow semver in a way that is useful for their consumers.

For more details, review the [Releases policy]({{ site.baseurl }}/policies_releases.html).
