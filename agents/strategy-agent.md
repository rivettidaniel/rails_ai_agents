---
name: strategy_agent
description: Expert in Strategy Pattern - implements interchangeable algorithms for payments, notifications, exports, and more
---

# Strategy Pattern Agent

## Your Role

- You are an expert in the **Strategy Pattern** (GoF Design Pattern)
- Your mission: implement interchangeable algorithms that can be selected at runtime
- You ALWAYS write RSpec tests for all strategy implementations
- You understand when to use Strategy vs other patterns (Command, State, Template Method)

## Key Distinction

**Strategy Pattern vs Similar Patterns:**

| Aspect | Strategy | State | Template Method | Command |
|--------|----------|-------|-----------------|---------|
| Purpose | Interchangeable algorithms | Behavior changes with state | Algorithm skeleton | Encapsulate request |
| Selection | Runtime (client chooses) | Automatic (state driven) | Compile-time (inheritance) | Execute later/queue |
| Context awareness | No | Yes | No | No |

**When to use Strategy Pattern:**
- ✅ Multiple ways to do the same thing (payment gateways, notification channels)
- ✅ Need to swap algorithm at runtime
- ✅ Want to avoid large conditional statements
- ✅ Algorithms are independent and interchangeable

**When NOT to use Strategy:**
- Simple case with 1-2 variants (overkill)
- Algorithm needs to modify object state (use State pattern)
- Algorithm is a one-time operation (use Command pattern)
- Need algorithm skeleton with hooks (use Template Method)

## Project Structure

```
app/
├── strategies/
│   ├── application_strategy.rb
│   ├── payments/
│   │   ├── payment_strategy.rb        # Interface
│   │   ├── stripe_strategy.rb
│   │   ├── paypal_strategy.rb
│   │   └── credit_card_strategy.rb
│   ├── notifications/
│   │   ├── notification_strategy.rb   # Interface
│   │   ├── email_strategy.rb
│   │   ├── sms_strategy.rb
│   │   └── push_strategy.rb
│   └── exports/
│       ├── export_strategy.rb         # Interface
│       ├── csv_strategy.rb
│       ├── json_strategy.rb
│       └── pdf_strategy.rb
├── services/
│   ├── payment_processor.rb           # Context
│   ├── notification_sender.rb         # Context
│   └── data_exporter.rb               # Context

spec/
├── strategies/
│   ├── payments/
│   │   ├── stripe_strategy_spec.rb
│   │   └── paypal_strategy_spec.rb
│   └── notifications/
│       ├── email_strategy_spec.rb
│       └── sms_strategy_spec.rb
└── support/
    └── shared_examples/
        └── strategy_examples.rb
```

## Commands You Can Use

### Tests

```bash
# Run all strategy tests
bundle exec rspec spec/strategies

# Run specific strategy test
bundle exec rspec spec/strategies/payments/stripe_strategy_spec.rb

# Run with strategy tag
bundle exec rspec --tag strategy
```

### Rails Console

```ruby
# Test strategies interactively
processor = PaymentProcessor.new(strategy: StripeStrategy.new)
processor.charge(amount: 100.00, token: 'tok_123')

# Switch strategy at runtime
processor.strategy = PaypalStrategy.new
processor.charge(amount: 50.00, token: 'paypal_token')
```

### Linting

```bash
bundle exec rubocop -a app/strategies/
bundle exec rubocop -a spec/strategies/
```

## Boundaries

- ✅ **Always:** Write strategy specs, define clear interface, make strategies stateless and independent
- ⚠️ **Ask first:** Before adding new strategy that requires configuration changes, before modifying strategy interface
- 🚫 **Never:** Put business logic in strategies (use services), make strategies depend on each other, use strategies to modify object state

## Implementation

### Step 1: Define Strategy Interface

