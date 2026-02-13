---
name: chain-of-responsibility-pattern
description: Passes requests along handler chain with Chain of Responsibility Pattern. Use for approval workflows, validation chains, error handling, or when multiple objects can handle a request.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Chain of Responsibility Pattern in Rails

## Overview

The Chain of Responsibility Pattern passes a request along a chain of handlers. Upon receiving a request, each handler decides either to process the request or to pass it to the next handler in the chain.

**Key Insight**: Decouple sender from receivers by giving multiple objects a chance to handle the request. Chain handlers and pass request along until handled.

## Core Components

```
Client → Handler1 → Handler2 → Handler3 → ... → HandlerN
         (check → process or pass to next)
```

1. **Handler Interface** - Defines handling method and successor link
2. **Base Handler** - Implements chain mechanics (optional but recommended)
3. **Concrete Handlers** - Actual handling logic with can_handle? checks
4. **Client** - Initiates request to first handler in chain

## When to Use Chain of Responsibility

✅ **Use Chain of Responsibility when you need:**

- **Runtime handler selection** - Don't know which handler will process request
- **Multiple handlers** - Several objects can handle request
- **Decouple sender/receiver** - Client doesn't need to know handler
- **Dynamic chain** - Handler set changes at runtime
- **Try until handled** - Pass along chain until one succeeds

❌ **Don't use Chain of Responsibility for:**

- Single known handler (use direct call)
- All handlers must execute (use Observer)
- Fixed sequence, no skipping (use Template Method)
- Simple if/else (over-engineering)

## Difference from Similar Patterns

| Aspect | Chain of Responsibility | Observer | Decorator | Template Method |
|--------|-------------------------|----------|-----------|-----------------|
| Purpose | Conditional handling | Notify all | Add behavior | Algorithm skeleton |
| Processing | One processes | All notified | All execute | Fixed sequence |
| Early exit | Yes | No | No | No |
| Mechanism | Pass along chain | Broadcast | Wrap | Inheritance |

## Common Rails Use Cases

### 1. Multi-Level Approval Workflow

```ruby
# Base handler
class ApprovalHandler
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def approve(purchase_order)
    if can_approve?(purchase_order)
      process_approval(purchase_order)
    elsif successor
      successor.approve(purchase_order)
    else
      raise ApprovalError, "No handler can approve this order"
    end
  end

  private

  def can_approve?(purchase_order)
    raise NotImplementedError
  end

  def process_approval(purchase_order)
    raise NotImplementedError
  end
end

# Concrete handlers
class ManagerApprovalHandler < ApprovalHandler
  LIMIT = 1_000

  private

  def can_approve?(purchase_order)
    purchase_order.amount < LIMIT
  end

  def process_approval(purchase_order)
    purchase_order.update!(
      approved: true,
      approved_by: "Manager",
      approved_at: Time.current
    )
    PurchaseOrderMailer.approved(purchase_order).deliver_later
  end
end

class DirectorApprovalHandler < ApprovalHandler
  LIMIT = 10_000

  private

  def can_approve?(purchase_order)
    purchase_order.amount < LIMIT
  end

  def process_approval(purchase_order)
    purchase_order.update!(
      approved: true,
      approved_by: "Director",
      approved_at: Time.current
    )
    PurchaseOrderMailer.approved(purchase_order).deliver_later
  end
end

class VpApprovalHandler < ApprovalHandler
  private

  def can_approve?(purchase_order)
    true  # VP can approve anything
  end

  def process_approval(purchase_order)
    purchase_order.update!(
      approved: true,
      approved_by: "VP",
      approved_at: Time.current
    )
    PurchaseOrderMailer.approved(purchase_order).deliver_later
  end
end

# Build chain
class ApprovalChain
  def self.build
    vp = VpApprovalHandler.new
    director = DirectorApprovalHandler.new(successor: vp)
    manager = ManagerApprovalHandler.new(successor: director)
    manager  # Return first handler
  end
end

# Usage
ApprovalChain.build.approve(purchase_order)
```

### 2. Validation Chain

