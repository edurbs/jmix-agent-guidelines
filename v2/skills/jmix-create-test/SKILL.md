---
name: jmix-create-test
description: Create or update Jmix unit, integration, UI integration, or end-to-end tests for services, entity listeners, security behavior, views, fragments, and persistence workflows.
---

# Create Test

Use this skill when adding or changing tests for Jmix application behavior.

## Steps

1. Choose the smallest test type that proves the behavior.
2. Use a plain JUnit test for pure Java logic without Spring/Jmix services.
3. Use `@SpringBootTest` for services, DataManager persistence, entity listeners, security, and transactions.
4. Use `@UiTest` with `FlowuiTestAssistConfiguration` for Flow UI controller/component behavior without a browser.
5. Use end-to-end browser tests only for real browser behavior, routing, login, theme, or Vaadin client-side interactions.
6. Create test data through `DataManager.create()` and `DataManager.save()`.
7. Set authentication with the project's `AuthenticatedAsAdmin` extension or `SystemAuthenticator`.
8. Clean up created persistent data in `@AfterEach`.
9. Mock external systems at the boundary; prefer `@MockitoBean` on Spring Boot 3.4+ projects and follow the project's existing compiled pattern otherwise.
10. Run the smallest relevant Gradle test command.

## Unit Test Pattern

```java
class PriceCalculatorTest {
    private final PriceCalculator calculator = new PriceCalculator();

    @Test
    void appliesDiscount() {
        assertThat(calculator.applyDiscount(100, 10)).isEqualTo(90);
    }
}
```

## Integration Test Pattern

```java
@SpringBootTest
@ExtendWith(AuthenticatedAsAdmin.class)
class CustomerServiceTest {
    @Autowired
    private DataManager dataManager;

    @Autowired
    private CustomerService customerService;

    private final List<Object> cleanup = new ArrayList<>();

    @Test
    void findsCustomerByEmail() {
        Customer customer = dataManager.create(Customer.class);
        customer.setEmail("customer@test.com");
        cleanup.add(dataManager.save(customer));

        assertThat(customerService.findByEmail("customer@test.com")).isPresent();
    }

    @AfterEach
    void tearDown() {
        cleanup.forEach(dataManager::remove);
    }
}
```

## Security Test Pattern

```java
Optional<Customer> result = systemAuthenticator.withUser(
        username,
        () -> customerService.findByEmail("customer@test.com")
);
```

Use this when the expected result depends on Jmix security policies.

## UI Integration Test Pattern

```java
@UiTest
@SpringBootTest(classes = {AppApplication.class, FlowuiTestAssistConfiguration.class})
class CustomerUiTest {
    @Autowired
    private ViewNavigators viewNavigators;

    @Test
    void opensCustomerList() {
        viewNavigators.view(UiTestUtils.getCurrentView(), CustomerListView.class).navigate();
        CustomerListView view = UiTestUtils.getCurrentView();
        DataGrid<Customer> grid = UiTestUtils.getComponent(view, "customersDataGrid");
        assertThat(grid).isNotNull();
    }
}
```

Use the project's helper for component lookup if it exists. Otherwise keep a local typed helper small and explicit.

## Test Type Comparison

| | Unit | Integration | UI Integration (`@UiTest`) | End-to-End (Masquerade) |
|---|---|---|---|---|
| **Spring context** | No | Yes | Yes | No (runs against live app) |
| **Browser** | No | No | No | Yes (Selenide/Selenium) |
| **Speed** | Fastest | Fast | Medium | Slowest |
| **What it tests** | Pure logic | Service layer + DB | Server-side Vaadin component tree | Real user-facing UI |
| **Best for** | Calculators, validators, mappers | Service methods, data access | View logic, navigation, data binding | Critical user flows, login, CRUD |

## UI Integration Tests (`@UiTest`)

Tests the server-side Vaadin component tree directly — no real browser. The `@UiTest` annotation starts Vaadin, configures application views, and sets up authentication.

### Key API

- `UiTestUtils.getCurrentView()` — returns the view currently opened by navigation
- `UiComponentUtils.getComponent(view, componentId)` — finds a component by its XML `id` attribute
- `ViewNavigators` — programmatic navigation between views
- `@UiTest` — JUnit 5 extension that starts Vaadin + configures views + sets up auth

Ref: https://docs.jmix.io/jmix/testing/ui-integration-tests.html

## End-to-End UI Tests (Masquerade)

Page Object pattern library built on **Selenide/Selenium WebDriver**. Drives a **real browser** against a running app. Supports views, components, dialogs, notifications, and composites. Requires Jmix 2.6+.

Ref: https://docs.jmix.io/jmix/testing/masquerade.html