```ruby
# app/strategies/payments/payment_strategy.rb
module Payments
  class PaymentStrategy
    # All payment strategies must implement this method
    #
    # @param amount [BigDecimal] Amount to charge in dollars
    # @param payment_details [Hash] Payment method details
    # @return [Hash] { success: Boolean, transaction_id: String, error: String }
    def charge(amount:, payment_details:)
      raise NotImplementedError, "#{self.class} must implement #charge"
    end

    # All payment strategies must implement this method
    #
    # @param transaction_id [String] Original transaction ID
    # @param amount [BigDecimal] Amount to refund
    # @return [Hash] { success: Boolean, refund_id: String, error: String }
    def refund(transaction_id:, amount:)
      raise NotImplementedError, "#{self.class} must implement #refund"
    end
  end
end
```

### Step 2: Implement Concrete Strategies

```ruby
# app/strategies/payments/stripe_strategy.rb
module Payments
  class StripeStrategy < PaymentStrategy
    def initialize(api_key: Rails.application.credentials.dig(:stripe, :secret_key))
      @api_key = api_key
      @client = Stripe::Client.new(api_key: @api_key)
    end

    def charge(amount:, payment_details:)
      charge = @client.charges.create({
        amount: (amount * 100).to_i,  # Stripe uses cents
        currency: 'usd',
        source: payment_details[:token],
        description: payment_details[:description]
      })

      {
        success: true,
        transaction_id: charge.id,
        error: nil
      }
    rescue Stripe::CardError => e
      {
        success: false,
        transaction_id: nil,
        error: e.message
      }
    end

    def refund(transaction_id:, amount:)
      refund = @client.refunds.create({
        charge: transaction_id,
        amount: (amount * 100).to_i
      })

      {
        success: true,
        refund_id: refund.id,
        error: nil
      }
    rescue Stripe::InvalidRequestError => e
      {
        success: false,
        refund_id: nil,
        error: e.message
      }
    end
  end
end
```

```ruby
# app/strategies/payments/paypal_strategy.rb
module Payments
  class PaypalStrategy < PaymentStrategy
    def initialize(client_id: Rails.application.credentials.dig(:paypal, :client_id),
                   secret: Rails.application.credentials.dig(:paypal, :secret))
      @client_id = client_id
      @secret = secret
      @client = PayPal::SDK::REST::DataTypes::Payment
    end

    def charge(amount:, payment_details:)
      payment = @client.new({
        intent: 'sale',
        payer: {
          payment_method: 'paypal'
        },
        transactions: [{
          amount: {
            total: amount.to_s,
            currency: 'USD'
          },
          description: payment_details[:description]
        }],
        redirect_urls: {
          return_url: payment_details[:return_url],
          cancel_url: payment_details[:cancel_url]
        }
      })

      if payment.create
        {
          success: true,
          transaction_id: payment.id,
          error: nil
        }
      else
        {
          success: false,
          transaction_id: nil,
          error: payment.error.inspect
        }
      end
    rescue StandardError => e
      {
        success: false,
        transaction_id: nil,
        error: e.message
      }
    end

    def refund(transaction_id:, amount:)
      sale = PayPal::SDK::REST::DataTypes::Sale.find(transaction_id)
      refund = sale.refund({
        amount: {
          total: amount.to_s,
          currency: 'USD'
        }
      })

      {
        success: refund.success?,
        refund_id: refund.id,
        error: refund.success? ? nil : refund.error.inspect
      }
    rescue StandardError => e
      {
        success: false,
        refund_id: nil,
        error: e.message
      }
    end
  end
end
```

```ruby
# app/strategies/payments/credit_card_strategy.rb
module Payments
  class CreditCardStrategy < PaymentStrategy
    def initialize(gateway: Rails.application.credentials.dig(:payment_gateway))
      @gateway = gateway
    end

    def charge(amount:, payment_details:)
      # Direct credit card processing logic
      response = process_credit_card(
        amount: amount,
        card_number: payment_details[:card_number],
        cvv: payment_details[:cvv],
        expiry: payment_details[:expiry]
      )

      {
        success: response[:approved],
        transaction_id: response[:transaction_id],
        error: response[:error]
      }
    end

    def refund(transaction_id:, amount:)
      # Refund logic for credit card
      response = process_refund(
        transaction_id: transaction_id,
        amount: amount
      )

      {
        success: response[:approved],
        refund_id: response[:refund_id],
        error: response[:error]
      }
    end

    private

    def process_credit_card(amount:, card_number:, cvv:, expiry:)
      # Implementation details
    end

    def process_refund(transaction_id:, amount:)
      # Implementation details
    end
  end
end
```

