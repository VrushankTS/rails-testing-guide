### 6.1 Focus on the Rails behaviors that break apps

For a Rails application, the most important tests are usually not generic "Ruby tests" but tests for actual Rails behavior:
- model validation and association rules
- ActiveRecord queries and scopes
- callbacks and lifecycle hooks
- state transitions and status changes
- background jobs and delayed work
- request/response behavior for controller actions

These are the places where bugs tend to hide in production.

### 6.2 Validations: test the business rule, not just the Rails API

Validate the actual rule you care about:

```ruby
it "requires a unique registration number" do
  create(:vehicle, registration_number: "KA01AB1234")
  duplicate = build(:vehicle, registration_number: "KA01AB1234")

  expect(duplicate).not_to be_valid
end
```

Do not write shallow tests like "model is valid" without checking the real constraint.

### 6.3 Associations and related records

Test the behavior of relationships that affect business flow:
- `belongs_to` / `has_many` correctness
- dependent destroy behavior
- foreign key constraints
- cascading status updates when parent/child records change

Example:

```ruby
it "removes the vehicle's projections when the vehicle is deleted" do
  vehicle = create(:vehicle)
  create(:projection, vehicle: vehicle)

  vehicle.destroy

  expect(Projection.where(vehicle_id: vehicle.id)).to be_empty
end
```

### 6.4 Scopes and queries must be tested as business logic

In Rails, query methods often contain logic that is easy to break silently. Test:
- filters by status, owner type, due date
- ordering rules
- edge cases like nil or empty results
- the exact records returned

```ruby
it "returns only vehicles with pending projections" do
  pending = create(:vehicle, projection_status: "PENDING")
  projected = create(:vehicle, projection_status: "PROJECTED")

  expect(Vehicle.with_pending_projection).to contain_exactly(pending)
end
```

### 6.5 Callbacks and lifecycle hooks

Callbacks are a critical Rails feature, but they are also a common source of hidden side effects.

Test:
- whether a callback actually triggers the expected job or status update
- both the success and failure paths
- whether the callback runs only when expected

```ruby
it "enqueues projection job after a vehicle is created" do
  expect {
    create(:vehicle)
  }.to have_enqueued_job(ProjectServiceJob)
end
```

### 6.6 Time and date handling is a major test risk

Business logic like service due dates, lapsing, and buffer windows is date-sensitive. Do not rely on real current time in tests.

Use:
- `travel_to`
- `freeze_time`
- explicit date values in factories

```ruby
travel_to Date.new(2026, 8, 20) do
  vehicle = create(:vehicle, invoice_date: Date.new(2025, 8, 15))
  expect(vehicle.next_service_due_date).to eq(Date.new(2026, 8, 29))
end
```

This avoids flaky, calendar-dependent tests.

### 6.7 Transactions and test isolation are critical

Rails tests run in a transactional database context by default. This is good, but do not assume it solves every test isolation problem.

Be careful when:
- running background jobs that execute after the transaction commits
- creating records in callbacks that happen outside the ordinary request flow
- using external services or queueing systems that are not isolated from the test DB

If a test needs real queue behavior, use the test adapter or inline execution explicitly.

### 6.8 External dependencies must be stubbed

Bad production behavior often comes from external calls: SMS, email, invoice APIs, third-party retrieval services.

Test the application logic without calling the real outside system:
- `WebMock` for HTTP calls
- VCR for recorded API responses
- fake adapters for notification services

```ruby
stub_request(:post, "https://sms.example.com/send")
  .to_return(status: 200)
```

### 6.9 Test the actual contract of your app

Rails tests should check what the app is supposed to do as a system, not just what a method returns internally.

Most important contract checks:
- request returns the correct HTTP status
- response body contains the expected JSON/data
- database state changes as expected
- async job is enqueued or executed
- user-facing status changes are reflected correctly

This is often more valuable than testing every implementation detail.

### 6.10 Common anti-patterns to avoid

Avoid these patterns — they create slow, brittle, low-value tests:
- testing implementation details or private method calls instead of user-visible behavior
- writing giant tests that mix multiple unrelated scenarios
- creating brittle tests tied to exact HTML/CSS structure in system tests
- depending on test execution order or database record order
- reusing or relying on global shared state across examples
- testing the same rule in several layers unnecessarily
- relying on real third-party/external systems in CI
- testing current date/time without freezing it

If a test is hard to understand, it's usually too broad or too coupled to internals. If it doesn't protect a real business risk or lifecycle behavior, it's probably not worth keeping at high priority.

### 6.11 Coverage: useful, but not the only objective

We use `SimpleCov` to measure how much of the application code is exercised by tests (setup covered in Section 1.7). This is a helpful signal for whether key business logic is being covered consistently — but it should not become the sole goal.

For this project, a sensible focus is:
- model logic: high coverage
- service logic: high coverage
- projection/sweeper/notification code: high coverage
- request and job flows: strong but not necessarily 100%
- system tests: limited and selective

A suite with 90%+ coverage in the rule-heavy and background-job-heavy parts of the app is generally much more valuable than a suite that hits 100% of trivial code but misses the risky cases.

### 6.12 Recommended team policy

- Do not merge code that breaks a relevant existing test.
- Add tests when changing business logic, especially in projection rules and status transitions.
- Use coverage as a review aid, not as a target to chase blindly.
- Prefer meaningful business coverage over inflated percentages.

### 6.13 Final principle

Keep the test suite readable, realistic, and focused on the actual business contract. A good Rails test answers one business question clearly and fails for one valid reason — if a test doesn't protect a real business risk, it's not worth keeping at high priority.

---