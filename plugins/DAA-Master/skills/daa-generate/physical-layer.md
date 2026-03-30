# Generating Physical Layer Code

## Purpose

The Physical Layer is a **thin, "dumb" wrapper** around the underlying system interaction library (HTTP client, WebDriver, Playwright, database driver, etc.). It knows HOW to talk to the system but knows NOTHING about business logic or expected outcomes.

## Structure Pattern

```
PhysicalLayerClass:
    driver = underlying_library_instance

    operation(params):
        return driver.operation(params)
```

## Rules

### R1: Pure Execution — Zero Intelligence

The Physical Layer must contain:
- **NO assertions** (`assert`, `expect`, `should`, etc.)
- **NO conditionals** (`if`, `switch`, ternary)
- **NO loops** (`for`, `while`)
- **NO error handling** (`try/catch`) — let errors propagate to Action Layer
- **NO business logic** (no data transformation, no decision-making)
- **NO retry logic** — retries are a business decision for the Action Layer

### R2: Thin Wrapping

Each method should be a direct, 1-2 line delegation to the underlying library:

```
// API Physical Layer
APIClient:
    http = requests_library

    get(url, params=None, headers=None):
        return http.get(url, params=params, headers=headers)

    post(url, data=None, headers=None):
        return http.post(url, json=data, headers=headers)

    put(url, data=None, headers=None):
        return http.put(url, json=data, headers=headers)

    delete(url, headers=None):
        return http.delete(url, headers=headers)
```

```
// Web Physical Layer (Playwright/Selenium)
WebDriver:
    page = browser_page_instance

    goto(url):
        page.goto(url)

    fill(selector, text):
        page.fill(selector, text)

    click(selector):
        page.click(selector)

    get_text(selector):
        return page.text_content(selector)

    get_title():
        return page.title()

    is_visible(selector):
        return page.is_visible(selector)

    get_count(selector):
        return page.locator(selector).count()

    wait_for_selector(selector):
        page.wait_for_selector(selector)

    get_attribute(selector, attribute):
        return page.get_attribute(selector, attribute)
```

### R3: One Method Per Primitive

Each Physical Layer method wraps exactly one primitive operation. Don't combine multiple operations:

```
// BAD — multiple operations in one method
fill_and_submit(selector, text, submit_button):
    page.fill(selector, text)
    page.click(submit_button)    // This is a second operation

// GOOD — one operation per method
fill(selector, text):
    page.fill(selector, text)

click(selector):
    page.click(selector)
```

### R4: Technology Swappability

The Physical Layer is the ONLY place where the underlying technology is referenced. This means:

- Switching from Selenium to Playwright? Only Physical Layer changes.
- Switching from `requests` to `httpx`? Only Physical Layer changes.
- Action Layer and Test Layer remain untouched.

Design the Physical Layer interface to be technology-agnostic:

```
// Both implementations expose the same interface
// Action Layer doesn't know (or care) which one is used

SeleniumDriver:
    fill(selector, text):
        self.driver.find_element(By.CSS_SELECTOR, selector).send_keys(text)

PlaywrightDriver:
    fill(selector, text):
        self.page.fill(selector, text)
```

## Constants / Selectors

UI selectors and URLs should be extracted into a constants file, separate from the Physical Layer:

```
// Constants file — centralized selector management
Selectors:
    BASE_URL = "https://example.com"
    SEARCH_INPUT = "#twotabsearchtextbox"
    SEARCH_BUTTON = "#nav-search-submit-button"
    RESULT_ITEM = "[data-component-type='s-search-result']"
    ADD_TO_CART = "#add-to-cart-button"
    CART_COUNT = "#nav-cart-count"
```

This ensures that when a UI element's selector changes, you update ONE file — not scattered references across the Physical Layer.

## Complete Example

```
// Physical Layer — API
APIClient:
    """Pure execution wrapper for HTTP operations."""

    __init__():
        self.http = requests_library

    get(url, params=None, headers=None):
        return self.http.get(url, params=params, headers=headers)

    post(url, data=None, headers=None):
        return self.http.post(url, json=data, headers=headers)

    delete(url, headers=None):
        return self.http.delete(url, headers=headers)


// Physical Layer — Web (Playwright)
PlaywrightDriver:
    """Pure execution wrapper for browser operations."""

    __init__(page):
        self.page = page

    goto(url):
        self.page.goto(url)

    fill(selector, text):
        self.page.fill(selector, text)

    click(selector):
        self.page.click(selector)

    get_text(selector):
        return self.page.text_content(selector)

    get_title():
        return self.page.title()

    is_visible(selector):
        return self.page.is_visible(selector)

    get_count(selector):
        return self.page.locator(selector).count()

    wait_for_selector(selector):
        self.page.wait_for_selector(selector)
```

Notice: every method is a 1-line delegation. No logic, no assertions, no decisions.
