# DAA Scaling Patterns

Advanced design patterns for managing complexity in the Action Layer as your test suite grows.

## Pattern 1: Composite Action (Core DAA)

**When to use**: The same sequence of 3+ Atomic Actions appears in multiple tests.

**Problem**: Repetition creates maintenance burden. A business process change requires updating N tests.

**Solution**: Encapsulate the repeated sequence into a Composite Action (Level 2).

```
// BEFORE — same sequence in 20 tests
test_upgrade_flow():
    old = get_object_and_verify(url, old_id)
    data = get_object_data(old)
    new = create_object_and_verify(url, "iPhone 15", data)
    delete_object_and_verify(url, old_id)
    get_object_and_expect_not_found(url, old_id)
    assert new.data == data

// AFTER — one Composite Action, used in 20 tests
test_upgrade_flow():
    old = create_object_and_verify(url, "iPhone 12", old_data)
    perform_device_upgrade(url, get_object_id(old), "iPhone 15")
```

**DAA Integration**: The Composite Action lives in the Action Layer. It calls only Atomic Actions. It includes its own business-level verification. Tests call it as a single step.

---

## Pattern 2: Builder Pattern

**When to use**: An Action method has too many parameters, making calls hard to read and maintain.

**Problem**: Methods with 8+ parameters are error-prone and hard to understand.

```
// PROBLEM — too many parameters
create_user_and_verify(
    url, first_name, last_name, email, phone,
    role, department, manager_id, permissions, avatar_url
)
```

**Solution**: Use a Builder to construct the complex input incrementally.

```
// Builder Pattern
UserBuilder:
    __init__():
        self.data = {}

    with_name(first, last):
        self.data["first_name"] = first
        self.data["last_name"] = last
        return self

    with_email(email):
        self.data["email"] = email
        return self

    with_role(role, department):
        self.data["role"] = role
        self.data["department"] = department
        return self

    build():
        return self.data

// Usage in Action Layer
create_user_and_verify(url, user_data):
    response = physical.post(url, user_data)
    assert response.status == 201
    return response.body

// Usage in Test Layer
test_create_admin_user():
    user_data = UserBuilder()
        .with_name("Jane", "Doe")
        .with_email("jane@example.com")
        .with_role("admin", "engineering")
        .build()
    create_user_and_verify(URL, user_data)
```

**DAA Integration**: The Builder is a data construction helper — it lives alongside the Action Layer but doesn't perform system interaction. The Action method still delegates to Physical Layer and self-verifies.

---

## Pattern 3: Adapter Pattern

**When to use**: Your Action Layer needs to work with an API that doesn't match your expected interface.

**Problem**: A third-party API returns data in a different format than your actions expect, or two different systems use different conventions for the same concept.

```
// PROBLEM — API returns flat structure but your actions expect nested
// External API: {"user_name": "Jane", "user_email": "jane@ex.com"}
// Your actions expect: {"name": "Jane", "email": "jane@ex.com"}
```

**Solution**: Create an Adapter in the Physical Layer (or between Physical and Action) that normalizes the interface.

```
// Adapter — normalizes external API response
ExternalAPIAdapter:
    raw_client = ExternalAPIClient()

    get_user(user_id):
        raw = raw_client.get_user(user_id)
        return {
            "name": raw["user_name"],
            "email": raw["user_email"],
            "id": raw["user_identifier"]
        }
```

**DAA Integration**: The Adapter lives in or next to the Physical Layer. The Action Layer calls the Adapter as if it were the Physical Layer — it doesn't know about the format mismatch. Self-verification still happens in the Action Layer.

---

## Pattern 4: Strategy Pattern

**When to use**: An Action method has `if/else` branches selecting between different execution paths based on context.

**Problem**: Conditional logic in the Action Layer makes actions hard to understand and test.

```
// PROBLEM — branching logic in action
submit_payment_and_verify(payment_type, amount):
    if payment_type == "credit_card":
        fill(CARD_NUMBER, card)
        fill(EXPIRY, expiry)
        click(PAY_BUTTON)
        assert is_visible(CC_SUCCESS)
    elif payment_type == "paypal":
        click(PAYPAL_BUTTON)
        // ... different flow
    elif payment_type == "bank_transfer":
        // ... yet another flow
```

**Solution**: Extract each branch into its own strategy (action method), and let the caller choose.

```
// Strategy — separate action per payment type
submit_credit_card_payment_and_verify(card, expiry, amount):
    fill(CARD_NUMBER, card)
    fill(EXPIRY, expiry)
    fill(AMOUNT, amount)
    click(PAY_BUTTON)
    assert is_visible(CC_SUCCESS_MESSAGE)

submit_paypal_payment_and_verify(email, amount):
    click(PAYPAL_BUTTON)
    fill(PAYPAL_EMAIL, email)
    click(PAYPAL_CONFIRM)
    assert is_visible(PAYPAL_SUCCESS)

submit_bank_transfer_and_verify(account, routing, amount):
    // ...
```

```
// Test Layer — caller selects the strategy
test_checkout_with_credit_card():
    // ...
    submit_credit_card_payment_and_verify("4111...", "12/25", 99.99)

test_checkout_with_paypal():
    // ...
    submit_paypal_payment_and_verify("user@example.com", 99.99)
```

**DAA Integration**: Each strategy is an Atomic Action with its own self-verification. The Test Layer remains declarative — it calls the specific action it needs, no `if/else` required.

---

## Pattern 5: Chain Pattern

**When to use**: A business process is a strict sequence of steps where each step depends on the previous one's output (pipeline).

**Problem**: The pipeline logic is complex and the steps are tightly ordered.

```
// Pipeline: provision → configure → activate → verify
```

**Solution**: Each step is an Atomic Action. A Composite Action chains them, passing output from one to the next.

```
// Chain — each step feeds the next
provision_and_activate_service(service_config):
    // Step 1: Provision
    instance = provision_service_and_verify(service_config)

    // Step 2: Configure (uses provision output)
    configured = configure_service_and_verify(
        instance.id, service_config.settings
    )

    // Step 3: Activate (uses configure output)
    activate_service_and_verify(configured.id)

    // Step 4: End-to-end verification
    verify_service_is_running(configured.id)

    return configured
```

**DAA Integration**: The Chain is a Composite Action. Each link in the chain is an Atomic Action with self-verification. The Composite adds end-to-end verification. The Test Layer calls the chain as one step.

---

## Choosing the Right Pattern

| Symptom | Pattern | Apply When |
|---------|---------|------------|
| Same action sequence repeated across tests | **Composite** | 3+ tests share the same sequence |
| Action method has 6+ parameters | **Builder** | Parameters represent a complex domain object |
| External API format doesn't match internal expectations | **Adapter** | Physical Layer needs format normalization |
| Action method has `if/else` branches | **Strategy** | Different execution paths based on type/mode |
| Steps must execute in strict order, each depending on previous output | **Chain** | Pipeline or provisioning workflows |

These patterns are **not mutually exclusive**. A large framework might use:
- Builder to construct complex test data
- Strategy to handle different payment methods
- Chain to model the end-to-end flow
- Composite to wrap the chain for reuse across tests
