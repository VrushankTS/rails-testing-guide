### 5.1 Why system tests are useful, but not the main layer

System tests, usually implemented with Capybara, simulate a real browser session and exercise the application the way a user would: click links, fill forms, submit pages, wait for content, and assert that the page changes as expected.

These tests are valuable because they catch failures that only show up when the entire application stack works together:

- the page loads with the right layout
- the form submits correctly
- validation messages appear in the browser
- JavaScript-driven UI updates work as expected
- navigation and route flow remain intact

However, they are not the primary layer for most Rails business logic. In a rules-heavy backend project, the majority of confidence should come from:

- model/unit tests for domain logic
- request/integration tests for controller and API behavior
- job tests for background processing

System tests are a smaller, more expensive layer and should be used selectively.

### 5.2 When to use Capybara/system tests

Use them when the user journey is important enough that a backend-only test would be insufficient.

Good candidates include:

- vehicle registration flow
- posting a completed service / updating service status
- mapping invoice to vehicle and confirming the result on screen
- login or role-based access flows
- approval or rejection workflows
- any multi-step journey that users actually rely on

These are the kinds of workflows where a request spec may pass but the real browser flow still fails because of UI or JavaScript issues.

### 5.3 When not to use them aggressively

Avoid making Capybara the default test layer for every rule or every database validation. This usually leads to:

- slow test runs
- brittle tests that break due to minor markup changes
- excessive maintenance cost
- lower confidence in the actual business rules because the tests are too high-level and too slow to run often

For a backend-heavy Rails app or a Service Projections system, it is usually better to keep system tests limited to a few critical journeys rather than trying to cover every rule through the browser.

### 5.4 Common Capybara patterns

A basic Capybara example:

```ruby
# spec/system/vehicle_registration_spec.rb
require 'rails_helper'

RSpec.describe "Vehicle registration", type: :system do
  before do
    driven_by(:rack_test) # For fast non-JS testing
  end

  it "allows a user to register a vehicle" do
    model = create(:model, name: "Honda Activa")

    visit "/vehicles/new"
    fill_in "Registration number", with: "KA01AB1234"
    select "Honda Activa", from: "Model"
    select "RB", from: "Owner type"
    fill_in "Invoice date", with: "2026-08-15"
    click_button "Create Vehicle"

    expect(page).to have_content("Vehicle was successfully created")
    expect(page).to have_content("KA01AB1234")
  end
end
```

This checks real browser behavior, not just app logic. It verifies the workflow from the user's point of view.

### 5.5 Typical browser interactions to test

System tests should cover the critical user actions, for example:

- `visit` a page
- `fill_in` form fields
- `select` dropdown values
- `check` or `uncheck` checkboxes
- `click_button` or `click_link`
- `have_content` to assert page text appears
- `have_current_path` to assert navigation
- `have_css` or `have_selector` for more structured DOM assertions

Example:

```ruby
it "shows validation errors for invalid input" do
  visit "/vehicles/new"
  click_button "Create Vehicle"

  expect(page).to have_content("Registration number can't be blank")
  expect(page).to have_current_path("/vehicles/new")
end
```

### 5.6 For backend-heavy apps, keep them small and purposeful

In projects like this one, system tests are best used for very targeted workflows, not for exhaustive business coverage.

A practical rule:

- Unit tests = cover the business rules
- Request/integration tests = cover API and controller contracts
- Job tests = cover asynchronous behavior
- System tests = cover 2–5 critical user journeys only

Example critical journeys for this project:

- Register a vehicle and see it appear as live / pending invoice
- Map an invoice and confirm projection is triggered
- View the service due state on the dashboard
- Submit a service completion and verify the correct status updates

### 5.7 Avoiding brittle system tests

Capybara tests can become flaky if overused. To keep them maintainable:

- write only for the most important user paths
- avoid testing every text variation or repeated HTML structure
- prefer stable selectors like visible labels or data-test attributes
- use `have_text`/`have_content` only for user-visible text
- avoid depending on exact timing unless necessary
- keep tests isolated and independent from one another

A more stable practice is to use specific selectors in the app, for example:

```html
<button data-testid="save-vehicle-button">Save</button>
```

Then test with:

```ruby
find("[data-testid='save-vehicle-button']").click
```

This is far less likely to break than relying on page structure or vague CSS selectors.

### 5.8 A realistic testing balance for this application

For a Service Projections app, a balanced strategy is:

- Use unit tests to cover vehicle type rules, lapse logic, service thresholds, and projection decisions
- Use request specs to test controller/API actions and state changes
- Use job specs to cover sweeper, notification, projection workers
- Use a few Capybara/system tests for the workflows that matter most to the business and the user experience

This is the sane default for Rails teams that do significant backend development but also have real browser-based actions.

### 5.9 Final guidance

Capybara/system tests are significant because they validate the browser experience, but they are not the backbone of a rules-heavy Rails application. They are a valuable top layer, not the foundation.

For this project, do not feel pressured to treat them as mandatory for every feature. Instead, keep them focused on the critical user journeys where a browser-level test adds real value.

---