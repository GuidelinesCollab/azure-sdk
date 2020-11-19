---
title: "Python Guidelines: Implementation"
keywords: guidelines python
permalink: python_implementation.html
folder: python
sidebar: general_sidebar
---

## Configuration

{% include requirement/MUST id="python-envvars-global" %} honor the following environment variables for global configuration settings:

{% include tables/environment_variables.md %}

## Azure Core

The `azure-core` package provides common functionality for client libraries. Documentation and usage examples can be found in the [azure/azure-sdk-for-python] repository.

### HTTP pipeline

The HTTP pipeline is an HTTP transport that is wrapped by multiple policies. Each policy is a control point that can modify either the request or response. A default set of policies is provided to standardize how client libraries interact with Azure services.

For more information on the Python implementation of the pipeline, see the [documentation](https://github.com/Azure/azure-sdk-for-python/tree/master/sdk/core/azure-core).

#### Custom Policies

Some services may require custom policies to be implemented. For example, custom policies may implement fall back to secondary endpoints during retry, request signing, or other specialized authentication techniques.

{% include requirement/SHOULD id="python-pipeline-core-policies" %} use the policy implementations in `azure-core` whenever possible.

{% include requirement/MUST id="python-pipeline-document-policies" %} document any custom policies in your package. The documentation should make it clear how a user of your library is supposed to use the policy.

{% include requirement/MUST id="python-pipeline-policy-namespace" %} add the policies to the `azure.<package name>.pipeline.policies` namespace.

### Protocols

#### LROPoller

```python
T = TypeVar("T")
class LROPoller(Protocol):

    def result(self, timeout=None) -> T:
        """ Retreive the final result of the long running operation.

        :param timeout: How long to wait for operation to complete (in seconds). If not specified, there is no timeout.
        :raises TimeoutException: If the operation has not completed before it timed out.
        """
        ...

    def wait(self, timeout=None) -> None:
        """ Wait for the operation to complete.

        :param timeout: How long to wait for operation to complete (in seconds). If not specified, there is no timeout.
        """

    def done(self) -> boolean:
        """ Check if long running operation has completed.
        """

    def add_done_callback(self, func) -> None:
        """ Register callback to be invoked when operation completes.

        :param func: Callable that will be called with the eventual result ('T') of the operation.
        """
        ...
```

`azure.core.LROPoller` implements the `LROPoller` protocol.

#### Paged

```python
T = TypeVar("T")
class ByPagePaged(Protocol, Iterable[Iterable[T]]):
    continuation_token: "str"

class Paged(Protocol, Iterable[T]):
    continuation_token: "str"

    def by_page(self) -> ByPagePaged[T] ...
```

`azure.core.Paged` implements the `Paged` protocol.

#### DiagnosticsResponseHook

```python
class ResponseHook(Protocol):

    __call__(self, headers, deserialized_response): -> None ...

```

## Testing

{% include requirement/MUST id="python-testing-pytest" %} use [pytest](https://docs.pytest.org/en/latest/) as the test framework.

{% include requirement/SHOULD id="python-testing-async" %} use [pytest-asyncio](https://github.com/pytest-dev/pytest-asyncio) for testing of async code.

{% include requirement/MUST id="python-testing-live" %} make your scenario tests runnable against live services. Strongly consider using the [Python Azure-DevTools](https://github.com/Azure/azure-python-devtools) package for scenario tests.

{% include requirement/MUST id="python-testing-record" %} provide recordings to allow running tests offline/without an Azure subscription

{% include requirement/MUST id="python-testing-parallel" %} support simultaneous test runs in the same subscription.

{% include requirement/MUST id="python-testing-independent" %} make each test case independent of other tests.

## Recommended Tools

{% include requirement/MUST id="python-tooling-pylint" %} use [pylint](https://www.pylint.org/) for your code. Use the pylintrc file in the [root of the repository](https://github.com/Azure/azure-sdk-for-python/blob/master/pylintrc).

{% include requirement/MUST id="python-tooling-flake8" %} use [flake8-docstrings](https://gitlab.com/pycqa/flake8-docstrings) to verify doc comments.

{% include requirement/MUST id="python-tooling-black" %} use [Black](https://black.readthedocs.io/en/stable/) for formatting your code.

{% include requirement/SHOULD id="python-tooling-mypy" %} use [MyPy](https://mypy.readthedocs.io/en/latest/) to statically check the public surface area of your library.

You don't need to check non-shipping code such as tests.

{% include refs.md %}
{% include_relative refs.md %}