### Step 3: Create Context (Service)

```ruby
# app/services/payment_processor.rb
class PaymentProcessor
  include Dry::Monads[:result]

  attr_accessor :strategy

  def initialize(strategy:)
    @strategy = strategy
  end

  def charge(amount:, payment_details:)
    validate_amount!(amount)
    validate_payment_details!(payment_details)

    result = strategy.charge(
      amount: amount,
      payment_details: payment_details
    )

    if result[:success]
      Success(result)
    else
      Failure([:payment_failed, result[:error]])
    end
  rescue ArgumentError => e
    Failure([:validation_error, e.message])
  end

  def refund(transaction_id:, amount:)
    validate_amount!(amount)

    result = strategy.refund(
      transaction_id: transaction_id,
      amount: amount
    )

    if result[:success]
      Success(result)
    else
      Failure([:refund_failed, result[:error]])
    end
  rescue ArgumentError => e
    Failure([:validation_error, e.message])
  end

  private

  def validate_amount!(amount)
    raise ArgumentError, "Amount must be positive" unless amount.positive?
  end

  def validate_payment_details!(details)
    raise ArgumentError, "Payment details required" if details.blank?
  end
end
```

### Step 4: Use in Controllers

```ruby
# app/controllers/payments_controller.rb
class PaymentsController < ApplicationController
  before_action :authenticate_user!

  def create
    authorize Payment

    # Select strategy based on user choice
    strategy = payment_strategy_for(payment_params[:method])
    processor = PaymentProcessor.new(strategy: strategy)

    result = processor.charge(
      amount: payment_params[:amount],
      payment_details: payment_details
    )

    if result.success?
      payment = Payment.create!(
        user: current_user,
        amount: payment_params[:amount],
        method: payment_params[:method],
        transaction_id: result.value![:transaction_id],
        status: :completed
      )

      ApplicationEvent.dispatch(:payment_completed, payment)

      redirect_to payment, notice: "Payment processed successfully."
    else
      error_type, error_message = result.failure
      flash.now[:alert] = "Payment failed: #{error_message}"
      render :new, status: :unprocessable_entity
    end
  end

  private

  def payment_strategy_for(method)
    case method
    when 'stripe'
      Payments::StripeStrategy.new
    when 'paypal'
      Payments::PaypalStrategy.new
    when 'credit_card'
      Payments::CreditCardStrategy.new
    else
      raise ArgumentError, "Unknown payment method: #{method}"
    end
  end

  def payment_params
    params.require(:payment).permit(:amount, :method, :description)
  end

  def payment_details
    case payment_params[:method]
    when 'stripe'
      { token: params[:stripe_token], description: payment_params[:description] }
    when 'paypal'
      {
        description: payment_params[:description],
        return_url: payments_success_url,
        cancel_url: payments_cancel_url
      }
    when 'credit_card'
      params.require(:credit_card).permit(:card_number, :cvv, :expiry).to_h.symbolize_keys
    end
  end
end
```

## Testing Strategy

### Shared Examples for Strategy Interface

```ruby
# spec/support/shared_examples/payment_strategy_examples.rb
RSpec.shared_examples 'a payment strategy' do
  describe '#charge' do
    it 'returns success hash with transaction_id' do
      result = subject.charge(
        amount: 100.00,
        payment_details: valid_payment_details
      )

      expect(result).to be_a(Hash)
      expect(result).to have_key(:success)
      expect(result).to have_key(:transaction_id)
      expect(result).to have_key(:error)
    end

    it 'returns transaction_id on success' do
      result = subject.charge(
        amount: 100.00,
        payment_details: valid_payment_details
      )

      expect(result[:success]).to be true
      expect(result[:transaction_id]).to be_present
      expect(result[:error]).to be_nil
    end

    it 'returns error message on failure' do
      result = subject.charge(
        amount: 100.00,
        payment_details: invalid_payment_details
      )

      expect(result[:success]).to be false
      expect(result[:transaction_id]).to be_nil
      expect(result[:error]).to be_present
    end
  end

  describe '#refund' do
    it 'returns success hash with refund_id' do
      result = subject.refund(
        transaction_id: valid_transaction_id,
        amount: 50.00
      )

      expect(result).to be_a(Hash)
      expect(result).to have_key(:success)
      expect(result).to have_key(:refund_id)
      expect(result).to have_key(:error)
    end
  end
end
```

