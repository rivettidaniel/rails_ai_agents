---
name: template-method-pattern
description: Defines algorithm skeleton with customizable steps using Template Method Pattern. Use for data importers, exporters, multi-step processors, or any process with fixed flow but variant steps.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

# Template Method Pattern in Rails

## Overview

The Template Method Pattern defines the skeleton of an algorithm in a base class, but lets subclasses override specific steps without changing the algorithm's structure.

**Key Insight**: Extract common algorithm structure to superclass, delegate variant steps to subclasses through inheritance.

## Core Components

```
Client → Abstract Class (template method + hooks)
              ↓
         Concrete Classes (implement abstract steps)
```

1. **Abstract Class** - Defines template method and abstract/concrete steps
2. **Template Method** - Defines algorithm skeleton (sequence of steps)
3. **Abstract Steps** - Must be implemented by subclasses
4. **Concrete Steps** - Shared implementation, can be overridden
5. **Hooks** - Optional extension points (empty methods)
6. **Concrete Classes** - Implement abstract steps, optionally override others

## When to Use Template Method

✅ **Use Template Method when you need:**

- **Eliminate duplicate code** - Multiple classes share algorithm structure
- **Control algorithm flow** - Superclass controls sequence, subclasses customize steps
- **Extension points** - Provide hooks for optional customization
- **Fixed algorithm** - Sequence doesn't change, only specific steps vary

❌ **Don't use Template Method for:**

- Runtime algorithm selection (use Strategy)
- Completely independent steps (use Strategy)
- Simple algorithms (no pattern needed)
- Operation queuing/undo (use Command)

## Difference from Similar Patterns

| Aspect | Template Method | Strategy | Factory Method | Command |
|--------|-----------------|----------|----------------|---------|
| Purpose | Algorithm skeleton | Interchangeable algorithms | Polymorphic creation | Encapsulate request |
| Mechanism | Inheritance | Composition | Inheritance | Composition |
| Flexibility | Compile-time | Runtime | Compile-time | Runtime |
| Use case | Multi-step process | Algorithm variants | Object creation | Operations |

## Common Rails Use Cases

### 1. Data Importers

Fixed import flow with format-specific parsing:

```ruby
# Base class with template method
class BaseImporter
  def import
    validate_file!
    open_file
    parse_data      # Abstract: CSV vs JSON vs XML
    validate_data
    transform_data  # Abstract: format-specific mapping
    save_data
    cleanup
  end

  private

  def parse_data
    raise NotImplementedError
  end

  def transform_data
    raise NotImplementedError
  end

  def validate_file!
    # Shared implementation
  end

  def save_data
    # Shared implementation
  end
end

# Concrete implementations
class CsvImporter < BaseImporter
  def parse_data
    @data = CSV.parse(@content, headers: true)
  end

  def transform_data
    @data = @data.map { |row| normalize_row(row) }
  end
end

class JsonImporter < BaseImporter
  def parse_data
    @data = JSON.parse(@content)
  end

  def transform_data
    @data = @data.map { |row| map_json_keys(row) }
  end
end
```

### 2. Data Exporters

Fixed export flow with format-specific generation:

```ruby
class BaseExporter
  def export
    validate_data!
    prepare_data
    format_data    # Abstract: CSV vs PDF vs Excel
    generate_output # Abstract: format-specific rendering
    add_metadata
    finalize_output
  end

  private

  def format_data
    raise NotImplementedError
  end

  def generate_output
    raise NotImplementedError
  end

  def prepare_data
    # Shared implementation
  end
end

class PdfExporter < BaseExporter
  def format_data
    @formatted = @data.map { |row| escape_for_pdf(row) }
  end

  def generate_output
    Prawn::Document.new { |pdf| render_pdf(pdf) }.render
  end
end

class CsvExporter < BaseExporter
  def format_data
    @formatted = @data
  end

  def generate_output
    CSV.generate { |csv| @formatted.each { |row| csv << row } }
  end
end
```

### 3. Order Processors

Fixed processing flow with customer-type variations:

```ruby
class BaseOrderProcessor
  def process
    validate_order!
    calculate_totals
    apply_discounts    # Abstract: regular vs premium vs wholesale
    calculate_tax      # Abstract: varies by customer type
    process_payment
    update_inventory
    send_notifications
  end

  private

  def apply_discounts
    raise NotImplementedError
  end

  def calculate_tax
    raise NotImplementedError
  end

  def validate_order!
    # Shared
  end

  def process_payment
    # Shared
  end
end

class PremiumOrderProcessor < BaseOrderProcessor
  def apply_discounts
    @order.discount = @order.subtotal * 0.15  # 15% off
  end

  def calculate_tax
    @order.tax = @order.subtotal * 0.05  # Reduced tax
  end
end

class RegularOrderProcessor < BaseOrderProcessor
  def apply_discounts
    @order.discount = 0  # No discount
  end

  def calculate_tax
    @order.tax = @order.subtotal * 0.10  # Standard tax
  end
end
```

