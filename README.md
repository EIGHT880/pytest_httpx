<h2 align="center">Send responses to HTTPX using pytest</h2>

<p align="center">
<a href="https://pypi.org/project/pytest-httpx/"><img alt="pypi version" src="https://img.shields.io/pypi/v/pytest_httpx"></a>
<a href="https://github.com/Colin-b/pytest_httpx/actions"><img alt="Build status" src="https://github.com/Colin-b/pytest_httpx/workflows/Release/badge.svg"></a>
<a href="https://codecov.io/gh/Colin-b/pytest_httpx" ><img alt="Coverage" src="https://codecov.io/gh/Colin-b/pytest_httpx/branch/master/graph/badge.svg?token=U2PJFVMCAL"/></a>
<a href="https://codecov.io/gh/Colin-b/pytest_httpx/tests"><img alt="Number of tests" src="https://img.shields.io/badge/tests-338 passed-blue"></a>
<a href="https://pypi.org/project/pytest-httpx/"><img alt="Number of downloads" src="https://img.shields.io/pypi/dm/pytest_httpx"></a>
</p>

> [!NOTE]
> Version 1.0.0 will be released once httpx is considered as stable (release of 1.0.0).
>
> However, current state can be considered as stable.

Once installed, `httpx_mock` [`pytest`](https://docs.pytest.org/en/latest/) fixture will make sure every [`httpx`](https://www.python-httpx.org) request will be replied to with user provided responses ([unless some hosts are explicitly skipped](#do-not-mock-some-requests)).

- [Add responses](#add-responses)
  - [JSON body](#add-json-response)
  - [Custom body](#reply-with-custom-body)
  - [Multipart body (files, ...)](#add-multipart-response)
  - [HTTP status code](#add-non-200-response)
  - [HTTP headers](#reply-with-custom-headers)
  - [HTTP/2.0](#add-http/2.0-response)
- [Add dynamic responses](#dynamic-responses)
- [Raising exceptions](#raising-exceptions)
- [Check requests](#check-sent-requests)
- [Configuration](#configuring-httpx_mock)
  - [Register more responses than requested](#allow-to-register-more-responses-than-what-will-be-requested)
  - [Register less responses than requested](#allow-to-not-register-responses-for-every-request)
  - [Allow to register a response for more than one request](#allow-to-register-a-response-for-more-than-one-request)
  - [Do not mock some requests](#do-not-mock-some-requests)
- [Migrating](#migrating-to-pytest-httpx)
  - [responses](#from-responses)
  - [aioresponses](#from-aioresponses)

## Add responses

You can register responses for both sync and async [`HTTPX`](https://www.python-httpx.org) requests.

```python
import pytest
import httpx


def test_something(httpx_mock):
    httpx_mock.add_response()

    with httpx.Client() as client:
        response = client.get("https://test_url")


@pytest.mark.asyncio
async def test_something_async(httpx_mock):
    httpx_mock.add_response()

    async with httpx.AsyncClient() as client:
        response = await client.get("https://test_url")
```

If all registered responses are not sent back during test execution, the test case will fail at teardown [(unless you turned `assert_all_responses_were_requested` option off)](#allow-to-register-more-responses-than-what-will-be-requested).

Default response is a `HTTP/1.1` `200 (OK)` without any body.

### How response is selected

In case more than one response match request, the first one not yet sent (according to the registration order) will be sent.

In case all matching responses have been sent once, the request will [not be considered as matched](#in-case-no-response-can-be-found) [(unless you turned `can_send_already_matched_responses` option on)](#allow-to-register-a-response-for-more-than-one-request).

You can add criteria so that response will be sent only in case of a more specific matching.

#### Matching on URL

`url` parameter can either be a string, a python [re.Pattern](https://docs.python.org/3/library/re.html) instance or a [httpx.URL](https://www.python-httpx.org/api/#url) instance.

Matching is performed on the full URL, query parameters included.

Order of parameters in the query string does not matter, however order of values do matter if the same parameter is provided more than once.

```python
import httpx
import re
from pytest_httpx import HTTPXMock


def test_url(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url="https://test_url?a=1&b=2")

    with httpx.Client() as client:
        response1 = client.delete("https://test_url?a=1&b=2")
        response2 = client.get("https://test_url?b=2&a=1")


def test_url_as_pattern(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url=re.compile(".*test.*"))

    with httpx.Client() as client:
        response = client.get("https://test_url")


def test_url_as_httpx_url(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url=httpx.URL("https://test_url", params={"a": "1", "b": "2"}))

    with httpx.Client() as client:
        response = client.get("https://test_url?a=1&b=2")
```

##### Ignoring query parameters

Use a python [re.Pattern](https://docs.python.org/3/library/re.html) instance to ignore query parameters while matching on the URL.

```python
import httpx
import re
from pytest_httpx import HTTPXMock


def test_url_as_pattern_ignoring_query_parameters(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url=re.compile("https://test_url/something.*"))

    with httpx.Client() as client:
        response = client.get("https://test_url/something?a=1&b=2")
        assert response.content == b""
```

#### Matching on query parameters

Use `match_params` to partially match query parameters without having to provide a regular expression as `url`.

If this parameter is provided, `url` parameter must not contain any query parameter.

All query parameters have to be provided. You can however use `unittest.mock.ANY` to do partial matching.

```python
import httpx
from pytest_httpx import HTTPXMock
from unittest.mock import ANY

def test_partial_params_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url="https://test_url", match_params={"a": 1, "b": ANY})

    with httpx.Client() as client:
        response = client.get("https://test_url?a=1&b=2")

def test_partial_multi_params_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(url="https://test_url", match_params={"a": ["1", 3], "b": ["2", ANY]})

    with httpx.Client() as client:
        response = client.get("https://test_url?a=1&b=2&a=3&b=4")
```

#### Matching on HTTP method

Use `method` parameter to specify the HTTP method (POST, PUT, DELETE, PATCH, HEAD) to reply to.

`method` parameter must be a string. It will be upper-cased, so it can be provided lower cased.

Matching is performed on equality.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_post(httpx_mock: HTTPXMock):
    httpx_mock.add_response(method="POST")

    with httpx.Client() as client:
        response = client.post("https://test_url")


def test_put(httpx_mock: HTTPXMock):
    httpx_mock.add_response(method="PUT")

    with httpx.Client() as client:
        response = client.put("https://test_url")


def test_delete(httpx_mock: HTTPXMock):
    httpx_mock.add_response(method="DELETE")

    with httpx.Client() as client:
        response = client.delete("https://test_url")


def test_patch(httpx_mock: HTTPXMock):
    httpx_mock.add_response(method="PATCH")

    with httpx.Client() as client:
        response = client.patch("https://test_url")


def test_head(httpx_mock: HTTPXMock):
    httpx_mock.add_response(method="HEAD")

    with httpx.Client() as client:
        response = client.head("https://test_url")

```

#### Matching on proxy URL

`proxy_url` parameter can either be a string, a python [re.Pattern](https://docs.python.org/3/library/re.html) instance or a [httpx.URL](https://www.python-httpx.org/api/#url) instance.

Matching is performed on the full proxy URL, query parameters included.

Order of parameters in the query string does not matter, however order of values do matter if the same parameter is provided more than once.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_proxy_url(httpx_mock: HTTPXMock):
    httpx_mock.add_response(proxy_url="http://test_proxy_url?b=1&a=2")

    with httpx.Client(proxy="http://test_proxy_url?a=2&b=1") as client:
        response = client.get("https://test_url")
```

#### Matching on HTTP headers

Use `match_headers` parameter to specify the HTTP headers (as a dict) to reply to.

Matching is performed on equality for each provided header.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_headers_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_headers={'User-Agent': 'python-httpx/0.25.0'})

    with httpx.Client() as client:
        response = client.get("https://test_url")
```

#### Matching on HTTP body

Use `match_content` parameter to specify the full HTTP body (as bytes) to reply to.

Matching is performed on equality.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_content_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_content=b"This is the body")

    with httpx.Client() as client:
        response = client.post("https://test_url", content=b"This is the body")
```

##### Matching on HTTP JSON body

Use `match_json` parameter to specify the JSON decoded HTTP body to reply to.

Matching is performed on equality. You can however use `unittest.mock.ANY` to do partial matching.

```python
import httpx
from pytest_httpx import HTTPXMock
from unittest.mock import ANY

def test_json_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_json={"a": "json", "b": 2})

    with httpx.Client() as client:
        response = client.post("https://test_url", json={"a": "json", "b": 2})


def test_partial_json_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_json={"a": "json", "b": ANY})

    with httpx.Client() as client:
        response = client.post("https://test_url", json={"a": "json", "b": 2})
```

Note that `match_content` or `match_files` cannot be provided if `match_json` is also provided.

##### Matching on HTTP multipart body

Use `match_files` and `match_data` parameters to specify the full multipart body to reply to.

Matching is performed on equality.

```python
import httpx
from pytest_httpx import HTTPXMock

def test_multipart_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_files={"name": ("file_name", b"File content")}, match_data={"field": "value"})

    with httpx.Client() as client:
        response = client.post("https://test_url", files={"name": ("file_name", b"File content")}, data={"field": "value"})
```

Note that `match_content` or `match_json` cannot be provided if `match_files` is also provided.

#### Matching on extensions

Use `match_extensions` parameter to specify the extensions (as a dict) to reply to.

Matching is performed on equality for each provided extension.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_extensions_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_extensions={'test': 'value'})

    with httpx.Client() as client:
        response = client.get("https://test_url", extensions={"test": "value"})
```

##### Matching on HTTP timeout(s)

Use `match_extensions` parameter to specify the timeouts (as a dict) to reply to.

Matching is performed on the full timeout dict equality.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_timeout_matching(httpx_mock: HTTPXMock):
    httpx_mock.add_response(match_extensions={'timeout': {'connect': 10, 'read': 10, 'write': 10, 'pool': 10}})

    with httpx.Client() as client:
        response = client.get("https://test_url", timeout=10)
```

### Add JSON response

Use `json` parameter to add a JSON response using python values.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_json(httpx_mock: HTTPXMock):
    httpx_mock.add_response(json=[{"key1": "value1", "key2": "value2"}])

    with httpx.Client() as client:
        assert client.get("https://test_url").json() == [{"key1": "value1", "key2": "value2"}]

```

Note that the `content-type` header will be set to `application/json` by default in the response.

### Reply with custom body

Use `text` parameter to reply with a custom body by providing UTF-8 encoded string.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_str_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response(text="This is my UTF-8 content")

    with httpx.Client() as client:
        assert client.get("https://test_url").text == "This is my UTF-8 content"

```

Use `content` parameter to reply with a custom body by providing bytes.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_bytes_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response(content=b"This is my bytes content")

    with httpx.Client() as client:
        assert client.get("https://test_url").content == b"This is my bytes content"

```

Use `html` parameter to reply with a custom body by providing UTF-8 encoded string.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_html_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response(html="<body>This is <p> HTML content</body>")

    with httpx.Client() as client:
        assert client.get("https://test_url").text == "<body>This is <p> HTML content</body>"

```

### Reply by streaming chunks

Use `stream` parameter (as `httpx.SyncByteStream` or `httpx.AsyncByteStream`) to stream chunks that you specify.

Note that `pytest_httpx.IteratorStream` can be used to provide an iterable.

```python
import httpx
import pytest
from pytest_httpx import HTTPXMock, IteratorStream

def test_sync_streaming(httpx_mock: HTTPXMock):
    httpx_mock.add_response(stream=IteratorStream([b"part 1", b"part 2"]))

    with httpx.Client() as client:
        with client.stream(method="GET", url="https://test_url") as response:
            assert list(response.iter_raw()) == [b"part 1", b"part 2"]


@pytest.mark.asyncio
async def test_async_streaming(httpx_mock: HTTPXMock):
    httpx_mock.add_response(stream=IteratorStream([b"part 1", b"part 2"]))

    async with httpx.AsyncClient() as client:
        async with client.stream(method="GET", url="https://test_url") as response:
            assert [part async for part in response.aiter_raw()] == [b"part 1", b"part 2"]

```

### Add multipart response

Use the httpx `MultipartStream` via the `stream` parameter to send a multipart response.

Reach out to `httpx` developers if you need this publicly exposed as [this is not a standard use case](https://github.com/encode/httpx/issues/872#issuecomment-633584819).

```python
import httpx
from httpx._multipart import MultipartStream
from pytest_httpx import HTTPXMock


def test_multipart_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response(stream=MultipartStream(data={"key1": "value1"}, files={"file1": b"content of file 1"}, boundary=b"2256d3a36d2a61a1eba35a22bee5c74a"))

    with httpx.Client() as client:
        assert client.get("https://test_url").text == '''--2256d3a36d2a61a1eba35a22bee5c74a\r
Content-Disposition: form-data; name="key1"\r
\r
value1\r
--2256d3a36d2a61a1eba35a22bee5c74a\r
Content-Disposition: form-data; name="file1"; filename="upload"\r
Content-Type: application/octet-stream\r
\r
content of file 1\r
--2256d3a36d2a61a1eba35a22bee5c74a--\r
'''

```

### Add non 200 response

Use `status_code` parameter to specify the HTTP status code (as an int) of the response.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_status_code(httpx_mock: HTTPXMock):
    httpx_mock.add_response(status_code=404)

    with httpx.Client() as client:
        assert client.get("https://test_url").status_code == 404

```

### Reply with custom headers

Use `headers` parameter to specify the extra headers of the response.

Any valid httpx headers type is supported, you can submit headers as a dict (str or bytes), a list of 2-tuples (str or bytes) or a [`httpx.Header`](https://www.python-httpx.org/api/#headers) instance.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_headers_as_str_dict(httpx_mock: HTTPXMock):
    httpx_mock.add_response(headers={"X-Header1": "Test value"})

    with httpx.Client() as client:
        assert client.get("https://test_url").headers["x-header1"] == "Test value"


def test_headers_as_str_tuple_list(httpx_mock: HTTPXMock):
    httpx_mock.add_response(headers=[("X-Header1", "Test value")])

    with httpx.Client() as client:
        assert client.get("https://test_url").headers["x-header1"] == "Test value"


def test_headers_as_httpx_headers(httpx_mock: HTTPXMock):
    httpx_mock.add_response(headers=httpx.Headers({b"X-Header1": b"Test value"}))

    with httpx.Client() as client:
        assert client.get("https://test_url").headers["x-header1"] == "Test value"

```

#### Reply with cookies

Cookies are sent in the `set-cookie` HTTP header.

You can then send cookies in the response by setting the `set-cookie` header with [the value following key=value format](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie).

```python
import httpx
from pytest_httpx import HTTPXMock


def test_cookie(httpx_mock: HTTPXMock):
    httpx_mock.add_response(headers={"set-cookie": "key=value"})

    with httpx.Client() as client:
        response = client.get("https://test_url")
    assert dict(response.cookies) == {"key": "value"}


def test_cookies(httpx_mock: HTTPXMock):
    httpx_mock.add_response(headers=[("set-cookie", "key=value"), ("set-cookie", "key2=value2")])

    with httpx.Client() as client:
        response = client.get("https://test_url")
    assert dict(response.cookies) == {"key": "value", "key2": "value2"}

```


### Add HTTP/2.0 response

Use `http_version` parameter to specify the HTTP protocol version (as a string) of the response.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_http_version(httpx_mock: HTTPXMock):
    httpx_mock.add_response(http_version="HTTP/2.0")

    with httpx.Client() as client:
        assert client.get("https://test_url").http_version == "HTTP/2.0"

```

## Add callbacks

You can perform custom manipulation upon request reception by registering callbacks.

Callback should expect one parameter, the received [`httpx.Request`](https://www.python-httpx.org/api/#request).

If all callbacks are not executed during test execution, the test case will fail at teardown [(unless you turned `assert_all_responses_were_requested` option off)](#allow-to-register-more-responses-than-what-will-be-requested).

Note that callbacks are considered as responses, and thus are [selected the same way](#how-response-is-selected).
Meaning that you can transpose `httpx_mock.add_response` calls in the related examples into `httpx_mock.add_callback`.

### Dynamic responses

Callback should return a [`httpx.Response`](https://www.python-httpx.org/api/#response) instance.

```python
import httpx
from pytest_httpx import HTTPXMock


def test_dynamic_response(httpx_mock: HTTPXMock):
    def custom_response(request: httpx.Request):
        return httpx.Response(
            status_code=200, json={"url": str(request.url)},
        )

    httpx_mock.add_callback(custom_response)

    with httpx.Client() as client:
        response = client.get("https://test_url")
        assert response.json() == {"url": "https://test_url"}

```

Alternatively, callbacks can also be asynchronous.

As in the following sample simulating network latency on some responses only.

```python
import asyncio
import httpx
import pytest
from pytest_httpx import HTTPXMock


@pytest.mark.asyncio
async def test_dynamic_async_response(httpx_mock: HTTPXMock):
    async def simulate_network_latency(request: httpx.Request):
        await asyncio.sleep(1)
        return httpx.Response(
            status_code=200, json={"url": str(request.url)},
        )

    httpx_mock.add_callback(simulate_network_latency)
    httpx_mock.add_response()

    async with httpx.AsyncClient() as client:
        responses = await asyncio.gather(
            # Response will be received after one second
            client.get("https://test_url"),
            # Response will instantly be received (1 second before the first request)
            client.get("https://test_url")
        )

```

### Raising exceptions

You can simulate HTTPX exception throwing by raising an exception in your callback or use `httpx_mock.add_exception` with the exception instance.

This can be useful if you want to assert that your code handles HTTPX exceptions properly.

```python
import httpx
import pytest
from pytest_httpx import HTTPXMock


def test_exception_raising(httpx_mock: HTTPXMock):
    httpx_mock.add_exception(httpx.ReadTimeout("Unable to read within timeout"))

    with httpx.Client() as client:
        with pytest.raises(httpx.ReadTimeout):
            client.get("https://test_url")

```

#### In case no response can be found

The default behavior is to instantly raise a [`httpx.TimeoutException`](https://www.python-httpx.org/advanced/timeouts/) in case no matching response can be found.

The exception message will display the request and every registered responses to help you identify any possible mismatch.

```python
import httpx
import pytest
from pytest_httpx import HTTPXMock


@pytest.mark.httpx_mock(assert_all_requests_were_expected=False)
def test_timeout(httpx_mock: HTTPXMock):
    with httpx.Client() as client:
        with pytest.raises(httpx.TimeoutException):
            client.get("https://test_url")

```

## Check sent requests

The best way to ensure the content of your requests is still to use the `match_headers` and / or `match_content` parameters when adding a response.
In the same spirit, ensuring that no request was issued does not necessarily require any code [(unless you turned `assert_all_requests_were_expected` option off)](#allow-to-not-register-responses-for-every-request).

In any case, you always have the ability to retrieve the requests that were issued.

As in the following samples:

```python
import httpx
from pytest_httpx import HTTPXMock


def test_many_requests(httpx_mock: HTTPXMock):
    httpx_mock.add_response()

    with httpx.Client() as client:
        response1 = client.get("https://test_url")
        response2 = client.get("https://test_url")

    requests = httpx_mock.get_requests()


def test_single_request(httpx_mock: HTTPXMock):
    httpx_mock.add_response()

    with httpx.Client() as client:
        response = client.get("https://test_url")

    request = httpx_mock.get_request()


def test_no_request(httpx_mock: HTTPXMock):
    assert not httpx_mock.get_request()
```

### How requests are selected

You can add criteria so that requests will be returned only in case of a more specific matching.

Note that requests are [selected the same way as responses](#how-response-is-selected).
Meaning that you can transpose `httpx_mock.add_response` calls in the related examples into `httpx_mock.get_requests` or `httpx_mock.get_request`.

## Configuring httpx_mock

The `httpx_mock` marker is available and can be used to change the default behavior of the `httpx_mock` fixture.

Refer to [available options](#available-options) for an exhaustive list of options that can be set [per test](#per-test), [per module](#per-module) or even [on the whole test suite](#for-the-whole-test-suite).

### Per test

```python
import pytest

@pytest.mark.httpx_mock(assert_all_responses_were_requested=False)
def test_something(httpx_mock):
    ...
```

### Per module

```python
import pytest

pytestmark = pytest.mark.httpx_mock(assert_all_responses_were_requested=False)
```

### For the whole test suite

This should be set in the root `conftest.py` file.
```python
import pytest

def pytest_collection_modifyitems(session, config, items):
    for item in items:
        item.add_marker(pytest.mark.httpx_mock(assert_all_responses_were_requested=False))
```

> [!IMPORTANT]
> Note that [there currently is a bug in pytest](https://github.com/pytest-dev/pytest/issues/10406) where `pytest_collection_modifyitems` will actually add the marker AFTER its `module` and `class` registration.
>
> Meaning the order is currently:
> module -> class -> test suite -> test
>
> instead of:
> test suite -> module -> class -> test

### Available options

#### Allow to register more responses than what will be requested

By default, `pytest-httpx` will ensure that every response was requested during test execution.

If you want to add an optional response, you can use the `is_optional` parameter when [registering a response](#add-responses) or [a callback](#add-callbacks).

```python
def test_fewer_requests_than_expected(httpx_mock):
    # Even if this response never received a corresponding request, the test will not fail at teardown
    httpx_mock.add_response(is_optional=True)
```

If you don't have control over the response registration process (shared fixtures),
and you want to allow fewer requests than what you registered responses for,
you can use the `httpx_mock` marker `assert_all_responses_were_requested` option.

> [!CAUTION]
> Use this option at your own risk of not spotting regression (requests not sent) in your code base!

```python
import pytest

@pytest.mark.httpx_mock(assert_all_responses_were_requested=False)
def test_fewer_requests_than_expected(httpx_mock):
    # Even if this response never received a corresponding request, the test will not fail at teardown
    httpx_mock.add_response()
```

Note that the `is_optional` parameter will take precedence over the `assert_all_responses_were_requested` option.
Meaning you can still register a response that will be checked for execution at teardown even if `assert_all_responses_were_requested` was set to `False`.

```python
import pytest

@pytest.mark.httpx_mock(assert_all_responses_were_requested=False)
def test_force_expected_request(httpx_mock):
    # Even if the assert_all_responses_were_requested option is set, the test will fail at teardown if this is not matched
    httpx_mock.add_response(is_optional=False)
```

#### Allow to not register responses for every request

By default, `pytest-httpx` will ensure that every request that was issued was expected.

You can use the `httpx_mock` marker `assert_all_requests_were_expected` option to allow more requests than what you registered responses for.

> [!CAUTION]
> Use this option at your own risk of not spotting regression (unexpected requests) in your code base!

```python
import pytest
import httpx

@pytest.mark.httpx_mock(assert_all_requests_were_expected=False)
def test_more_requests_than_expected(httpx_mock):
    with httpx.Client() as client:
        # Even if this request was not expected, the test will not fail at teardown
        with pytest.raises(httpx.TimeoutException):
            client.get("https://test_url")
```

#### Allow to register a response for more than one request

By default, `pytest-httpx` will ensure that every request that was issued was expected.

If you want to add a response once, while allowing it to match more than once, you can use the `is_reusable` parameter when [registering a response](#add-responses) or [a callback](#add-callbacks).

```python
import httpx

def test_more_requests_than_responses(httpx_mock):
    httpx_mock.add_response(is_reusable=True)
    with httpx.Client() as client:
        client.get("https://test_url")
        # Even if only one response was registered, the test will not fail at teardown as this request will also be matched
        client.get("https://test_url")
```

If you don't have control over the response registration process (shared fixtures),
and you want to allow multiple requests to match the same registered response,
you can use the `httpx_mock` marker `can_send_already_matched_responses` option.

With this option, in case all matching responses have been sent at least once, the last one (according to the registration order) will be sent.

> [!CAUTION]
> Use this option at your own risk of not spotting regression (requests issued more than the expected number of times) in your code base!

```python
import pytest
import httpx

@pytest.mark.httpx_mock(can_send_already_matched_responses=True)
def test_more_requests_than_responses(httpx_mock):
    httpx_mock.add_response()
    with httpx.Client() as client:
        client.get("https://test_url")
        # Even if only one response was registered, the test will not fail at teardown as this request will also be matched
        client.get("https://test_url")
```

#### Do not mock some requests

By default, `pytest-httpx` will mock every request.

But, for instance, in case you want to write integration tests with other servers, you might want to let some requests go through.

To do so, you can use the `httpx_mock` marker `should_mock` option and provide a callable expecting the [`httpx.Request`](https://www.python-httpx.org/api/#request) as parameter and returning a boolean.

Returning `True` will ensure that the request is handled by `pytest-httpx` (mocked), `False` will let the request pass through (not mocked).

```python
import pytest
import httpx

@pytest.mark.httpx_mock(should_mock=lambda request: request.url.host != "www.my_local_test_host")
def test_partial_mock(httpx_mock):
    httpx_mock.add_response()

    with httpx.Client() as client:
        # This request will NOT be mocked
        response1 = client.get("https://www.my_local_test_host/sub?param=value")
        # This request will be mocked
        response2 = client.get("https://test_url")
```

## Migrating to pytest-httpx

Here is how to migrate from well-known testing libraries to `pytest-httpx`.

### From responses

| Feature           | responses                  | pytest-httpx                |
|:------------------|:---------------------------|:----------------------------|
| Add a response    | `responses.add()`          | `httpx_mock.add_response()` |
| Add a callback    | `responses.add_callback()` | `httpx_mock.add_callback()` |
| Retrieve requests | `responses.calls`          | `httpx_mock.get_requests()` |

#### Add a response or a callback

Undocumented parameters means that they are unchanged between `responses` and `pytest-httpx`.
Below is a list of parameters that will require a change in your code.

| Parameter            | responses                           | pytest-httpx                                                         |
|:---------------------|:------------------------------------|:---------------------------------------------------------------------|
| method               | `method=responses.GET`              | `method="GET"`                                                       |
| body (as bytes)      | `body=b"sample"`                    | `content=b"sample"`                                                  |
| body (as str)        | `body="sample"`                     | `text="sample"`                                                      |
| status code          | `status=201`                        | `status_code=201`                                                    |
| headers              | `adding_headers={"name": "value"}`  | `headers={"name": "value"}`                                          |
| content-type header  | `content_type="application/custom"` | `headers={"content-type": "application/custom"}`                     |
| Match the full query | `match_querystring=True`            | The full query is always matched when providing the `url` parameter. |

Sample adding a response with `responses`:
```python
from responses import RequestsMock

def test_response(responses: RequestsMock):
    responses.add(
        method=responses.GET,
        url="https://test_url",
        body=b"This is the response content",
        status=400,
    )

```

Sample adding the same response with `pytest-httpx`:
```python
from pytest_httpx import HTTPXMock

def test_response(httpx_mock: HTTPXMock):
    httpx_mock.add_response(
        method="GET",
        url="https://test_url",
        content=b"This is the response content",
        status_code=400,
    )

```

### From aioresponses

| Feature        | aioresponses            | pytest-httpx                               |
|:---------------|:------------------------|:-------------------------------------------|
| Add a response | `aioresponses.method()` | `httpx_mock.add_response(method="METHOD")` |
| Add a callback | `aioresponses.method()` | `httpx_mock.add_callback(method="METHOD")` |

#### Add a response or a callback

Undocumented parameters means that they are unchanged between `responses` and `pytest-httpx`.
Below is a list of parameters that will require a change in your code.

| Parameter       | responses            | pytest-httpx        |
|:----------------|:---------------------|:--------------------|
| body (as bytes) | `body=b"sample"`     | `content=b"sample"` |
| body (as str)   | `body="sample"`      | `text="sample"`     |
| body (as JSON)  | `payload=["sample"]` | `json=["sample"]`   |
| status code     | `status=201`         | `status_code=201`   |

Sample adding a response with `aioresponses`:
```python
import pytest
from aioresponses import aioresponses


@pytest.fixture
def mock_aioresponse():
    with aioresponses() as m:
        yield m


def test_response(mock_aioresponse):
    mock_aioresponse.get(
        url="https://test_url",
        body=b"This is the response content",
        status=400,
    )

```

Sample adding the same response with `pytest-httpx`:
```python
def test_response(httpx_mock):
    httpx_mock.add_response(
        method="GET",
        url="https://test_url",
        content=b"This is the response content",
        status_code=400,
    )

```
#!/usr/bin/env bash

{ # this ensures the entire script is downloaded #

nvm_has() {
  type "$1" > /dev/null 2>&1
}

nvm_echo() {
  command printf %s\\n "$*" 2>/dev/null
}

if [ -z "${BASH_VERSION}" ] || [ -n "${ZSH_VERSION}" ]; then
  # shellcheck disable=SC2016
  nvm_echo >&2 'Error: the install instructions explicitly say to pipe the install script to `bash`; please follow them'
  exit 1
fi

nvm_grep() {
  GREP_OPTIONS='' command grep "$@"
}

nvm_default_install_dir() {
  [ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm"
}

nvm_install_dir() {
  if [ -n "$NVM_DIR" ]; then
    printf %s "${NVM_DIR}"
  else
    nvm_default_install_dir
  fi
}

nvm_latest_version() {
  nvm_echo "v0.40.4"
}

nvm_profile_is_bash_or_zsh() {
  local TEST_PROFILE
  TEST_PROFILE="${1-}"
  case "${TEST_PROFILE-}" in
    *"/.bashrc" | *"/.bash_profile" | *"/.zshrc" | *"/.zprofile")
      return
    ;;
    *)
      return 1
    ;;
  esac
}

#
# Outputs the location to NVM depending on:
# * The availability of $NVM_SOURCE
# * The presence of $NVM_INSTALL_GITHUB_REPO
# * The method used ("script" or "git" in the script, defaults to "git")
# NVM_SOURCE always takes precedence unless the method is "script-nvm-exec"
#
nvm_source() {
  local NVM_GITHUB_REPO
  NVM_GITHUB_REPO="${NVM_INSTALL_GITHUB_REPO:-nvm-sh/nvm}"
  if [ "${NVM_GITHUB_REPO}" != 'nvm-sh/nvm' ]; then
    { nvm_echo >&2 "$(cat)" ; } << EOF
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE REPO IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!

The default repository for this install is \`nvm-sh/nvm\`,
but the environment variables \`\$NVM_INSTALL_GITHUB_REPO\` is
currently set to \`${NVM_GITHUB_REPO}\`.

If this is not intentional, interrupt this installation and
verify your environment variables.
EOF
  fi
  local NVM_VERSION
  NVM_VERSION="${NVM_INSTALL_VERSION:-$(nvm_latest_version)}"
  local NVM_METHOD
  NVM_METHOD="$1"
  local NVM_SOURCE_URL
  NVM_SOURCE_URL="$NVM_SOURCE"
  if [ "_$NVM_METHOD" = "_script-nvm-exec" ]; then
    NVM_SOURCE_URL="https://raw.githubusercontent.com/${NVM_GITHUB_REPO}/${NVM_VERSION}/nvm-exec"
  elif [ "_$NVM_METHOD" = "_script-nvm-bash-completion" ]; then
    NVM_SOURCE_URL="https://raw.githubusercontent.com/${NVM_GITHUB_REPO}/${NVM_VERSION}/bash_completion"
  elif [ -z "$NVM_SOURCE_URL" ]; then
    if [ "_$NVM_METHOD" = "_script" ]; then
      NVM_SOURCE_URL="https://raw.githubusercontent.com/${NVM_GITHUB_REPO}/${NVM_VERSION}/nvm.sh"
    elif [ "_$NVM_METHOD" = "_git" ] || [ -z "$NVM_METHOD" ]; then
      NVM_SOURCE_URL="https://github.com/${NVM_GITHUB_REPO}.git"
    else
      nvm_echo >&2 "Unexpected value \"$NVM_METHOD\" for \$NVM_METHOD"
      return 1
    fi
  fi
  nvm_echo "$NVM_SOURCE_URL"
}

#
# Node.js version to install
#
nvm_node_version() {
  nvm_echo "$NODE_VERSION"
}

nvm_download() {
  if nvm_has "curl"; then
    curl --fail --compressed -q "$@"
  elif nvm_has "wget"; then
    # Emulate curl with wget
    ARGS=$(nvm_echo "$@" | command sed -e 's/--progress-bar /--progress=bar /' \
                            -e 's/--compressed //' \
                            -e 's/--fail //' \
                            -e 's/-L //' \
                            -e 's/-I /--server-response /' \
                            -e 's/-s /-q /' \
                            -e 's/-sS /-nv /' \
                            -e 's/-o /-O /' \
                            -e 's/-C - /-c /')
    # shellcheck disable=SC2086
    eval wget $ARGS
  fi
}

install_nvm_from_git() {
  local INSTALL_DIR
  INSTALL_DIR="$(nvm_install_dir)"
  local NVM_VERSION
  NVM_VERSION="${NVM_INSTALL_VERSION:-$(nvm_latest_version)}"
  if [ -n "${NVM_INSTALL_VERSION:-}" ]; then
    # Check if version is an existing ref
    if command git ls-remote "$(nvm_source "git")" "$NVM_VERSION" | nvm_grep -q "$NVM_VERSION" ; then
      :
    # Check if version is an existing changeset
    elif ! nvm_download -o /dev/null "$(nvm_source "script-nvm-exec")"; then
      nvm_echo >&2 "Failed to find '$NVM_VERSION' version."
      exit 1
    fi
  fi

  local fetch_error
  if [ -d "$INSTALL_DIR/.git" ]; then
    # Updating repo
    nvm_echo "=> nvm is already installed in $INSTALL_DIR, trying to update using git"
    command printf '\r=> '
    fetch_error="Failed to update nvm with $NVM_VERSION, run 'git fetch' in $INSTALL_DIR yourself."
  else
    fetch_error="Failed to fetch origin with $NVM_VERSION. Please report this!"
    nvm_echo "=> Downloading nvm from git to '$INSTALL_DIR'"
    command printf '\r=> '
    mkdir -p "${INSTALL_DIR}"
    if [ "$(ls -A "${INSTALL_DIR}")" ]; then
      # Initializing repo
      command git init "${INSTALL_DIR}" || {
        nvm_echo >&2 'Failed to initialize nvm repo. Please report this!'
        exit 2
      }
      command git --git-dir="${INSTALL_DIR}/.git" remote add origin "$(nvm_source)" 2> /dev/null \
        || command git --git-dir="${INSTALL_DIR}/.git" remote set-url origin "$(nvm_source)" || {
        nvm_echo >&2 'Failed to add remote "origin" (or set the URL). Please report this!'
        exit 2
      }
    else
      # Cloning repo
      command git clone -o origin "$(nvm_source)" --depth=1 "${INSTALL_DIR}" || {
        nvm_echo >&2 'Failed to clone nvm repo. Please report this!'
        exit 2
      }
    fi
  fi
  # Try to fetch tag
  if command git --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" fetch origin tag "$NVM_VERSION" --depth=1 2>/dev/null; then
    :
  # Fetch given version
  elif ! command git --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" fetch origin "$NVM_VERSION" --depth=1; then
    nvm_echo >&2 "$fetch_error"
    exit 1
  fi
  command git -c advice.detachedHead=false --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" checkout -f --quiet FETCH_HEAD || {
    nvm_echo >&2 "Failed to checkout the given version $NVM_VERSION. Please report this!"
    exit 2
  }
  if [ -n "$(command git --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" show-ref refs/heads/master)" ]; then
    if command git --no-pager --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" branch --quiet 2>/dev/null; then
      command git --no-pager --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" branch --quiet -D master >/dev/null 2>&1
    else
      nvm_echo >&2 "Your version of git is out of date. Please update it!"
      command git --no-pager --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" branch -D master >/dev/null 2>&1
    fi
  fi

  nvm_echo "=> Compressing and cleaning up git repository"
  if ! command git --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" reflog expire --expire=now --all; then
    nvm_echo >&2 "Your version of git is out of date. Please update it!"
  fi
  if ! command git --git-dir="$INSTALL_DIR"/.git --work-tree="$INSTALL_DIR" gc --auto --aggressive --prune=now ; then
    nvm_echo >&2 "Your version of git is out of date. Please update it!"
  fi
  return
}

#
# Automatically install Node.js
#
nvm_install_node() {
  local NODE_VERSION_LOCAL
  NODE_VERSION_LOCAL="$(nvm_node_version)"

  if [ -z "$NODE_VERSION_LOCAL" ]; then
    return 0
  fi

  nvm_echo "=> Installing Node.js version $NODE_VERSION_LOCAL"
  nvm install "$NODE_VERSION_LOCAL"
  local CURRENT_NVM_NODE

  CURRENT_NVM_NODE="$(nvm_version current)"
  if [ "$(nvm_version "$NODE_VERSION_LOCAL")" == "$CURRENT_NVM_NODE" ]; then
    nvm_echo "=> Node.js version $NODE_VERSION_LOCAL has been successfully installed"
  else
    nvm_echo >&2 "Failed to install Node.js $NODE_VERSION_LOCAL"
  fi
}

install_nvm_as_script() {
  local INSTALL_DIR
  INSTALL_DIR="$(nvm_install_dir)"
  local NVM_SOURCE_LOCAL
  NVM_SOURCE_LOCAL="$(nvm_source script)"
  local NVM_EXEC_SOURCE
  NVM_EXEC_SOURCE="$(nvm_source script-nvm-exec)"
  local NVM_BASH_COMPLETION_SOURCE
  NVM_BASH_COMPLETION_SOURCE="$(nvm_source script-nvm-bash-completion)"

  # Downloading to $INSTALL_DIR
  mkdir -p "$INSTALL_DIR"
  if [ -f "$INSTALL_DIR/nvm.sh" ]; then
    nvm_echo "=> nvm is already installed in $INSTALL_DIR, trying to update the script"
  else
    nvm_echo "=> Downloading nvm as script to '$INSTALL_DIR'"
  fi
  nvm_download -s "$NVM_SOURCE_LOCAL" -o "$INSTALL_DIR/nvm.sh" || {
    nvm_echo >&2 "Failed to download '$NVM_SOURCE_LOCAL'"
    return 1
  } &
  nvm_download -s "$NVM_EXEC_SOURCE" -o "$INSTALL_DIR/nvm-exec" || {
    nvm_echo >&2 "Failed to download '$NVM_EXEC_SOURCE'"
    return 2
  } &
  nvm_download -s "$NVM_BASH_COMPLETION_SOURCE" -o "$INSTALL_DIR/bash_completion" || {
    nvm_echo >&2 "Failed to download '$NVM_BASH_COMPLETION_SOURCE'"
    return 2
  } &
  for job in $(jobs -p | command sort)
  do
    wait "$job" || return $?
  done
  chmod a+x "$INSTALL_DIR/nvm-exec" || {
    nvm_echo >&2 "Failed to mark '$INSTALL_DIR/nvm-exec' as executable"
    return 3
  }
}

nvm_try_profile() {
  if [ -z "${1-}" ] || [ ! -f "${1}" ]; then
    return 1
  fi
  nvm_echo "${1}"
}

#
# Detect profile file if not specified as environment variable
# (eg: PROFILE=~/.myprofile)
# The echo'ed path is guaranteed to be an existing file
# Otherwise, an empty string is returned
#
nvm_detect_profile() {
  if [ "${PROFILE-}" = '/dev/null' ]; then
    # the user has specifically requested NOT to have nvm touch their profile
    return
  fi

  if [ -n "${PROFILE}" ] && [ -f "${PROFILE}" ]; then
    nvm_echo "${PROFILE}"
    return
  fi

  local DETECTED_PROFILE
  DETECTED_PROFILE=''

  if [ "${SHELL#*bash}" != "$SHELL" ]; then
    if [ -f "$HOME/.bashrc" ]; then
      DETECTED_PROFILE="$HOME/.bashrc"
    elif [ -f "$HOME/.bash_profile" ]; then
      DETECTED_PROFILE="$HOME/.bash_profile"
    fi
  elif [ "${SHELL#*zsh}" != "$SHELL" ]; then
    if [ -f "${ZDOTDIR:-${HOME}}/.zshrc" ]; then
      DETECTED_PROFILE="${ZDOTDIR:-${HOME}}/.zshrc"
    elif [ -f "${ZDOTDIR:-${HOME}}/.zprofile" ]; then
      DETECTED_PROFILE="${ZDOTDIR:-${HOME}}/.zprofile"
    fi
  fi

  if [ -z "$DETECTED_PROFILE" ]; then
    for EACH_PROFILE in ".profile" ".bashrc" ".bash_profile" ".zprofile" ".zshrc"
    do
      if DETECTED_PROFILE="$(nvm_try_profile "${ZDOTDIR:-${HOME}}/${EACH_PROFILE}")"; then
        break
      fi
    done
  fi

  if [ -n "$DETECTED_PROFILE" ]; then
    nvm_echo "$DETECTED_PROFILE"
  fi
}

#
# Check whether the user has any globally-installed npm modules in their system
# Node, and warn them if so.
#
nvm_check_global_modules() {
  local NPM_COMMAND
  NPM_COMMAND="$(command -v npm 2>/dev/null)" || return 0
  [ -n "${NVM_DIR}" ] && [ -z "${NPM_COMMAND%%"$NVM_DIR"/*}" ] && return 0

  local NPM_VERSION
  NPM_VERSION="$(npm --version)"
  NPM_VERSION="${NPM_VERSION:--1}"
  [ "${NPM_VERSION%%[!-0-9]*}" -gt 0 ] || return 0

  local NPM_GLOBAL_MODULES
  NPM_GLOBAL_MODULES="$(
    npm list -g --depth=0 |
    command sed -e '/ npm@/d' -e '/ (empty)$/d'
  )"

  local MODULE_COUNT
  MODULE_COUNT="$(
    command printf %s\\n "$NPM_GLOBAL_MODULES" |
    command sed -ne '1!p' |                     # Remove the first line
    wc -l | command tr -d ' '                   # Count entries
  )"

  if [ "${MODULE_COUNT}" != '0' ]; then
    # shellcheck disable=SC2016
    nvm_echo '=> You currently have modules installed globally with `npm`. These will no'
    # shellcheck disable=SC2016
    nvm_echo '=> longer be linked to the active version of Node when you install a new node'
    # shellcheck disable=SC2016
    nvm_echo '=> with `nvm`; and they may (depending on how you construct your `$PATH`)'
    # shellcheck disable=SC2016
    nvm_echo '=> override the binaries of modules installed with `nvm`:'
    nvm_echo

    command printf %s\\n "$NPM_GLOBAL_MODULES"
    nvm_echo '=> If you wish to uninstall them at a later point (or re-install them under your'
    # shellcheck disable=SC2016
    nvm_echo '=> `nvm` node installs), you can remove them from the system Node as follows:'
    nvm_echo
    nvm_echo '     $ nvm use system'
    nvm_echo '     $ npm uninstall -g a_module'
    nvm_echo
  fi
}

nvm_do_install() {
  if [ -n "${NVM_DIR-}" ] && ! [ -d "${NVM_DIR}" ]; then
    if [ -e "${NVM_DIR}" ]; then
      nvm_echo >&2 "File \"${NVM_DIR}\" has the same name as installation directory."
      exit 1
    fi

    if [ "${NVM_DIR}" = "$(nvm_default_install_dir)" ]; then
      mkdir "${NVM_DIR}"
    else
      nvm_echo >&2 "You have \$NVM_DIR set to \"${NVM_DIR}\", but that directory does not exist. Check your profile files and environment."
      exit 1
    fi
  fi
  # Disable the optional which check, https://www.shellcheck.net/wiki/SC2230
  # shellcheck disable=SC2230
  if nvm_has xcode-select && [ "$(xcode-select -p >/dev/null 2>/dev/null ; echo $?)" = '2' ] && [ "$(which git)" = '/usr/bin/git' ] && [ "$(which curl)" = '/usr/bin/curl' ]; then
    nvm_echo >&2 'You may be on a Mac, and need to install the Xcode Command Line Developer Tools.'
    # shellcheck disable=SC2016
    nvm_echo >&2 'If so, run `xcode-select --install` and try again. If not, please report this!'
    exit 1
  fi
  if [ -z "${METHOD}" ]; then
    # Autodetect install method
    if nvm_has git; then
      install_nvm_from_git
    elif nvm_has curl || nvm_has wget; then
      install_nvm_as_script
    else
      nvm_echo >&2 'You need git, curl, or wget to install nvm'
      exit 1
    fi
  elif [ "${METHOD}" = 'git' ]; then
    if ! nvm_has git; then
      nvm_echo >&2 "You need git to install nvm"
      exit 1
    fi
    install_nvm_from_git
  elif [ "${METHOD}" = 'script' ]; then
    if ! nvm_has curl && ! nvm_has wget; then
      nvm_echo >&2 "You need curl or wget to install nvm"
      exit 1
    fi
    install_nvm_as_script
  else
    nvm_echo >&2 "The environment variable \$METHOD is set to \"${METHOD}\", which is not recognized as a valid installation method."
    exit 1
  fi

  nvm_echo

  local NVM_PROFILE
  NVM_PROFILE="$(nvm_detect_profile)"
  local PROFILE_INSTALL_DIR
  PROFILE_INSTALL_DIR="$(nvm_install_dir | command sed "s:^$HOME:\$HOME:")"

  SOURCE_STR="\\nexport NVM_DIR=\"${PROFILE_INSTALL_DIR}\"\\n[ -s \"\$NVM_DIR/nvm.sh\" ] && \\. \"\$NVM_DIR/nvm.sh\"  # This loads nvm\\n"

  # shellcheck disable=SC2016
  COMPLETION_STR='[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion\n'
  BASH_OR_ZSH=false

  if [ "${PROFILE-}" = '/dev/null' ] ; then
    # the user has specifically requested NOT to have nvm touch their profile
    echo
  elif [ -z "${NVM_PROFILE-}" ] ; then
    local TRIED_PROFILE
    if [ -n "${PROFILE}" ]; then
      TRIED_PROFILE="${NVM_PROFILE} (as defined in \$PROFILE), "
    fi
    nvm_echo "=> Profile not found. Tried ${TRIED_PROFILE-}~/.bashrc, ~/.bash_profile, ~/.zprofile, ~/.zshrc, and ~/.profile."
    nvm_echo "=> Create one of them and run this script again"
    nvm_echo "   OR"
    nvm_echo "=> Append the following lines to the correct file yourself:"
    command printf "${SOURCE_STR}"
    nvm_echo
  else
    if nvm_profile_is_bash_or_zsh "${NVM_PROFILE-}"; then
      BASH_OR_ZSH=true
    fi
    if ! command grep -qc '/nvm.sh' "$NVM_PROFILE"; then
      nvm_echo "=> Appending nvm source string to $NVM_PROFILE"
      command printf "${SOURCE_STR}" >> "$NVM_PROFILE"
    else
      nvm_echo "=> nvm source string already in ${NVM_PROFILE}"
    fi
    # shellcheck disable=SC2016
    if ${BASH_OR_ZSH} && ! command grep -qc '$NVM_DIR/bash_completion' "$NVM_PROFILE"; then
      nvm_echo "=> Appending bash_completion source string to $NVM_PROFILE"
      command printf "$COMPLETION_STR" >> "$NVM_PROFILE"
    else
      nvm_echo "=> bash_completion source string already in ${NVM_PROFILE}"
    fi
  fi
  if ${BASH_OR_ZSH} && [ -z "${NVM_PROFILE-}" ] ; then
    nvm_echo "=> Please also append the following lines to the if you are using bash/zsh shell:"
    command printf "${COMPLETION_STR}"
  fi

  # Source nvm
  # shellcheck source=/dev/null
  \. "$(nvm_install_dir)/nvm.sh"

  nvm_check_global_modules

  nvm_install_node

  nvm_reset

  nvm_echo "=> Close and reopen your terminal to start using nvm or run the following to use it now:"
  command printf "${SOURCE_STR}"
  if ${BASH_OR_ZSH} ; then
    command printf "${COMPLETION_STR}"
  fi
}

#
# Unsets the various functions defined
# during the execution of the install script
#
nvm_reset() {
  unset -f nvm_has nvm_install_dir nvm_latest_version nvm_profile_is_bash_or_zsh \
    nvm_source nvm_node_version nvm_download install_nvm_from_git nvm_install_node \
    install_nvm_as_script nvm_try_profile nvm_detect_profile nvm_check_global_modules \
    nvm_do_install nvm_reset nvm_default_install_dir nvm_grep
}

[ "_$NVM_ENV" = "_testing" ] || nvm_do_install

} # this ensures the entire script is downloaded #