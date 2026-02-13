---
name: state-pattern
description: Manages state-dependent behavior with State Pattern (Finite State Machine). Use for order workflows, document approval, user accounts, or when object behavior changes based on internal state.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# State Pattern in Rails

## Overview

The State Pattern allows an object to alter its behavior when its internal state changes. The object will appear to change its class.

**Key Insight**: Encapsulate each state as a class. Delegate state-specific behavior to current state object. Transitions change which state object is active.

## Core Components

```
Context → State (interface)
           ↓
    Concrete States (implement behavior + transitions)
```

1. **Context** - Maintains reference to current state, delegates behavior
2. **State Interface** - Defines state-specific methods all states must implement
3. **Concrete States** - Implement behavior for each state, handle transitions
4. **Transitions** - State objects can trigger state changes in context

## When to Use State Pattern

✅ **Use State Pattern when you need:**

- **State-dependent behavior** - Object behavior changes based on state
- **Eliminate conditionals** - Replace state-checking if/case statements
- **State machine** - Clear transitions between well-defined states
- **Encapsulate states** - Each state has complex logic worth separating

❌ **Don't use State Pattern for:**

- Simple flags (use boolean attribute)
- Algorithm selection (use Strategy)
- Fixed algorithm steps (use Template Method)
- Operation queuing (use Command)

## Difference from Similar Patterns

| Aspect | State | Strategy | Template Method | Command |
|--------|-------|----------|-----------------|---------|
| Purpose | State-dependent behavior | Algorithm selection | Algorithm skeleton | Encapsulate request |
| Switching | Automatic (internal) | Manual (external) | No switching | No switching |
| Awareness | States know each other | Strategies independent | Steps independent | Commands independent |
| Use case | Workflows, FSM | Interchangeable algorithms | Multi-step process | Operations |

## Common Rails Use Cases

### 1. Order Workflow

```ruby
# Context
class Order < ApplicationRecord
  # State: draft, submitted, paid, shipped, delivered, cancelled

  def current_state
    @current_state ||= state_class.new(self)
  end

  def transition_to(new_state)
    self.state = new_state
    @current_state = nil  # Reset cached state
    save!
  end

  # Delegate to current state
  def submit
    current_state.submit
  end

  def pay
    current_state.pay
  end

  def ship
    current_state.ship
  end

  def deliver
    current_state.deliver
  end

  def cancel
    current_state.cancel
  end

  def can_be_modified?
    current_state.can_be_modified?
  end

  private

  def state_class
    "Orders::#{state.to_s.camelize}State".constantize
  rescue NameError
    raise "Unknown state: #{state}"
  end
end

# Base state
module Orders
  class BaseState
    attr_reader :order

    def initialize(order)
      @order = order
    end

    # Default implementations (raise errors)
    def submit
      raise StateTransitionError, "Cannot submit from #{state_name}"
    end

    def pay
      raise StateTransitionError, "Cannot pay from #{state_name}"
    end

    def ship
      raise StateTransitionError, "Cannot ship from #{state_name}"
    end

    def deliver
      raise StateTransitionError, "Cannot deliver from #{state_name}"
    end

    def cancel
      raise StateTransitionError, "Cannot cancel from #{state_name}"
    end

    def can_be_modified?
      false
    end

    private

    def state_name
      self.class.name.demodulize.gsub('State', '').underscore
    end
  end
end

# Concrete states
module Orders
  class DraftState < BaseState
    def submit
      validate_order!
      order.transition_to(:submitted)
      OrderMailer.submitted(order).deliver_later
    end

    def cancel
      order.transition_to(:cancelled)
    end

    def can_be_modified?
      true
    end

    private

    def validate_order!
      raise ValidationError, "Order must have items" if order.items.empty?
      raise ValidationError, "Order must have shipping address" unless order.shipping_address
    end
  end

  class SubmittedState < BaseState
    def pay
      process_payment!
      order.transition_to(:paid)
      OrderMailer.paid(order).deliver_later
    end

    def cancel
      order.transition_to(:cancelled)
      refund_payment if order.payment_started?
    end

    private

    def process_payment!
      PaymentProcessor.charge(order)
    end

    def refund_payment
      PaymentProcessor.refund(order)
    end
  end

  class PaidState < BaseState
    def ship
      validate_inventory!
      update_inventory!
      order.transition_to(:shipped)
      OrderMailer.shipped(order).deliver_later
    end

    def cancel
      refund_payment!
      order.transition_to(:cancelled)
      OrderMailer.cancelled(order).deliver_later
    end

    private

    def validate_inventory!
      order.items.each do |item|
        raise InventoryError unless item.product.in_stock?
      end
    end

    def update_inventory!
      order.items.each { |item| item.product.decrement_inventory! }
    end

    def refund_payment!
      PaymentProcessor.refund(order)
    end
  end

  class ShippedState < BaseState
    def deliver
      order.update!(delivered_at: Time.current)
      order.transition_to(:delivered)
      OrderMailer.delivered(order).deliver_later
    end

    def cancel
      raise StateTransitionError, "Cannot cancel shipped order. Request return instead."
    end
  end

  class DeliveredState < BaseState
    # Terminal state - no transitions
    def cancel
      raise StateTransitionError, "Cannot cancel delivered order. Request return instead."
    end
  end

  class CancelledState < BaseState
    # Terminal state - no transitions
    def submit
      raise StateTransitionError, "Cannot resubmit cancelled order. Create new order."
    end
  end
end

# Usage in controller
class OrdersController < ApplicationController
  def submit
    @order = current_user.orders.find(params[:id])
    @order.submit
    redirect_to @order, notice: "Order submitted successfully"
  rescue StateTransitionError => e
    redirect_to @order, alert: e.message
  end

  def pay
    @order = current_user.orders.find(params[:id])
    @order.pay
    redirect_to @order, notice: "Payment processed successfully"
  rescue StateTransitionError => e
    redirect_to @order, alert: e.message
  end
end
```

