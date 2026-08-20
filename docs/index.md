# Rails Testing Methodology & Practice Guide

**Purpose:** A hands-on guide to writing and running tests in a Ruby on Rails application. This complements [testing-strategy-service-projections.md](testing-strategy-service-projections.md), which covers *what* to test and *why*; this document covers *how* to actually do it.

**Audience:** Developers building features in the Planned Service module and future modules.

---

## 1. Testing Environment & Setup

### 1.1 The Rails Testing Ecosystem

A standard Rails project uses several layers of testing tools that work together. Here's what each does:

| Tool | Purpose | Installed by? |
|------|---------|---------------|
| **RSpec** | Test runner and assertion library (replaces Rails' built-in Minitest) | Explicitly added gem |
| **Factory Bot** | Generates test data on demand (instead of static fixtures) | Explicitly added gem |
| **Shoulda Matchers** | Pre-built assertions for common Rails patterns (validations, associations, callbacks) | Explicitly added gem |
| **VCR** | Records and replays HTTP interactions to avoid hitting real APIs during tests | Explicitly added gem |
| **Capybara** | Browser-like test driver (clicks links, fills forms, waits for content) | Comes with Rails; activated when RSpec system tests are used |
| **SimpleCov** | Measures test coverage (% of code actually exercised by tests) | Explicitly added gem |
| **Faker** | Generates realistic fake data (names, emails, phone numbers, etc.) | Often paired with Factory Bot |

### 1.2 Gemfile Dependencies for Testing

Your test group in `Gemfile` should include:

```ruby
group :test do
  gem 'rspec-rails'              # RSpec integration with Rails
  gem 'factory_bot_rails'        # Generate test data
  gem 'shoulda-matchers'         # Pre-built matchers for validations, associations
  gem 'faker'                    # Realistic fake data generation
  gem 'vcr'                      # Record/replay HTTP responses
  gem 'webmock'                  # Stub HTTP requests (required by VCR)
  gem 'simplecov'                # Code coverage measurement
  gem 'rspec-its'                # One-liner test syntax (optional, but nice for simple checks)
end
```

Install these with:
```bash
bundle install
```

### 1.3 Directory Structure

After generating a Rails app, your test files live in the `spec/` folder (because we use RSpec, not the default `test/` folder):

```
spec/
├── models/              # Model tests (validations, associations, methods)
├── requests/            # Integration tests (API endpoints, controllers)
├── jobs/                # Background job tests
├── services/            # Service/PORO tests (business logic classes)
├── system/              # End-to-end browser tests
├── support/             # Helper code (shared examples, custom matchers)
│   ├── factory_bot.rb
│   └── shared_examples.rb
├── fixtures/            # Static test data (used by VCR)
└── spec_helper.rb       # Main RSpec configuration
```

### 1.4 Setting Up RSpec in a Rails Project

If you're starting from scratch, generate RSpec config:

```bash
bundle exec rails generate rspec:install
```

This creates:
- `.rspec` — command-line options for the `rspec` command
- `spec/spec_helper.rb` — shared RSpec setup
- `spec/rails_helper.rb` — Rails-specific RSpec setup

#### Key Configuration in `spec/rails_helper.rb`

```ruby
# spec/rails_helper.rb

require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('config/environment', __dir__)

abort("The Rails environment is running in production mode!") if Rails.env.production?

require 'rspec/rails'

# Database configuration is automatic; Rails maintains a separate test database

RSpec.configure do |config|
  # Use ActiveRecord's database transactions to roll back changes after each test
  config.use_transactional_fixtures = true

  # Infer spec location from file path (models/user_spec.rb tests the User model)
  config.infer_spec_type_from_file_location!
end
```

This does several important things:
- Loads your Rails app with `ENV['RAILS_ENV'] = 'test'`
- Enables automatic transaction rollback so each test starts with a clean database
- Maps file locations to spec types (so RSpec knows a file in `spec/models/` is a model test)

### 1.5 Setting Up Factory Bot

Factory Bot needs to know where your factories are defined. Create `spec/support/factory_bot.rb`:

```ruby
# spec/support/factory_bot.rb
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```

Then require it in `spec/rails_helper.rb`:

```ruby
# spec/rails_helper.rb
require 'support/factory_bot'
```

This gives you access to `create`, `build`, `build_stubbed` etc. in any test without prefixing them with `FactoryBot.`.

### 1.6 Setting Up Shoulda Matchers

In `spec/rails_helper.rb`, add:

```ruby
# spec/rails_helper.rb
require 'shoulda/matchers'

Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
```

### 1.7 Setting Up SimpleCov for Coverage Reports

In `spec/spec_helper.rb`, at the **very top** (before any other requires):

```ruby
# spec/spec_helper.rb
require 'simplecov'

SimpleCov.start 'rails' do
  add_filter '/spec/'
  add_filter '/config/'
  add_filter '/db/'
end
```

After running tests, SimpleCov generates an HTML report in `coverage/index.html`:

```bash
bundle exec rspec
# ... tests run ...
# Then open coverage/index.html in your browser to see coverage %
```

### 1.8 The Test Database

Rails automatically manages a **test database** that:
- Is separate from your development database
- Gets wiped and rebuilt from your schema before tests run
- Is wrapped in a transaction for each test, so changes are rolled back automatically

No configuration needed — Rails handles this automatically. But occasionally you may need to:

```bash
# Rebuild the test database schema (if you modified migrations)
bundle exec rails db:test:prepare

# Or, explicitly drop and recreate it
RAILS_ENV=test bundle exec rails db:drop db:create db:schema:load
```

### 1.9 Running Tests

#### Run all tests:
```bash
bundle exec rspec
```

#### Run a specific file:
```bash
bundle exec rspec spec/models/vehicle_spec.rb
```

#### Run a specific line (test case):
```bash
bundle exec rspec spec/models/vehicle_spec.rb:25
```

#### Run all tests matching a pattern:
```bash
bundle exec rspec --pattern "*projection*"
```

#### Run only one type of test:
```bash
bundle exec rspec spec/models/           # Only model tests
bundle exec rspec spec/requests/         # Only integration tests
bundle exec rspec spec/jobs/             # Only job tests
bundle exec rspec spec/system/           # Only system tests
```

#### Run tests with verbose output:
```bash
bundle exec rspec --format documentation
```

#### Run with coverage:
```bash
COVERAGE=true bundle exec rspec
```

#### Run in parallel (if you install the `parallel_tests` gem):
```bash
bundle exec parallel_test spec/
```

### 1.10 Configuration in `.rspec`

The `.rspec` file sets default options for the `rspec` command:

```
--require spec_helper
--format progress
--color
--order random
```

- `--require spec_helper`: Always load spec_helper before running tests
- `--format progress`: Show a dot for each passing test, F for failures (can be `documentation` for verbose names)
- `--color`: Colorize output
- `--order random`: Run tests in random order (catches tests that accidentally depend on order)

### 1.11 CI/CD Integration

Before merging any code, tests should run automatically. In your CI (GitHub Actions, CircleCI, etc.):

```yaml
# .github/workflows/tests.yml (GitHub Actions example)
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres

    steps:
      - uses: actions/checkout@v2
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.1
          bundler-cache: true
      
      - run: bundle exec rails db:test:prepare
      - run: bundle exec rspec
      - run: bundle exec rubocop  # Optional: static code quality
```

This ensures:
- Tests run on every push and pull request
- If tests fail, the PR can't be merged
- Developers get immediate feedback

### 1.12 Debugging Tests

#### See actual vs. expected values:
```ruby
expect(actual).to eq(expected)  # Shows diff if not equal
```

#### Add debug output to tests:
```ruby
it "does something" do
  result = some_method
  pp result  # Pretty-print the result
  puts result.inspect
  expect(result).to eq(expected)
end
```

#### Use a debugger within tests:
```ruby
it "does something" do
  result = some_method
  binding.pry  # Execution pauses here; inspect variables, call methods
  expect(result).to eq(expected)
end
```

Then run the test with:
```bash
bundle exec rspec spec/models/vehicle_spec.rb --pry
```

#### See what SQL is being executed:
```ruby
it "queries the database efficiently" do
  expect {
    Vehicle.find_by(registration_number: 'KA01AB1234')
  }.not_to exceed_query_limit(1)  # Requires rspec-rails and should_not gem
end
```

---


## 2. Unit Testing in Rails — Patterns & Practices

### 2.1 Overview: What Unit Tests Do

A unit test checks one isolated piece of business logic in complete isolation. It doesn't:
- Call other business logic methods
- Hit the real database (or pretends not to, by using transactions)
- Make HTTP requests
- Touch the filesystem
- Depend on the current time

It does:
- Check that a single rule or calculation works correctly
- Verify error handling and edge cases
- Run in milliseconds
- Give pinpoint feedback when it fails

### 2.2 The Basic Structure of a Unit Test

Every RSpec test follows this pattern:

```ruby
describe "what thing is being tested" do
  context "in this scenario" do
    it "should do this specific thing" do
      # Arrange — set up the data you need
      vehicle = build(:vehicle, registration_number: "KA01AB1234")
      
      # Act — call the thing you're testing
      result = vehicle.valid?
      
      # Assert — check the result
      expect(result).to be(true)
    end
  end
end
```

**Breaking it down:**
- `describe` — top-level grouping for a class or feature
- `context` — a specific scenario or condition (optional, but improves readability)
- `it` — a single test case (should be one assertion, conceptually)
- Arrange → Act → Assert — the three steps of every test

### 2.3 Testing Models — Validations

Rails models have built-in validation rules. Here's how to test them:

#### Example Model:
```ruby
# app/models/vehicle.rb
class Vehicle < ApplicationRecord
  validates :registration_number, presence: true, uniqueness: true
  validates :model_id, presence: true
  validates :owner_type, inclusion: { in: %w(RB MBSI) }
end
```

#### Example Tests:
```ruby
# spec/models/vehicle_spec.rb
describe Vehicle do
  describe "validations" do
    it { should validate_presence_of(:registration_number) }
    it { should validate_uniqueness_of(:registration_number) }
    it { should validate_presence_of(:model_id) }
    it { should validate_inclusion_of(:owner_type).in_array(%w(RB MBSI)) }
  end
end
```

These `should validate_*` lines are **Shoulda Matchers** — pre-written assertions for common Rails patterns. They're faster to write than manual assertions, and you get for free:
- Proper error messages when they fail
- Checking edge cases (like what happens with nil, empty string, etc.)

If you need a custom validation test that Shoulda doesn't have a matcher for:

```ruby
describe "validations" do
  context "when registration_number is already taken" do
    it "is invalid" do
      create(:vehicle, registration_number: "KA01AB1234")
      duplicate = build(:vehicle, registration_number: "KA01AB1234")
      
      expect(duplicate).not_to be_valid
      expect(duplicate.errors[:registration_number]).to include("has already been taken")
    end
  end
end
```

### 2.4 Testing Models — Associations

Your models likely have relationships (a Vehicle belongs to a Model, a Projection belongs to a Vehicle, etc.). Test them:

```ruby
# app/models/vehicle.rb
class Vehicle < ApplicationRecord
  belongs_to :model
  has_many :projections, dependent: :destroy
end

# app/models/projection.rb
class Projection < ApplicationRecord
  belongs_to :vehicle
end
```

#### Example Tests:
```ruby
# spec/models/vehicle_spec.rb
describe Vehicle do
  describe "associations" do
    it { should belong_to(:model) }
    it { should have_many(:projections).dependent(:destroy) }
  end
end

# spec/models/projection_spec.rb
describe Projection do
  describe "associations" do
    it { should belong_to(:vehicle) }
  end
end
```

If an association has custom logic, test it explicitly:

```ruby
# spec/models/vehicle_spec.rb
describe "associations" do
  it "destroys associated projections when the vehicle is deleted" do
    vehicle = create(:vehicle)
    projection = create(:projection, vehicle: vehicle)
    
    vehicle.destroy
    
    expect(Projection.find_by(id: projection.id)).to be_nil
  end
end
```

### 2.5 Testing Models — Scopes and Query Methods

Scopes are query shortcuts that return an ActiveRecord relation:

```ruby
# app/models/vehicle.rb
class Vehicle < ApplicationRecord
  scope :by_owner_type, ->(owner_type) { where(owner_type: owner_type) }
  scope :with_pending_projection, -> { where("projection_status = ?", "PENDING") }
  
  # Also test custom query methods
  def self.overdue_for_service
    joins(:projections)
      .where("projections.due_date < ?", Date.today)
      .where(projections: { status: "UPCOMING" })
  end
end
```

#### Example Tests:
```ruby
# spec/models/vehicle_spec.rb
describe Vehicle, type: :model do
  describe ".by_owner_type" do
    it "returns only vehicles with the specified owner type" do
      rb_vehicle = create(:vehicle, owner_type: "RB")
      mbsi_vehicle = create(:vehicle, owner_type: "MBSI")
      
      result = Vehicle.by_owner_type("RB")
      
      expect(result).to include(rb_vehicle)
      expect(result).not_to include(mbsi_vehicle)
    end
  end
  
  describe ".with_pending_projection" do
    it "returns vehicles waiting for a projection to be created" do
      pending = create(:vehicle, projection_status: "PENDING")
      projected = create(:vehicle, projection_status: "PROJECTED")
      
      result = Vehicle.with_pending_projection
      
      expect(result).to contain_exactly(pending)
    end
  end
  
  describe ".overdue_for_service" do
    it "includes vehicles with upcoming services past their due date" do
      overdue_vehicle = create(:vehicle)
      overdue_projection = create(:projection, 
        vehicle: overdue_vehicle,
        status: "UPCOMING",
        due_date: 10.days.ago
      )
      
      result = Vehicle.overdue_for_service
      
      expect(result).to include(overdue_vehicle)
    end
    
    it "excludes vehicles without overdue services" do
      future_vehicle = create(:vehicle)
      future_projection = create(:projection,
        vehicle: future_vehicle,
        status: "UPCOMING", 
        due_date: 10.days.from_now
      )
      
      result = Vehicle.overdue_for_service
      
      expect(result).not_to include(future_vehicle)
    end
  end
end
```

### 2.6 Testing Models — Custom Methods and Business Logic

This is where the core rules of your system get tested. For the Service Projections app, this includes methods that calculate projections, determine vehicle type rules, check lapse status, etc.

```ruby
# app/models/projection.rb
class Projection < ApplicationRecord
  belongs_to :vehicle
  
  # Core business logic: is this service upcoming, due, or lapsed?
  def status_classification
    today = Date.today
    
    if due_date.nil? || km_due.nil?
      return :incomplete
    end
    
    # Whichever (date or km) comes first determines the status
    trigger_point = [due_date, km_due].min
    
    if today < trigger_point - 7.days
      :upcoming
    elsif today >= trigger_point
      :lapsed
    else
      :due  # Within 7 days of trigger point
    end
  end
end
```

#### Example Tests:
```ruby
# spec/models/projection_spec.rb
describe Projection do
  describe "#status_classification" do
    context "when both date and km thresholds are set" do
      it "returns :upcoming when far from the trigger point" do
        projection = build(:projection,
          due_date: 30.days.from_now,
          km_due: 3000
        )
        
        expect(projection.status_classification).to eq(:upcoming)
      end
      
      it "returns :due when within 7 days of the trigger point" do
        projection = build(:projection,
          due_date: 3.days.from_now,
          km_due: 3000
        )
        
        expect(projection.status_classification).to eq(:due)
      end
      
      it "returns :lapsed when past the trigger point" do
        projection = build(:projection,
          due_date: 10.days.ago,
          km_due: 3000
        )
        
        expect(projection.status_classification).to eq(:lapsed)
      end
      
      it "uses whichever comes first (date or km)" do
        # Date comes first
        projection = build(:projection,
          due_date: 5.days.from_now,
          km_due: 50000  # Far in the future
        )
        expect(projection.status_classification).to eq(:due)
        
        # Km comes first
        projection = build(:projection,
          due_date: 50.days.from_now,
          km_due: 100  # Already passed
        )
        expect(projection.status_classification).to eq(:lapsed)
      end
    end
    
    context "when date or km is missing" do
      it "returns :incomplete if due_date is nil" do
        projection = build(:projection, due_date: nil, km_due: 3000)
        expect(projection.status_classification).to eq(:incomplete)
      end
      
      it "returns :incomplete if km_due is nil" do
        projection = build(:projection, due_date: 30.days.from_now, km_due: nil)
        expect(projection.status_classification).to eq(:incomplete)
      end
    end
  end
end
```

### 2.7 Using Factory Bot — Creating Test Data Efficiently

Factory Bot replaces hardcoded test fixtures with reusable "recipes" for building test objects.

#### Basic Factory Definition:
```ruby
# spec/factories/vehicle.rb
FactoryBot.define do
  factory :vehicle do
    registration_number { Faker::Vehicle.license_plate }
    owner_type { "RB" }
    model { association :model }  # Creates a related Model too
    invoice_date { 6.months.ago }
    projection_status { "PENDING_INVOICE" }
  end
end
```

#### Using Factories in Tests:

```ruby
# Create and save to database
vehicle = create(:vehicle)

# Build but don't save (faster, for simple tests)
vehicle = build(:vehicle)

# Build but stub database methods (for pure logic tests)
vehicle = build_stubbed(:vehicle)

# Create multiple at once
vehicles = create_list(:vehicle, 5)

# Override specific attributes
gearless_vehicle = create(:vehicle, 
  transmission_type: "gearless",
  model: create(:model, name: "Honda Activa")
)
```

#### Traits — Reusable Variations:

Rather than creating a new factory for every variation, use traits:

```ruby
# spec/factories/vehicle.rb
FactoryBot.define do
  factory :vehicle do
    # ... base attributes ...
    
    trait :gearless do
      model { association :model, :gearless }
    end
    
    trait :geared do
      model { association :model, :geared }
    end
    
    trait :mbsi do
      model { association :model, :mbsi }
      owner_type { "MBSI" }
    end
    
    trait :overdue_for_service do
      association :projection, :lapsed
    end
    
    trait :projected do
      projection_status { "PROJECTED" }
    end
  end
  
  factory :model do
    name { "Honda Activa 6G" }
    transmission_type { "gearless" }
    
    trait :gearless do
      transmission_type { "gearless" }
    end
    
    trait :geared do
      transmission_type { "geared" }
    end
    
    trait :mbsi do
      owner_type { "MBSI" }
    end
  end
end
```

#### Using Traits in Tests:

```ruby
# Create a gearless, projected vehicle
vehicle = create(:vehicle, :gearless, :projected)

# Create multiple with a trait
gearless_vehicles = create_list(:vehicle, 3, :gearless)

# Combine traits
lapsed_geared_vehicle = create(:vehicle, :geared, :overdue_for_service)
```

### 2.8 Mocking and Stubbing External Dependencies

Real code often depends on things outside your control:
- Current time (Date.today, Time.now)
- External APIs (invoice scanning service, SMS provider)
- Randomness
- File operations

In tests, you want to isolate your logic from these. Use mocking:

#### Stubbing Time (Very Common):

```ruby
# app/models/projection.rb
class Projection < ApplicationRecord
  def days_until_due
    (due_date - Date.today).to_i
  end
end
```

```ruby
# spec/models/projection_spec.rb
describe "#days_until_due" do
  it "calculates the number of days correctly" do
    # Freeze time at a specific point
    travel_to Date.new(2026, 8, 19) do
      projection = build(:projection, due_date: Date.new(2026, 8, 29))
      
      expect(projection.days_until_due).to eq(10)
    end
  end
  
  it "handles past dates" do
    travel_to Date.new(2026, 8, 29) do
      projection = build(:projection, due_date: Date.new(2026, 8, 19))
      
      expect(projection.days_until_due).to eq(-10)
    end
  end
end
```

The `travel_to` method (built into Rails tests) freezes time at a specific date. Your code sees `Date.today` as that frozen date.

#### Stubbing HTTP Calls (Using WebMock):

```ruby
# app/services/invoice_scanner.rb
class InvoiceScanner
  def self.extract_date(invoice_id)
    response = HTTParty.get("https://invoice-api.example.com/scan/#{invoice_id}")
    response['date']
  end
end
```

```ruby
# spec/services/invoice_scanner_spec.rb
describe InvoiceScanner do
  describe ".extract_date" do
    it "extracts the date from an invoice" do
      # Stub the HTTP call to return a fake response
      stub_request(:get, "https://invoice-api.example.com/scan/12345")
        .to_return(body: { date: "2026-08-15" }.to_json)
      
      result = InvoiceScanner.extract_date(12345)
      
      expect(result).to eq("2026-08-15")
    end
    
    it "handles API errors gracefully" do
      stub_request(:get, "https://invoice-api.example.com/scan/invalid")
        .to_return(status: 500, body: "Server Error")
      
      expect {
        InvoiceScanner.extract_date("invalid")
      }.to raise_error(StandardError)
    end
  end
end
```

#### Stubbing Method Calls:

Sometimes you want to replace a method's behavior entirely:

```ruby
# app/models/vehicle.rb
class Vehicle < ApplicationRecord
  def update_projection
    calculator = ProjectionCalculator.new(self)
    new_projection = calculator.calculate  # Call external service
    self.projection = new_projection
  end
end
```

```ruby
# spec/models/vehicle_spec.rb
describe "#update_projection" do
  it "stores the calculated projection" do
    vehicle = build(:vehicle)
    fake_projection = build(:projection)
    
    # Stub the entire calculator
    allow_any_instance_of(ProjectionCalculator)
      .to receive(:calculate)
      .and_return(fake_projection)
    
    vehicle.update_projection
    
    expect(vehicle.projection).to eq(fake_projection)
  end
end
```

### 2.9 Testing Callbacks and Hooks

ActiveRecord callbacks (before_save, after_create, etc.) are tested by triggering the action that fires the callback:

```ruby
# app/models/vehicle.rb
class Vehicle < ApplicationRecord
  after_create :trigger_projection
  
  private
  
  def trigger_projection
    ProjectServiceJob.perform_later(self.id)
  end
end
```

```ruby
# spec/models/vehicle_spec.rb
describe "callbacks" do
  it "schedules a projection job when the vehicle is created" do
    expect {
      create(:vehicle)
    }.to have_enqueued_job(ProjectServiceJob)
  end
end
```

### 2.10 Testing Services/POROs (Business Logic Classes)

Not all business logic belongs in models. Complex calculations or workflows often live in standalone classes (POROs — Plain Old Ruby Objects):

```ruby
# app/services/projection_calculator.rb
class ProjectionCalculator
  def initialize(vehicle)
    @vehicle = vehicle
    @model_rule = vehicle.model.service_rule
  end
  
  def calculate
    raise "Vehicle must have a model rule" unless @model_rule
    
    case @model_rule.vehicle_type
    when "gearless"
      calculate_gearless_projection
    when "geared"
      calculate_geared_projection
    when "mbsi"
      calculate_mbsi_projection
    else
      raise "Unknown vehicle type: #{@model_rule.vehicle_type}"
    end
  end
  
  private
  
  def calculate_gearless_projection
    # Complex logic here
  end
end
```

#### Example Tests:

```ruby
# spec/services/projection_calculator_spec.rb
describe ProjectionCalculator do
  describe "#calculate" do
    context "for a gearless vehicle" do
      it "calculates the next service based on the ladder" do
        gearless_model = create(:model, :gearless)
        vehicle = build_stubbed(:vehicle, model: gearless_model)
        calculator = ProjectionCalculator.new(vehicle)
        
        result = calculator.calculate
        
        expect(result.due_date).to be_present
        expect(result.km_due).to be_present
      end
    end
    
    context "when the vehicle has no model rule" do
      it "raises an error" do
        vehicle = build_stubbed(:vehicle)
        allow(vehicle.model).to receive(:service_rule).and_return(nil)
        calculator = ProjectionCalculator.new(vehicle)
        
        expect {
          calculator.calculate
        }.to raise_error("Vehicle must have a model rule")
      end
    end
  end
end
```

### 2.11 Best Practices for Unit Tests

#### 1. One Assertion Per Test (Conceptually)
```ruby
# Good — focuses on one behavior
it "returns true for valid vehicles" do
  vehicle = build(:vehicle)
  expect(vehicle.valid?).to be(true)
end

# Avoid — mixes multiple unrelated checks
it "validates the vehicle" do
  vehicle = build(:vehicle)
  expect(vehicle).to be_valid
  expect(vehicle.registration_number).to be_present
  expect(vehicle.model).to be_present
  expect(vehicle.owner_type).to eq("RB")
end
```

(Exception: Multiple assertions checking related aspects of one outcome are fine.)

#### 2. Use Descriptive Test Names
```ruby
# Good — tells you exactly what's being tested
it "marks a service as lapsed if not completed by the due date"

# Avoid — too vague
it "checks lapse status"
```

#### 3. Avoid Test Interdependencies
```ruby
# Bad — tests depend on execution order
describe "vehicles" do
  it "can be created" do
    @vehicle = create(:vehicle)
    expect(@vehicle).to be_persisted
  end
  
  it "can be updated" do
    @vehicle.update(owner_type: "MBSI")  # Depends on @vehicle from previous test
    expect(@vehicle.owner_type).to eq("MBSI")
  end
end

# Good — each test is independent
describe "vehicles" do
  it "can be created" do
    vehicle = create(:vehicle)
    expect(vehicle).to be_persisted
  end
  
  it "can be updated" do
    vehicle = create(:vehicle)
    vehicle.update(owner_type: "MBSI")
    expect(vehicle.owner_type).to eq("MBSI")
  end
end
```

#### 4. Keep Tests Fast
- Use `build_stubbed` for pure logic tests (doesn't hit database)
- Avoid unnecessary creates; use build when you can
- Don't make HTTP calls; stub them
- Run tests frequently to catch regressions early

#### 5. Organize with Nested Contexts
```ruby
# Good — logical grouping, easy to find related tests
describe Projection do
  describe "#status_classification" do
    context "when both date and km thresholds are set" do
      # ... tests ...
    end
    
    context "when only date is set" do
      # ... tests ...
    end
  end
end
```

### 2.12 Code Coverage — Measuring Test Completeness

After running tests with SimpleCov, you get a coverage report showing which code is exercised:

```bash
bundle exec rspec
open coverage/index.html
```

**Coverage targets:**
- **Models:** 90%+ (these contain business logic; high coverage is crucial)
- **Services/POROs:** 90%+ (same reason)
- **Controllers/Requests:** 80%+ (integration tests often cover these)
- **Don't aim for 100%** — some code is genuinely hard to test (error handling in edge cases) and the effort-to-value ratio gets poor

Focus on:
- ✅ All business logic paths
- ✅ Error/edge cases
- ❌ Trivial getters/setters
- ❌ Boilerplate Rails configuration

---


## 3. Integration Testing in Rails — Request/Controller Testing

### 3.1 Overview: What Integration Tests Do

An integration test (called a "request spec" in RSpec) simulates a real HTTP request coming into your Rails application and verifies:
1. The request is routed correctly
2. The controller/action executes without error
3. Database changes happen as expected
4. The response (status, headers, body) is correct

Unlike unit tests, integration tests exercise multiple layers at once: routing → controller → model → database → response. This catches problems that unit tests can't — like a perfectly correct model method that's never actually called because of a missing route.

### 3.2 Basic Structure of an Integration Test

Integration tests live in `spec/requests/` and follow the same Arrange-Act-Assert pattern:

```ruby
# spec/requests/vehicle_registrations_spec.rb
describe "Vehicle Registration", type: :request do
  describe "POST /vehicles" do
    it "creates a new vehicle" do
      # Arrange
      params = {
        registration_number: "KA01AB1234",
        model_id: create(:model).id,
        owner_type: "RB",
        invoice_date: "2026-08-15"
      }
      
      # Act
      post "/vehicles", params: params
      
      # Assert
      expect(response).to have_http_status(:created)
      expect(Vehicle.count).to eq(1)
      expect(Vehicle.last.registration_number).to eq("KA01AB1234")
    end
  end
end
```

**Key differences from unit tests:**
- Uses `post`, `get`, `patch`, `delete` to simulate HTTP verbs
- Checks `response` status and body
- Verifies database state after the request
- Tests the actual URL path, not just the method

### 3.3 Testing HTTP Requests — The Five Verbs

Rails applications handle five main HTTP verbs. Here's how to test each:

#### GET — Retrieving Data

```ruby
describe "GET /vehicles/:id" do
  it "returns the vehicle details" do
    vehicle = create(:vehicle)
    
    get "/vehicles/#{vehicle.id}"
    
    expect(response).to have_http_status(:ok)
    # Response body is automatically parsed from JSON
  end
  
  it "returns 404 if vehicle not found" do
    get "/vehicles/999"
    
    expect(response).to have_http_status(:not_found)
  end
end

describe "GET /vehicles" do
  it "returns a list of all vehicles" do
    vehicle1 = create(:vehicle)
    vehicle2 = create(:vehicle)
    
    get "/vehicles"
    
    expect(response).to have_http_status(:ok)
    # We'll cover response body checking in section 3.5
  end
end
```

#### POST — Creating Data

```ruby
describe "POST /vehicles" do
  it "creates a vehicle with valid params" do
    model = create(:model)
    
    expect {
      post "/vehicles", params: {
        registration_number: "KA01AB1234",
        model_id: model.id,
        owner_type: "RB",
        invoice_date: "2026-08-15"
      }
    }.to change { Vehicle.count }.by(1)
    
    expect(response).to have_http_status(:created)
  end
  
  it "rejects invalid params" do
    expect {
      post "/vehicles", params: {
        registration_number: "",  # Empty, should fail validation
        model_id: 999
      }
    }.not_to change { Vehicle.count }
    
    expect(response).to have_http_status(:unprocessable_entity)
  end
end
```

#### PATCH — Updating Existing Data

```ruby
describe "PATCH /vehicles/:id" do
  it "updates the vehicle" do
    vehicle = create(:vehicle, owner_type: "RB")
    
    patch "/vehicles/#{vehicle.id}", params: {
      owner_type: "MBSI"
    }
    
    expect(response).to have_http_status(:ok)
    expect(vehicle.reload.owner_type).to eq("MBSI")
  end
  
  it "returns 404 if vehicle doesn't exist" do
    patch "/vehicles/999", params: { owner_type: "MBSI" }
    
    expect(response).to have_http_status(:not_found)
  end
end
```

#### DELETE — Removing Data

```ruby
describe "DELETE /vehicles/:id" do
  it "deletes the vehicle" do
    vehicle = create(:vehicle)
    
    expect {
      delete "/vehicles/#{vehicle.id}"
    }.to change { Vehicle.count }.by(-1)
    
    expect(response).to have_http_status(:no_content)
  end
  
  it "returns 404 if vehicle doesn't exist" do
    delete "/vehicles/999"
    
    expect(response).to have_http_status(:not_found)
  end
end
```

### 3.4 Testing Request Parameters and Headers

Requests can include:
- **Query string** (`?page=1&sort=date`)
- **URL path parameters** (`/vehicles/123`)
- **Body parameters** (POST/PATCH)
- **Headers** (authentication, content-type, etc.)

#### Query String Parameters:

```ruby
describe "GET /vehicles?owner_type=RB" do
  it "filters vehicles by owner type" do
    rb_vehicle = create(:vehicle, owner_type: "RB")
    mbsi_vehicle = create(:vehicle, owner_type: "MBSI")
    
    get "/vehicles", params: { owner_type: "RB" }
    
    expect(response).to have_http_status(:ok)
    # Verify the response contains only RB vehicles (see 3.5)
  end
end
```

#### Path Parameters (Captured from URL):

```ruby
# Route: /vehicles/:id/projections/:projection_id
describe "GET /vehicles/:id/projections/:projection_id" do
  it "returns the specific projection for a vehicle" do
    vehicle = create(:vehicle)
    projection = create(:projection, vehicle: vehicle)
    
    get "/vehicles/#{vehicle.id}/projections/#{projection.id}"
    
    expect(response).to have_http_status(:ok)
  end
end
```

#### Request Headers:

```ruby
describe "POST /vehicles" do
  it "accepts authorization headers" do
    model = create(:model)
    
    post "/vehicles", params: {
      registration_number: "KA01AB1234",
      model_id: model.id
    }, headers: {
      "Authorization" => "Bearer token123",
      "Content-Type" => "application/json"
    }
    
    expect(response).to have_http_status(:created)
  end
end
```

### 3.5 Testing Response Status Codes

HTTP status codes tell the client whether the request succeeded, and if not, why. Test them explicitly:

```ruby
describe "Response status codes" do
  describe "2xx Success" do
    it "returns 200 OK for successful GET" do
      vehicle = create(:vehicle)
      get "/vehicles/#{vehicle.id}"
      expect(response).to have_http_status(:ok)
      # OR: expect(response.status).to eq(200)
    end
    
    it "returns 201 Created for successful POST" do
      post "/vehicles", params: { registration_number: "KA01AB1234" }
      expect(response).to have_http_status(:created)  # 201
    end
    
    it "returns 204 No Content for DELETE" do
      vehicle = create(:vehicle)
      delete "/vehicles/#{vehicle.id}"
      expect(response).to have_http_status(:no_content)  # 204
    end
  end
  
  describe "4xx Client Errors" do
    it "returns 400 Bad Request for malformed data" do
      post "/vehicles", params: { registration_number: nil }
      expect(response).to have_http_status(:bad_request)  # 400
    end
    
    it "returns 401 Unauthorized if no auth token" do
      get "/admin/dashboard", headers: {}
      expect(response).to have_http_status(:unauthorized)  # 401
    end
    
    it "returns 404 Not Found for missing resource" do
      get "/vehicles/999"
      expect(response).to have_http_status(:not_found)  # 404
    end
    
    it "returns 422 Unprocessable Entity for validation failure" do
      post "/vehicles", params: { registration_number: "" }
      expect(response).to have_http_status(:unprocessable_entity)  # 422
    end
  end
  
  describe "5xx Server Errors" do
    it "returns 500 Internal Server Error on exception" do
      # This usually happens if the controller has a bug
      # Test it by triggering the bug scenario
      allow(Vehicle).to receive(:create!).and_raise(StandardError)
      
      post "/vehicles", params: { registration_number: "KA01AB1234" }
      expect(response).to have_http_status(:internal_server_error)  # 500
    end
  end
end
```

Common status code symbols in RSpec:
- `:ok` (200), `:created` (201), `:no_content` (204)
- `:bad_request` (400), `:unauthorized` (401), `:forbidden` (403), `:not_found` (404)
- `:unprocessable_entity` (422)
- `:internal_server_error` (500)

### 3.6 Testing Response Body — JSON Structure and Content

For APIs, the response body (usually JSON) is as important as the status code:

```ruby
# app/controllers/vehicles_controller.rb
class VehiclesController < ApplicationController
  def show
    vehicle = Vehicle.find(params[:id])
    render json: {
      id: vehicle.id,
      registration_number: vehicle.registration_number,
      owner_type: vehicle.owner_type,
      model: {
        id: vehicle.model.id,
        name: vehicle.model.name
      }
    }
  end
  
  def index
    vehicles = Vehicle.all
    render json: vehicles, each_serializer: VehicleSerializer
  end
end
```

#### Testing JSON Response Structure:

```ruby
# spec/requests/vehicles_spec.rb
describe "GET /vehicles/:id" do
  it "returns the vehicle as JSON with the correct structure" do
    model = create(:model, name: "Honda Activa")
    vehicle = create(:vehicle, 
      registration_number: "KA01AB1234",
      owner_type: "RB",
      model: model
    )
    
    get "/vehicles/#{vehicle.id}"
    
    expect(response).to have_http_status(:ok)
    
    # Parse the JSON response
    json_response = JSON.parse(response.body)
    
    # Check structure
    expect(json_response).to have_key("id")
    expect(json_response).to have_key("registration_number")
    expect(json_response).to have_key("owner_type")
    expect(json_response).to have_key("model")
    expect(json_response["model"]).to have_key("id")
    expect(json_response["model"]).to have_key("name")
  end
  
  it "returns the correct vehicle data" do
    model = create(:model, name: "Honda Activa")
    vehicle = create(:vehicle, 
      registration_number: "KA01AB1234",
      owner_type: "RB",
      model: model
    )
    
    get "/vehicles/#{vehicle.id}"
    
    json_response = JSON.parse(response.body)
    
    # Check values
    expect(json_response["registration_number"]).to eq("KA01AB1234")
    expect(json_response["owner_type"]).to eq("RB")
    expect(json_response["model"]["name"]).to eq("Honda Activa")
  end
end
```

#### Testing JSON Arrays:

```ruby
describe "GET /vehicles" do
  it "returns an array of vehicles" do
    vehicle1 = create(:vehicle, registration_number: "KA01AB1234")
    vehicle2 = create(:vehicle, registration_number: "KA01AB5678")
    
    get "/vehicles"
    
    json_response = JSON.parse(response.body)
    
    expect(json_response).to be_an(Array)
    expect(json_response.length).to eq(2)
    expect(json_response.map { |v| v["registration_number"] })
      .to contain_exactly("KA01AB1234", "KA01AB5678")
  end
  
  it "returns an empty array when no vehicles exist" do
    get "/vehicles"
    
    json_response = JSON.parse(response.body)
    expect(json_response).to eq([])
  end
end
```

#### Using Response Matcher (Cleaner):

RSpec has a built-in matcher for JSON responses:

```ruby
describe "GET /vehicles/:id" do
  it "returns the vehicle as JSON" do
    model = create(:model, name: "Honda Activa")
    vehicle = create(:vehicle, 
      registration_number: "KA01AB1234",
      model: model
    )
    
    get "/vehicles/#{vehicle.id}"
    
    expect(response).to have_http_status(:ok)
    expect(response.body).to include_json(
      id: vehicle.id,
      registration_number: "KA01AB1234",
      model: {
        name: "Honda Activa"
      }
    )
  end
end
```

(Requires the `rspec_json_expectations` gem for `include_json`. Alternatively, manual parsing with `JSON.parse` is more standard.)

### 3.7 Testing State Changes — Database Verification

Integration tests verify that the request not only returns the right response, but also causes the right database changes:

```ruby
describe "POST /vehicles" do
  it "creates a vehicle and schedules projection job" do
    model = create(:model)
    
    expect {
      post "/vehicles", params: {
        registration_number: "KA01AB1234",
        model_id: model.id,
        owner_type: "RB",
        invoice_date: "2026-08-15"
      }
    }.to change { Vehicle.count }.by(1)
    
    # Verify the vehicle was created correctly
    vehicle = Vehicle.last
    expect(vehicle.registration_number).to eq("KA01AB1234")
    expect(vehicle.owner_type).to eq("RB")
    expect(vehicle.invoice_date).to eq(Date.new(2026, 8, 15))
  end
  
  it "schedules a background projection job" do
    model = create(:model)
    
    expect {
      post "/vehicles", params: {
        registration_number: "KA01AB1234",
        model_id: model.id
      }
    }.to have_enqueued_job(ProjectServiceJob)
  end
end

describe "PATCH /invoices/:id/map_to_vehicle" do
  it "updates the projection status and schedules re-calculation" do
    vehicle = create(:vehicle, projection_status: "PENDING_INVOICE")
    invoice = create(:invoice)
    
    expect {
      patch "/invoices/#{invoice.id}/map_to_vehicle", params: {
        vehicle_id: vehicle.id,
        invoice_date: "2026-08-15"
      }
    }.to change { vehicle.reload.projection_status }.to("PROJECTED")
    
    # Verify the background job was scheduled
    expect(ProjectServiceJob).to have_been_enqueued
  end
  
  it "does not create duplicate projections" do
    vehicle = create(:vehicle, projection_status: "PENDING_INVOICE")
    invoice = create(:invoice)
    
    patch "/invoices/#{invoice.id}/map_to_vehicle", params: {
      vehicle_id: vehicle.id
    }
    
    expect(Projection.where(vehicle: vehicle).count).to eq(1)
  end
end
```

### 3.8 Common Integration Test Patterns

#### Pattern: Going Live and Mapping Invoices are Independent

One of your key requirements is that vehicle go-live and invoice mapping can fail independently. Test this:

```ruby
describe "Vehicle Go-Live and Invoice Mapping (independence)" do
  it "can go live even if projection calculation fails later" do
    # Go-live should succeed immediately
    post "/vehicles", params: {
      registration_number: "KA01AB1234",
      model_id: create(:model).id
    }
    
    expect(response).to have_http_status(:created)
    vehicle = Vehicle.last
    expect(vehicle.projection_status).to eq("PENDING_INVOICE")
    
    # Later, mapping invoice fails
    allow(ProjectionCalculator).to receive(:new)
      .and_raise(StandardError, "Calculation failed")
    
    patch "/invoices", params: {
      vehicle_id: vehicle.id,
      invoice_date: "2026-08-15"
    }
    
    # Mapping fails, but the vehicle still exists as "live"
    expect(response).to have_http_status(:internal_server_error)
    expect(vehicle.reload).to be_persisted
    expect(vehicle.projection_status).to eq("PENDING_INVOICE")
  end
end
```

#### Pattern: Filtering and Pagination

Many endpoints support filtering and pagination:

```ruby
describe "GET /vehicles with filtering and pagination" do
  before do
    create_list(:vehicle, 15, owner_type: "RB")
    create_list(:vehicle, 10, owner_type: "MBSI")
  end
  
  it "filters by owner type" do
    get "/vehicles", params: { owner_type: "RB" }
    
    json_response = JSON.parse(response.body)
    expect(json_response.length).to eq(15)
    expect(json_response.all? { |v| v["owner_type"] == "RB" }).to be(true)
  end
  
  it "paginates results" do
    get "/vehicles", params: { page: 1, per_page: 10 }
    
    json_response = JSON.parse(response.body)
    expect(json_response.length).to eq(10)
    
    get "/vehicles", params: { page: 2, per_page: 10 }
    
    json_response = JSON.parse(response.body)
    expect(json_response.length).to eq(10)
  end
end
```

#### Pattern: Authorization and Authentication

Test that endpoints enforce access control:

```ruby
describe "Authorization" do
  it "blocks unauthenticated requests to admin endpoints" do
    get "/admin/vehicles"
    
    expect(response).to have_http_status(:unauthorized)
  end
  
  it "allows authenticated users" do
    user = create(:user, role: "admin")
    
    get "/admin/vehicles", headers: {
      "Authorization" => "Bearer #{user.auth_token}"
    }
    
    expect(response).to have_http_status(:ok)
  end
  
  it "prevents non-admin users from accessing admin endpoints" do
    user = create(:user, role: "operator")
    
    get "/admin/vehicles", headers: {
      "Authorization" => "Bearer #{user.auth_token}"
    }
    
    expect(response).to have_http_status(:forbidden)
  end
end
```

### 3.9 Testing Error Scenarios

Tests should verify that errors are handled gracefully and predictably:

```ruby
describe "Error Handling" do
  it "returns appropriate error message for validation failure" do
    post "/vehicles", params: {
      registration_number: "",  # Invalid
      model_id: 999
    }
    
    expect(response).to have_http_status(:unprocessable_entity)
    json_response = JSON.parse(response.body)
    expect(json_response["errors"]).to be_present
    expect(json_response["errors"]["registration_number"]).to include("can't be blank")
  end
  
  it "returns 404 instead of 500 for missing resource" do
    get "/vehicles/999"
    
    expect(response).to have_http_status(:not_found)
    json_response = JSON.parse(response.body)
    expect(json_response["error"]).to include("not found")
  end
  
  it "handles database constraint violations" do
    vehicle1 = create(:vehicle, registration_number: "KA01AB1234")
    
    post "/vehicles", params: {
      registration_number: "KA01AB1234",  # Duplicate
      model_id: create(:model).id
    }
    
    expect(response).to have_http_status(:unprocessable_entity)
    json_response = JSON.parse(response.body)
    expect(json_response["errors"]["registration_number"])
      .to include("has already been taken")
  end
end
```

### 3.10 Best Practices for Integration Tests

#### 1. Test Happy Path and Key Error Paths
```ruby
# Good — covers success and main failure modes
describe "POST /vehicles" do
  it "creates a vehicle successfully" do
    # happy path
  end
  
  it "rejects duplicate registration numbers" do
    # key error: business constraint violation
  end
  
  it "rejects missing required fields" do
    # key error: validation failure
  end
end

# Avoid — testing every conceivable error
describe "POST /vehicles" do
  it "rejects registration_number as nil" { }
  it "rejects registration_number as empty string" { }
  it "rejects registration_number with 100 chars" { }
  it "rejects registration_number with special chars" { }
  # ... 20 more variations
end
```

#### 2. Use Factories to Build Realistic Scenarios
```ruby
# Good — realistic data
it "projects service correctly for a gearless vehicle" do
  model = create(:model, :gearless)
  vehicle = create(:vehicle, model: model, invoice_date: 6.months.ago)
  
  post "/vehicles/#{vehicle.id}/project_service"
  
  expect(response).to have_http_status(:ok)
end

# Avoid — minimal/fake data
it "projects service" do
  post "/vehicles/1/project_service"
  expect(response).to have_http_status(:ok)
end
```

#### 3. One Scenario Per Test
```ruby
# Good — each test is independent and clear
it "filters vehicles by owner type" do
  # ... test filtering by owner_type only
end

it "filters vehicles by registration number prefix" do
  # ... test filtering by registration prefix only
end

# Avoid — mixing multiple scenarios
it "filters and sorts vehicles" do
  # ... filter by owner_type AND sort by registration AND paginate
  # If it fails, which part was wrong?
end
```

#### 4. Clean Up After Yourself
Most of the time, Rails transactions handle this automatically, but for side effects (files, external APIs):

```ruby
# Good — clean up explicitly
it "exports vehicles to CSV" do
  create(:vehicle)
  
  get "/vehicles/export.csv"
  
  expect(response).to have_http_status(:ok)
  
  # Clean up the temp file if you created one
  # (Usually handled automatically by Rails)
end
```

#### 5. Keep Integration Tests Focused, Not Exhaustive
```ruby
# Good — integration tests for key workflows
describe "Vehicle Registration Workflow" do
  it "allows BD to register a vehicle" do
    # ... full workflow test
  end
end

# Model tests verify all the business logic details separately
describe Vehicle do
  it "validates uniqueness of registration_number"
  it "requires owner_type to be RB or MBSI"
  # ... 20 more model-level tests
end
```

### 3.11 Structuring Request Specs for Your App

For the Service Projections app, organize your request specs by feature/endpoint:

```
spec/requests/
├── vehicles_spec.rb              # GET/POST/PATCH vehicles
├── invoice_mappings_spec.rb      # POST /invoices/:id/map_to_vehicle
├── projections_spec.rb           # GET /projections, /vehicles/:id/projection
├── admin/
│   ├── model_rules_spec.rb       # Admin CRUD for model rules
│   └── dashboard_spec.rb         # Admin dashboard endpoints
└── shared_examples/
    ├── authentication.rb         # Shared auth tests
    └── error_handling.rb         # Shared error response tests
```

#### Example: Shared Authentication Tests

```ruby
# spec/requests/shared_examples/authentication.rb
RSpec.shared_examples "requires authentication" do
  it "returns 401 for unauthenticated request" do
    make_request  # This is provided by the including spec

    expect(response).to have_http_status(:unauthorized)
  end
  
  it "allows authenticated request" do
    user = create(:user)
    make_request(headers: { "Authorization" => "Bearer #{user.token}" })

    expect(response).not_to have_http_status(:unauthorized)
  end
end

# spec/requests/admin/model_rules_spec.rb
describe "Admin Model Rules", type: :request do
  describe "DELETE /admin/model_rules/:id" do
    it_behaves_like "requires authentication" do
      def make_request(headers: {})
        delete "/admin/model_rules/1", headers: headers
      end
    end
  end
end
```

---

## 4. Background Job / Worker Testing

### 4.1 Overview

Background processing is a core part of the Service Projections app: projections are calculated in the background, the sweeper periodically scans the fleet, and notification workers alert users when a projection enters a buffer window. Tests for jobs focus on two things:
- Was the job enqueued when it should have been? (contract test)
- When the job runs, does it produce the expected side effects? (behavior test)

Jobs can be tested at both levels without touching the real queueing backend (Sidekiq/Resque) using the Rails test adapters or Sidekiq's testing helpers.

### 4.2 Configure Test Adapters

For ActiveJob (Rails' abstraction over background backends) use the test adapter in `spec/rails_helper.rb` or per-spec:

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.before(:each) do
    ActiveJob::Base.queue_adapter = :test
  end
end
```

This prevents jobs from actually running in a background worker and enables matchers like `have_enqueued_job`.

If using Sidekiq directly, in your test helper:

```ruby
# spec/rails_helper.rb
require 'sidekiq/testing'
Sidekiq::Testing.fake!   # Jobs are added to queues but not executed
# or
Sidekiq::Testing.inline! # Jobs are executed immediately when pushed
```

Use `fake!` for testing enqueueing and `inline!` for testing execution. Remember to reset the setting if you change it globally.

### 4.3 Asserting Jobs Were Enqueued

ActiveJob matcher examples:

```ruby
RSpec.describe "Invoice mapping" do
  it "schedules a projection job when an invoice is mapped" do
    vehicle = create(:vehicle, projection_status: "PENDING_INVOICE")

    expect {
      patch "/invoices/1/map_to_vehicle", params: { vehicle_id: vehicle.id }
    }.to have_enqueued_job(ProjectServiceJob).with(vehicle.id)
  end
end
```

For Sidekiq (fake mode):

```ruby
RSpec.describe InvoiceMappingWorker do
  it "pushes a job to Sidekiq" do
    InvoiceMappingWorker.perform_async(1, 2)
    expect(InvoiceMappingWorker.jobs.size).to eq(1)
  end
end
```

### 4.4 Executing Jobs in Tests (Behavior Tests)

To verify side effects (database updates, HTTP calls, scheduled notifications), run the job inline in the test.

ActiveJob / RSpec pattern:

```ruby
RSpec.describe ProjectServiceJob, type: :job do
  include ActiveJob::TestHelper

  it "calculates a projection and stores it" do
    vehicle = create(:vehicle)

    perform_enqueued_jobs do
      ProjectServiceJob.perform_later(vehicle.id)
    end

    # Inspect side effects
    expect(Projection.where(vehicle: vehicle).exists?).to be(true)
  end
end
```

Sidekiq pattern with inline execution:

```ruby
require 'sidekiq/testing'
Sidekiq::Testing.inline!

RSpec.describe ProjectionWorker do
  it "calculates projections when run" do
    vehicle = create(:vehicle)

    ProjectionWorker.perform_async(vehicle.id)

    expect(vehicle.reload.projection_status).to eq("PROJECTED")
  end
end
```

### 4.5 Testing Scheduled / Periodic Jobs (Sweeper)

The sweeper (scheduled job) should be tested by calling the worker's perform method directly with controlled test data. Treat it like a service test that happens to be executed by a scheduler in production.

```ruby
RSpec.describe ProjectionSweeperJob, type: :job do
  it "scans vehicles and schedules recalculations for missed ones" do
    # Setup: vehicles in various states
    missed = create_list(:vehicle, 3, projection_status: "PENDING")
    healthy = create_list(:vehicle, 2, projection_status: "PROJECTED")

    perform_enqueued_jobs do
      ProjectionSweeperJob.perform_now
    end

    # Sweeper should have enqueued or created projections for missed vehicles
    missed.each do |v|
      expect(v.reload.projection_status).to eq("PROJECTED").or eq("FAILED")
    end
  end
end
```

If the worker is invoked by a scheduler like sidekiq-cron or the OS cron, unit tests should call the worker directly. Integration tests (capable infra tests) can exercise the scheduler separately.

### 4.6 Testing Notification Workers

Notification workers generally send outbound messages (SMS, email, push). Don’t call real external services in tests — stub or capture them.

Example using a fake notification adapter or WebMock:

```ruby
RSpec.describe NotificationWorker, type: :job do
  include ActiveJob::TestHelper

  it "enqueues a notification for vehicles entering the buffer" do
    vehicle = create(:vehicle)
    projection = create(:projection, vehicle: vehicle, due_date: 3.days.from_now)

    perform_enqueued_jobs do
      NotificationWorker.perform_now
    end

    # Verify that a notification record was created (or that the external adapter was called)
    expect(Notification.where(vehicle: vehicle).count).to be >= 1
  end
end
```

Or assert an outbound HTTP call was made (WebMock):

```ruby
stub_request(:post, "https://sms.example.com/send").to_return(status: 200)

perform_enqueued_jobs do
  NotificationWorker.perform_now
end

expect(a_request(:post, "https://sms.example.com/send")).to have_been_made.once
```

### 4.7 Testing Retries and Failure Handling

To test retry behavior (backoff, max attempts), simulate failures and verify the retry count or status change:

```ruby
RSpec.describe UnstableWorker, type: :job do
  include ActiveJob::TestHelper

  it "retries on transient error" do
    allow(ExternalApi).to receive(:call).and_raise(TransientError)

    ActiveJob::Base.queue_adapter = :test

    expect {
      perform_enqueued_jobs do
        UnstableWorker.perform_later(1)
      end
    }.to raise_error(TransientError)

    # Many queue backends will re-enqueue; assert the retry logic is scheduled
    # For Sidekiq-specific retries, check Sidekiq::RetrySet or the job's retry count
  end
end
```

Sidekiq-specific retry tests can assert that the job ended up in the retry set when using `Sidekiq::Testing.fake!`, or use `Sidekiq::Testing.inline!` and check behavior on exception.

### 4.8 Common Pitfalls & How to Avoid Them

- Transactional tests and background jobs: If a job runs in a separate thread or process, it may not see records created inside an RSpec transaction. Solutions:
  - Use the test adapter (`ActiveJob::Base.queue_adapter = :test`) and `perform_enqueued_jobs` so jobs execute inline in the same thread.
  - For Sidekiq, use `Sidekiq::Testing.inline!` for tests that must execute jobs immediately.
  - When a job is queued in an `after_commit` callback, ensure the commit happens before asserting the job was enqueued — use `create(:record)` (not build) so callbacks run.
  - If needed, disable transactional fixtures for a test and use database_cleaner or explicit cleanup.

- Flaky tests from external services: Always stub HTTP calls (WebMock/VCR) or provide a fake adapter for sending SMS/email.

- Over-testing the queue: Prefer asserting that a job was enqueued, not that Sidekiq internals were exercised. The scheduling contract is usually enough; separate job tests can cover the job behavior itself.

### 4.9 Best Practices

- Assert enqueueing at the controller/request layer and behavior in the job spec itself. This keeps tests focused and easy to diagnose.
- Use `perform_enqueued_jobs` for behavior tests that need to execute jobs and assert side effects.
- Keep long-running job behavior small and testable — extract complex logic into POROs and test those as unit tests.
- For periodic/sweeper jobs, write focused tests that call `perform_now` and assert that the expected set of records were processed.
- Stub external calls and verify interactions with fake adapters where possible.

---

## 5. System / End-to-End Testing

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

## 6. Rails-Specific Testing Patterns and Critical Practices

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

### 6.10 Avoid the most common anti-patterns

Avoid these patterns in a Rails codebase:
- testing implementation details instead of behavior
- writing giant tests that do multiple unrelated things
- creating brittle tests tied to exact HTML structure
- depending on test order
- reusing global state across examples
- testing the same rule in five different layers unnecessarily
- relying on real external systems in CI

A good Rails test answers one business question clearly and fails for one valid reason.

### 6.11 The highest-value test mix for this app

For the Service Projections app, the most valuable test stack is:
- Unit tests for projection rules, service thresholds, owner-type logic, and lapse logic
- Request tests for register / map invoice / project service / status change flows
- Job tests for sweeper and notification workers
- A few system tests only for critical user journeys

This gives strong confidence without over-investing in slow browser-level tests.

### 6.12 Final principle

Keep the test suite readable, realistic, and focused on the actual business contract.

If a test does not protect a real business risk or a real lifecycle behavior, it is probably not worth keeping at high priority.

---

## 7. Common Testing Patterns, Anti-Patterns, and Coverage

### 7.1 Why a dedicated section is useful

Some of the guidance in this section overlaps with earlier material, but a short dedicated section is still useful because it captures the team standards that matter most in daily work:
- what good tests look like
- what to avoid
- how to measure whether the suite is strong enough

This helps keep the testing culture consistent as the app grows.

### 7.2 Good testing patterns to follow

Use these as default rules for the team:
- Test behavior, not implementation details.
- One test should answer one business question.
- Prefer realistic data over synthetic shortcuts.
- Use factories for variations and traits for common scenarios.
- Keep model tests focused on rules and edge cases.
- Keep request tests focused on contracts and state changes.
- Keep job tests focused on side effects and failure handling.
- Keep system tests limited to critical user journeys.

### 7.3 Common anti-patterns to avoid

Avoid these practices because they create slow, brittle, and low-value tests:
- asserting on private method calls instead of user-visible output
- testing the same rule in too many layers
- writing giant tests that mix multiple scenarios
- relying on global shared state between examples
- depending on database order or test execution order
- hard-coding fragile HTML or CSS selectors in system tests
- using real third-party APIs in CI
- testing current date/time without freezing it

If a test is hard to understand, it is usually too broad or too coupled to internals.

### 7.4 Coverage: useful, but not the only objective

We use `SimpleCov` to measure how much of the application code is exercised by tests. This is useful because it tells us whether key business logic is being covered consistently.

Example setup from Section 1:

```ruby
# spec/spec_helper.rb
require 'simplecov'

SimpleCov.start 'rails' do
  add_filter '/spec/'
  add_filter '/config/'
  add_filter '/db/'
end
```

### 7.5 Practical coverage guidance

Coverage is a helpful signal, but it should not become the sole goal.

For this project, a sensible focus is:
- model logic: high coverage
- service logic: high coverage
- projection/sweeper/notification code: high coverage
- request and job flows: strong but not necessarily 100%
- system tests: limited and selective

A suite with 90%+ coverage in the rule-heavy and background-job-heavy parts of the app is generally much more valuable than a suite that hits 100% of trivial code but misses the risky cases.

### 7.6 Recommended team policy

The team should follow this policy:
- Do not merge code that breaks a relevant existing test.
- Add tests when changing business logic, especially in projection rules and status transitions.
- Use coverage as a review aid, not as a target to chase blindly.
- Prefer meaningful business coverage over inflated percentages.

This keeps the suite useful, maintainable, and aligned with real production risk.

---