```ruby
# Base validator
class ValidationHandler
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def validate(user)
    result = validate_step(user)

    if result.failure?
      result  # Stop on failure
    elsif successor
      successor.validate(user)
    else
      Success(user)  # All valid
    end
  end

  private

  def validate_step(user)
    raise NotImplementedError
  end
end

# Concrete validators
class EmailValidationHandler < ValidationHandler
  private

  def validate_step(user)
    if user.email.blank?
      Failure("Email is required")
    elsif !valid_email?(user.email)
      Failure("Email format is invalid")
    elsif User.exists?(email: user.email)
      Failure("Email already taken")
    else
      Success(user)
    end
  end

  def valid_email?(email)
    email.match?(URI::MailTo::EMAIL_REGEXP)
  end
end

class PasswordValidationHandler < ValidationHandler
  private

  def validate_step(user)
    if user.password.blank?
      Failure("Password is required")
    elsif user.password.length < 8
      Failure("Password must be at least 8 characters")
    elsif !strong_password?(user.password)
      Failure("Password must include uppercase, lowercase, and number")
    else
      Success(user)
    end
  end

  def strong_password?(password)
    password.match?(/(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  end
end

class AgeValidationHandler < ValidationHandler
  private

  def validate_step(user)
    if user.birth_date.blank?
      Failure("Birth date is required")
    elsif user.age < 18
      Failure("Must be 18 or older")
    else
      Success(user)
    end
  end
end

# Build chain
class UserValidationChain
  def self.build
    age = AgeValidationHandler.new
    password = PasswordValidationHandler.new(successor: age)
    email = EmailValidationHandler.new(successor: password)
    email
  end
end

# Usage
result = UserValidationChain.build.validate(user)
if result.success?
  user.save!
else
  flash[:alert] = result.failure
end
```

### 3. Support Ticket Routing

```ruby
class TicketRouter
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def route(ticket)
    if can_handle?(ticket)
      assign_to_team(ticket)
    elsif successor
      successor.route(ticket)
    else
      raise RoutingError, "No team can handle this ticket"
    end
  end

  private

  def can_handle?(ticket)
    raise NotImplementedError
  end

  def assign_to_team(ticket)
    raise NotImplementedError
  end
end

class UrgentTicketRouter < TicketRouter
  private

  def can_handle?(ticket)
    ticket.priority == 'urgent'
  end

  def assign_to_team(ticket)
    ticket.update!(assigned_to_team: 'Emergency Response')
    Team.find_by(name: 'Emergency Response').notify_urgent(ticket)
  end
end

class BillingTicketRouter < TicketRouter
  private

  def can_handle?(ticket)
    ticket.category == 'billing'
  end

  def assign_to_team(ticket)
    ticket.update!(assigned_to_team: 'Billing')
  end
end

class TechnicalTicketRouter < TicketRouter
  private

  def can_handle?(ticket)
    ticket.category == 'technical'
  end

  def assign_to_team(ticket)
    ticket.update!(assigned_to_team: 'Engineering')
  end
end

class GeneralTicketRouter < TicketRouter
  private

  def can_handle?(ticket)
    true  # Catch-all
  end

  def assign_to_team(ticket)
    ticket.update!(assigned_to_team: 'Customer Support')
  end
end

# Build chain
class TicketRoutingChain
  def self.build
    general = GeneralTicketRouter.new
    technical = TechnicalTicketRouter.new(successor: general)
    billing = BillingTicketRouter.new(successor: technical)
    urgent = UrgentTicketRouter.new(successor: billing)
    urgent
  end
end

# Usage
TicketRoutingChain.build.route(ticket)
```

### 4. Error Handler Chain

