# DAA Project Scaffold Template

## Standard Directory Structure

```
project-root/
│
├── lib/                              # Framework core (non-test code)
│   │
│   ├── api/                          # API testing domain
│   │   ├── __init__.py
│   │   ├── action_layer.py           # Action Layer: API business actions
│   │   │                             # - Atomic: create_object_and_verify()
│   │   │                             # - Composite: perform_device_upgrade()
│   │   │                             # - Self-verification in every method
│   │   │
│   │   └── physical_layer.py         # Physical Layer: HTTP client wrapper
│   │                                 # - Pure execution: get(), post(), put(), delete()
│   │                                 # - No assertions, no business logic
│   │
│   ├── web/                          # Web/UI testing domain
│   │   ├── __init__.py
│   │   ├── action_layer.py           # Action Layer: UI business actions
│   │   │                             # - navigate_to_home_and_verify_title()
│   │   │                             # - search_and_verify_results_not_empty()
│   │   │
│   │   ├── physical_layer.py         # Physical Layer: browser driver wrapper
│   │   │                             # - Pure execution: click(), fill(), goto()
│   │   │                             # - No assertions, no business logic
│   │   │
│   │   └── constants.py              # UI selectors and URLs
│   │                                 # - Centralized selector management
│   │                                 # - One place to update when UI changes
│   │
│   └── __init__.py
│
├── tests/                            # Test Layer (100% declarative)
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── test_basic_crud.py        # Basic CRUD test scenarios
│   │   └── test_business_workflows.py # Composite workflow scenarios
│   │
│   ├── web/
│   │   ├── __init__.py
│   │   └── test_search_flows.py      # Web UI test scenarios
│   │
│   └── conftest.py                   # Shared fixtures
│                                     # - Physical Layer initialization
│                                     # - Environment configuration
│                                     # - No business logic
│
├── config/                           # Environment configuration (optional)
│   ├── dev.yaml
│   ├── staging.yaml
│   └── production.yaml
│
├── requirements.txt                  # Dependencies
├── pytest.ini                        # Test runner configuration (or equivalent)
└── README.md                         # Project documentation
```

## Layer Placement Rules

| Content Type | Belongs In | Never In |
|-------------|------------|----------|
| Test scenarios | `tests/` | `lib/` |
| Business actions + verification | `lib/*/action_layer.py` | `tests/`, `physical_layer.py` |
| System interaction wrappers | `lib/*/physical_layer.py` | `tests/`, `action_layer.py` |
| UI selectors / URLs | `lib/*/constants.py` | Scattered across any file |
| Test fixtures / setup | `tests/conftest.py` | `lib/` action files |
| Environment config | `config/` | Hardcoded in code |

## Scaling the Structure

### Small project (< 50 tests)

Single action layer file per domain is sufficient:

```
lib/
├── api/
│   ├── action_layer.py      # All API actions in one file
│   └── physical_layer.py
```

### Medium project (50-200 tests)

Split action layer by business domain:

```
lib/
├── api/
│   ├── actions/
│   │   ├── user_actions.py       # User CRUD actions
│   │   ├── order_actions.py      # Order workflow actions
│   │   └── payment_actions.py    # Payment actions
│   └── physical_layer.py
```

### Large project (200+ tests)

Add composite actions directory and shared base classes:

```
lib/
├── api/
│   ├── actions/
│   │   ├── base_action.py        # Shared action utilities
│   │   ├── user_actions.py
│   │   ├── order_actions.py
│   │   └── payment_actions.py
│   ├── composites/
│   │   ├── checkout_flow.py      # Multi-step business workflows
│   │   └── onboarding_flow.py
│   └── physical_layer.py
```

## Fixture / Configuration Patterns

### Fixture Responsibilities

Fixtures handle Physical Layer initialization and environment setup. They must NOT contain business logic.

```
// conftest.py — GOOD
fixture api_client():
    """Provides an initialized API client."""
    client = APIClient(base_url=config.API_URL)
    return client

fixture web_driver(playwright_page):
    """Provides an initialized browser driver."""
    driver = PlaywrightDriver(playwright_page)
    return driver
```

### Base Test Class Pattern

Use a base class to wire fixtures into the Action Layer:

```
// action_layer.py
BaseAPITest:
    fixture setup(api_client):
        self.api_client = api_client

    // Actions can now use self.api_client
    create_object_and_verify(url, name, data):
        response = self.api_client.post(url, {name, data})
        assert response.status == 201
        return response.body
```

```
// test_crud.py — Test Layer inherits from Action Layer
TestCRUD(BaseAPITest):
    URL = "https://api.example.com/objects"

    test_create_and_retrieve():
        obj = self.create_object_and_verify(URL, "Widget", {"color": "red"})
        self.get_object_and_verify(URL, obj.id, expected_name="Widget")
```

## Multi-Domain Projects

For projects testing multiple interfaces (API + Web + Mobile):

```
lib/
├── api/                    # API domain
│   ├── action_layer.py
│   └── physical_layer.py
├── web/                    # Web domain
│   ├── action_layer.py
│   ├── physical_layer.py
│   └── constants.py
├── mobile/                 # Mobile domain
│   ├── action_layer.py
│   ├── physical_layer.py
│   └── constants.py
└── shared/                 # Cross-domain utilities
    └── data_helpers.py     # Data generation, format conversion (no system calls)
```

Each domain has its own complete Action + Physical stack. The Test Layer can compose across domains if needed.
