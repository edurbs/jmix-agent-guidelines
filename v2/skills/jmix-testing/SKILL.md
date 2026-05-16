---
name: jmix-testing
description: Writing reliable tests for Jmix application — Unit, Integration, UI Integration, and End-to-End (Masquerade) tests with proper authentication and cleanup.
---

# Testing

## When to Use

Use this Skill when:
- Writing tests for Jmix services and views
- Testing with authentication context
- Creating page object wrappers for end-to-end UI tests
- Choosing between UI integration tests and Masquerade tests

## Test Type Comparison

| | Unit | Integration | UI Integration (`@UiTest`) | End-to-End (Masquerade) |
|---|---|---|---|---|
| **Spring context** | No | Yes | Yes | No (runs against live app) |
| **Browser** | No | No | No | Yes (Selenide/Selenium) |
| **Speed** | Fastest | Fast | Medium | Slowest |
| **What it tests** | Pure logic | Service layer + DB | Server-side Vaadin component tree | Real user-facing UI |
| **Best for** | Calculators, validators, mappers | Service methods, data access | View logic, navigation, data binding | Critical user flows, login, CRUD |

## Unit Tests (No Spring Context)

```java
class PriceCalculatorTest {
    private final PriceCalculator calculator = new PriceCalculator();

    @ParameterizedTest
    @CsvSource({"100, 10, 90", "100, 0, 100"})
    void shouldApplyDiscount(int price, int discount, int expected) {
        assertThat(calculator.applyDiscount(price, discount)).isEqualTo(expected);
    }
}
```

## Integration Tests

```java
@SpringBootTest
@ExtendWith(AuthenticatedAsAdmin.class)
class OrderServiceTest {

    @Autowired OrderService orderService;
    @MockitoBean PaymentGateway paymentGateway;  // NOT @MockBean!

    @Test
    void shouldProcessPayment() {
        when(paymentGateway.charge(any())).thenReturn(true);
        orderService.checkout(order);
        verify(paymentGateway).charge(any());
    }
}
```

```java
@SpringBootTest
@ExtendWith(AuthenticatedAsAdmin.class)
class CustomerServiceTest {
    @Autowired
    CustomerService customerService;
    @Autowired
    DataManager dataManager;

    @Test
    void testFindByEmail() {
        // given
        Customer customer = dataManager.create(Customer.class);
        customer.setEmail("customer@test.com");
        dataManager.save(customer);

        // when
        Optional<Customer> foundCustomer = customerService.findByEmail("customer@test.com");

        // then
        assertThat(foundCustomer)
                .isPresent();
    }
}
```

## UI Integration Tests (`@UiTest`)

Tests the server-side Vaadin component tree directly — no real browser. The `@UiTest` annotation starts Vaadin, configures application views, and sets up authentication.

Ref: https://docs.jmix.io/jmix/testing/ui-integration-tests.html

```java
@UiTest
@SpringBootTest(classes = {SampleApplication.class, FlowuiTestAssistConfiguration.class})
class UserUiTest {
    @Autowired
    DataManager dataManager;

    @Autowired
    ViewNavigators viewNavigators;

    @Test
    void test_createUser() {
        // Navigate to user list view
        viewNavigators.view(UiTestUtils.getCurrentView(), UserListView.class).navigate();

        UserListView userListView = UiTestUtils.getCurrentView();

        // click "Create" button
        JmixButton createBtn = findComponent(userListView, "createBtn");
        createBtn.click();

        // Get detail view
        UserDetailView userDetailView = UiTestUtils.getCurrentView();

        // Set username and password in the fields
        TypedTextField<String> usernameField = findComponent(userDetailView, "usernameField");
        String username = "test-user-" + System.currentTimeMillis();
        usernameField.setValue(username);

        JmixPasswordField passwordField = findComponent(userDetailView, "passwordField");
        passwordField.setValue("test-passwd");

        JmixPasswordField confirmPasswordField = findComponent(userDetailView, "confirmPasswordField");
        confirmPasswordField.setValue("test-passwd");

        // Click "OK"
        JmixButton commitAndCloseBtn = findComponent(userDetailView, "saveAndCloseBtn");
        commitAndCloseBtn.click();

        // Get navigated user list view
        userListView = UiTestUtils.getCurrentView();

        // Check the created user is shown in the table
        DataGrid<User> usersDataGrid = findComponent(userListView, "usersDataGrid");

        DataGridItems<User> usersDataGridItems = usersDataGrid.getItems();
        Assertions.assertNotNull(usersDataGridItems);

        usersDataGridItems.getItems().stream()
                .filter(u -> u.getUsername().equals(username))
                .findFirst()
                .orElseThrow();
    }

    @AfterEach
    void tearDown() {
        dataManager.load(User.class)
                .query("e.username like ?1", "test-user-%")
                .list()
                .forEach(u -> dataManager.remove(u));
    }

    @SuppressWarnings("unchecked")
    private static <T> T findComponent(View<?> view, String componentId) {
        return (T) UiComponentUtils.getComponent(view, componentId);
    }
}
```

### Key API

- `UiTestUtils.getCurrentView()` — returns the view currently opened by navigation
- `UiComponentUtils.getComponent(view, componentId)` — finds a component by its XML `id` attribute
- `ViewNavigators` — programmatic navigation between views
- `@UiTest` — JUnit 5 extension that starts Vaadin + configures views + sets up auth

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

