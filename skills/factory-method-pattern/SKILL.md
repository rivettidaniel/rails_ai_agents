---
name: factory-method-pattern
description: Creates objects through factory methods with Factory Method Pattern. Use for polymorphic object creation, notification systems, report generators, authentication providers, or when you need framework extensibility.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Factory Method Pattern in Rails

## Overview

The Factory Method Pattern defines an interface for creating objects, but lets subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

**Key Insight**: Replace direct object creation with factory methods that can be overridden to produce different product types polymorphically.

## Core Components

```
Client → Creator (base) → Product (interface)
           ↓                  ↓
    Concrete Creators → Concrete Products
```

1. **Product Interface** - Declares operations all products must implement
2. **Concrete Products** - Different implementations of product interface
3. **Creator (Base)** - Declares factory method returning products
4. **Concrete Creators** - Override factory method to return specific products

## When to Use Factory Method

✅ **Use Factory Method when you need:**

- **Polymorphic object creation** - Don't know exact types beforehand
- **Framework extensibility** - Let users extend with new product types
- **Decouple creation from usage** - Client doesn't depend on concrete classes
- **Open/Closed Principle** - Add new products without modifying existing code

❌ **Don't use Factory Method for:**

- Simple object creation (use constructor)
- Creating families of related objects (use Abstract Factory)
- Complex step-by-step construction (use Builder)
- Just centralizing creation logic (use Simple Factory)

## Difference from Similar Patterns

| Aspect | Factory Method | Abstract Factory | Builder | Simple Factory |
|--------|----------------|------------------|---------|----------------|
| Purpose | Polymorphic creation | Family creation | Step-by-step | Centralized creation |
| Inheritance | Uses subclasses | Composition | Composition | Static method |
| Products | Single product type | Multiple products | Complex product | Single product |
| Extensibility | High (subclass) | High (new factory) | High (new steps) | Low (modify method) |

## Common Rails Use Cases

### 1. Notification System

```ruby
# Product interface
class Notification
  def send
    raise NotImplementedError
  end

  def recipient
    raise NotImplementedError
  end
end

# Concrete products
class EmailNotification < Notification
  def initialize(user:, message:, subject:)
    @user, @message, @subject = user, message, subject
  end

  def send
    NotificationMailer.send_notification(
      to: @user.email,
      subject: @subject,
      body: @message
    ).deliver_later
  end

  def recipient
    @user.email
  end
end

class SmsNotification < Notification
  def initialize(user:, message:)
    @user, @message = user, message
  end

  def send
    TwilioClient.send_sms(to: @user.phone, body: @message)
  end

  def recipient
    @user.phone
  end
end

# Base factory (Creator)
class NotificationFactory
  # Factory method to be overridden
  def create_notification(user:, message:, **options)
    raise NotImplementedError
  end

  # Template method using factory method
  def send_notification(user:, message:, **options)
    notification = create_notification(user: user, message: message, **options)
    notification.send
  end

  # Static factory to get appropriate factory
  def self.for(type)
    case type
    when :email then EmailNotificationFactory.new
    when :sms then SmsNotificationFactory.new
    when :push then PushNotificationFactory.new
    end
  end
end

# Concrete factories
class EmailNotificationFactory < NotificationFactory
  def create_notification(user:, message:, subject: "Notification", **options)
    EmailNotification.new(user: user, message: message, subject: subject)
  end
end

class SmsNotificationFactory < NotificationFactory
  def create_notification(user:, message:, **options)
    SmsNotification.new(user: user, message: message)
  end
end

# Usage
factory = NotificationFactory.for(user.notification_preference)
factory.send_notification(user: user, message: "Hello!")
```

### 2. Report Generators

```ruby
# Product interface
class Report
  def generate
    raise NotImplementedError
  end

  def content_type
    raise NotImplementedError
  end
end

# Concrete products
class PdfReport < Report
  def initialize(data:)
    @data = data
  end

  def generate
    Prawn::Document.new { |pdf| # PDF generation }.render
  end

  def content_type
    'application/pdf'
  end
end

class ExcelReport < Report
  def initialize(data:)
    @data = data
  end

  def generate
    # Excel generation
  end

  def content_type
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
  end
end

# Factory
class ReportFactory
  def create_report(data:)
    raise NotImplementedError
  end

  def self.for(format)
    case format
    when :pdf then PdfReportFactory.new
    when :excel then ExcelReportFactory.new
    when :csv then CsvReportFactory.new
    end
  end
end

class PdfReportFactory < ReportFactory
  def create_report(data:)
    PdfReport.new(data: data)
  end
end

# Usage
factory = ReportFactory.for(params[:format])
report = factory.create_report(data: @sales_data)
send_data report.generate, type: report.content_type
```

### 3. Authentication Providers

