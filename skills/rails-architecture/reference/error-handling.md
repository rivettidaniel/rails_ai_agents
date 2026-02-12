# Error Handling Strategies

## dry-monads Result Pattern (Preferred)

Services use **dry-monads** for Result handling:

**Installation:**

Add to `Gemfile`:
```ruby
gem 'dry-monads', '~> 1.6'
```

**Base Service:**

```ruby
# app/services/application_service.rb
class ApplicationService
  include Dry::Monads[:result]

  def self.call(...)
    new(...).call
  end
end
```

All services return:
- `Success(data)` for successful operations
- `Failure(error)` for failures

Check results with `.success?` and `.failure?`
Extract values with `.value!` or `.value_or(default)`

## Error Handling Approaches

### Simple String Messages

```ruby
module Orders
  class CreateService
    include Dry::Monads[:result]

    def call(params)
      return Failure("Your cart is empty") if params[:items].empty?
      return Failure("Item out of stock") unless inventory_available?(params[:items])

      order = create_order(params)
      Success(order)
    rescue PaymentGateway::Declined => e
      Failure("Payment declined: #{e.message}")
    rescue ActiveRecord::RecordInvalid => e
      Failure("Validation failed: #{e.message}")
    end
  end
end
```

### Structured Error Hashes

For programmatic error handling:

```ruby
def call(params)
  return Failure(code: :empty_cart, message: "Your cart is empty") if params[:items].empty?
  return Failure(code: :out_of_stock, message: "Item unavailable") unless inventory_available?(params[:items])

  order = create_order(params)
  Success(order)
rescue PaymentGateway::Declined => e
  Failure(code: :payment_declined, message: e.message)
rescue ActiveRecord::RecordInvalid => e
  Failure(code: :validation_failed, message: e.message, details: e.record.errors.to_hash)
end
```

## Controller Error Handling

### Simple Handling

```ruby
class OrdersController < ApplicationController
  def create
    result = Orders::CreateService.new.call(order_params)

    if result.success?
      redirect_to result.value!, notice: "Order created"
    else
      flash.now[:alert] = result.failure
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Structured Error Handling

```ruby
class OrdersController < ApplicationController
  def create
    result = Orders::CreateService.new.call(order_params)

    if result.success?
      redirect_to result.value!, notice: "Order created"
    else
      handle_error(result.failure)
    end
  end

  private

  def handle_error(error)
    case error[:code]
    when :empty_cart
      redirect_to cart_path, alert: error[:message]
    when :out_of_stock
      flash.now[:alert] = error[:message]
      render :new, status: :unprocessable_entity
    when :payment_declined
      redirect_to payment_path, alert: error[:message]
    else
      flash.now[:alert] = error[:message] || error
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Pattern Matching (Ruby 3+)

```ruby
def create
  case Orders::CreateService.new.call(order_params)
  in Dry::Monads::Success(order)
    redirect_to order, notice: "Order created"
  in Dry::Monads::Failure(code: :empty_cart, message: msg)
    redirect_to cart_path, alert: msg
  in Dry::Monads::Failure(code: :payment_declined, message: msg)
    redirect_to payment_path, alert: msg
  in Dry::Monads::Failure(message)
    flash.now[:alert] = message
    render :new, status: :unprocessable_entity
  end
end
```

## API Error Responses

### Consistent Error Format

```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ApplicationController
    private

    def render_service_result(result, status_success: :ok, status_failure: :unprocessable_entity)
      if result.success?
        render json: { data: result.value! }, status: status_success
      else
        error = result.failure
        render json: {
          error: error.is_a?(Hash) ? error : { message: error }
        }, status: status_failure
      end
    end
  end
end
```

### HTTP Status Mapping

```ruby
ERROR_STATUS_MAP = {
  not_found: :not_found,
  unauthorized: :unauthorized,
  forbidden: :forbidden,
  validation_failed: :unprocessable_entity,
  conflict: :conflict,
  rate_limited: :too_many_requests
}.freeze

def render_service_result(result)
  if result.success?
    render_success(result.data)
  else
    status = ERROR_STATUS_MAP.fetch(result.code, :unprocessable_entity)
    render_error(result, status: status)
  end
end
```

## Exception Handling Layers

### Service Layer (Catch and Wrap)

```ruby
class ExternalApiService
  include Dry::Monads[:result]

  def call(params)
    response = client.request(params)
    Success(response.data)
  rescue Faraday::TimeoutError
    Failure("External service timed out")
  rescue Faraday::ConnectionFailed
    Failure("Could not connect to service")
  rescue JSON::ParserError
    Failure("Invalid response from service")
  end
end
```

### Controller Layer (Rescue From)

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  private

  def not_found
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  def forbidden
    respond_to do |format|
      format.html { redirect_to root_path, alert: t("errors.forbidden") }
      format.json { render json: { error: "Forbidden" }, status: :forbidden }
    end
  end
end
```

### Global Error Handler

```ruby
# config/initializers/error_handler.rb
Rails.application.config.exceptions_app = ->(env) {
  ErrorsController.action(:show).call(env)
}

# app/controllers/errors_controller.rb
class ErrorsController < ApplicationController
  skip_before_action :authenticate_user!

  def show
    @status = request.env["PATH_INFO"].delete("/").to_i
    render status: @status
  end
end
```

## Validation Errors

### Model Validations to Result

**Simple version:**

```ruby
def call(params)
  record = Model.new(params)

  if record.save
    Success(record)
  else
    Failure(record.errors.full_messages.join(", "))
  end
end
```

**Structured version:**

```ruby
def call(params)
  record = Model.new(params)

  if record.save
    Success(record)
  else
    Failure(
      code: :validation_failed,
      message: record.errors.full_messages.join(", "),
      details: record.errors.to_hash
    )
  end
end
```

### Display Validation Errors

```ruby
# In controller
if result.failure?
  error = result.failure
  if error.is_a?(Hash) && error[:details]
    @errors = error[:details] # Hash of field => [messages]
  end
end

# In view
<% if @errors&.dig(:email) %>
  <p class="text-red-500"><%= @errors[:email].join(", ") %></p>
<% end %>
```

## Logging Errors

```ruby
class ApplicationService
  include Dry::Monads[:result]

  private

  def log_and_fail(message, exception: nil)
    Rails.logger.error({
      service: self.class.name,
      message: message,
      exception: exception&.class&.name,
      backtrace: exception&.backtrace&.first(5)
    }.to_json)

    Failure(message)
  end
end

# Usage:
def call(params)
  do_something
  Success(result)
rescue SomeError => e
  log_and_fail("Operation failed", exception: e)
end
```

## Error Tracking Integration

```ruby
# With Sentry/Rollbar
def call(params)
  do_something
  Success(result)
rescue UnexpectedError => e
  Sentry.capture_exception(e, extra: { service: self.class.name, params: params })
  Failure("An unexpected error occurred")
rescue ExpectedError => e
  # Don't report expected errors
  Failure(e.message)
end
```

## Checklist

- [ ] Services include `Dry::Monads[:result]`
- [ ] Services return `Success(data)` or `Failure(error)`
- [ ] Controllers handle both success and failure cases
- [ ] API responses have consistent format
- [ ] Unexpected errors logged with context
- [ ] Sensitive data not exposed in errors
- [ ] User-facing messages use I18n
- [ ] Use `.value!` to unwrap Success (raises on Failure)
- [ ] Use `.value_or(default)` for safe unwrapping