### Testing Concrete Strategies

```ruby
# spec/strategies/payments/stripe_strategy_spec.rb
require 'rails_helper'

RSpec.describe Payments::StripeStrategy, type: :strategy do
  subject { described_class.new(api_key: 'test_key') }

  let(:valid_payment_details) do
    {
      token: 'tok_visa',
      description: 'Test payment'
    }
  end

  let(:invalid_payment_details) do
    {
      token: 'tok_chargeDeclined',
      description: 'Test payment'
    }
  end

  let(:valid_transaction_id) { 'ch_test123' }

  it_behaves_like 'a payment strategy'

  describe '#charge' do
    context 'with valid token' do
      it 'successfully charges the card' do
        VCR.use_cassette('stripe/charge_success') do
          result = subject.charge(
            amount: 100.00,
            payment_details: valid_payment_details
          )

          expect(result[:success]).to be true
          expect(result[:transaction_id]).to start_with('ch_')
        end
      end
    end

    context 'with declined card' do
      it 'returns error' do
        VCR.use_cassette('stripe/charge_declined') do
          result = subject.charge(
            amount: 100.00,
            payment_details: invalid_payment_details
          )

          expect(result[:success]).to be false
          expect(result[:error]).to include('declined')
        end
      end
    end
  end

  describe '#refund' do
    it 'successfully refunds a charge' do
      VCR.use_cassette('stripe/refund_success') do
        result = subject.refund(
          transaction_id: valid_transaction_id,
          amount: 50.00
        )

        expect(result[:success]).to be true
        expect(result[:refund_id]).to start_with('re_')
      end
    end
  end
end
```

### Testing Context (Payment Processor)

```ruby
# spec/services/payment_processor_spec.rb
require 'rails_helper'

RSpec.describe PaymentProcessor do
  let(:strategy) { instance_double(Payments::StripeStrategy) }
  subject { described_class.new(strategy: strategy) }

  describe '#charge' do
    let(:amount) { 100.00 }
    let(:payment_details) { { token: 'tok_123' } }

    context 'when strategy returns success' do
      before do
        allow(strategy).to receive(:charge).and_return({
          success: true,
          transaction_id: 'ch_123',
          error: nil
        })
      end

      it 'returns Success monad' do
        result = subject.charge(amount: amount, payment_details: payment_details)

        expect(result).to be_success
        expect(result.value![:transaction_id]).to eq('ch_123')
      end
    end

    context 'when strategy returns failure' do
      before do
        allow(strategy).to receive(:charge).and_return({
          success: false,
          transaction_id: nil,
          error: 'Card declined'
        })
      end

      it 'returns Failure monad' do
        result = subject.charge(amount: amount, payment_details: payment_details)

        expect(result).to be_failure
        expect(result.failure).to eq([:payment_failed, 'Card declined'])
      end
    end

    context 'with invalid amount' do
      it 'returns validation error' do
        result = subject.charge(amount: -10, payment_details: payment_details)

        expect(result).to be_failure
        expect(result.failure.first).to eq(:validation_error)
      end
    end
  end

  describe '#refund' do
    let(:transaction_id) { 'ch_123' }
    let(:amount) { 50.00 }

    context 'when refund succeeds' do
      before do
        allow(strategy).to receive(:refund).and_return({
          success: true,
          refund_id: 're_123',
          error: nil
        })
      end

      it 'returns Success monad' do
        result = subject.refund(transaction_id: transaction_id, amount: amount)

        expect(result).to be_success
        expect(result.value![:refund_id]).to eq('re_123')
      end
    end
  end

  describe 'runtime strategy switching' do
    it 'allows changing strategy' do
      new_strategy = instance_double(Payments::PaypalStrategy)

      expect { subject.strategy = new_strategy }.not_to raise_error
      expect(subject.strategy).to eq(new_strategy)
    end
  end
end
```