```ruby
# Product interface
class AuthProvider
  def authenticate(credentials)
    raise NotImplementedError
  end
end

# Concrete products
class PasswordAuthProvider < AuthProvider
  def authenticate(credentials)
    user = User.find_by(email: credentials[:email])
    user&.authenticate(credentials[:password]) ? user : nil
  end
end

class OauthAuthProvider < AuthProvider
  def initialize(provider)
    @provider = provider
  end

  def authenticate(credentials)
    User.find_or_create_by_oauth(@provider, credentials[:auth_hash])
  end
end

class TokenAuthProvider < AuthProvider
  def authenticate(credentials)
    decoded = JWT.decode(credentials[:token], Rails.application.secret_key_base)
    User.find(decoded[0]['user_id'])
  rescue JWT::DecodeError
    nil
  end
end

# Factory
class AuthProviderFactory
  def create_provider
    raise NotImplementedError
  end

  def self.for(type, **options)
    case type
    when :password then PasswordAuthProviderFactory.new
    when :oauth then OauthAuthProviderFactory.new(options[:provider])
    when :token then TokenAuthProviderFactory.new
    end
  end
end

# Usage in SessionsController
factory = AuthProviderFactory.for(auth_type, provider: params[:provider])
provider = factory.create_provider

if user = provider.authenticate(credentials)
  sign_in(user)
  redirect_to dashboard_path
else
  render :new, alert: "Authentication failed"
end
```

### 4. Payment Processors

```ruby
class PaymentProcessorFactory
  def create_processor
    raise NotImplementedError
  end

  def self.for(gateway)
    case gateway
    when :stripe then StripeProcessorFactory.new
    when :paypal then PaypalProcessorFactory.new
    when :braintree then BraintreeProcessorFactory.new
    end
  end
end

# Usage
factory = PaymentProcessorFactory.for(user.preferred_gateway)
processor = factory.create_processor
result = processor.process(amount: order.total, details: payment_details)
```

## Implementation Guidelines

### 1. Define Clear Product Interface

```ruby
# All products must implement common interface
class Product
  def operation
    raise NotImplementedError, "#{self.class} must implement #operation"
  end
end
```

### 2. Use Factory Method in Template Methods

```ruby
class Creator
  # Factory method
  def create_product
    raise NotImplementedError
  end

  # Template method using factory
  def execute
    product = create_product  # Factory method
    product.operation
    log_result(product)
  end

  private

  def log_result(product)
    # Common logic
  end
end
```

### 3. Provide Static Factory for Convenience

```ruby
class NotificationFactory
  # Instance factory method (to be overridden)
  def create_notification(user:, message:, **options)
    raise NotImplementedError
  end

  # Static factory (convenience)
  def self.for(type)
    case type
    when :email then EmailNotificationFactory.new
    when :sms then SmsNotificationFactory.new
    when :push then PushNotificationFactory.new
    else raise ArgumentError, "Unknown type: #{type}"
    end
  end
end
```

### 4. Keep Factories Simple

```ruby
# ✅ Good: Factory just creates
class EmailNotificationFactory < NotificationFactory
  def create_notification(user:, message:, **options)
    EmailNotification.new(user: user, message: message)
  end
end

# ❌ Bad: Factory has business logic
class BadEmailNotificationFactory < NotificationFactory
  def create_notification(user:, message:, **options)
    message = add_branding(message) if user.premium?  # Business logic!
    EmailNotification.new(user: user, message: message)
  end
end
```

## Testing Factory Method

```ruby
# Shared examples for product interface
RSpec.shared_examples 'a notification' do
  it 'responds to send' do
    expect(subject).to respond_to(:send)
  end

  it 'responds to recipient' do
    expect(subject).to respond_to(:recipient)
  end
end

# Test concrete product
RSpec.describe EmailNotification do
  it_behaves_like 'a notification'

  describe '#send' do
    it 'enqueues email' do
      expect { subject.send }.to have_enqueued_mail
    end
  end
end

# Test factory
RSpec.describe EmailNotificationFactory do
  describe '#create_notification' do
    it 'creates EmailNotification' do
      notification = subject.create_notification(
        user: user,
        message: "Test"
      )

      expect(notification).to be_a(EmailNotification)
    end
  end
end

# Test static factory
RSpec.describe NotificationFactory do
  describe '.for' do
    it 'returns EmailNotificationFactory for :email' do
      factory = described_class.for(:email)
      expect(factory).to be_a(EmailNotificationFactory)
    end

    it 'raises error for unknown type' do
      expect {
        described_class.for(:unknown)
      }.to raise_error(ArgumentError)
    end
  end
end
```

## Anti-Patterns to Avoid

### ❌ Don't Put Business Logic in Factories