```ruby
class ErrorHandler
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def handle(error, context)
    if can_handle?(error)
      process_error(error, context)
    elsif successor
      successor.handle(error, context)
    else
      raise error  # Re-raise if no handler
    end
  end

  private

  def can_handle?(error)
    raise NotImplementedError
  end

  def process_error(error, context)
    raise NotImplementedError
  end
end

class ValidationErrorHandler < ErrorHandler
  private

  def can_handle?(error)
    error.is_a?(ActiveRecord::RecordInvalid)
  end

  def process_error(error, context)
    context[:controller].render json: {
      error: 'Validation failed',
      details: error.record.errors.full_messages
    }, status: :unprocessable_entity
  end
end

class AuthenticationErrorHandler < ErrorHandler
  private

  def can_handle?(error)
    error.is_a?(AuthenticationError)
  end

  def process_error(error, context)
    context[:controller].redirect_to login_path, alert: 'Please log in'
  end
end

class NotFoundErrorHandler < ErrorHandler
  private

  def can_handle?(error)
    error.is_a?(ActiveRecord::RecordNotFound)
  end

  def process_error(error, context)
    context[:controller].render file: 'public/404.html', status: :not_found
  end
end

class GenericErrorHandler < ErrorHandler
  private

  def can_handle?(error)
    true  # Catch all
  end

  def process_error(error, context)
    ErrorTracker.report(error)
    context[:controller].render json: {
      error: 'Internal server error'
    }, status: :internal_server_error
  end
end

# Build chain
class ErrorHandlerChain
  def self.build
    generic = GenericErrorHandler.new
    not_found = NotFoundErrorHandler.new(successor: generic)
    auth = AuthenticationErrorHandler.new(successor: not_found)
    validation = ValidationErrorHandler.new(successor: auth)
    validation
  end
end

# Usage in ApplicationController
class ApplicationController < ActionController::Base
  rescue_from StandardError do |error|
    ErrorHandlerChain.build.handle(error, { controller: self })
  end
end
```

## Implementation Guidelines

### 1. Define Handler Interface

```ruby
class Handler
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def handle(request)
    if can_handle?(request)
      process(request)
    elsif successor
      successor.handle(request)
    else
      handle_end_of_chain(request)
    end
  end

  private

  def can_handle?(request)
    raise NotImplementedError, "#{self.class} must implement #can_handle?"
  end

  def process(request)
    raise NotImplementedError, "#{self.class} must implement #process"
  end

  def handle_end_of_chain(request)
    raise NoHandlerError, "No handler for #{request.inspect}"
  end
end
```

### 2. Keep Handlers Independent

```ruby
# ✅ Good: Handler is self-contained
class ManagerHandler < ApprovalHandler
  LIMIT = 1_000

  def can_approve?(order)
    order.amount < LIMIT
  end

  def process_approval(order)
    order.update!(approved_by: 'Manager')
  end
end

# ❌ Bad: Handler depends on other handlers
class BadManagerHandler < ApprovalHandler
  def can_approve?(order)
    # Don't reference other handlers!
    !DirectorHandler.new.can_approve?(order)
  end
end
```

### 3. Provide Chain Builder

```ruby
# ✅ Good: Centralized chain building
class ChainBuilder
  def self.build
    # Build from last to first
    handler3 = ConcreteHandler3.new
    handler2 = ConcreteHandler2.new(successor: handler3)
    handler1 = ConcreteHandler1.new(successor: handler2)
    handler1  # Return first handler
  end
end

# Usage
chain = ChainBuilder.build
chain.handle(request)
```

### 4. Handle End of Chain

```ruby
# Option 1: Raise error
def handle(request)
  if can_handle?(request)
    process(request)
  elsif successor
    successor.handle(request)
  else
    raise NoHandlerError, "No handler found"
  end
end

# Option 2: Default behavior
def handle(request)
  if can_handle?(request)
    process(request)
  elsif successor
    successor.handle(request)
  else
    default_process(request)
  end
end

# Option 3: Return result (dry-monads)
def handle(request)
  if can_handle?(request)
    Success(process(request))
  elsif successor
    successor.handle(request)
  else
    Failure("Not handled")
  end
end
```

## Testing Chain of Responsibility