### Key Packages

| Class | Package | Purpose |
|---|---|---|
| `Masquerade` | `io.jmix.masquerade` | Entry point — static `$j()` methods |
| `@TestView` | `io.jmix.masquerade` | Marks a view wrapper class |
| `@TestComponent` | `io.jmix.masquerade` | Marks a component field in a wrapper (wires by `j-test-id`) |
| `View<T>` | `io.jmix.masquerade.sys` | Base class for view wrappers |
| `Button`, `TextField`, etc. | `io.jmix.masquerade.component` | Component wrappers |
| `@FindBy` | `org.openqa.selenium.support` | Selenium annotation — locates by CSS/ID/XPath (NOT a Masquerade annotation) |

### How Component Lookup Works

Masquerade uses the HTML attribute `j-test-id` to locate components. When `jmix.ui.ui-test-mode=true` is set, Jmix generates this attribute on every UI component, using the component's XML `id` as the value.

Two ways to declare component fields in wrappers:

1. **`@TestComponent`** (preferred) — matches field name to `j-test-id`. Use `path` when the j-test-id differs from the field name or for nested elements.
2. **`@FindBy`** (Selenium) — locates by CSS selector, ID, or XPath. Use for components not covered by `j-test-id` (e.g., the login form's native Vaadin elements).

### Setup

**build.gradle:**
```gradle
testImplementation 'io.jmix.masquerade:jmix-masquerade'
```

**application.properties** (enables `j-test-id` attribute generation on all UI components):
```properties
jmix.ui.ui-test-mode=true
```

**Selenide configuration:**
```java
@BeforeAll
static void setupSelenide() {
    baseUrl = "http://localhost:8080";
    timeout = 10000;
    browser = CHROME;
    headless = true;
}
```

### View Wrappers (Page Objects)

Each Jmix view gets a wrapper class annotated with `@TestView`, extending `View<Self>`. Use `@TestComponent` for Jmix components (matched by `j-test-id`) and `@FindBy` for raw HTML elements.

```java
@TestView(id = "MyView")
public class MyView extends View<MyView> {

    // Field name matches j-test-id by default
    @TestComponent
    private EntityComboBox entityComboBox;

    // Use path when j-test-id differs from field name or for nested lookup
    @TestComponent(path = "myButton")
    private Button button;

    // @FindBy for elements not accessible via j-test-id
    @FindBy(xpath = "//vaadin-text-area[@class='my-text-area']")
    private TextArea textArea;
}
```

### Writing Tests

```java
public class LoginUiTest {

    @BeforeAll
    static void setup() { /* Selenide config */ }

    @Test
    void loginAsAdmin() {
        Selenide.open("/");
        LoginView loginView = Masquerade.$j(LoginView.class);

        loginView.getUsernameField().shouldHave(value("admin"))
                .setValue("").setValue("admin");
        loginView.getPasswordField().shouldHave(value("admin"))
                .setValue("").setValue("admin");
        loginView.getButton().shouldHave(text("Log in")).click();
    }
}
```

### Key API

- `Masquerade.$j(Class<T> clazz)` — gets a view wrapper for the current page
- `Masquerade.$j(String uiTestId)` — access element directly by `j-test-id` value
- `Masquerade.$j(Class<T> clazz, String... path)` — get wrapper class by nested `j-test-id` path
- `@TestComponent` — wires field to component by `j-test-id` (field name = test ID by default)
- `@TestComponent(path = "...")` — wires field with explicit `j-test-id` path
- `@FindBy(css/id/xpath = "...")` — Selenium locator for raw HTML elements outside `j-test-id` scope

### Accessing Fragments

For fragments not declared in the view wrapper, use `$j` with the wrapper class directly:

```java
$j(TestFragment.class)
        .getTestField()
        .shouldHave(value(""))
        .setValue("Fragment_2")
        .shouldHave(value("Fragment_2"));
```

## Cleanup Audit

Before finishing, check:

- Every created persistent record is removed in `@AfterEach`.
- Cleanup uses the same authentication level needed for deletion.
- Test data has unique values to avoid collisions.
- Assertions verify persisted or visible behavior, not just absence of exceptions.
- The test command can run one class or method without running the full suite.

## Forbidden

- `new Entity()` or constructor-created Jmix entities in persistence tests.
- Tests that depend on data left by previous tests.
- Cleanup only at the end of the test method.
- UI tests that assert only that navigation did not throw.
- Browser tests for behavior that `@UiTest` or service tests can prove.
- Hardcoded sleeps when framework waits or component assertions are available.
- `@WithUserDetails` for Jmix security tests when `SystemAuthenticator` or the project auth extension is available.
