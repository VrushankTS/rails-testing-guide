## 1. Testing Environment & Setup

### 1.1 The Rails Testing Ecosystem

A standard Rails project uses several layers of testing tools that work together. Here's what each does:

| Tool | Purpose | Installed by? |
|------|---------|---------------|
| **RSpec** | Test runner and assertion library (replaces Rails' built-in Minitest) | Explicitly added gem |
| **Factory Bot** | Generates test data on demand (instead of static fixtures) | Explicitly added gem |
| **Shoulda Matchers** | Pre-built assertions for common Rails patterns (validations, associations, callbacks) | Explicitly added gem |
| **VCR** | Records and replays HTTP interactions to avoid hitting real APIs during tests | Explicitly added gem |
| **Capybara** | Provides a browser-like DSL for system tests (visiting pages, clicking links, filling forms, etc.) | Added/configured as part of the system-testing stack
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
  gem 'vcr'                      # Record/replay HTTP interactions
  gem 'webmock'                  # Stub HTTP requests; commonly used with VCR
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
├── models/              # Model specs
├── requests/            # Request/integration specs
├── jobs/                # Background job specs
├── services/            # Service/PORO specs
├── system/              # System/end-to-end specs
├── support/             # Shared examples, helpers, custom matchers
├── fixtures/            # Optional static test data
├── vcr_cassettes/       # VCR recordings, if VCR is used
└── spec_helper.rb
```
(Please note that **the exact directory structure is not a mandatory Rails/RSpec requirement**. They're a recommended organizational convention.)


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
- Enables transactional test isolation, so database changes made within a test are rolled back after the test completes.
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
# Then open coverage/index.html in your browser to see percentage of measured application code executed by the test suite
```

### 1.8 The Test Database

- Rails uses a separate database for the `test` environment, so tests do not normally operate on your development data.
- When transactional fixtures are enabled, database changes made during a test are wrapped in a transaction and rolled back after the test completes. This provides isolation between tests without requiring the entire database to be recreated for every test.
- The test database must still have the expected schema. When migrations or schema-related configuration change, you may need to prepare or rebuild the test database.

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
        image: postgres:YOUR_POSTGRES_VERSION
        env:
          POSTGRES_PASSWORD: postgres

    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 'YOUR_PROJECT_RUBY_VERSION'
          bundler-cache: true
      
      - run: bundle exec rails db:test:prepare
      - run: bundle exec rspec
      - run: bundle exec rubocop  # Optional: static code quality
```

This ensures:
- Tests run on every push and pull request
- When configured as a required status check, failing tests can prevent a PR from being merged
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
bundle exec rspec spec/models/vehicle_spec.rb
```
When execution reaches `binding.pry`, the test process pauses and opens a Pry session.

#### See what SQL is being executed:
```ruby
it "queries the database efficiently" do
  expect {
    Vehicle.find_by(registration_number: 'KA01AB1234')
  }.not_to exceed_query_limit(1)  # Requires rspec-rails and should_not gem
end
```

---
