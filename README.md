# Rails AI Suite (Agents, Commands, Skills)

A curated collection of specialized AI agents for Rails development, organized into three complementary components:

1. **Standard Rails Agents** - Modern Rails patterns with service objects, query objects, and presenters
2. **Feature Specification Agents** - High-level planning and feature management
3. **Skills Library** - Reusable knowledge modules for specific Rails patterns and technologies

> 🆕 **New:** Check out the [Claude Code Setup Plan](CLAUDE_CODE_SETUP_PLAN.md) for a complete guide on configuring Claude Code with Rails projects, including security hooks, custom commands, and MCP servers. Based on [The Complete Guide to Claude Code V4](https://thedecipherist.com/articles/claude-code-guide-v4/).

Built using insights from [GitHub's analysis of 2,500+ agent.md files](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/).

## Why This Exists

Most AI coding assistants lack deep Rails context. This suite provides a comprehensive architectural approach:

- 🏗️ **Standard Rails**: Service objects, query objects, presenters, form objects
- 📋 **Feature Planning**: Requirements analysis and implementation orchestration
- 📚 **Skills Library**: Deep knowledge modules for Rails patterns and technologies

---

## Standard Rails Agents

Modern Rails architecture with clear separation of concerns, SOLID principles, and comprehensive testing.

### Core Philosophy

- **Thin models & controllers** - Delegate to service/query/presenter objects
- **Service objects** - Encapsulate business logic with dry-monads Result pattern
- **No callback side effects** - Side effects (emails, notifications) in controllers, not model callbacks
- **Query objects** - Complex queries in dedicated classes (prevents N+1)
- **Presenters** - View logic separated from models
- **Form objects** - Multi-model forms
- **Pundit policies** - Authorization (deny by default)
- **ViewComponent** - Tested, reusable components
- **TDD workflow** - RED → GREEN → REFACTOR

### Available Agents

#### Testing & TDD
- **`@tdd_red_agent`** - Writes failing tests FIRST (RED phase)
- **`@rspec_agent`** - RSpec expert for all test types
- **`@tdd_refactoring_agent`** - Refactors while keeping tests green

#### Implementation
- **`@model_agent`** - Thin ActiveRecord models
- **`@controller_agent`** - Thin RESTful controllers
- **`@service_agent`** - Business logic service objects
- **`@command_agent`** - Command Pattern with undo/redo, command queues
- **`@query_agent`** - Complex query objects
- **`@form_agent`** - Multi-model form objects
- **`@presenter_agent`** - View logic presenters
- **`@policy_agent`** - Pundit authorization
- **`@view_component_agent`** - ViewComponent + Hotwire
- **`@job_agent`** - Background jobs (Solid Queue)
- **`@mailer_agent`** - Mailers with previews
- **`@migration_agent`** - Safe migrations
- **`@implementation_agent`** - General implementation

#### Frontend
- **`@stimulus_agent`** - Stimulus controllers
- **`@turbo_agent`** - Turbo Frames/Streams
- **`@tailwind_agent`** - Tailwind CSS styling

#### Quality
- **`@review_agent`** - Code quality analysis
- **`@lint_agent`** - Style fixes (no logic changes)
- **`@security_agent`** - Security audits (Brakeman)

### Standard Rails Workflow Example

```
1. @feature_planner_agent analyze the user authentication feature

2. @tdd_red_agent write failing tests for User model

3. @model_agent implement the User model

4. @tdd_red_agent write failing tests for AuthenticationService

5. @service_agent implement AuthenticationService

6. @controller_agent create SessionsController

7. @policy_agent create authorization policies

8. @review_agent check implementation

9. @tdd_refactoring_agent improve code structure

10. @lint_agent fix style issues
```

### Key Patterns

**Service Objects:**
```ruby
module Users
  class CreateService < ApplicationService
    def call
      # Returns Success(data) or Failure(error)
      # Uses dry-monads for Result handling
    end
  end
end
```

**Thin Controllers:**
```ruby
def create
  authorize User
  result = Users::CreateService.call(params: user_params)

  if result.success?
    redirect_to result.value!
  else
    render :new, status: :unprocessable_entity
  end
end
```

**Query Objects:**
```ruby
class Users::SearchQuery
  def call(params)
    @relation
      .then { |rel| filter_by_status(rel, params[:status]) }
      .then { |rel| search_by_name(rel, params[:q]) }
  end
end
```

---

## Feature Specification Agents

High-level planning and orchestration for complex features.

### Available Agents

- **`@feature_specification_agent`** - Writes detailed feature specifications
- **`@feature_planner_agent`** - Breaks features into tasks, recommends agents
- **`@feature_reviewer_agent`** - Reviews completed features against specs

### Feature Planning Workflow

```
1. @feature_specification_agent write spec for blog with comments

2. @feature_planner_agent create implementation plan

3. [Use recommended agents to implement]

4. @feature_reviewer_agent verify implementation matches spec
```

---

## Skills Library

Reusable knowledge modules that provide deep context on specific Rails patterns and technologies. Skills are referenced by agents and can be used directly to provide comprehensive guidance.

### What Are Skills?

Skills are focused knowledge documents that contain:
- **Patterns & best practices** - Proven approaches for specific domains
- **Code examples** - Real-world implementations
- **Reference materials** - Detailed documentation for complex topics
- **Decision guidance** - When and how to use specific patterns

### Available Skills

#### Architecture & Patterns
- **`rails-architecture`** - Code organization decisions, layered architecture
- **`rails-concern`** - Shared behavior with concerns
- **`rails-service-object`** - Business logic encapsulation
- **`command-pattern`** - Command Pattern with undo/redo, command queues, and history
- **`rails-query-object`** - Complex database queries
- **`rails-presenter`** - View logic separation
- **`rails-controller`** - RESTful controller patterns
- **`rails-model-generator`** - Model creation with validations
- **`form-object-patterns`** - Multi-model forms, wizards

#### Frontend & Hotwire
- **`hotwire-patterns`** - Turbo Frames, Turbo Streams, Stimulus integration
- **`viewcomponent-patterns`** - Reusable UI components with ViewComponent

#### Authentication & Authorization
- **`authentication-flow`** - Rails 8 built-in authentication
- **`authorization-pundit`** - Pundit policies and permissions

#### Data & Storage
- **`database-migrations`** - Safe, reversible migrations
- **`active-storage-setup`** - File uploads and attachments
- **`caching-strategies`** - Fragment, action, and HTTP caching

#### Background Jobs & Real-time
- **`solid-queue-setup`** - Background job processing
- **`action-cable-patterns`** - WebSocket real-time features
- **`action-mailer-patterns`** - Transactional emails

#### API & Internationalization
- **`api-versioning`** - Versioned REST APIs
- **`i18n-patterns`** - Internationalization and localization

#### Performance & Testing
- **`performance-optimization`** - N+1 prevention, eager loading, optimization
- **`tdd-cycle`** - Test-driven development workflow

### Using Skills

Skills provide context that agents can reference. Each skill includes:

```
skills/
├── skill-name/
│   ├── SKILL.md           # Main skill documentation
│   └── reference/         # Optional detailed references
│       ├── patterns.md
│       └── examples.md
```

### Example: Architecture Decision

The `rails-architecture` skill provides a decision tree for where code should live:

```
Where should this code go?
├─ Complex business logic?      → Service Object
├─ Complex database query?      → Query Object
├─ View/display formatting?     → Presenter
├─ Shared behavior across models? → Concern
├─ Authorization logic?         → Policy
├─ Reusable UI with logic?      → ViewComponent
└─ Async/background work?       → Job
```

---

## Agent Design Principles

All agents follow best practices from GitHub's analysis:

### ✅ What Makes These Agents Effective

- **YAML Frontmatter** - Each has `name` and `description`
- **Executable Commands** - Specific commands (e.g., `bundle exec rspec spec/models/user_spec.rb:25`)
- **Three-Tier Boundaries**:
  - ✅ **Always** - Must do
  - ⚠️ **Ask first** - Requires confirmation
  - 🚫 **Never** - Hard limits
- **Code Examples** - Real good/bad patterns
- **Concise** - Under 1,000 lines per agent

---

## Tech Stack Support

### Standard Agents Stack
- Ruby 3.3+
- Rails 7.x
- PostgreSQL
- Hotwire (Turbo + Stimulus)
- ViewComponent
- Tailwind CSS
- Solid Queue
- Pundit
- **dry-monads** (Result monad for service objects)
- RSpec + FactoryBot

---

## Claude Code Setup

For a complete guide on setting up Claude Code with Rails projects, see [CLAUDE_CODE_SETUP_PLAN.md](CLAUDE_CODE_SETUP_PLAN.md). This includes:

- **Global Configuration** - CLAUDE.md, settings.json, directory structure
- **Security Hooks** - Block secrets, dangerous commands, Rails-specific protections
- **Custom Commands** - `/ci`, `/lint`, `/security`, `/rails-test`, `/generate`
- **Skills** - Rails model, controller, and Stimulus patterns
- **Custom Agents** - Rails reviewer, test writer, migration expert
- **MCP Servers** - GitHub, PostgreSQL, sequential thinking integration

---

## Contributing

These agents are designed to be customized. Feel free to:

- Adjust boundaries based on your workflow
- Add project-specific commands
- Include custom validation rules
- Extend with your own coding standards

## Credits

Built using insights from:
- [How to write a great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/) by GitHub

## License

MIT License - Feel free to use and adapt these agents for your projects.
