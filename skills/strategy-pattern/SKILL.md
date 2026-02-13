---
name: strategy-pattern
description: Implements interchangeable algorithms with Strategy Pattern. Use when implementing payment gateways, notification channels, export formats, authentication methods, or any scenario requiring runtime algorithm selection.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Strategy Pattern in Rails

## Overview

The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**Key Insight**: Replace conditional logic with polymorphic strategy objects that can be swapped at runtime.

## Core Components

```
Client → Context → Strategy (Interface)
                       ↓
            ConcreteStrategyA
            ConcreteStrategyB
            ConcreteStrategyC
```

1. **Strategy (Interface)** - Declares method all strategies must implement
2. **Concrete Strategies** - Implement different algorithm variations
3. **Context** - Maintains reference to strategy, delegates work to it
4. **Client** - Selects and configures the appropriate strategy

## When to Use Strategy Pattern

✅ **Use Strategy Pattern when you need:**

- **Multiple ways to do the same thing** - Payment gateways (Stripe, PayPal, Credit Card)
- **Runtime algorithm selection** - Choose behavior based on user preference
- **Eliminate large conditionals** - Replace case/if statements with strategy objects
- **Independent, interchangeable algorithms** - Notification channels, export formats

❌ **Don't use Strategy Pattern for:**

- Simple cases with 1-2 variants (overkill)
- Algorithm needs to modify object state (use State pattern)
- One-time operations (use Command pattern)
- Need algorithm skeleton with hooks (use Template Method)

## Difference from Similar Patterns

| Aspect | Strategy | State | Template Method | Command |
|--------|----------|-------|-----------------|---------|
| Purpose | Interchangeable algorithms | Behavior changes with state | Algorithm skeleton | Encapsulate request |
| Selection | Runtime (client chooses) | Automatic (state driven) | Compile-time (inheritance) | Execute later/queue |
| Context awareness | No | Yes | No | No |
| Typical use | Payments, exports | Order states, workflow | Import/export base class | Undo/redo |

## Common Rails Use Cases

### 1. Payment Processing

```ruby
# Bad: Conditional logic
class PaymentProcessor
  def charge(amount, method, details)
    case method
    when 'stripe'
      Stripe::Charge.create(amount: amount * 100, source: details[:token])
    when 'paypal'
      # PayPal logic
    when 'credit_card'
      # Credit card logic
    end
  end
end

# Good: Strategy pattern
class PaymentProcessor
  def initialize(strategy:)
    @strategy = strategy
  end

  def charge(amount:, details:)
    @strategy.charge(amount: amount, payment_details: details)
  end
end

# Strategies
class StripeStrategy < PaymentStrategy
  def charge(amount:, payment_details:)
    # Stripe-specific logic
  end
end

class PaypalStrategy < PaymentStrategy
  def charge(amount:, payment_details:)
    # PayPal-specific logic
  end
end

# Usage
strategy = params[:method] == 'stripe' ? StripeStrategy.new : PaypalStrategy.new
processor = PaymentProcessor.new(strategy: strategy)
processor.charge(amount: 100, details: payment_details)
```

### 2. Notification Channels

```ruby
# Strategy interface
class NotificationStrategy
  def deliver(recipient:, message:)
    raise NotImplementedError
  end
end

# Concrete strategies
class EmailStrategy < NotificationStrategy
  def deliver(recipient:, message:)
    NotificationMailer.send_notification(
      to: recipient.email,
      subject: message[:subject],
      body: message[:body]
    ).deliver_later
  end
end

class SmsStrategy < NotificationStrategy
  def deliver(recipient:, message:)
    TwilioClient.send_sms(
      to: recipient.phone,
      body: message[:body]
    )
  end
end

class PushStrategy < NotificationStrategy
  def deliver(recipient:, message:)
    FCM.send_notification(
      device_token: recipient.device_token,
      title: message[:subject],
      body: message[:body]
    )
  end
end

# Context
class NotificationSender
  def initialize(strategy:)
    @strategy = strategy
  end

  def send(recipient:, message:)
    @strategy.deliver(recipient: recipient, message: message)
  end
end

# Usage based on user preference
strategy = case user.notification_preference
           when 'email' then EmailStrategy.new
           when 'sms' then SmsStrategy.new
           when 'push' then PushStrategy.new
           end

sender = NotificationSender.new(strategy: strategy)
sender.send(recipient: user, message: notification_message)
```