**Selenide configuration** (e.g., in test class or JUnit 5 extension):
```java
import static com.codeborne.selenide.Configuration.*;
import static com.codeborne.selenide.Browsers.*;

@BeforeAll
static void setupSelenide() {
    baseUrl = "http://localhost:8080";
    timeout = 10000;
    browser = CHROME;
    headless = true;
}
```

### Project Structure

Place Masquerade tests separate from unit/integration tests:

```
src/test/java/
├── com.company.testproject/
│   └── test_support/
│       └── view/                  # View wrappers (page objects)
│           └── LoginView.java
└── ui_autotest/                   # End-to-end test classes
    └── LoginUiTest.java
```

### View Wrappers (Page Objects)

Each Jmix view gets a wrapper class annotated with `@TestView`, extending `View<Self>`. Use `@TestComponent` for Jmix components (matched by `j-test-id`) and `@FindBy` for raw HTML elements.

```java
import io.jmix.masquerade.TestView;
import io.jmix.masquerade.TestComponent;
import io.jmix.masquerade.component.Button;
import io.jmix.masquerade.component.TextField;
import io.jmix.masquerade.component.PasswordField;
import io.jmix.masquerade.sys.View;
import org.openqa.selenium.support.FindBy;

@TestView
public class LoginView extends View<LoginView> {

    // @FindBy for native Vaadin login elements (no j-test-id available)
    @FindBy(css = "[slot='submit']")
    private Button button;

    @FindBy(id = "vaadinLoginUsername")
    private TextField username;

    @FindBy(id = "vaadinLoginPassword")
    private PasswordField password;

    public Button getButton() { return button; }
    public TextField getUsernameField() { return username; }
    public PasswordField getPasswordField() { return password; }
}
```

For Jmix views with `j-test-id` enabled, prefer `@TestComponent`:

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

    public EntityComboBox getEntityComboBox() { return entityComboBox; }
    public Button getButton() { return button; }
}
```

### Writing Tests (JUnit 5)

```java
import com.codeborne.selenide.Selenide;
import io.jmix.masquerade.Masquerade;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static com.codeborne.selenide.Condition.text;
import static com.codeborne.selenide.Condition.value;

public class LoginUiTest {

    @BeforeAll
    static void setup() {
        // Selenide config (or use selenide.properties file)
    }

    @Test
    void loginAsAdmin() {
        Selenide.open("/");

        LoginView loginView = Masquerade.$j(LoginView.class);

        loginView.getUsernameField()
                .shouldHave(value("admin"))
                .setValue("")
                .setValue("admin");

        loginView.getPasswordField()
                .shouldHave(value("admin"))
                .setValue("")
                .setValue("admin");

        loginView.getButton()
                .shouldHave(text("Log in"))
                .click();
    }
}
```

### Key API

- `Masquerade.$j(Class<T> clazz)` — gets a view wrapper for the current page
- `Masquerade.$j(String uiTestId)` — access element directly by `j-test-id` value (returns `SelenideElement`)
- `Masquerade.$j(Class<T> clazz, String... path)` — get wrapper class by nested `j-test-id` path
- `@TestComponent` — wires field to component by `j-test-id` (field name = test ID by default)
- `@TestComponent(path = "...")` — wires field with explicit `j-test-id` path (for nested or renamed elements)
- `@FindBy(css/id/xpath = "...")` — Selenium locator for raw HTML elements outside `j-test-id` scope
- Selenide assertions: `.shouldHave(value(...))`, `.shouldBe(visible)`, `.shouldHave(text(...))`

### Accessing Fragments / Composites

For fragments not declared in the view wrapper, use `$j` with the wrapper class directly:

```java
$j(TestFragment.class)
        .getTestField()
        .shouldHave(value(""))
        .setValue("Fragment_2")
        .shouldHave(value("Fragment_2"));
```

## Authentication in Tests

**Integration / UI Integration tests:** Use `SystemAuthenticator` via `AuthenticatedAsAdmin` JUnit extension available in the project at `src/test/java/**/test_support/AuthenticatedAsAdmin.java`:

```java
@SpringBootTest
@ExtendWith(AuthenticatedAsAdmin.class)
class MyServiceTest { ... }
```

**Masquerade tests:** No special auth setup — tests run against a live app. Login is part of the test flow (navigate to login view and enter credentials).

## Cleanup

Always in `@AfterEach` (applies to Unit, Integration, and UI Integration tests):

```java
@AfterEach
void tearDown() {
    dataManager.remove(createdEntities);
}
```

Masquerade tests typically clean up via the UI (e.g., deleting created records through the view) or via REST API calls to the running app.

## Checklist
- [ ] NO `@Transactional` on tests
- [ ] Cleanup in `@AfterEach`
- [ ] Use `@MockitoBean` (not `@MockBean`)
- [ ] Use `AuthenticatedAsAdmin` for auth (not `@WithUserDetails`)
- [ ] AssertJ assertions
- [ ] `@UiTest` for server-side UI tests, Masquerade for browser E2E tests
- [ ] `jmix.ui.ui-test-mode=true` when using Masquerade

## Forbidden
- `@Transactional` on test classes/methods
- `@MockBean` (deprecated, use `@MockitoBean`)
- `@WithUserDetails` (use `SystemAuthenticator` / `AuthenticatedAsAdmin`)
- Cleanup at the end of test method (use `@AfterEach`)