### 4. Report Generators

```ruby
class BaseReportGenerator
  def generate
    validate_params!
    fetch_data
    filter_data       # Abstract: report-specific filters
    aggregate_data    # Abstract: report-specific calculations
    format_output     # Abstract: presentation format
    apply_styling
  end
end

class SalesReportGenerator < BaseReportGenerator
  def filter_data
    @data = @data.where(type: 'sale')
  end

  def aggregate_data
    @aggregated = @data.group_by(&:category).transform_values(&:sum)
  end

  def format_output
    # Sales-specific formatting
  end
end
```

## Implementation Guidelines

### 1. Define Clear Template Method

```ruby
class BaseImporter
  # Template method - documents the algorithm flow
  def import
    # Step 1: Validation
    validate_file!

    # Step 2: Read
    open_file

    # Step 3: Parse (abstract - varies by format)
    parse_data

    # Step 4: Validate
    validate_data

    # Step 5: Transform (abstract - format-specific)
    transform_data

    # Step 6: Save
    save_data

    # Step 7: Cleanup
    cleanup
  end
end
```

### 2. Mark Abstract vs Concrete Steps

```ruby
class BaseImporter
  private

  # Abstract step - MUST be implemented
  def parse_data
    raise NotImplementedError, "#{self.class} must implement #parse_data"
  end

  # Concrete step - CAN be overridden
  def validate_data
    # Default implementation
    @data.each { |row| validate_row(row) }
  end

  # Hook - MAY be overridden for customization
  def before_save
    # Empty by default
  end
end
```

### 3. Provide Hooks for Extension

```ruby
class BaseImporter
  def import
    validate_file!
    before_parse    # Hook
    parse_data
    after_parse     # Hook
    validate_data
    before_save     # Hook
    save_data
    after_save      # Hook
  end

  private

  def before_parse; end  # Hook: override for custom logic
  def after_parse; end   # Hook: override for custom logic
  def before_save; end   # Hook: override for custom logic
  def after_save; end    # Hook: override for custom logic
end

# Subclass uses hooks
class CsvImporter < BaseImporter
  def before_parse
    Rails.logger.info("Starting CSV parse")
  end

  def after_save
    CacheService.invalidate('users')
  end
end
```

### 4. Keep Template Method Simple

```ruby
# ✅ Good: Clear, sequential steps
def import
  validate_file!
  parse_data
  transform_data
  save_data
end

# ❌ Bad: Too complex, too many conditionals
def import
  if valid_file?
    if format == :csv
      parse_csv
    elsif format == :json
      parse_json
    end
    if premium_user?
      apply_premium_transform
    end
  end
end
```

## Testing Template Method

```ruby
# Test template method flow
RSpec.describe BaseImporter do
  # Use concrete class for testing
  let(:concrete_importer) do
    Class.new(described_class) do
      def parse_data
        @data = [{ name: 'Test' }]
      end

      def transform_data
        @data = @data.map { |d| d.merge(email: 'test@example.com') }
      end
    end
  end

  subject { concrete_importer.new(file: file) }

  describe '#import' do
    it 'calls methods in correct order' do
      expect(subject).to receive(:validate_file!).ordered
      expect(subject).to receive(:open_file).ordered
      expect(subject).to receive(:parse_data).ordered
      expect(subject).to receive(:validate_data).ordered
      expect(subject).to receive(:transform_data).ordered
      expect(subject).to receive(:save_data).ordered

      subject.import
    end

    it 'handles errors' do
      allow(subject).to receive(:parse_data).and_raise("Error")
      expect { subject.import }.to raise_error("Error")
    end
  end
end

# Test concrete implementation
RSpec.describe CsvImporter do
  describe '#parse_data' do
    it 'parses CSV into array of hashes' do
      subject.send(:open_file)
      subject.send(:parse_data)

      data = subject.instance_variable_get(:@data)
      expect(data).to be_an(Array)
      expect(data.first).to be_a(Hash)
    end
  end

  describe '#transform_data' do
    it 'normalizes data' do
      subject.instance_variable_set(:@data, [{ 'Email' => 'TEST@EXAMPLE.COM' }])
      subject.send(:transform_data)

      data = subject.instance_variable_get(:@data)
      expect(data.first[:email]).to eq('test@example.com')
    end
  end
end
```

## Anti-Patterns to Avoid

### ❌ Don't Let Subclasses Override Template Method