### 3. Export Formats

```ruby
# Strategy interface
class ExportStrategy
  def export(data:)
    raise NotImplementedError
  end

  def content_type
    raise NotImplementedError
  end
end

# Concrete strategies
class CsvStrategy < ExportStrategy
  def export(data:)
    CSV.generate(headers: true) do |csv|
      csv << data.first.keys
      data.each { |row| csv << row.values }
    end
  end

  def content_type
    'text/csv'
  end
end

class JsonStrategy < ExportStrategy
  def export(data:)
    data.to_json
  end

  def content_type
    'application/json'
  end
end

class PdfStrategy < ExportStrategy
  def export(data:)
    Prawn::Document.new do |pdf|
      # PDF generation logic
    end.render
  end

  def content_type
    'application/pdf'
  end
end

# Context
class DataExporter
  def initialize(strategy:)
    @strategy = strategy
  end

  def export(data:)
    {
      content: @strategy.export(data: data),
      content_type: @strategy.content_type
    }
  end
end

# Usage
strategy = case params[:format]
           when 'csv' then CsvStrategy.new
           when 'json' then JsonStrategy.new
           when 'pdf' then PdfStrategy.new
           end

exporter = DataExporter.new(strategy: strategy)
result = exporter.export(data: @records)

send_data result[:content], type: result[:content_type]
```

### 4. Authentication Methods

```ruby
# Multiple authentication strategies
class PasswordStrategy < AuthStrategy
  def authenticate(credentials:)
    user = User.find_by(email: credentials[:email])
    user&.authenticate(credentials[:password]) ? { success: true, user: user } : { success: false }
  end
end

class OauthStrategy < AuthStrategy
  def authenticate(credentials:)
    # OAuth logic
  end
end

class TokenStrategy < AuthStrategy
  def authenticate(credentials:)
    # JWT token logic
  end
end

class SsoStrategy < AuthStrategy
  def authenticate(credentials:)
    # SSO logic
  end
end
```

## Implementation Guidelines

### 1. Define Clear Strategy Interface

```ruby
# app/strategies/payments/payment_strategy.rb
module Payments
  class PaymentStrategy
    # All strategies must implement these methods
    def charge(amount:, payment_details:)
      raise NotImplementedError
    end

    def refund(transaction_id:, amount:)
      raise NotImplementedError
    end
  end
end
```

### 2. Keep Strategies Stateless

```ruby
# ❌ Bad: Stateful strategy
class BadStrategy
  attr_accessor :last_result  # State!

  def execute
    @last_result = do_something  # Storing state
  end
end

# ✅ Good: Stateless strategy
class GoodStrategy
  def execute
    do_something  # Returns result, doesn't store it
  end
end
```

### 3. Make Strategies Independent

```ruby
# ❌ Bad: Strategy depends on another
class BadEmailStrategy
  def deliver(recipient:, message:)
    if email_fails
      SmsStrategy.new.deliver(recipient, message)  # Coupling!
    end
  end
end

# ✅ Good: Context handles fallback
class NotificationSender
  def initialize(primary:, fallback: nil)
    @primary = primary
    @fallback = fallback
  end

  def send(recipient:, message:)
    result = @primary.deliver(recipient: recipient, message: message)
    @fallback.deliver(recipient: recipient, message: message) if !result[:success] && @fallback
  end
end
```

### 4. Don't Put Business Logic in Strategies

```ruby
# ❌ Bad: Business logic in strategy
class BadPaymentStrategy
  def charge(amount:, details:)
    fee = amount * 0.029 + 0.30  # Business logic!
    total = amount + fee
    validate_business_rules!(total)  # Business logic!
    process_charge(total, details)
  end
end

# ✅ Good: Business logic in service
class PaymentProcessor
  def charge(amount:, details:)
    fee = calculate_fee(amount)  # Business logic here
    total = amount + fee
    validate_amount!(total)      # Business logic here
    @strategy.charge(amount: total, details: details)  # Strategy just executes
  end
end
```

## Testing Strategies

