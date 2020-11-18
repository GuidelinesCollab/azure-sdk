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

## Testing TypeScript libraries

{% include requirement/SHOULD id="ts-use-mocha-karma" %} use [Mocha](https://mochajs.org/) and [Karma](http://karma-runner.github.io/4.0/index.html) as these tools support build pipelines and work in browsers and node.