```ruby
# Test individual handler
RSpec.describe ManagerApprovalHandler do
  describe '#approve' do
    context 'with order under limit' do
      let(:order) { create(:purchase_order, amount: 500) }
      subject { described_class.new }

      it 'approves the order' do
        expect { subject.approve(order) }
          .to change { order.reload.approved }.to(true)
      end

      it 'sets approver' do
        subject.approve(order)
        expect(order.reload.approved_by).to eq('Manager')
      end

      it 'does not call successor' do
        subject.approve(order)
        # No successor called
      end
    end

    context 'with order over limit' do
      let(:order) { create(:purchase_order, amount: 5000) }
      let(:successor) { instance_double(DirectorApprovalHandler) }
      subject { described_class.new(successor: successor) }

      it 'passes to successor' do
        expect(successor).to receive(:approve).with(order)
        subject.approve(order)
      end

      it 'does not approve itself' do
        allow(successor).to receive(:approve)
        expect { subject.approve(order) }
          .not_to change { order.reload.approved }
      end
    end

    context 'with no successor' do
      let(:order) { create(:purchase_order, amount: 5000) }
      subject { described_class.new }

      it 'raises error' do
        expect { subject.approve(order) }
          .to raise_error(ApprovalError, /No handler/)
      end
    end
  end
end

# Test full chain
RSpec.describe 'Approval chain' do
  let(:chain) { ApprovalChain.build }

  it 'routes $500 to manager' do
    order = create(:purchase_order, amount: 500)
    chain.approve(order)
    expect(order.reload.approved_by).to eq('Manager')
  end

  it 'routes $5000 to director' do
    order = create(:purchase_order, amount: 5000)
    chain.approve(order)
    expect(order.reload.approved_by).to eq('Director')
  end

  it 'routes $50000 to VP' do
    order = create(:purchase_order, amount: 50000)
    chain.approve(order)
    expect(order.reload.approved_by).to eq('VP')
  end
end

# Test chain builder
RSpec.describe ApprovalChain do
  describe '.build' do
    it 'builds chain in correct order' do
      chain = described_class.build
      expect(chain).to be_a(ManagerApprovalHandler)
      expect(chain.successor).to be_a(DirectorApprovalHandler)
      expect(chain.successor.successor).to be_a(VpApprovalHandler)
    end

    it 'has no successor after VP' do
      chain = described_class.build
      expect(chain.successor.successor.successor).to be_nil
    end
  end
end
```

## Anti-Patterns to Avoid

### ❌ Don't Make Handlers Depend on Each Other

```ruby
# ❌ Bad: Handler knows about others
class BadHandler < Handler
  def can_handle?(request)
    !OtherHandler.new.can_handle?(request)  # Tight coupling!
  end
end

# ✅ Good: Handler is independent
class GoodHandler < Handler
  def can_handle?(request)
    request.type == :specific_type
  end
end
```

### ❌ Don't Put Business Logic in Builder

```ruby
# ❌ Bad: Business logic in builder
class BadChainBuilder
  def self.build(user)
    if user.premium?
      # Business logic here!
      PremiumHandler.new
    else
      StandardHandler.new
    end
  end
end

# ✅ Good: Builder just builds
class GoodChainBuilder
  def self.build_premium
    PremiumHandler.new(successor: FallbackHandler.new)
  end

  def self.build_standard
    StandardHandler.new(successor: FallbackHandler.new)
  end
end

# Logic in service
class Service
  def process(user, request)
    chain = user.premium? ? ChainBuilder.build_premium : ChainBuilder.build_standard
    chain.handle(request)
  end
end
```

### ❌ Don't Skip End-of-Chain Handling

```ruby
# ❌ Bad: Silent failure
def handle(request)
  if can_handle?(request)
    process(request)
  elsif successor
    successor.handle(request)
  end
  # What if no handler? Silent failure!
end

# ✅ Good: Explicit end handling
def handle(request)
  if can_handle?(request)
    process(request)
  elsif successor
    successor.handle(request)
  else
    raise NoHandlerError, "No handler for #{request.inspect}"
  end
end
```

## Decision Tree

### When to use Chain of Responsibility vs alternatives:

**Single known handler at compile time?**
→ YES: Use direct call
→ NO: Keep reading

**All handlers must execute?**
→ YES: Use Observer Pattern
→ NO: Keep reading

**Fixed sequence with no skipping?**
→ YES: Use Template Method Pattern
→ NO: Keep reading

**Multiple possible handlers, determined at runtime?**
→ YES: Use Chain of Responsibility ✅

**Handler set changes dynamically?**
→ YES: Use Chain of Responsibility ✅