```ruby
# Shared examples for strategy interface
RSpec.shared_examples 'a payment strategy' do
  it 'responds to charge' do
    expect(subject).to respond_to(:charge)
  end

  it 'responds to refund' do
    expect(subject).to respond_to(:refund)
  end

  it 'returns success hash on charge' do
    result = subject.charge(amount: 100, payment_details: valid_details)
    expect(result).to have_key(:success)
    expect(result).to have_key(:transaction_id)
  end
end

# Test concrete strategy
RSpec.describe StripeStrategy do
  it_behaves_like 'a payment strategy'

  describe '#charge' do
    it 'successfully charges card' do
      result = subject.charge(amount: 100, payment_details: valid_details)
      expect(result[:success]).to be true
    end
  end
end

# Test context with mock strategies
RSpec.describe PaymentProcessor do
  let(:strategy) { instance_double(PaymentStrategy) }
  subject { described_class.new(strategy: strategy) }

  it 'delegates to strategy' do
    expect(strategy).to receive(:charge).with(amount: 100, payment_details: anything)
    subject.charge(amount: 100, details: {})
  end
end
```

## Decision Tree

### When to use Strategy vs other patterns:

**Need multiple algorithms that are interchangeable?**
→ YES: Use Strategy Pattern
→ NO: Keep reading

**Need undo/redo functionality?**
→ YES: Use Command Pattern
→ NO: Keep reading

**Algorithm needs to know object's internal state?**
→ YES: Use State Pattern
→ NO: Keep reading

**Need algorithm skeleton with customizable steps?**
→ YES: Use Template Method Pattern
→ NO: Keep reading

**Simple conditional with 2 options?**
→ YES: Use simple if/else or ternary
→ NO: Consider Strategy

## Benefits

✅ **Runtime flexibility** - Swap algorithms at runtime
✅ **Eliminates conditionals** - No large case/if statements
✅ **Open/Closed Principle** - Add strategies without modifying context
✅ **Single Responsibility** - Each strategy handles one algorithm
✅ **Easy to test** - Test strategies independently

## Drawbacks

❌ **Increased complexity** - More classes to maintain
❌ **Client must know strategies** - Client needs to choose appropriate strategy
❌ **Overkill for simple cases** - 2-3 variants might not justify pattern

## Real-World Rails Examples

### Shipping Calculators

```ruby
class UpsStrategy < ShippingStrategy
  def calculate(package:, destination:)
    # UPS API call
  end
end

class FedexStrategy < ShippingStrategy
  def calculate(package:, destination:)
    # FedEx API call
  end
end

class UspsStrategy < ShippingStrategy
  def calculate(package:, destination:)
    # USPS API call
  end
end

# Usage
calculator = ShippingCalculator.new(strategy: UpsStrategy.new)
cost = calculator.calculate(package: package, destination: address)
```

### Tax Calculators

```ruby
class UsTaxStrategy < TaxStrategy
  def calculate(amount:, state:)
    # US tax logic
  end
end

class EuTaxStrategy < TaxStrategy
  def calculate(amount:, country:)
    # EU VAT logic
  end
end

class InternationalTaxStrategy < TaxStrategy
  def calculate(amount:, country:)
    # International tax logic
  end
end
```

### Search Engines

```ruby
class ElasticsearchStrategy < SearchStrategy
  def search(query:, filters:)
    # Elasticsearch query
  end
end

class PostgresFullTextStrategy < SearchStrategy
  def search(query:, filters:)
    # PostgreSQL full-text search
  end
end

class AlgoliaStrategy < SearchStrategy
  def search(query:, filters:)
    # Algolia API
  end
end
```

## Summary

**Use Strategy Pattern when:**
- You have 3+ interchangeable algorithms
- You need to swap behavior at runtime
- You want to eliminate large conditional statements
- Algorithms are independent and don't need object state

**Avoid Strategy Pattern when:**
- You have only 1-2 variants (overkill)
- Algorithm needs object state (use State instead)
- You need undo/redo (use Command instead)
- You need algorithm skeleton (use Template Method instead)

**Most common Rails use cases:**
1. Payment gateways (Stripe, PayPal, etc.)
2. Notification channels (Email, SMS, Push)
3. Export formats (CSV, JSON, PDF)
4. Authentication methods (Password, OAuth, Token, SSO)
5. Shipping calculators (UPS, FedEx, USPS)
6. Tax calculators (US, EU, International)
7. Search engines (Elasticsearch, PostgreSQL, Algolia)