### 2. Document Approval Workflow

```ruby
class Document < ApplicationRecord
  # States: draft, pending_review, approved, rejected, published

  def current_state
    @current_state ||= "Documents::#{state.to_s.camelize}State".constantize.new(self)
  end

  def submit_for_review
    current_state.submit_for_review
  end

  def approve
    current_state.approve
  end

  def reject(reason:)
    current_state.reject(reason: reason)
  end

  def publish
    current_state.publish
  end
end

module Documents
  class DraftState < BaseState
    def submit_for_review
      validate_document!
      document.transition_to(:pending_review)
      notify_reviewers
    end

    private

    def validate_document!
      raise ValidationError unless document.complete?
    end

    def notify_reviewers
      document.reviewers.each { |r| ReviewerMailer.new_document(r, document).deliver_later }
    end
  end

  class PendingReviewState < BaseState
    def approve
      document.update!(approved_by: Current.user, approved_at: Time.current)
      document.transition_to(:approved)
      DocumentMailer.approved(document).deliver_later
    end

    def reject(reason:)
      document.update!(rejected_by: Current.user, rejection_reason: reason)
      document.transition_to(:rejected)
      DocumentMailer.rejected(document).deliver_later
    end
  end

  class ApprovedState < BaseState
    def publish
      document.update!(published_at: Time.current)
      document.transition_to(:published)
      DocumentMailer.published(document).deliver_later
    end
  end

  class RejectedState < BaseState
    def submit_for_review
      document.update!(rejection_reason: nil)
      document.transition_to(:pending_review)
    end
  end

  class PublishedState < BaseState
    # Terminal state - can only unpublish
  end
end
```

### 3. User Account States

```ruby
class User < ApplicationRecord
  # States: pending, active, suspended, deactivated

  def current_state
    @current_state ||= "Users::#{account_state.to_s.camelize}State".constantize.new(self)
  end

  def activate
    current_state.activate
  end

  def suspend(reason:)
    current_state.suspend(reason: reason)
  end

  def deactivate
    current_state.deactivate
  end

  def can_login?
    current_state.can_login?
  end
end

module Users
  class PendingState < BaseState
    def activate
      user.update!(activated_at: Time.current)
      user.transition_to(:active)
      UserMailer.welcome(user).deliver_later
    end

    def can_login?
      false
    end
  end

  class ActiveState < BaseState
    def suspend(reason:)
      user.update!(suspended_at: Time.current, suspension_reason: reason)
      user.transition_to(:suspended)
      UserMailer.suspended(user).deliver_later
    end

    def deactivate
      user.transition_to(:deactivated)
    end

    def can_login?
      true
    end
  end

  class SuspendedState < BaseState
    def activate
      user.update!(suspended_at: nil, suspension_reason: nil)
      user.transition_to(:active)
      UserMailer.reactivated(user).deliver_later
    end

    def can_login?
      false
    end
  end

  class DeactivatedState < BaseState
    def activate
      user.transition_to(:active)
      UserMailer.reactivated(user).deliver_later
    end

    def can_login?
      false
    end
  end
end
```

