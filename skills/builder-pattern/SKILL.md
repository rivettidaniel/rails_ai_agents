---
name: builder-pattern
description: Constructs complex objects step-by-step with Builder Pattern. Use when creating query builders, test data builders, configuration objects, or any object with many optional parameters requiring fluent API.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Builder Pattern in Rails

## Overview

The Builder Pattern constructs complex objects step-by-step, separating the construction logic from the representation. It provides a fluent interface for creating objects with many optional parameters.

**Key Insight**: Build complex objects incrementally using chainable methods instead of large constructor calls or telescoping constructors.

## Core Components

```
Client → Builder → Product
          ↓
      Director (optional)
```

1. **Builder** - Provides methods to construct product step-by-step
2. **Product** - Complex object being constructed
3. **Director** (optional) - Orchestrates build steps for common configurations
4. **Client** - Uses builder to construct product

## When to Use Builder Pattern

✅ **Use Builder Pattern when you need:**

- **Many optional parameters** - Object has 5+ optional attributes
- **Step-by-step construction** - Object requires multiple setup steps
- **Fluent/chainable API** - Want readable, chainable method calls
- **Different representations** - Same construction process creates different objects
- **Complex test data** - Need flexible test object creation

❌ **Don't use Builder Pattern for:**

- Simple objects with few parameters (use constructor)
- Objects with all required parameters (use factory)
- One-step object creation (no incremental building needed)

## Difference from Similar Patterns

| Aspect | Builder | Factory Method | Constructor | Abstract Factory |
|--------|---------|----------------|-------------|------------------|
| Purpose | Step-by-step construction | Single-step creation | Direct instantiation | Family creation |
| Complexity | High (many options) | Low-Medium | Low | Medium |
| Flexibility | Very high | Medium | Low | Medium |
| Fluent API | Yes | No | No | No |
| Use case | Complex objects | Simple objects | Basic objects | Related objects |

## Common Rails Use Cases

### 1. Query Builder

**Problem**: Complex ActiveRecord queries with many optional filters.

```ruby
# Bad: Messy conditional chaining
@users = User.all
@users = @users.where(status: params[:status]) if params[:status].present?
@users = @users.where(role: params[:role]) if params[:role].present?
@users = @users.where('created_at >= ?', params[:from_date]) if params[:from_date].present?
@users = @users.where('email ILIKE ?', "%#{params[:search]}%") if params[:search].present?
@users = @users.order(created_at: :desc)

# Good: Builder pattern
@users = UserSearchBuilder.new
  .with_status(params[:status])
  .with_role(params[:role])
  .created_after(params[:from_date])
  .with_email(params[:search])
  .sorted_by(:created_at, direction: :desc)
  .build
```

**Implementation:**

```ruby
class UserSearchBuilder
  def initialize(relation = User.all)
    @relation = relation
  end

  def with_status(status)
    return self if status.blank?
    @relation = @relation.where(status: status)
    self
  end

  def with_role(role)
    return self if role.blank?
    @relation = @relation.where(role: role)
    self
  end

  def with_email(email)
    return self if email.blank?
    @relation = @relation.where('email ILIKE ?', "%#{email}%")
    self
  end

  def created_after(date)
    return self if date.blank?
    @relation = @relation.where('created_at >= ?', date)
    self
  end

  def sorted_by(field, direction: :asc)
    @relation = @relation.order(field => direction)
    self
  end

  def build
    @relation
  end
end
```

### 2. Test Data Builder

**Problem**: Tests need complex object setup with many attributes.

```ruby
# Bad: FactoryBot with many traits/params
user = create(:user,
  email: 'test@example.com',
  first_name: 'John',
  last_name: 'Doe',
  role: :admin,
  status: :active,
  verified: true,
  subscription_plan: :premium,
  posts_count: 10
)

# Good: Builder pattern
user = UserBuilder.new
  .with_email('test@example.com')
  .with_name('John', 'Doe')
  .admin
  .active
  .verified
  .premium
  .with_posts(10)
  .create
```

**Implementation:**

