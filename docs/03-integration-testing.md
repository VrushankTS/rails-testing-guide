## 3. Integration Testing in Rails — Request/Controller Testing

### 3.1 Overview: What Integration Tests Do

In Rails applications using RSpec, request specs are commonly used to test
the application through its HTTP interface.

A request spec sends an HTTP request to a Rails endpoint and verifies the
observable result of that request, such as:

1. The request reaches the expected endpoint.
2. The application processes the request correctly.
3. The appropriate database or other application state changes occur.
4. The response has the expected status, headers, and body.

Unlike a focused unit test, a request spec exercises several application
layers together. A typical flow may involve:

HTTP request → routing → controller → domain/model/service logic → database → response

This makes request specs useful for verifying that the different parts of
the application work together correctly.

### 3.2 Basic Structure of an Integration Test

Integration tests live in `spec/requests/` and follow the same Arrange-Act-Assert pattern:

```ruby
# spec/requests/vehicle_registrations_spec.rb
describe "Vehicle Registration", type: :request do
  describe "POST /vehicles" do
    it "creates a new vehicle" do
      # Arrange
      model = create(:model)

      params = {
        registration_number: "KA01AB1234",
        model_id: model.id,
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

Rails applications handle five main HTTP verbs. The following examples cover the HTTP methods most commonly used by Rails CRUD endpoints. Here's how to test each:

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
- **Query parameters** (`?page=1&sort=date`)
- **URL path parameters** (`/vehicles/123`)
- **Body parameters** (submitted with POST/PATCH/etc.)
- **Headers** (authentication, content-type, etc.)

#### Query Parameters:

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

#### Using a JSON Matcher (Cleaner):

A JSON matcher such as `include_json` can make response assertions more concise. The matcher shown below is provided by the `rspec_json_expectations` gem.

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

### 3.7 Testing the HTTP Contract

A request spec should primarily verify what a client of the application can observe:
- HTTP status
- response body and structure
- relevant headers
- persisted state resulting from the request
- externally visible side effects

Avoid coupling request specs to controller implementation details such as private methods, internal helper calls, or the exact sequence of method calls.

For example, if `POST /vehicles` creates a vehicle and schedules a projection job, the request spec should verify those observable outcomes rather than asserting that a particular controller method called `ProjectionCalculator` directly.


### 3.8 Testing State Changes — Database Verification

Integration tests verify that the request not only returns the right response, but also causes the right database changes:

```ruby
describe "POST /vehicles" do
  it "creates a vehicle" do
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

### 3.9 Common Integration Test Patterns

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

### 3.10 Testing Error Scenarios

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

### 3.11 Best Practices for Integration Tests

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
Rails transactional fixtures normally clean up database state between tests. Tests that create external side effects — temporary files, uploaded files, external resources, etc. — may require explicit cleanup.

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

### 3.12 Structuring Request Specs for Your App

Application-specific example: Service Projections request-spec organization by feature/endpoint:

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