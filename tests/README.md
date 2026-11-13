# FHIR MCP Server Test Suite

This directory contains the test suite for the FHIR MCP server. Unit and
integration tests mock HTTP. End-to-end tests start the MCP server in a
subprocess and talk to it over MCP, either against a local stub FHIR server or
against the public [HAPI FHIR](https://hapi.fhir.org/) test server.

**The default live FHIR server for tests is public HAPI FHIR**
(`https://hapi.fhir.org/baseR4`). An optional acceptance test can also run
against Aidbox; see [Optional Aidbox acceptance test](#optional-aidbox-acceptance-test).

## Test Structure

```
tests/
├── unit/                                 # Offline unit tests
│   ├── oauth/                            # OAuth types, client provider, common functions
│   ├── field_filter/                     # FHIRPath response filtering
│   ├── test_utils.py                     # utils module
│   ├── test_interaction_validation.py    # Bundle and JSON Patch input validation
│   └── test_e2e_environment.py           # Test-server environment isolation
├── integration/                          # Offline integration tests
│   ├── test_integration.py               # Server configuration and OAuth wiring
│   ├── test_fhir_http.py                 # HTTP transport for execute_bundle/patch
│   └── test_interaction_handlers.py      # execute_bundle/patch handlers
├── e2e/                                  # MCP server in a subprocess
│   ├── conftest.py                       # Server launcher, local stub FHIR server
│   ├── test_tools.py                     # test_tool_*: live HAPI; others: local stub
│   ├── test_bundle_patch.py              # execute_bundle/patch against the local stub
│   └── test_credential_isolation.py      # No credentials leak to test servers
├── aidbox/
│   └── test_bundle_patch.py              # Optional live Aidbox acceptance (skipped by default)
└── test_utils.py
```

## Running Tests

`python run_tests.py` runs everything under `tests/`, as CI does. That includes the
live `test_tool_*` tests in `tests/e2e/test_tools.py`, which need internet access and
create, update and delete test Patient resources on public HAPI. To run without
contacting HAPI:

```bash
python -m pytest tests/ -k "not test_tool_"
```

The e2e launcher sets access-token and OAuth credential settings to empty values,
so neither inherited environment variables nor the project `.env` can
authenticate requests to test servers. Both the HAPI fixture and the local stub
fixture use this isolation. To check it without contacting a FHIR server:

```bash
python -m pytest tests/unit/test_e2e_environment.py tests/e2e/test_credential_isolation.py -q
```

### Option 1: Using the test runner script (Recommended)
```bash
python run_tests.py
```

### Option 2: Using pytest directly
```bash
# Set the Python path and run tests
PYTHONPATH=src python -m pytest tests/ -v --cov=src/fhir_mcp_server --cov-report=term-missing --cov-report=html:htmlcov
```

### Option 3: Running specific test files
```bash
# Run only unit tests
PYTHONPATH=src python -m pytest tests/unit/ -v

# Run only integration tests
PYTHONPATH=src python -m pytest tests/integration/ -v

# Run specific test file
PYTHONPATH=src python -m pytest tests/unit/test_utils.py -v
```

## Test Coverage

The test suite achieves the following coverage:

- **utils.py**: 100% coverage (41/41 statements)
- **oauth/types.py**: 99% coverage (79/80 statements)
- **oauth/common.py**: 100% coverage (62/62 statements)
- **oauth/client_provider.py**: 99% coverage (111/112 statements)
- **Overall coverage**: 53% (335/635 statements)

*Note: The server.py module is not tested as it contains the main application logic that would require a full server setup. The OAuth server provider is partially tested due to its complexity.*

## Test Categories

### Unit Tests (118 tests)

#### Utils Module Tests (21 tests)
- Tests for FHIR client creation with various configurations
- Bundle entry extraction and processing
- Resource trimming functionality
- Operation outcome error generation
- Capability statement discovery
- Default headers generation

#### OAuth Types Tests (34 tests)
- Configuration classes validation
- OAuth metadata handling
- Token management and scope validation
- Authorization code handling
- URL generation and validation

#### OAuth Client Provider Tests (30 tests)
- OAuth flow execution with PKCE
- Token validation and refresh
- HTTP request mocking
- Error handling and edge cases
- Callback handling
- Scope validation

#### OAuth Common Functions Tests (33 tests)
- OAuth metadata discovery
- Token expiration checking
- Endpoint URL extraction
- Code verifier/challenge generation
- Token flow execution
- Authentication response handling

### Integration Tests (5 tests)
- Server configuration integration
- Provider initialization
- OAuth flow coordination
- URL generation consistency
- Cross-component communication

## Test Features

### Mocking
- Unit and integration tests mock all external HTTP requests
- Local end-to-end tests use a stub FHIR server on localhost
- Only the live `test_tool_*` tests (public HAPI) and the optional Aidbox test use a real FHIR server

### Async Testing Support
- Full support for async/await patterns
- Proper async test fixtures
- Mock async functions and coroutines

### Edge Case Coverage
- Error conditions and exception handling
- Invalid input validation
- Network failure scenarios
- Configuration edge cases

### Fixtures and Utilities
- Reusable test fixtures
- Mock data generators
- Common test utilities

## Dependencies

Test dependencies are installed via:
```bash
pip install pytest pytest-asyncio pytest-mock pytest-cov
```

## Configuration

Test configuration is managed through `pytest.ini`:
- Test discovery patterns
- Coverage settings
- Async mode configuration
- Test markers

## Best Practices

1. **Isolation**: Each test is isolated and doesn't depend on others
2. **Mocking**: External dependencies are properly mocked
3. **Coverage**: High test coverage with meaningful assertions
4. **Documentation**: Each test is well-documented with clear purpose
5. **Performance**: Tests run quickly with minimal overhead

## Contributing

When adding new tests:
1. Follow the existing naming conventions
2. Add appropriate docstrings
3. Mock external dependencies
4. Test both success and failure scenarios
5. Update this README if adding new test categories

## Bundle and PATCH tests

`execute_bundle` and `patch` are covered by the default suite without a live
FHIR server:

- `tests/unit/test_interaction_validation.py`: Bundle and JSON Patch validation.
- `tests/integration/test_fhir_http.py` and `test_interaction_handlers.py`: HTTP
  transport, limits, timeouts and result classification.
- `tests/e2e/test_bundle_patch.py`: the tools called over MCP against a local stub
  FHIR server that records each request and returns scripted responses.

```bash
python -m pytest tests/unit tests/integration tests/test_utils.py tests/e2e/test_bundle_patch.py tests/e2e/test_credential_isolation.py -q
```

The stub sees only dummy credentials. Tests that exercise writes enable
`FHIR_MCP_ENABLE_WRITE_TOOLS` explicitly for their own server process.

### Optional Aidbox acceptance test

[Aidbox](https://www.health-samurai.io/aidbox) is a FHIR server made by Health
Samurai. It is commercial software with a free development license, and it runs
locally in Docker. `tests/aidbox/test_bundle_patch.py` uses it because Aidbox lets
you give a client access policies that allow some resource types and deny
others. That makes it possible to check behavior the public HAPI server cannot
show, such as a transaction rolling back because one of its entries is forbidden,
or a batch reporting a mix of successes and 403s.

The test is skipped unless you set these environment variables:

- `FHIR_TEST_AIDBOX_ENABLE=true`
- `FHIR_TEST_AIDBOX_URL`: the Aidbox FHIR endpoint, ending in `/fhir`.
- `FHIR_TEST_AIDBOX_TOKEN`: an access token for a restricted Aidbox client. Never
  commit it.

Use a disposable Aidbox instance. The token's client must be allowed Patient and
Observation create/read/update/delete, FHIRPath PATCH and Bundle execution, and
must be denied Condition writes and administration of Client, AccessPolicy and
User. The test checks those denials before it writes anything, uses unique
synthetic IDs and deletes its resources afterwards. Their version history may
remain in Aidbox.

```bash
python -m pytest tests/aidbox -q --no-cov --tb=short -s
```

The test runs its own MCP server on port 8001. It checks linked transactions,
transaction rollback, batch partial success, conditional create, current and
stale ETags, failing JSON Patch `test` operations, FHIRPath Parameters patches
inside Bundles and `Prefer: return` handling. Run it on its own, not in parallel
with other end-to-end tests, which also use port 8001.