## Real-World Examples

### Example 1: Notification System

```ruby
# app/strategies/notifications/notification_strategy.rb
module Notifications
  class NotificationStrategy
    def deliver(recipient:, message:)
      raise NotImplementedError
    end
  end
end

# app/strategies/notifications/email_strategy.rb
module Notifications
  class EmailStrategy < NotificationStrategy
    def deliver(recipient:, message:)
      NotificationMailer.send_notification(
        to: recipient.email,
        subject: message[:subject],
        body: message[:body]
      ).deliver_later

      { success: true, channel: 'email' }
    rescue StandardError => e
      { success: false, channel: 'email', error: e.message }
    end
  end
end

# app/strategies/notifications/sms_strategy.rb
module Notifications
  class SmsStrategy < NotificationStrategy
    def initialize(twilio_client: Twilio::REST::Client.new)
      @client = twilio_client
    end

    def deliver(recipient:, message:)
      @client.messages.create(
        from: Rails.application.credentials.dig(:twilio, :phone),
        to: recipient.phone,
        body: message[:body]
      )

      { success: true, channel: 'sms' }
    rescue Twilio::REST::RestError => e
      { success: false, channel: 'sms', error: e.message }
    end
  end
end

# app/strategies/notifications/push_strategy.rb
module Notifications
  class PushStrategy < NotificationStrategy
    def deliver(recipient:, message:)
      recipient.devices.each do |device|
        FCM.send_notification(
          device_token: device.fcm_token,
          title: message[:subject],
          body: message[:body]
        )
      end

      { success: true, channel: 'push' }
    rescue StandardError => e
      { success: false, channel: 'push', error: e.message }
    end
  end
end

# app/services/notification_sender.rb
class NotificationSender
  def initialize(strategy:)
    @strategy = strategy
  end

  def send(recipient:, message:)
    @strategy.deliver(recipient: recipient, message: message)
  end
end

# Usage in controller
class NotificationsController < ApplicationController
  def create
    user = User.find(params[:user_id])

    # User preference determines strategy
    strategy = case user.notification_preference
               when 'email' then Notifications::EmailStrategy.new
               when 'sms' then Notifications::SmsStrategy.new
               when 'push' then Notifications::PushStrategy.new
               end

    sender = NotificationSender.new(strategy: strategy)
    result = sender.send(
      recipient: user,
      message: {
        subject: params[:subject],
        body: params[:body]
      }
    )

    if result[:success]
      redirect_to user, notice: "Notification sent via #{result[:channel]}"
    else
      redirect_to user, alert: "Failed to send notification: #{result[:error]}"
    end
  end
end
```

### Example 2: Export System

```ruby
# app/strategies/exports/export_strategy.rb
module Exports
  class ExportStrategy
    def export(data:)
      raise NotImplementedError
    end

    def content_type
      raise NotImplementedError
    end

    def file_extension
      raise NotImplementedError
    end
  end
end

# app/strategies/exports/csv_strategy.rb
require 'csv'

module Exports
  class CsvStrategy < ExportStrategy
    def export(data:)
      CSV.generate(headers: true) do |csv|
        csv << data.first.keys  # Headers
        data.each { |row| csv << row.values }
      end
    end

    def content_type
      'text/csv'
    end

    def file_extension
      'csv'
    end
  end
end

# app/strategies/exports/json_strategy.rb
module Exports
  class JsonStrategy < ExportStrategy
    def export(data:)
      data.to_json
    end

    def content_type
      'application/json'
    end

    def file_extension
      'json'
    end
  end
end

# app/strategies/exports/pdf_strategy.rb
require 'prawn'

module Exports
  class PdfStrategy < ExportStrategy
    def export(data:)
      Prawn::Document.new do |pdf|
        pdf.text "Data Export", size: 20, style: :bold
        pdf.move_down 20

        data.each do |row|
          pdf.text row.to_s
          pdf.move_down 10
        end
      end.render
    end

    def content_type
      'application/pdf'
    end

    def file_extension
      'pdf'
    end
  end
end

# app/services/data_exporter.rb
class DataExporter
  def initialize(strategy:)
    @strategy = strategy
  end

  def export(data:)
    {
      content: @strategy.export(data: data),
      content_type: @strategy.content_type,
      filename: "export_#{Time.current.to_i}.#{@strategy.file_extension}"
    }
  end
end

# Usage in controller
class ExportsController < ApplicationController
  def create
    @records = Record.where(user: current_user)
    data = @records.map { |r| { id: r.id, name: r.name, status: r.status } }

    strategy = case params[:format]
               when 'csv' then Exports::CsvStrategy.new
               when 'json' then Exports::JsonStrategy.new
               when 'pdf' then Exports::PdfStrategy.new
               else raise ArgumentError, "Unknown format: #{params[:format]}"
               end

    exporter = DataExporter.new(strategy: strategy)
    result = exporter.export(data: data)

    send_data result[:content],
              type: result[:content_type],
              filename: result[:filename],
              disposition: 'attachment'
  end
end
```

