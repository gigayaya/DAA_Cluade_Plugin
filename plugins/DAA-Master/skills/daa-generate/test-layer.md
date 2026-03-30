# Generating Test Layer Code

## Purpose

The Test Layer defines test scenarios in a **100% declarative** format. It serves as "Living Documentation" — readable by anyone, including non-technical stakeholders.

## Structure Pattern

```
TestClass(inherits ActionLayer):

    CONSTANTS:
        BASE_URL = "..."
        TEST_DATA = {...}

    test_scenario_description():
        // Setup (using Action methods only)
        action_setup_step()

        // Execute (using Action methods only)
        action_primary_step()

        // NO assertions here — they live in Action Layer
```

## Rules

### R1: No Logic — Absolute Zero

The Test Layer must contain NO executable logic:

| Forbidden | Why |
|-----------|-----|
| `if / else` | Conditional paths belong in Action Layer |
| `for / while` | Loops belong in Action Layer |
| `try / catch` | Error handling belongs in Action Layer |
| `assert` | Verification belongs in Action Layer |
| Raw API calls | System interaction belongs in Physical Layer |
| Variable manipulation | Data processing belongs in Action Layer |

**Exception**: Assigning return values from Action methods to variables for passing to subsequent Action methods is acceptable.

```
// Acceptable — passing data between actions
old_phone = create_object_and_verify(url, name, data)
old_id = get_object_id(old_phone)
perform_device_upgrade(url, old_id, "iPhone 15")
```

### R2: Test Method Naming

Test methods describe the **scenario**, not the implementation:

```
// GOOD — describes the scenario
test_upgrade_old_phone_to_new_phone()
test_search_for_existing_product()
test_proceed_to_checkout_with_empty_cart()
test_filter_results_by_brand()

// BAD — describes implementation details
test_post_and_delete_api()
test_fill_form_click_submit()
test_http_200_response()
```

### R3: Inheritance / Composition

The Test class must access Action Layer methods through inheritance or composition:

```
// Inheritance approach (simpler)
TestCheckout(inherits CheckoutActionLayer):
    test_guest_checkout():
        self.navigate_to_home_and_verify_title()
        self.search_for_product_and_verify_results("MacBook")

// Composition approach (more flexible)
TestCheckout:
    actions = CheckoutActionLayer()

    test_guest_checkout():
        actions.navigate_to_home_and_verify_title()
        actions.search_for_product_and_verify_results("MacBook")
```

### R4: Test Data

Test data should be defined as class-level constants or fixtures, not inline magic values:

```
// GOOD — named constants
TestDeviceUpgrade(BaseAPITest):
    DEMO_URL = "https://api.example.com/objects"
    OLD_PHONE_NAME = "iPhone 12"
    OLD_PHONE_DATA = {"color": "Blue", "storage": "64GB"}
    NEW_PHONE_NAME = "iPhone 15"

    test_upgrade():
        old = create_object_and_verify(DEMO_URL, OLD_PHONE_NAME, OLD_PHONE_DATA)
        perform_device_upgrade(DEMO_URL, get_object_id(old), NEW_PHONE_NAME)

// BAD — magic values inline
test_upgrade():
    old = create_object_and_verify("https://api.example.com/objects", "iPhone 12", {"color": "Blue"})
```

## Complete Example

```
TestAmazonSearch(inherits AmazonActionLayer):
    """Test Layer: E2E tests for product search functionality."""

    test_search_for_existing_product():
        navigate_to_home_and_verify_title()
        search_for_product_and_verify_result_list_not_empty("iPhone 15")

    test_search_for_non_existent_product():
        navigate_to_home_and_verify_title()
        search_for_product_and_expect_no_results("xyz_nonexistent_random_123")

    test_search_with_special_characters():
        navigate_to_home_and_verify_title()
        search_for_product_and_verify_handling("@#$%^&*")

    test_product_detail_consistency():
        navigate_to_home_and_verify_title()
        search_for_product_and_verify_result_list_not_empty("Apple MacBook Air")
        title, price = get_first_result_title_and_price()
        click_first_result_and_verify_detail_page_matches(title, price)
```

Notice: every test is a short sequence of action calls. No logic. No assertions. Pure intent.
