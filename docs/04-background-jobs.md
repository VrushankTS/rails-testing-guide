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