### 4. Subscription States

```ruby
class Subscription < ApplicationRecord
  # States: trial, active, past_due, cancelled, expired

  def current_state
    @current_state ||= state_class.new(self)
  end

  def activate
    current_state.activate
  end

  def mark_past_due
    current_state.mark_past_due
  end

  def cancel
    current_state.cancel
  end

  def expire
    current_state.expire
  end

  def renew
    current_state.renew
  end

  def has_access?
    current_state.has_access?
  end
end

module Subscriptions
  class TrialState < BaseState
    def activate
      process_payment!
      subscription.transition_to(:active)
    end

    def cancel
      subscription.transition_to(:cancelled)
    end

    def expire
      subscription.transition_to(:expired)
    end

    def has_access?
      !trial_expired?
    end

    private

    def trial_expired?
      subscription.trial_ends_at < Time.current
    end

    def process_payment!
      PaymentProcessor.charge(subscription)
    end
  end

  class ActiveState < BaseState
    def mark_past_due
      subscription.transition_to(:past_due)
      SubscriptionMailer.payment_failed(subscription).deliver_later
    end

    def cancel
      subscription.update!(cancelled_at: Time.current)
      subscription.transition_to(:cancelled)
    end

    def has_access?
      true
    end
  end

  class PastDueState < BaseState
    def activate
      retry_payment!
      subscription.transition_to(:active)
    end

    def expire
      subscription.transition_to(:expired)
    end

    def has_access?
      grace_period_active?
    end

    private

    def retry_payment!
      PaymentProcessor.charge(subscription)
    end

    def grace_period_active?
      subscription.past_due_at + 7.days > Time.current
    end
  end

  class CancelledState < BaseState
    def renew
      process_payment!
      subscription.transition_to(:active)
    end

    def has_access?
      false
    end

    private

    def process_payment!
      PaymentProcessor.charge(subscription)
    end
  end

  class ExpiredState < BaseState
    def renew
      process_payment!
      subscription.transition_to(:active)
    end

    def has_access?
      false
    end

    private

    def process_payment!
      PaymentProcessor.charge(subscription)
    end
  end
end
```

## Implementation Guidelines

### 1. Use Enum for State Column

```ruby
class Order < ApplicationRecord
  enum state: {
    draft: 0,
    submitted: 1,
    paid: 2,
    shipped: 3,
    delivered: 4,
    cancelled: 5
  }
end
```

### 2. Create Base State Class

```ruby
module Orders
  class BaseState
    attr_reader :order

    def initialize(order)
      @order = order
    end

    # Define all state-specific methods
    def submit
      raise StateTransitionError, "Cannot submit from #{state_name}"
    end

    def pay
      raise StateTransitionError, "Cannot pay from #{state_name}"
    end

    # ... other actions

    private

    def state_name
      self.class.name.demodulize.gsub('State', '').underscore
    end
  end
end
```

### 3. Validate Transitions

```ruby
# ✅ Good: Validate before transition
class SubmittedState < BaseState
  def pay
    validate_payment!  # Validate first
    process_payment!
    order.transition_to(:paid)
  end

  private

  def validate_payment!
    raise ValidationError unless order.payment_method.present?
  end
end

# ❌ Bad: No validation
class BadSubmittedState < BaseState
  def pay
    order.transition_to(:paid)  # No validation!
  end
end
```

### 4. Keep States Independent

```ruby
# ✅ Good: States don't depend on each other
class DraftState < BaseState
  def submit
    order.transition_to(:submitted)
  end
end

class SubmittedState < BaseState
  def pay
    order.transition_to(:paid)
  end
end

# ❌ Bad: State depends on other state internals
class BadDraftState < BaseState
  def submit
    submitted_state = SubmittedState.new(order)
    submitted_state.setup  # Don't call other state methods!
    order.transition_to(:submitted)
  end
end
```