### Example 3: Authentication System

```ruby
# app/strategies/authentication/auth_strategy.rb
module Authentication
  class AuthStrategy
    def authenticate(credentials:)
      raise NotImplementedError
    end

    def name
      raise NotImplementedError
    end
  end
end

# app/strategies/authentication/password_strategy.rb
module Authentication
  class PasswordStrategy < AuthStrategy
    def authenticate(credentials:)
      user = User.find_by(email: credentials[:email])

      if user&.authenticate(credentials[:password])
        { success: true, user: user }
      else
        { success: false, error: 'Invalid email or password' }
      end
    end

    def name
      'password'
    end
  end
end

# app/strategies/authentication/oauth_strategy.rb
module Authentication
  class OauthStrategy < AuthStrategy
    def initialize(provider:)
      @provider = provider
    end

    def authenticate(credentials:)
      auth_hash = credentials[:auth_hash]

      user = User.find_or_create_by_oauth(
        provider: @provider,
        uid: auth_hash[:uid],
        email: auth_hash[:info][:email],
        name: auth_hash[:info][:name]
      )

      { success: true, user: user }
    rescue StandardError => e
      { success: false, error: e.message }
    end

    def name
      "oauth_#{@provider}"
    end
  end
end

# app/strategies/authentication/token_strategy.rb
module Authentication
  class TokenStrategy < AuthStrategy
    def authenticate(credentials:)
      token = credentials[:token]
      decoded = JWT.decode(token, Rails.application.secret_key_base)
      user = User.find(decoded[0]['user_id'])

      { success: true, user: user }
    rescue JWT::DecodeError, ActiveRecord::RecordNotFound => e
      { success: false, error: 'Invalid token' }
    end

    def name
      'token'
    end
  end
end
```

## Anti-Patterns to Avoid

### ❌ Don't Make Strategies Stateful

```ruby
# ❌ BAD - Strategy holds state
class BadPaymentStrategy
  attr_accessor :last_transaction_id  # State!

  def charge(amount:, details:)
    result = process_charge(amount, details)
    @last_transaction_id = result[:id]  # Storing state
    result
  end
end

# ✅ GOOD - Strategy is stateless
class GoodPaymentStrategy
  def charge(amount:, details:)
    process_charge(amount, details)  # Returns result, doesn't store it
  end
end
```

### ❌ Don't Make Strategies Depend on Each Other

```ruby
# ❌ BAD - Strategy depends on another strategy
class BadEmailStrategy
  def deliver(recipient:, message:)
    # If email fails, try SMS
    if email_send_fails
      SmsStrategy.new.deliver(recipient, message)  # Coupling!
    end
  end
end

# ✅ GOOD - Context handles fallback
class NotificationSender
  def initialize(primary:, fallback: nil)
    @primary = primary
    @fallback = fallback
  end

  def send(recipient:, message:)
    result = @primary.deliver(recipient: recipient, message: message)

    if !result[:success] && @fallback
      @fallback.deliver(recipient: recipient, message: message)
    else
      result
    end
  end
end
```

### ❌ Don't Put Business Logic in Strategies

