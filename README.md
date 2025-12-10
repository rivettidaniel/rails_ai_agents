# Rails 8 AI Agent Suite

A collection of specialized AI agents for modern Rails development, for AI driven-development and follow TDD best practices.

Built using insights from [GitHub's analysis of 2,500+ agent.md files](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/).

## Why This Exists

Most AI coding assistants treat Rails like any other framework. These agents understand:

- 🏗️ **Rails Architecture**: Service Objects, Query Objects, Presenters, Form Objects
- 🔒 **Authorization**: Pundit policies with least privilege principle
- ⚡ **Rails 8 Features**: Solid Queue, `normalizes`, `generates_token_for`, `authenticate_by`
- 🎨 **Modern Stack**: Hotwire (Turbo + Stimulus), ViewComponent, Tailwind CSS
- ✅ **TDD Workflow**: RED → GREEN → REFACTOR

## The Agent Suite

### Planning & Orchestration

- **`@feature_planner_agent`** - Analyzes feature specs, breaks down into tasks, recommends which specialist agents to use

### Testing Agents

- **`@tdd_red_agent`** - Writes failing tests FIRST (RED phase of TDD)
- **`@rspec_agent`** - Expert in RSpec testing for models, services, controllers, etc.
- **`@tdd_refactoring_agent`** - Refactors code while keeping all tests green (REFACTOR phase)

### Implementation Specialists

- **`@model_agent`** - Creates thin ActiveRecord models with validations, associations, scopes
- **`@controller_agent`** - Creates thin RESTful controllers that delegate to services
- **`@service_agent`** - Creates business service objects with Result patterns
- **`@query_agent`** - Creates query objects for complex database queries (prevents N+1)
- **`@form_agent`** - Creates form objects for multi-model forms
- **`@presenter_agent`** - Creates presenters/decorators for view logic
- **`@policy_agent`** - Creates Pundit authorization policies (deny by default)
- **`@view_component_agent`** - Creates tested ViewComponents with Hotwire
- **`@job_agent`** - Creates background jobs with Solid Queue
- **`@mailer_agent`** - Creates mailers with HTML/text templates and previews
- **`@migration_agent`** - Creates safe, reversible, production-ready migrations

### Quality & Security

- **`@review_agent`** - Analyzes code quality, runs static analysis (read-only)
- **`@lint_agent`** - Fixes code style and formatting (no logic changes)
- **`@security_agent`** - Audits security with Brakeman and Bundler Audit

## TDD Workflow Example

Here's how to use the agents together for a new feature:

```
1. @feature_planner_agent analyze the user authentication feature

2. @tdd_red_agent write failing tests for User model

3. @model_agent implement the User model to pass the tests

4. @tdd_red_agent write failing tests for AuthenticationService

5. @service_agent implement AuthenticationService

6. @controller_agent create SessionsController with authorization

7. @review_agent check the implementation

8. @tdd_refactoring_agent improve the code structure

9. @lint_agent fix any style issues
```

## Agent Design Principles

Each agent follows best practices:

### ✅ What Makes These Agents Effective

- **YAML Frontmatter**: Each agent has `name` and `description`
- **Executable Commands**: Specific commands with flags (e.g., `bundle exec rspec spec/models/user_spec.rb:25`)
- **Three-Tier Boundaries**:
  - ✅ **Always**: Things the agent must do
  - ⚠️ **Ask first**: Actions requiring confirmation
  - 🚫 **Never**: Hard limits
- **Code Examples**: Real good/bad examples instead of abstract descriptions
- **Concise**: Each under 1,000 lines to stay within AI context limits

### 🎯 Core Features

- **Tech Stack Specified**: Ruby 3.3, Rails 8.1, PostgreSQL, Pundit, ViewComponent
- **Project Structure**: Clear file paths and architecture
- **Commands First**: Testing, linting, and verification commands
- **Rails Conventions**: Follows Rails way and modern best practices

## Tech Stack

These agents are optimized for the **Rails 8.x** stack:

- Ruby 3.3+
- Rails 8.x
- PostgreSQL
- Hotwire (Turbo + Stimulus)
- ViewComponent
- Tailwind CSS
- Solid Queue (database-backed jobs)
- Pundit (authorization)
- RSpec + FactoryBot (testing)

## Key Rails Patterns Implemented

### Service Objects

```ruby
module Users
  class CreateService < ApplicationService
    def call
      # Returns Result.new(success:, data:, error:)
    end
  end
end
```

### Thin Controllers

```ruby
def create
  authorize User

  result = Users::CreateService.call(params: user_params)

  if result.success?
    redirect_to result.data
  else
    # Handle error
  end
end
```

### Query Objects

```ruby
class Users::SearchQuery
  def call(params)
    @relation
      .then { |rel| filter_by_status(rel, params[:status]) }
      .then { |rel| search_by_name(rel, params[:q]) }
  end
end
```

### Authorization (Pundit)

```ruby
class UserPolicy < ApplicationPolicy
  def update?
    user.present? && (record.id == user.id || user.admin?)
  end
end
```

## Agent Boundaries

All agents follow strict boundaries to prevent mistakes:

| Agent Type | Never Does |
|------------|------------|
| **TDD Red** | Never modifies source code, only writes tests |
| **Model** | Never adds business logic (delegates to services) |
| **Controller** | Never implements business logic (delegates to services) |
| **Service** | Never skips error handling or Result objects |
| **Refactoring** | Never changes behavior, never refactors with failing tests |
| **Migration** | Never modifies migrations that have already run |
| **Security** | Never modifies credentials, secrets, or production configs |
| **Lint** | Never changes business logic, only formatting |

## Examples

### Create a New Feature

```
@feature_planner_agent I need to add a blog post feature with comments
```

The planner will:
1. Analyze requirements
2. Break down into models, services, controllers
3. Recommend the sequence of agents to use
4. Suggest the TDD approach

### Write Failing Tests First

```
@tdd_red_agent write tests for a Post model with title, body, published_at, and belongs_to :author
```

The agent will:
1. Create `spec/models/post_spec.rb` with failing tests
2. Create `spec/factories/posts.rb`
3. Verify tests fail for the right reason
4. Document what code needs to be implemented

### Implement the Model

```
@model_agent implement the Post model to pass the tests
```

The agent will:
1. Create `app/models/post.rb`
2. Add validations, associations, scopes
3. Keep the model thin (no business logic)
4. Run tests to verify they pass

### Review Code Quality

```
@review_agent check the Post implementation
```

The agent will:
1. Run Brakeman for security issues
2. Run RuboCop for style violations
3. Check for SOLID principle violations
4. Provide actionable feedback (no modifications)

## Contributing

These agents are designed to be customized for your team's needs. Feel free to:

- Adjust boundaries based on your workflow
- Add project-specific commands
- Include custom validation rules
- Extend with your own coding standards

## Credits

Built following the insights from:
- [How to write a great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/) by GitHub

## License

MIT License - Feel free to use and adapt these agents for your projects.