### 5. Centralize State Loading

```ruby
class Order < ApplicationRecord
  def current_state
    @current_state ||= state_class.new(self)
  end

  def transition_to(new_state)
    self.state = new_state
    @current_state = nil  # Clear cache
    save!
  end

  private

  def state_class
    "Orders::#{state.to_s.camelize}State".constantize
  rescue NameError
    raise "Unknown state: #{state}"
  end
end
```

## Testing State Pattern

```ruby
# Test state transitions
RSpec.describe Orders::DraftState do
  let(:order) { create(:order, state: :draft) }
  subject { described_class.new(order) }

  describe '#submit' do
    context 'with valid order' do
      before do
        order.items << create(:item)
        order.shipping_address = create(:address)
      end

      it 'transitions to submitted' do
        expect { subject.submit }.to change { order.reload.state }
          .from('draft').to('submitted')
      end

      it 'sends notification email' do
        expect { subject.submit }.to have_enqueued_mail(OrderMailer, :submitted)
      end
    end

    context 'with invalid order' do
      it 'raises validation error' do
        expect { subject.submit }.to raise_error(ValidationError)
      end

      it 'does not change state' do
        expect { subject.submit rescue nil }.not_to change { order.reload.state }
      end
    end
  end

  describe '#cancel' do
    it 'transitions to cancelled' do
      expect { subject.cancel }.to change { order.reload.state }
        .from('draft').to('cancelled')
    end
  end

  describe '#pay' do
    it 'raises error' do
      expect { subject.pay }.to raise_error(StateTransitionError, /Cannot pay from draft/)
    end
  end
end

# Test context
RSpec.describe Order do
  describe '#submit' do
    let(:order) { create(:order, state: initial_state) }

    context 'from draft state' do
      let(:initial_state) { :draft }

      it 'delegates to state object' do
        expect_any_instance_of(Orders::DraftState).to receive(:submit)
        order.submit
      end
    end

    context 'from submitted state' do
      let(:initial_state) { :submitted }

      it 'raises error' do
        expect { order.submit }.to raise_error(StateTransitionError)
      end
    end
  end
end

# Test state machine as a whole
RSpec.describe 'Order state machine' do
  let(:order) { create(:order, state: :draft) }

  it 'follows happy path' do
    # Draft -> Submitted
    expect { order.submit }.to change { order.state }.from('draft').to('submitted')

    # Submitted -> Paid
    expect { order.pay }.to change { order.state }.from('submitted').to('paid')

    # Paid -> Shipped
    expect { order.ship }.to change { order.state }.from('paid').to('shipped')

    # Shipped -> Delivered
    expect { order.deliver }.to change { order.state }.from('shipped').to('delivered')
  end

  it 'handles cancellation from draft' do
    expect { order.cancel }.to change { order.state }.from('draft').to('cancelled')
  end

  it 'handles cancellation from submitted' do
    order.submit
    expect { order.cancel }.to change { order.state }.from('submitted').to('cancelled')
  end

  it 'prevents invalid transitions' do
    expect { order.pay }.to raise_error(StateTransitionError)
    expect { order.ship }.to raise_error(StateTransitionError)
    expect { order.deliver }.to raise_error(StateTransitionError)
  end
end
```

## Anti-Patterns to Avoid

### ❌ Don't Use God States

```ruby
# ❌ Bad: State has too much logic
class BadActiveState < BaseState
  def process_action(action, params)
    case action
    when :submit then submit_logic(params)
    when :pay then pay_logic(params)
    when :ship then ship_logic(params)
    # ... 50 more actions
    end
  end
end

# ✅ Good: Each state has focused behavior
class GoodDraftState < BaseState
  def submit
    # Only submit logic
  end

  def cancel
    # Only cancel logic
  end
end
```

### ❌ Don't Put Business Logic in Context

```ruby
# ❌ Bad: Context has business logic
class BadOrder < ApplicationRecord
  def submit
    if state == 'draft'
      # Complex validation logic
      # Payment logic
      # Notification logic
      self.state = 'submitted'
    end
  end
end

# ✅ Good: Delegate to state
class GoodOrder < ApplicationRecord
  def submit
    current_state.submit
  end
end
```