```ruby
# ❌ Bad: Allows overriding template
class BadBaseImporter
  def import  # Can be overridden
    parse_data
    save_data
  end
end

class BadCsvImporter < BadBaseImporter
  def import  # Changes entire flow!
    validate_first
    parse_data
    transform
    save_data
  end
end

# ✅ Good: Template can't be changed
class GoodBaseImporter
  def import  # Final (by convention in Ruby)
    parse_data
    save_data
  end
  # Document that this shouldn't be overridden
end
```

### ❌ Don't Make Steps Too Granular

```ruby
# ❌ Bad: Too many tiny steps
class BadImporter
  def import
    open_file
    read_line_1
    read_line_2
    parse_line_1
    parse_line_2
    # ... 50 more steps
  end
end

# ✅ Good: Balanced granularity
class GoodImporter
  def import
    validate_file!
    parse_data      # Internally reads all lines
    validate_data
    save_data
  end
end
```

### ❌ Don't Put Complex Logic in Hooks

```ruby
# ❌ Bad: Hook has business logic
class BadImporter < BaseImporter
  def before_save
    # Complex logic in hook!
    calculate_discounts
    apply_taxes
    send_notifications
    update_analytics
  end
end

# ✅ Good: Hook is simple
class GoodImporter < BaseImporter
  def before_save
    Rails.logger.info("Saving #{@data.count} records")
  end

  # Business logic in proper step
  def transform_data
    super
    apply_business_rules
  end
end
```

## Decision Tree

### When to use Template Method vs alternatives:

**Need runtime algorithm selection?**
→ YES: Use Strategy Pattern
→ NO: Keep reading

**Steps are completely independent?**
→ YES: Use Strategy Pattern (composition)
→ NO: Keep reading

**Need operation queuing/undo?**
→ YES: Use Command Pattern
→ NO: Keep reading

**Multiple classes share algorithm structure?**
→ YES: Use Template Method Pattern ✅

**Algorithm is simple (< 3 steps)?**
→ YES: No pattern needed
→ NO: Use Template Method Pattern ✅

## Benefits

✅ **Code reuse** - Extract common code to superclass
✅ **Control** - Superclass controls algorithm flow
✅ **Flexibility** - Subclasses customize specific steps
✅ **Maintainability** - Algorithm changes affect all subclasses
✅ **Extension points** - Hooks for optional customization

## Drawbacks

❌ **Inheritance coupling** - Subclasses tied to superclass
❌ **Rigid structure** - Can't change algorithm sequence
❌ **Liskov violation risk** - Subclasses might break expectations
❌ **Complexity** - More steps = harder to maintain

## Real-World Rails Examples

### Email Generators

```ruby
class BaseEmailGenerator
  def generate
    load_template
    inject_data
    apply_styling
    render_html
  end
end

class WelcomeEmailGenerator < BaseEmailGenerator
  def inject_data
    @template.merge!(user_name: @user.name, activation_link: @link)
  end
end
```

### Data Migration Scripts

```ruby
class BaseMigrationScript
  def migrate
    connect_to_source
    extract_data
    transform_data
    connect_to_target
    load_data
    verify_migration
  end
end

class LegacyDatabaseMigration < BaseMigrationScript
  def transform_data
    # Legacy-specific transformations
  end
end
```

### Multi-Step Wizards

```ruby
class BaseWizard
  def complete
    validate_step_1
    process_step_1
    validate_step_2
    process_step_2
    finalize
  end
end

class RegistrationWizard < BaseWizard
  def process_step_1
    # Create user account
  end

  def process_step_2
    # Set up user profile
  end
end
```

## Summary

**Use Template Method when:**
- Multiple classes share algorithm structure
- Want to eliminate duplicate code
- Need to control algorithm flow
- Sequence is fixed, steps vary

**Avoid Template Method when:**
- Need runtime algorithm selection (use Strategy)
- Steps are independent (use Strategy)
- Simple algorithm (no pattern needed)
- Need undo/queue operations (use Command)

**Most common Rails use cases:**
1. Data importers (CSV, JSON, XML parsers)
2. Data exporters (PDF, Excel, CSV generators)
3. Order processors (regular, premium, wholesale)
4. Report generators (sales, analytics, custom)
5. Email generators (welcome, notification, digest)
6. Multi-step wizards (registration, checkout, onboarding)
7. Data migration scripts (different sources)
8. Background job processors (different job types)

**Key Pattern Structure:**
```ruby
# 1. Base class with template method
class BaseClass
  def template_method
    step_1            # Concrete
    step_2            # Abstract
    step_3            # Concrete
  end

  private

  def step_1; end     # Shared implementation
  def step_2; raise NotImplementedError; end  # Must override
  def step_3; end     # Can override
end

# 2. Concrete class implements abstract steps
class ConcreteClass < BaseClass
  private

  def step_2
    # Specific implementation
  end
end
```