```ruby
class UserBuilder
  def initialize
    @attributes = {
      email: "user#{SecureRandom.hex(4)}@example.com",
      password: 'password123',
      status: :pending,
      role: :user,
      verified: false
    }
    @associations = {}
  end

  def with_email(email)
    @attributes[:email] = email
    self
  end

  def with_name(first_name, last_name)
    @attributes[:first_name] = first_name
    @attributes[:last_name] = last_name
    self
  end

  def admin
    @attributes[:role] = :admin
    @attributes[:verified] = true
    @attributes[:status] = :active
    self
  end

  def premium
    @associations[:subscription] = create(:subscription, plan: :premium)
    self
  end

  def with_posts(count)
    @posts_count = count
    self
  end

  def build
    User.new(@attributes).tap do |user|
      @associations.each { |key, value| user.send("#{key}=", value) }
    end
  end

  def create
    user = build
    user.save!
    create_list(:post, @posts_count, user: user) if @posts_count
    user
  end
end
```

### 3. Configuration Builder

**Problem**: Building complex configuration objects.

```ruby
# Bad: Large hash or many method calls
config = {
  name: "Campaign 1",
  subject: "Welcome!",
  from: "noreply@example.com",
  template: :welcome,
  recipients: User.where(verified: true),
  schedule: :immediate,
  tracking: true,
  analytics: true
}

# Good: Builder pattern
campaign = EmailCampaignBuilder.new
  .named("Campaign 1")
  .with_subject("Welcome!")
  .from("noreply@example.com")
  .using_template(:welcome)
  .to(User.where(verified: true))
  .send_immediately
  .with_tracking
  .with_analytics
  .build
```

### 4. Report Builder

```ruby
report = ReportBuilder.new
  .for_date_range(1.month.ago, Date.today)
  .for_user(current_user)
  .include_columns(:date, :amount, :category)
  .group_by(:category)
  .as_pdf
  .build
```

### 5. API Request Builder

```ruby
response = ApiRequestBuilder.new(base_url: 'https://api.example.com')
  .with_authentication(current_user.api_token)
  .with_params(page: 1, per_page: 20)
  .with_header('Accept', 'application/json')
  .get('/users')
  .execute
```

## Implementation Guidelines

### 1. Make Methods Chainable

```ruby
# ✅ Good: Returns self for chaining
def with_email(email)
  @attributes[:email] = email
  self  # Return self!
end

# ❌ Bad: Doesn't return self
def with_email(email)
  @attributes[:email] = email
  # Breaks chaining!
end
```

### 2. Handle Nil/Blank Values Gracefully

```ruby
def with_status(status)
  return self if status.blank?  # Skip if empty
  @relation = @relation.where(status: status)
  self
end
```

### 3. Provide Convenience Methods

```ruby
def active
  with_status(:active)  # Shortcut for common case
end

def verified
  @attributes[:verified] = true
  self
end

def admin
  with_role(:admin).verified.active  # Composite shortcut
end
```

### 4. Validate at Build Time

```ruby
def build
  validate!
  EmailCampaign.create!(@config)
end

private

def validate!
  raise ArgumentError, "Name required" if @config[:name].blank?
  raise ArgumentError, "Subject required" if @config[:subject].blank?
end
```

### 5. Separate Build and Create

```ruby
def build
  # Returns unsaved object
  User.new(@attributes)
end

def create
  # Returns saved object
  user = build
  user.save!
  user
end
```

## Testing Builders

```ruby
RSpec.describe UserSearchBuilder do
  let!(:active_user) { create(:user, status: :active) }
  let!(:suspended_user) { create(:user, status: :suspended) }

  describe '#with_status' do
    it 'filters by status' do
      users = described_class.new
        .with_status(:active)
        .build

      expect(users).to include(active_user)
      expect(users).not_to include(suspended_user)
    end
  end

  describe 'chaining filters' do
    it 'combines multiple filters' do
      users = described_class.new
        .active
        .with_role(:admin)
        .sorted_by(:name)
        .build

      expect(users.to_sql).to include('status')
      expect(users.to_sql).to include('role')
      expect(users.to_sql).to include('ORDER BY')
    end
  end
end

RSpec.describe UserBuilder do
  describe '#build' do
    it 'creates user with default attributes' do
      user = described_class.new.build

      expect(user).to be_a(User)
      expect(user).to be_new_record
    end
  end

  describe '#create' do
    it 'persists the user' do
      user = described_class.new.create

      expect(user).to be_persisted
    end
  end

  describe 'chaining' do
    it 'allows method chaining' do
      user = described_class.new
        .with_email('test@example.com')
        .admin
        .premium
        .create

      expect(user.email).to eq('test@example.com')
      expect(user).to be_admin
    end
  end
end
```

## Director Pattern (Optional)

For standardized build sequences:

```ruby
class UserBuilderDirector
  def self.build_admin(email:)
    UserBuilder.new
      .with_email(email)
      .admin
      .verified
      .create
  end

  def self.build_premium_user(email:)
    UserBuilder.new
      .with_email(email)
      .premium
      .verified
      .create
  end

  def self.build_test_user
    UserBuilder.new
      .with_email("test#{SecureRandom.hex(4)}@example.com")
      .active
      .create
  end
end

# Usage
admin = UserBuilderDirector.build_admin(email: 'admin@test.com')
```

## Anti-Patterns to Avoid

### ❌ Don't Mutate After Build

```ruby
# ❌ Bad
builder = UserBuilder.new.with_email('test@example.com')
user1 = builder.build
builder.with_role(:admin)  # Modifies previous build!
user2 = builder.build  # Now has admin role!

# ✅ Good
builder1 = UserBuilder.new.with_email('test@example.com')
user1 = builder1.build

builder2 = UserBuilder.new.with_email('test@example.com').with_role(:admin)
user2 = builder2.build
```

### ❌ Don't Break Chaining

```ruby
# ❌ Bad: Breaks chain
def with_email(email)
  @email = email
  # Missing return self!
end

# ✅ Good: Chainable
def with_email(email)
  @email = email
  self
end
```

### ❌ Don't Skip Validation

```ruby
# ❌ Bad
def build
  EmailCampaign.create!(@config)  # Might create invalid object
end

# ✅ Good
def build
  validate!
  EmailCampaign.create!(@config)
end
```

## Decision Tree

### When to use Builder vs other approaches:

**Object has 5+ optional parameters?**
→ YES: Use Builder Pattern
→ NO: Keep reading

**Need fluent/chainable API?**
→ YES: Use Builder Pattern
→ NO: Keep reading

**Need step-by-step construction?**
→ YES: Use Builder Pattern
→ NO: Keep reading

**Only 1-3 parameters?**
→ YES: Use constructor or keyword arguments
→ NO: Keep reading

**All parameters required?**
→ YES: Use factory method
→ NO: Use Builder Pattern

## Benefits

✅ **Fluent API** - Readable, chainable method calls
✅ **Flexible** - Easy to add new configuration options
✅ **Testable** - Easy to create test data with specific attributes
✅ **Prevents telescoping constructors** - No need for multiple constructor overloads
✅ **Step-by-step construction** - Build complex objects incrementally

## Drawbacks

❌ **Increased complexity** - More classes to maintain
❌ **Overkill for simple objects** - Simple objects don't need builders
❌ **Boilerplate** - Requires writing many small methods

## Real-World Rails Examples

### Search Filter Builder

```ruby
# Complex search with many filters
search = ProductSearchBuilder.new
  .with_category(params[:category])
  .in_price_range(params[:min_price], params[:max_price])
  .with_brand(params[:brand])
  .in_stock
  .on_sale
  .sorted_by(:popularity)
  .paginated(page: params[:page])
  .build
```

### Form Builder

```ruby
# Multi-step form
form = RegistrationFormBuilder.new
  .with_email(params[:email])
  .with_password(params[:password])
  .with_profile(params[:profile])
  .with_preferences(params[:preferences])
  .build
```

### SQL Query Builder

```ruby
# Complex SQL
query = SqlQueryBuilder.new('users')
  .select('users.*, COUNT(posts.id) as posts_count')
  .join('posts', 'posts.user_id = users.id')
  .where('users.status', '=', 'active')
  .where('users.created_at', '>=', 1.month.ago)
  .group_by('users.id')
  .having('COUNT(posts.id) > 5')
  .order_by('posts_count', :desc)
  .limit(10)
  .to_sql
```

## Summary

**Use Builder Pattern when:**
- Object has 5+ optional parameters
- You want fluent/chainable API
- Construction requires multiple steps
- Need flexible test data creation

**Avoid Builder Pattern when:**
- Object is simple (< 5 parameters)
- All parameters are required
- No need for step-by-step construction

**Most common Rails use cases:**
1. Query builders (complex filters and scopes)
2. Test data builders (FactoryBot alternative)
3. Configuration builders (email campaigns, API requests)
4. Form builders (multi-step forms)
5. Report builders (customizable reports)
6. Search builders (advanced search filters)
7. SQL query builders (complex SQL generation)

**Key Pattern:**
```ruby
builder = ObjectBuilder.new
  .with_attribute1(value1)  # Returns self
  .with_attribute2(value2)  # Returns self
  .convenience_method       # Returns self
  .build                    # Returns final object
```