### ❌ Don't Skip Validation

```ruby
# ❌ Bad: No validation before transition
class BadDraftState < BaseState
  def submit
    order.transition_to(:submitted)  # No checks!
  end
end

# ✅ Good: Validate before transition
class GoodDraftState < BaseState
  def submit
    validate_order!
    order.transition_to(:submitted)
  end

  private

  def validate_order!
    raise ValidationError, "Order must have items" if order.items.empty?
  end
end
```

## Decision Tree

### When to use State vs alternatives:

**Simple flag (on/off)?**
→ YES: Use boolean attribute
→ NO: Keep reading

**Need algorithm selection?**
→ YES: Use Strategy Pattern
→ NO: Keep reading

**Need fixed algorithm steps?**
→ YES: Use Template Method
→ NO: Keep reading

**Behavior changes based on internal state?**
→ YES: Use State Pattern ✅

**Multiple if/case checking state?**
→ YES: Use State Pattern ✅

**Well-defined state transitions?**
→ YES: Use State Pattern ✅

## Benefits

✅ **Eliminate conditionals** - No more if/case state checks
✅ **Single Responsibility** - Each state handles own logic
✅ **Open/Closed** - Add new states without modifying existing
✅ **Explicit transitions** - Clear state machine flow
✅ **Testability** - Test each state independently

## Drawbacks

❌ **More classes** - One class per state
❌ **Overkill** - Too complex for simple flags
❌ **State explosion** - Many states = many classes

## Real-World Rails Examples

### Blog Post States

```ruby
# States: draft, scheduled, published, archived
class Post < ApplicationRecord
  def current_state
    @current_state ||= "Posts::#{status.camelize}State".constantize.new(self)
  end

  def schedule(publish_at:)
    current_state.schedule(publish_at: publish_at)
  end

  def publish
    current_state.publish
  end

  def archive
    current_state.archive
  end

  def visible?
    current_state.visible?
  end
end
```

### Job Application States

```ruby
# States: applied, screening, interviewing, offer, hired, rejected
class JobApplication < ApplicationRecord
  def advance_to_screening
    current_state.advance_to_screening
  end

  def schedule_interview
    current_state.schedule_interview
  end

  def make_offer
    current_state.make_offer
  end

  def hire
    current_state.hire
  end

  def reject(reason:)
    current_state.reject(reason: reason)
  end
end
```

### Pull Request States

```ruby
# States: open, review_requested, changes_requested, approved, merged, closed
class PullRequest < ApplicationRecord
  def request_review
    current_state.request_review
  end

  def request_changes
    current_state.request_changes
  end

  def approve
    current_state.approve
  end

  def merge
    current_state.merge
  end

  def close
    current_state.close
  end
end
```

## Summary

**Use State Pattern when:**
- Object behavior changes based on internal state
- Multiple if/case statements checking state
- Well-defined state transitions (FSM)
- Each state has complex logic worth separating

**Avoid State Pattern when:**
- Simple boolean flag (use attribute)
- Algorithm selection (use Strategy)
- Fixed algorithm steps (use Template Method)
- Operation queuing (use Command)

**Most common Rails use cases:**
1. Order workflows (draft → submitted → paid → shipped → delivered)
2. Document approval (draft → review → approved → published)
3. User accounts (pending → active → suspended → deactivated)
4. Subscriptions (trial → active → past_due → cancelled)
5. Blog posts (draft → scheduled → published → archived)
6. Job applications (applied → screening → interviewing → hired)
7. Pull requests (open → review → approved → merged)
8. Support tickets (open → in_progress → resolved → closed)

**Key Pattern Structure:**
```ruby
# 1. Context with state delegation
class Context < ApplicationRecord
  enum state: [:state_a, :state_b]

  def current_state
    @current_state ||= state_class.new(self)
  end

  def action
    current_state.action
  end
end

# 2. Base state
class BaseState
  attr_reader :context

  def initialize(context)
    @context = context
  end

  def action
    raise StateTransitionError
  end
end

# 3. Concrete states
class StateA < BaseState
  def action
    context.transition_to(:state_b)
  end
end

class StateB < BaseState
  def action
    # Different behavior
  end
end
```