```ruby
# ❌ Bad
class BadFactory < NotificationFactory
  def create_notification(user:, message:, **options)
    # Business logic doesn't belong here!
    message = premium_template(message) if user.premium?
    schedule = user.timezone_adjusted_time

    EmailNotification.new(user: user, message: message, schedule: schedule)
  end
end

# ✅ Good: Factory creates, service has business logic
class GoodFactory < NotificationFactory
  def create_notification(user:, message:, **options)
    EmailNotification.new(user: user, message: message)
  end
end

class NotificationService
  def send(user:, message:)
    message = premium_template(message) if user.premium?
    factory = NotificationFactory.for(user.preference)
    notification = factory.create_notification(user: user, message: message)
    notification.send
  end
end
```

### ❌ Don't Skip Product Interface

```ruby
# ❌ Bad: No common interface
class EmailNotification
  def send_email; end
end

class SmsNotification
  def send_sms; end  # Different method name!
end

# ✅ Good: Common interface
class Notification
  def send
    raise NotImplementedError
  end
end

class EmailNotification < Notification
  def send
    # Email logic
  end
end
```

### ❌ Don't Create God Factories

```ruby
# ❌ Bad: Factory creates unrelated products
class GodFactory
  def create_user; end
  def create_post; end
  def create_notification; end
  def create_payment; end
end

# ✅ Good: Separate factories for each product family
class UserFactory; end
class NotificationFactory; end
class PaymentFactory; end
```

## Decision Tree

### When to use Factory Method vs alternatives:

**Creating a single object with known type?**
→ YES: Use constructor
→ NO: Keep reading

**Need step-by-step construction with many options?**
→ YES: Use Builder Pattern
→ NO: Keep reading

**Creating families of related objects?**
→ YES: Use Abstract Factory
→ NO: Keep reading

**Just centralizing simple creation logic?**
→ YES: Use Simple Factory (static method)
→ NO: Keep reading

**Need polymorphic creation with subclass extensibility?**
→ YES: Use Factory Method Pattern ✅

## Benefits

✅ **Decoupling** - Client doesn't depend on concrete classes
✅ **Extensibility** - Add new products without modifying existing code
✅ **Polymorphism** - Create objects without knowing exact types
✅ **Single Responsibility** - Creation logic separated from usage
✅ **Open/Closed** - Open for extension, closed for modification

## Drawbacks

❌ **More classes** - Requires factory hierarchy
❌ **Indirection** - Extra layer between client and product
❌ **Overkill** - Too complex for simple creation needs

## Real-World Rails Examples

### File Parsers

```ruby
class ParserFactory
  def self.for(format)
    case format
    when :csv then CsvParserFactory.new
    when :json then JsonParserFactory.new
    when :xml then XmlParserFactory.new
    end
  end
end

factory = ParserFactory.for(file.format)
parser = factory.create_parser
data = parser.parse(file.content)
```

### Shipping Calculators

```ruby
class ShippingCalculatorFactory
  def self.for(carrier)
    case carrier
    when :ups then UpsCalculatorFactory.new
    when :fedex then FedexCalculatorFactory.new
    when :usps then UspsCalculatorFactory.new
    end
  end
end

factory = ShippingCalculatorFactory.for(order.shipping_carrier)
calculator = factory.create_calculator
cost = calculator.calculate(order.weight, order.destination)
```

### Search Engines

```ruby
class SearchEngineFactory
  def self.for(engine)
    case engine
    when :elasticsearch then ElasticsearchFactory.new
    when :postgres then PostgresSearchFactory.new
    when :algolia then AlgoliaFactory.new
    end
  end
end

factory = SearchEngineFactory.for(Rails.configuration.search_engine)
search = factory.create_search
results = search.query(params[:q])
```

## Summary

**Use Factory Method when:**
- You don't know exact types beforehand
- You want framework extensibility
- You need polymorphic object creation
- Client shouldn't depend on concrete classes

**Avoid Factory Method when:**
- Simple object creation (use constructor)
- Creating families (use Abstract Factory)
- Complex construction (use Builder)
- Just centralizing (use Simple Factory)

**Most common Rails use cases:**
1. Notification systems (Email, SMS, Push, Slack)
2. Report generators (PDF, Excel, CSV, HTML)
3. Authentication providers (Password, OAuth, Token, SSO)
4. Payment processors (Stripe, PayPal, Braintree)
5. File parsers (CSV, JSON, XML, YAML)
6. Search engines (Elasticsearch, PostgreSQL, Algolia)
7. Shipping calculators (UPS, FedEx, USPS, DHL)
8. Logging adapters (File, Database, External service)

**Key Pattern:**
```ruby
# 1. Product interface
class Product
  def operation; raise NotImplementedError; end
end

# 2. Concrete products
class ConcreteProductA < Product
  def operation; end
end

# 3. Base factory
class Factory
  def create_product; raise NotImplementedError; end
  def self.for(type); end  # Static factory
end

# 4. Concrete factory
class ConcreteFactoryA < Factory
  def create_product
    ConcreteProductA.new
  end
end

# 5. Usage
factory = Factory.for(:a)
product = factory.create_product
product.operation
```
