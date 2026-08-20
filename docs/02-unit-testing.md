### 2.1 Overview: What Unit Tests Do

A unit test focuses on a relatively small unit of behavior and verifies that it behaves correctly under specific conditions.

In Rails, the "unit" is often a model, service object, PORO, or a specific method. A unit test may still interact with the test database when the behavior being tested depends on ActiveRecord, validations, associations, scopes, or persistence.

The goal is to keep the test focused on the behavior under examination and isolate unrelated external dependencies where practical.

Unit tests typically:

- Verify a specific rule, calculation, or behavior
- Cover expected, invalid, and edge-case scenarios
- Avoid unnecessary external dependencies such as real HTTP requests
- Use controlled time, randomness, or external service responses when needed
- Provide focused feedback when a particular behavior breaks

For example, a model validation test may use the test database, while a pure calculation can often use `build_stubbed` or plain Ruby objects without database access.


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
- `it` — defines a single example that describes one behavior or outcome
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

These `validate_*` lines use Shoulda Matchers to concisely verify common Rails validation behavior. They reduce boilerplate compared with manually constructing invalid records and inspecting errors. When the business rule has additional conditions or edge cases, write explicit examples for those cases.

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

Because query methods can depend on the current date/time, tests should control time when the boundary itself is important. See Section 2.8 for `travel_to`.

### 2.6 Testing Models — Custom Methods and Business Logic

This is where the core rules of your system get tested. In the Service Projections examples used throughout this guide, this includes methods that calculate projections, determine vehicle type rules, check lapse status, etc.

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

Use `build_stubbed` when the code under test only needs an object that behaves like a persisted record and does not require real database interaction.

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

### 2.8 Controlling Time and Stubbing External Dependencies

Real code often depends on things outside your control:

- Current time (Date.today, Time.now)
- External APIs (invoice scanning service, SMS provider)
- Randomness
- File operations

In tests, you want to isolate your logic from these. Use mocking:

#### Controlling Time in Tests (Very Common):

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

Don't test the third-party API itself. Test how your application behaves given the responses it expects from that API.

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
    calculator = instance_double(
      ProjectionCalculator,
      calculate: fake_projection
    )

    allow(ProjectionCalculator)
      .to receive(:new)
      .with(vehicle)
      .and_return(calculator)
    
    vehicle.update_projection
    
    expect(vehicle.projection).to eq(fake_projection)
  end
end
```

### 2.9 Testing Callbacks and Hooks

ActiveRecord callbacks (`before_save`, `after_create`, etc.) are tested by triggering the action that fires the callback:

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
Prefer testing the observable behavior caused by the callback rather than testing the callback method itself.


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

Service specs can exist at different levels of isolation. A service that contains pure calculations can often be tested with plain Ruby objects or doubles. A service that coordinates ActiveRecord objects may reasonably use factories and the test database. The important question is whether the test is focused on the service's behavior and its intended collaborators.

### 2.11 Best Practices for Unit Tests

#### 1. Focus Each Test on One Behavior

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

A test should answer one clear question. That may require multiple assertions when those assertions describe different aspects of the same outcome. For example, checking:

```ruby
expect(response).to have_http_status(:created)
expect(response.body).to include(vehicle.id.to_s)
```
can be perfectly reasonable if both verify the same behavior.


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
- Use `build` or `build_stubbed` when database persistence isn't required.
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

Coverage targets should be agreed upon based on the project's risk profile and testing strategy. High coverage is particularly valuable around business-critical models, services, calculations, and workflows, while a lower percentage may be reasonable for boilerplate or framework-driven code.

Focus on:

- ✅ All business logic paths
- ✅ Error/edge cases
- ❌ Trivial getters/setters
- ❌ Boilerplate Rails configuration

---