**Want to decouple sender from receiver?**
→ YES: Use Chain of Responsibility ✅

## Benefits

✅ **Decoupling** - Sender doesn't need to know receiver
✅ **Single Responsibility** - Each handler has one job
✅ **Open/Closed** - Add handlers without modifying existing
✅ **Flexibility** - Change chain order at runtime
✅ **Early exit** - Stop processing when handled

## Drawbacks

❌ **No guarantee** - Request might not be handled
❌ **Runtime uncertainty** - Don't know which handler will process
❌ **Debugging complexity** - Hard to trace chain execution
❌ **Performance** - Request passes through multiple handlers

## Real-World Rails Examples

### Discount Calculation Chain

```ruby
class DiscountHandler
  def calculate(order)
    if applies_to?(order)
      apply_discount(order)
    elsif successor
      successor.calculate(order)
    else
      0  # No discount
    end
  end
end

class CouponDiscountHandler < DiscountHandler
  def applies_to?(order)
    order.coupon.present?
  end

  def apply_discount(order)
    order.coupon.discount_amount
  end
end

class LoyaltyDiscountHandler < DiscountHandler
  def applies_to?(order)
    order.user.loyalty_points >= 100
  end

  def apply_discount(order)
    order.total * 0.1
  end
end
```

### Content Moderation Pipeline

```ruby
class ModerationHandler
  def moderate(content)
    if violates?(content)
      take_action(content)
    elsif successor
      successor.moderate(content)
    else
      approve(content)
    end
  end
end

class SpamModerationHandler < ModerationHandler
  def violates?(content)
    SpamDetector.spam?(content.text)
  end

  def take_action(content)
    content.mark_as_spam!
  end
end

class ProfanityModerationHandler < ModerationHandler
  def violates?(content)
    ProfanityFilter.contains_profanity?(content.text)
  end

  def take_action(content)
    content.censor_profanity!
  end
end
```

### Request Authorization Chain

```ruby
class AuthorizationHandler
  def authorize(user, resource)
    if can_authorize?(user, resource)
      grant_access
    elsif successor
      successor.authorize(user, resource)
    else
      deny_access
    end
  end
end

class OwnerAuthorizationHandler < AuthorizationHandler
  def can_authorize?(user, resource)
    resource.owner == user
  end
end

class AdminAuthorizationHandler < AuthorizationHandler
  def can_authorize?(user, resource)
    user.admin?
  end
end
```

## Summary

**Use Chain of Responsibility when:**
- Multiple objects can handle request
- Handler determined at runtime
- Want to decouple sender from receivers
- Set of handlers changes dynamically

**Avoid Chain of Responsibility when:**
- Single handler (use direct call)
- All handlers execute (use Observer)
- Fixed sequence (use Template Method)
- Simple conditions (over-engineering)

**Most common Rails use cases:**
1. Multi-level approval workflows (manager → director → VP)
2. Validation chains (email → password → age → spam check)
3. Support ticket routing (urgent → billing → technical → general)
4. Error handling chains (validation → auth → not found → generic)
5. Request filtering/middleware
6. Authorization chains (owner → admin → super admin)
7. Discount calculation (coupon → loyalty → seasonal)
8. Content moderation (spam → profanity → copyright)

**Key Pattern Structure:**
```ruby
# 1. Base handler
class Handler
  attr_reader :successor

  def initialize(successor: nil)
    @successor = successor
  end

  def handle(request)
    if can_handle?(request)
      process(request)
    elsif successor
      successor.handle(request)
    else
      raise "Not handled"
    end
  end

  private

  def can_handle?(request)
    raise NotImplementedError
  end

  def process(request)
    raise NotImplementedError
  end
end

# 2. Concrete handlers
class ConcreteHandlerA < Handler
  private

  def can_handle?(request)
    request.type == :a
  end

  def process(request)
    # Handle A
  end
end

# 3. Build chain
handler3 = ConcreteHandlerC.new
handler2 = ConcreteHandlerB.new(successor: handler3)
handler1 = ConcreteHandlerA.new(successor: handler2)

# 4. Use
handler1.handle(request)
```
