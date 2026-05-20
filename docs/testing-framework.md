# Testing Framework Documentation

This document describes the testing framework, test structure, and conventions used in libzmq.

## Overview

libzmq uses [Unity](https://github.com/ThrowTheSwitch/Unity), a lightweight C unit testing framework. The Unity source is embedded directly in the repository under `external/unity/`. Two compile-time definitions are applied:

- `UNITY_USE_COMMAND_LINE_ARGS` — enables command-line argument support for test filtering.
- `UNITY_EXCLUDE_FLOAT` — excludes floating-point assertion macros (not needed by libzmq).

## Test Architecture

Tests are organized into two layers:

### Integration Tests (`tests/`)

These tests exercise libzmq through its **public C API** (`zmq.h`), simulating real application usage. Each `test_*.cpp` file compiles into a standalone executable.

There are approximately 116 test files covering:

- **Transport protocols**: TCP, IPC, TIPC, VMCI, WebSocket (`test_pair_tcp`, `test_pair_ipc`, `test_ws_transport`, etc.)
- **Socket patterns**: REQ/REP, PUB/SUB, PUSH/PULL, DEALER/ROUTER, PAIR, CLIENT/SERVER, RADIO/DISH, SCATTER/GATHER, PEER, etc.
- **Security mechanisms**: CURVE, PLAIN, NULL, ZAP, GSSAPI (`test_security_curve`, `test_security_plain`, etc.)
- **Socket options and context options**: `test_setsockopt`, `test_sockopt_hwm`, `test_ctx_options`
- **Proxy functionality**: `test_proxy`, `test_proxy_hwm`, `test_proxy_terminate`
- **Connection management**: `test_reconnect_ivl`, `test_connect_rid`, `test_disconnect_inproc`, `test_reconnect_options`
- **Message handling**: `test_msg_flags`, `test_msg_ffn`, `test_msg_init`

Some tests are conditionally compiled based on build configuration:

| Condition | Tests enabled |
|-----------|---------------|
| `ZMQ_HAVE_CURVE` | `test_security_curve` |
| `ENABLE_DRAFTS` | `test_poller`, `test_client_server`, `test_radio_dish`, `test_peer`, etc. |
| `ZMQ_HAVE_IPC` | `test_ipc_wildcard`, `test_pair_ipc`, `test_reqrep_ipc`, `test_rebind_ipc` |
| `ZMQ_HAVE_TIPC` (Linux) | `test_pair_tipc`, `test_reqrep_tipc`, `test_address_tipc`, etc. |
| `ZMQ_HAVE_WS` | `test_ws_transport` |
| `ZMQ_HAVE_WSS` | `test_wss_transport` |
| `WITH_VMCI` | `test_pair_vmci`, `test_reqrep_vmci` |
| `HAVE_FORK` (non-Windows) | `test_fork` |
| `ENABLE_CAPSH` | `test_pair_tcp_cap_net_admin` |

### Unit Tests (`unittests/`)

These tests directly exercise **internal C++ implementation** classes by including private headers (e.g., `<poller.hpp>`, `<ip.hpp>`). They are statically linked against `libzmq-static` and are only built when `BUILD_STATIC` is enabled.

There are 7 unit test files:

| Test file | Target |
|-----------|--------|
| `unittest_ypipe.cpp` | Internal ypipe data structure |
| `unittest_poller.cpp` | Internal poller implementation |
| `unittest_mtrie.cpp` | Subscription matching trie |
| `unittest_ip_resolver.cpp` | IP address resolution |
| `unittest_udp_address.cpp` | UDP address parsing |
| `unittest_radix_tree.cpp` | Radix tree data structure |
| `unittest_curve_encoding.cpp` | CURVE encryption encoding |

## Test Utility Library

A static library `libtestutil.a` (or `testutil`/`testutil-static` in CMake) provides shared helper functions across all tests. It consists of the following modules:

### `testutil.hpp` / `testutil.cpp`

Core utilities:

- `bounce(server, client)` — sends a message from client to server and back, for REQ/REP or DEALER/DEALER pairs.
- `expect_bounce_fail(server, client)` — expects message delivery to fail (e.g., security rejection).
- `s_send_seq(socket, ...)` / `s_recv_seq(socket, ...)` — sends/receives a sequence of message frames, terminated by `SEQ_END`.
- `s_recv(socket)` — receives a ZMQ string and converts to a C string.
- `setup_test_environment(timeout)` — initializes the test environment; on POSIX systems, sets an alarm to kill tests that exceed the timeout (default 60 seconds).
- `msleep(milliseconds)` — portable millisecond sleep.
- `is_ipv6_available()` / `is_tipc_available()` — runtime capability checks.

Constants:

- `SETTLE_TIME` (300 ms) — standard delay for socket settle time in tests.
- `MAX_SOCKET_STRING` (256) — buffer size for endpoint strings.
- `ENDPOINT_0` through `ENDPOINT_5` — fixed test endpoints for non-random port testing.

### `testutil_unity.hpp` / `testutil_unity.cpp`

Unity integration layer providing ZMQ-specific assertion macros and test lifecycle management.

**Custom assertion macros:**

- `TEST_ASSERT_SUCCESS_ERRNO(expr)` — asserts a ZMQ API call succeeds (returns >= 0). On failure, prints the expression, `zmq_errno()`, and `zmq_strerror()`.
- `TEST_ASSERT_SUCCESS_MESSAGE_ERRNO(expr, msg)` — same as above with an additional context message.
- `TEST_ASSERT_FAILURE_ERRNO(error_code, expr)` — asserts a ZMQ API call fails with the expected errno.
- `TEST_ASSERT_SUCCESS_RAW_ERRNO(expr)` — for native socket API calls (uses `WSAGetLastError` on Windows, `errno` otherwise).
- `TEST_ASSERT_FAILURE_RAW_ERRNO(error_code, expr)` — asserts native socket API failure with expected error.

**Message send/receive helpers:**

- `send_string_expect_success(socket, str, flags)`
- `recv_string_expect_success(socket, str, flags)`
- `send_array_expect_success(socket, array, flags)` (template)
- `recv_array_expect_success(socket, array, flags)` (template)

**Context lifecycle management:**

- `SETUP_TEARDOWN_TESTCONTEXT` — macro that defines `setUp()` and `tearDown()` functions for automatic context creation and destruction per test case.
- `setup_test_context()` / `teardown_test_context()` — create and destroy the test ZMQ context.
- `get_test_context()` — returns the current test context.
- `test_context_socket(type)` — creates a socket on the test context and registers it for lifecycle tracking.
- `test_context_socket_close(socket)` / `test_context_socket_close_zero_linger(socket)` — closes and unregisters a socket.

If any sockets remain open when `tearDown` runs, they are forcibly closed with `linger=0` and a warning is printed.

**Bind helpers:**

- `test_bind(socket, address, endpoint_buf, len)` — binds and retrieves the bound endpoint.
- `bind_loopback_ipv4(socket, endpoint_buf, len)` — binds to `tcp://127.0.0.1:*`.
- `bind_loopback_ipv6(socket, endpoint_buf, len)` — binds to `tcp://[::1]:*` (skips if IPv6 unavailable).
- `bind_loopback_ipc(socket, endpoint_buf, len)` — binds to `ipc://*`.
- `bind_loopback_tipc(socket, endpoint_buf, len)` — binds to `tipc://<*>`.

### `testutil_monitoring.hpp` / `testutil_monitoring.cpp`

Socket monitor event utilities:

- `get_monitor_event(monitor, value, address)` — reads one event from a monitor socket.
- `get_monitor_event_with_timeout(monitor, value, address, timeout)` — same with timeout.
- `expect_monitor_event(monitor, expected_event)` — asserts the next event matches.
- `expect_monitor_event_multiple(monitor, expected_event, expected_err, optional)` — waits for one or more occurrences of an event.
- `get_monitor_event_v2()` / `expect_monitor_event_v2()` — v2 monitor event variants with local/remote addresses.

### `testutil_security.hpp` / `testutil_security.cpp`

Security testing utilities:

- `socket_config_null_client/server()` — configure NULL security.
- `socket_config_plain_client/server()` — configure PLAIN security.
- `socket_config_curve_client/server()` — configure CURVE security with pre-generated test keys.
- `setup_testutil_security_curve()` — generates random CURVE key pairs for testing.
- `zap_handler()` / `zap_handler_generic()` — implements a ZAP authentication handler that can simulate various protocol conditions (success, failure, protocol errors, etc.).

## Writing a Test

Each test file follows this structure:

```c
#include "testutil.hpp"
#include "testutil_unity.hpp"

// Automatic context setup/teardown for each test case
SETUP_TEARDOWN_TESTCONTEXT

void test_example ()
{
    void *socket = test_context_socket (ZMQ_PAIR);

    char endpoint[MAX_SOCKET_STRING];
    bind_loopback_ipv4 (socket, endpoint, sizeof endpoint);

    // ... test logic using TEST_ASSERT_SUCCESS_ERRNO, etc.

    test_context_socket_close (socket);
}

int main ()
{
    setup_test_environment ();

    UNITY_BEGIN ();
    RUN_TEST (test_example);
    return UNITY_END ();
}
```

Key conventions (also see `tests/README.md`):

- Use ANSI C99 style in test cases.
- Use `test_context_socket()` / `test_context_socket_close()` instead of raw `zmq_socket()` / `zmq_close()`.
- Use `msleep(SETTLE_TIME)` when a delay is needed for socket settling.
- Do not include internal headers (`src/`) in integration tests; use `unittests/` for internal testing.
- Wrap non-portable code in `#ifdef` guards.

## Build System Integration

Tests are integrated with both supported build systems.

### CMake

Controlled by `BUILD_TESTS` option (default `ON`):

```
cmake -DBUILD_TESTS=ON ..
make
ctest
```

- `tests/CMakeLists.txt` registers each integration test via `add_test()`.
- `unittests/CMakeLists.txt` registers unit tests (requires `BUILD_STATIC=ON`).
- Default test timeout: 10 seconds. Security tests: 60 seconds.
- A self-check at the end of `CMakeLists.txt` scans for `test_*.cpp` files not registered in CTest and emits a warning.

### Autotools

```
./autogen.sh
./configure
make check
```

- Tests are listed in `Makefile.am` as `check_PROGRAMS` and `TESTS`.
- Each test links against `libtestutil.a`, `libunity.a`, and `libzmq.la`.
- `make distcheck` runs the full test suite as part of distribution validation.

### CI

The `ci_build.sh` script runs `make distcheck` for default builds, which includes the full test suite. Various CI configurations test different combinations of options (CURVE, DRAFT APIs, sanitizers, etc.).

## Test Timeout Configuration

| Test | Timeout |
|------|---------|
| Default | 10 seconds |
| `test_security_curve` | 60 seconds |
| `test_heartbeats` | 60 seconds |
| `test_security_zap` | 60 seconds |
| `test_reconnect_ivl` | 15 seconds |
| `test_radio_dish` (Windows, DRAFTS) | 30 seconds |

Tests return exit code 77 to indicate a skip (e.g., when a required transport is unavailable). This is handled via `SKIP_RETURN_CODE 77` in CTest.