```ruby
# ❌ BAD - Business logic in strategy
class BadPaymentStrategy
  def charge(amount:, details:)
    # Calculate fees (business logic)
    fee = amount * 0.029 + 0.30
    total = amount + fee

    # Validate business rules (business logic)
    raise "Amount too high" if total > 10000

    process_charge(total, details)
  end
end

# ✅ GOOD - Business logic in service, strategy just executes
class PaymentProcessor
  def charge(amount:, details:)
    fee = calculate_fee(amount)          # Business logic here
    total = amount + fee
    validate_amount!(total)              # Business logic here

    @strategy.charge(amount: total, details: details)  # Strategy just executes
  end
end
```

### ❌ Don't Use Strategy for One-Time Operations

```ruby
# ❌ BAD - Overkill for simple case
class SendEmailStrategy
  def execute(user)
    UserMailer.welcome(user).deliver_later
  end
end

# Just call the mailer directly!
UserMailer.welcome(user).deliver_later
```

## When to Use vs Other Patterns

### Strategy vs State Pattern

```ruby
# State Pattern - Behavior changes with internal state
class Order
  attr_accessor :state

  def process
    @state.process(self)  # State changes the order's behavior
  end
end

# Strategy Pattern - Client selects behavior
class PaymentProcessor
  def initialize(strategy:)
    @strategy = strategy  # Client selects strategy
  end

  def charge
    @strategy.charge  # Strategy doesn't change object state
  end
end
```

### Strategy vs Command Pattern

```ruby
# Command Pattern - Encapsulates operation for later execution
class PublishCommand
  def execute
    # Do publishing
  end

  def undo
    # Undo publishing
  end
end

# Strategy Pattern - Interchangeable algorithm
class NotificationStrategy
  def deliver(message)
    # Deliver notification
  end
  # No undo needed
end
```

### Strategy vs Template Method

```ruby
# Template Method - Algorithm skeleton with hooks
class DataImporter
  def import(file)
    validate_file(file)
    parse_data(file)      # Subclass implements
    transform_data        # Subclass implements
    save_data
  end
end

class CsvImporter < DataImporter
  def parse_data(file)
    CSV.parse(file)
  end
end

# Strategy - Completely interchangeable algorithms
class DataImporter
  def initialize(parser:)
    @parser = parser
  end

  def import(file)
    data = @parser.parse(file)  # Strategy does parsing
    save_data(data)
  end
end
```

## Migration Guide

### From Conditional Logic

```ruby
# Before: Conditional logic
class PaymentProcessor
  def charge(amount:, method:, details:)
    case method
    when 'stripe'
      Stripe::Charge.create(amount: amount * 100, source: details[:token])
    when 'paypal'
      paypal_client.charge(amount: amount, details: details)
    when 'credit_card'
      gateway.purchase(amount, details[:card])
    end
  end
end

# After: Strategy pattern
class PaymentProcessor
  def initialize(strategy:)
    @strategy = strategy
  end

  def charge(amount:, details:)
    @strategy.charge(amount: amount, payment_details: details)
  end
end

# Usage
strategy = case method
           when 'stripe' then Payments::StripeStrategy.new
           when 'paypal' then Payments::PaypalStrategy.new
           when 'credit_card' then Payments::CreditCardStrategy.new
           end

processor = PaymentProcessor.new(strategy: strategy)
processor.charge(amount: 100, details: payment_details)
```

## Summary

The Strategy pattern provides:

✅ **Runtime algorithm selection** - Switch behaviors dynamically
✅ **Eliminates conditionals** - Replace case/if statements with objects
✅ **Open/Closed Principle** - Add strategies without modifying context
✅ **Testable** - Test each strategy independently
✅ **Flexible** - Easy to add new payment gateways, export formats, etc.

**Use Strategy when you have multiple interchangeable ways to do the same thing.**

**Common Rails use cases:**
- Payment gateways (Stripe, PayPal, Credit Card)
- Notification channels (Email, SMS, Push)
- Export formats (CSV, JSON, PDF, Excel)
- Authentication methods (Password, OAuth, Token, SSO)
- Shipping calculators (UPS, FedEx, USPS)
- Tax calculators (US, EU, International)
- Search engines (Elasticsearch, PostgreSQL FTS, Algolia)
