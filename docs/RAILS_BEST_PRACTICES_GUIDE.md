# Rails 8+ Best Practices Guide

> A deep-dive companion to the CLAUDE.md template for Rails 8+ projects. This guide explains the **why** behind every rule, provides alternatives and gotchas, and is meant for human onboarding. The CLAUDE.md template states the rules concisely; this guide gives the rationale.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Ruby Style](#2-ruby-style)
3. [Controllers and REST](#3-controllers-and-rest)
4. [Models and Concerns](#4-models-and-concerns)
5. [ActiveRecord Queries](#5-activerecord-queries)
6. [Callbacks and Jobs](#6-callbacks-and-jobs)
7. [Hotwire: Turbo](#7-hotwire-turbo)
8. [Hotwire: Stimulus](#8-hotwire-stimulus)
9. [Views and ERB](#9-views-and-erb)
10. [Error Handling](#10-error-handling)
11. [Performance and Caching](#11-performance-and-caching)
12. [Testing](#12-testing)
13. [Security and Authentication](#13-security-and-authentication)
14. [Migrations](#14-migrations)
15. [Tooling and DX](#15-tooling-and-dx)
16. [Rails 8+ Features and Deployment](#16-rails-8-features-and-deployment)
17. [Gotchas and Common Mistakes](#17-gotchas-and-common-mistakes)
18. [Recommended Stack](#18-recommended-stack)
19. [Quick Reference Cheat Sheet](#19-quick-reference-cheat-sheet)

---

## 1. Project Structure

### The Philosophy

Rails encodes architecture into directory structure. When you see `app/models/`, you know business logic lives there. When you see `app/controllers/`, you know those files are thin dispatchers. The structure *is* the architecture. The goal is predictability: when a bug report comes in, the file path should be obvious from the symptom.

### Canonical Layout

```
app/
  controllers/
    concerns/                  # Shared controller mixins (authentication, pagination)
    application_controller.rb
    posts_controller.rb
    posts/
      closures_controller.rb   # Nested resource controller
  models/
    concerns/                  # Cross-cutting model mixins (searchable, sluggable)
    application_record.rb
    current.rb                 # ActiveSupport::CurrentAttributes
    post.rb
    user.rb
    user/                      # Model-specific concerns
      authentication.rb
      avatarable.rb
  views/
    layouts/
    posts/
    shared/                    # Cross-resource partials
  jobs/
    application_job.rb
  mailers/
    application_mailer.rb
  javascript/
    controllers/               # Stimulus controllers
  helpers/                     # Keep minimal; prefer tag helpers in views
config/
  initializers/                # One concern per file, kept small
  deploy.yml                   # Kamal deployment
  queue.yml                    # Solid Queue workers
  recurring.yml                # Solid Queue scheduled tasks
  routes.rb
db/
  migrate/
lib/
  tasks/                       # Custom rake tasks
test/
  controllers/
  fixtures/
  models/
  system/                      # Capybara browser tests
bin/
  setup   dev   ci
```

### Where Concerns Live

**Cross-cutting concerns** go in `app/models/concerns/` or `app/controllers/concerns/`. A `Searchable` concern that any model can include belongs in `app/models/concerns/searchable.rb`.

**Model-specific concerns** go in `app/models/<model>/`. A `User::Authentication` concern belongs in `app/models/user/authentication.rb`. The module name mirrors the path. This distinction communicates scope: `include Searchable` is general-purpose; `include User::Authentication` is model-specific.

### When to Use `lib/`

`lib/` is for code that could be extracted into a gem: rake tasks, railties, pure libraries with no Rails model dependencies. If your code references `ApplicationRecord` or `Current`, it belongs in `app/`.

### Initializer Minimalism

Each initializer should do one thing. If `config/initializers/content_security_policy.rb` is 15 lines that configure CSP, that is perfect. A 400-line initializer configuring three services should be three files. Initializers run at boot -- they should configure, not compute.

### Why No `app/services/` by Default

Business logic lives on models. A model is not just a database wrapper -- it is a domain object. When a `Post` knows how to `publish` itself, that is not "fat model" -- it is a rich domain model.

Service objects add an indirection layer between controllers and models. The controller calls a service, which calls a model method. Why not call the model method directly? In most Rails apps, services do not pay for themselves.

**When services are justified:** The operation genuinely spans multiple aggregate roots (e.g., a `Signup` that creates a User, Session, and sends email) and does not fit any single model. When you add one, treat it as a regular Ruby class -- no base class, no gem.

```ruby
class Signup
  def initialize(email_address:, password:)
    @email_address = email_address
    @password = password
  end

  def create_identity
    User.transaction do
      user = User.create!(email_address: @email_address, password: @password)
      user.sessions.create!
      WelcomeMailer.with(user: user).deliver_later
    end
  end
end
```

---

## 2. Ruby Style

### Expanded Conditionals Over Guard Clauses

DHH prefers expanded `if/else` over stacked guard clauses. Guard clauses force readers to mentally track "what conditions have been eliminated so far." Expanded conditionals read top-to-bottom.

```ruby
# Avoid: stacked guards
def publish(post)
  return unless post
  return if post.draft?
  return if post.published_at.present?
  post.update!(published_at: Time.current)
end

# Prefer: expanded conditional
def publish(post)
  if post && !post.draft? && post.published_at.nil?
    post.update!(published_at: Time.current)
  end
end

# Also fine: single guard when the body is non-trivial
def publish(post)
  return if post.nil? || post.draft? || post.published_at.present?

  post.update!(published_at: Time.current)
  NotifySubscribersJob.perform_later(post)
  refresh_sitemap
end
```

**Assignment in conditions** is encouraged when it makes intent clearer:

```ruby
# Clear: "if we find a user, do something with it"
if user = User.find_by(email: params[:email])
  start_session_for(user)
else
  redirect_to new_registration_path
end
```

This reads as "if we can find a user and assign it, do this with it." Rubocop's omakase config allows it.

### Method Ordering

1. Class methods (`self.xxx`) first
2. Public instance methods, `initialize` at top
3. Private methods, ordered by call sequence -- readers follow execution top-to-bottom

```ruby
class Invoice
  def finalize
    charge_card
    send_receipt
  end

  private
    def charge_card
      PaymentGateway.charge(amount: total_cents, source: payment_token)
    end

    def send_receipt
      ReceiptMailer.with(invoice: self).deliver_later
    end
end
```

When a private method calls other private methods, nest sub-methods immediately after their caller.

### Visibility Modifier Indentation

No blank line between `private` and the first method. Indent methods two spaces under `private`:

```ruby
class User
  def full_name
    "#{first_name} #{last_name}"
  end

  private
    def normalize_email
      self.email = email.strip.downcase
    end
end
```

**Exception:** If a module contains *only* private methods, place `private` at top with a blank line, no indent:

```ruby
module User::PasswordNormalization
  private

  def normalize_password_fields
    # ...
  end
end
```

### Bang Methods

`!` means "this has a non-bang counterpart" (`save`/`save!`, `create`/`create!`). Do not use `!` to signal general destructiveness -- Ruby has many destructive methods without bangs (`push`, `delete`, `clear`).

### Other Conventions

- **Double quotes** by default. Single quotes only for literal backslashes or contractions.
- **`frozen_string_literal: true`** at the top of every Ruby file. Prevents accidental string mutation; enables memory optimizations.
- **Enumerables:** `.each` for side effects (not `.map`). `.find_each` for AR batches.
- **`Data.define`** (Ruby 3.2+) for immutable value objects; `Struct.new` for mutable ones. Promote to a full class when you need validation or behavior.
- **Plain Ruby over gems** when the problem fits in 20 lines. Every gem is a dependency to maintain.

---

## 3. Controllers and REST

### REST Purity

When every endpoint maps to CRUD on a resource, the team shares a mental model of the entire app. Custom actions (`post :close`, `post :approve`) break that model. Instead, model the state change as its own resource.

```ruby
# Bad: custom actions
resources :cards do
  member do
    post :close
    post :reopen
    post :archive
  end
end

# Good: each state change is its own resource
resources :cards do
  resource :closure,  only: [:create, :destroy]  # close = create, reopen = destroy
  resource :archival, only: [:create]
end
```

`POST /cards/42/closure` is self-documenting. The controller stays thin. You can add per-resource authorization without entangling unrelated actions.

### Thin Controllers

A controller action sets up data, calls a model method, and renders. Under 10 lines.

```ruby
class SubscriptionsController < ApplicationController
  def create
    @subscription = Current.user.subscriptions.start(subscription_params)

    if @subscription.persisted?
      redirect_to @subscription, notice: "Subscribed"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private
    def subscription_params
      params.expect(subscription: [:plan_id])
    end
end
```

### `params.expect` vs `params.require.permit`

Rails 8's `params.expect` is stricter than `params.require(...).permit(...)`. It raises `ActionController::ParameterMissing` on shape mismatches and returns stricter types. Prefer it in new code.

```ruby
params.expect(post: [:title, :body, :published, tag_ids: []])
```

### `before_action` and Concerns

Use `before_action` for record loading, authentication, and permission checks:

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :ensure_author, only: [:edit, :update, :destroy]

  # ...

  private
    def set_post
      @post = Current.user.posts.find(params[:id])
    end

    def ensure_author
      head :forbidden unless @post.author == Current.user
    end
end
```

Extract shared patterns into controller concerns. The authentication concern is the most important:

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?, :current_user
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end
  end

  private
    def authenticated?  = Current.session.present?
    def current_user    = Current.session&.user

    def require_authentication
      return if authenticated?
      redirect_to new_session_path
    end

    def start_new_session_for(user)
      user.sessions.create!(
        user_agent: request.user_agent,
        ip_address: request.remote_ip
      ).tap { |session| set_current_session(session) }
    end

    def set_current_session(session)
      Current.session = session
      cookies.signed.permanent[:session_token] = {
        value: session.signed_id, httponly: true, same_site: :lax
      }
    end

    def terminate_session
      Current.session.destroy
      cookies.delete(:session_token)
    end
end
```

### `head :status` and `:unprocessable_entity`

Use `head :forbidden`, `head :no_content` for bodyless responses. Return `status: :unprocessable_entity` (422) on failed form submissions -- Turbo Drive relies on 422 to re-render the form in place. Without it, Turbo treats the response as a success and navigates away, losing validation errors.

---

## 4. Models and Concerns

### Rich Models

Models are domain objects, not data transfer objects. A `Post` that knows how to `publish` itself is not "fat" -- it is taking responsibility for its domain. The alternative (service objects) distributes logic across many files.

### Concern Extraction

Extract concerns at ~100 lines. Include concerns alphabetically. Each concern handles one responsibility.

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include Authentication, Avatarable, Searchable, Tokenizable

  has_many :posts, dependent: :destroy
  normalizes :email, with: -> { _1.strip.downcase }
  validates :email, presence: true, uniqueness: true
end
```

A concern's internal structure: `included` block (associations, scopes, callbacks), then query/predicate methods, then action methods, then private helpers.

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed? = closure.present?
  def open?   = !closed?

  def close(user: Current.user)
    create_closure!(user: user) unless closed?
  end

  def reopen
    closure&.destroy if closed?
  end
end
```

### Modern ActiveRecord Features

**`normalizes`** -- Normalizes on assignment *and* in queries. Replaces `before_save` callbacks:

```ruby
class User < ApplicationRecord
  normalizes :email, with: -> { _1.strip.downcase }
  normalizes :phone, with: -> { _1.gsub(/\D/, "") }
end

user = User.new(email: "  ALICE@Example.COM  ")
user.email  # => "alice@example.com"

# Also normalizes in queries:
User.find_by(email: "  ALICE@Example.COM  ")
# => Finds the user, because the query value is normalized too
```

**`encrypts`** -- Column-level encryption (AES-256-GCM). Use `deterministic: true` when you need `find_by`. Run `bin/rails db:encryption:init` to generate keys:

```ruby
encrypts :two_factor_secret              # Non-deterministic: more secure, cannot query
encrypts :ssn, deterministic: true       # Deterministic: allows find_by
```

**`generates_token_for`** -- Signed, scoped, time-limited tokens. The block is a fingerprint -- if it changes (e.g., user changes password), the token is invalidated:

```ruby
generates_token_for :password_reset, expires_in: 15.minutes do
  password_salt.last(10)
end

generates_token_for :email_confirmation, expires_in: 24.hours do
  email
end

# Generate: token = user.generate_token_for(:password_reset)
# Consume:  user  = User.find_by_token_for(:password_reset, token)
# Returns nil if expired or fingerprint changed
```

**`has_secure_password`** -- Bcrypt-based password hashing. No gem beyond the bundled `bcrypt`. Rails 8 adds `User.authenticate_by(email:, password:)` for login:

```ruby
class User < ApplicationRecord
  has_secure_password
end

user = User.create!(email: "alice@example.com", password: "secret123")
user.authenticate("secret123")  # => user
user.authenticate("wrong")       # => false

# Rails 8 login:
User.authenticate_by(email: "alice@example.com", password: "secret123")
```

**Enums with prefix** -- Always prefix to avoid method name collisions:

```ruby
# Bad: Post.draft, post.draft? collide with any method called "draft"
enum status: { draft: 0, published: 1, archived: 2 }

# Good: Post.status_draft, post.status_draft? -- no collision risk
enum :status, { draft: 0, published: 1, archived: 2 }, prefix: true
```

**Delegate** -- Expose association attributes as part of the model's interface:

```ruby
delegate :name, :email, to: :author, prefix: true
# => post.author_name, post.author_email

delegate :identity, to: :session, allow_nil: true
# => Returns nil instead of NoMethodError when session is nil
```

**Lambda defaults** -- Set from request context (evaluated at assignment time, not boot):

```ruby
belongs_to :author, class_name: "User", default: -> { Current.user }
belongs_to :account, default: -> { author.account }
```

**Validations** -- Validate what the database cannot enforce. Always pair `validates :x, uniqueness: true` with a unique index (the validation provides a friendly error; the index prevents race conditions):

```ruby
validates :title, presence: true, length: { maximum: 255 }
validates :slug, uniqueness: { scope: :account_id }
validates :published_at, comparison: { greater_than: :created_at }, allow_nil: true
```

---

## 5. ActiveRecord Queries

### `inverse_of`

Without `inverse_of`, Rails creates separate in-memory objects for the same database row when traversing bidirectional associations:

```ruby
# Without inverse_of
post = Post.first
author = post.author
author.posts.first.author  # DIFFERENT Ruby object than `author`

# With inverse_of
class Post < ApplicationRecord
  belongs_to :author, class_name: "User", inverse_of: :posts
end
class User < ApplicationRecord
  has_many :posts, inverse_of: :author
end

post = Post.first
author = post.author
author.posts.first.author  # Same Ruby object as `author`
```

Rails infers `inverse_of` when association names match class names. It fails for custom names (`belongs_to :author, class_name: "User"`). Always specify it explicitly in those cases. Missing `inverse_of` causes stale data in memory, unnecessary queries, and inconsistent state during transactions.

### `touch`, `counter_cache`, `dependent`

**`touch: true`** on `belongs_to` bumps the parent's `updated_at` on child changes. Essential for cache key expiration -- without it, cached post fragments will not know a new comment was added.

**`counter_cache: true`** denormalizes `COUNT(*)` into a column on the parent:

```ruby
# Migration
add_column :posts, :comments_count, :integer, default: 0, null: false

# Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Usage: column read, no SQL query
post.comments_count  # => 42

# For existing data:
Post.find_each { |p| Post.reset_counters(p.id, :comments) }
```

**`dependent:`** on every `has_many`/`has_one`:

| Option | Behavior | Use When |
|---|---|---|
| `:destroy` | Loads each child, calls `destroy` | Children have callbacks |
| `:delete_all` | Single `DELETE` SQL, skips callbacks | Performance-critical |
| `:nullify` | Sets FK to NULL | Children should survive |
| `:restrict_with_error` | Prevents deletion | Business rule |

Never leave orphans.

### N+1 Prevention

The N+1 manifests as repeated queries in the dev log:

```
Post Load (0.5ms)  SELECT * FROM posts LIMIT 10
User Load (0.3ms)  SELECT * FROM users WHERE id = 1
User Load (0.2ms)  SELECT * FROM users WHERE id = 2
User Load (0.3ms)  SELECT * FROM users WHERE id = 3
... (one per post)
```

Rails provides three eager-loading methods:

**`includes`** -- The smart default. Rails decides separate query or JOIN:

```ruby
Post.includes(:author).where(published: true).each { |p| p.author.name }
# SQL: SELECT posts.* FROM posts WHERE published = 1
#      SELECT users.* FROM users WHERE id IN (1, 2, 3, ...)
```

**`preload`** -- Always separate queries. Use when the association is not in WHERE, or for polymorphic associations (JOINs do not work with polymorphic):

```ruby
Post.preload(:author, :tags).each { |p| p.author.name }
```

**`eager_load`** -- Always LEFT OUTER JOIN. Use when filtering or sorting by the association:

```ruby
Post.eager_load(:author).where(users: { role: "editor" })
# SQL: SELECT posts.*, users.* FROM posts
#      LEFT OUTER JOIN users ON users.id = posts.author_id
#      WHERE users.role = 'editor'
```

Enable `config.active_record.strict_loading_by_default = true` in development to raise `StrictLoadingViolationError` on any lazy-loaded association.

### Query Idioms

**`exists?` vs `present?`:**

```ruby
# Bad: loads every post into memory, then checks array emptiness
if @user.posts.present?  # SELECT * FROM posts WHERE user_id = 1

# Good: single fast check
if @user.posts.exists?   # SELECT 1 FROM posts WHERE user_id = 1 LIMIT 1
```

**`pluck` and `select`:**

```ruby
# Bad: instantiates AR objects just to get strings
emails = User.active.map(&:email)

# Good: returns Array<String>, no AR objects
emails = User.active.pluck(:email)

# Good: AR objects with subset of columns (for rendering)
users = User.active.select(:id, :email, :name)
```

**`where.missing` / `where.associated`:**

```ruby
# Old: User.left_joins(:subscription).where(subscriptions: { id: nil })
User.where.missing(:subscription)    # Users without a subscription
User.where.associated(:subscription) # Users with a subscription
```

**`find_each`** -- Iterates in batches (default 1,000). Constant memory:

```ruby
# Bad: loads all records at once
User.where(verified: false).each(&:send_reminder)

# Good: batches of 500, constant memory
User.where(verified: false).find_each(batch_size: 500, &:send_reminder)
```

**`insert_all` / `upsert_all`** -- Bulk writes that skip callbacks and validations. One SQL statement. Use for imports and backfills, not regular application writes.

### Scopes and Transactions

Prefer scopes over class methods for query fragments -- they compose naturally. Name by intent: `scope :recent`, not `scope :ordered_by_created_at_desc`.

Wrap related writes in `transaction`. Never enqueue jobs or call external services inside a transaction (use `_commit` callbacks instead).

---

## 6. Callbacks and Jobs

### The Mandatory Lambda Rule

**Always use lambda syntax for callbacks.** Never use symbol syntax.

```ruby
# Bad
after_create_commit :notify_author

# Good
after_create_commit -> { AuthorMailer.with(comment: self).notify_later }
```

**Why:** (1) The action is visible at the declaration -- no scrolling to a private method. (2) Grep for `perform_later` finds every job enqueue site, including callbacks. (3) No named-method indirection -- the lambda *is* the implementation.

### `after_*_commit` vs `after_*`

This is a correctness requirement, not a style preference. Plain `after_create`/`after_save` run **inside the database transaction**.

**Problem 1: Race condition.** The worker picks up the job before the transaction commits:

```
Timeline:
  1. INSERT INTO messages ...     (inside transaction)
  2. after_create fires
  3. Job enqueued                 (record not yet committed!)
  4. Worker picks up job
  5. Message.find(id) -> NOT FOUND  (transaction still open)
  6. COMMIT                       (too late, job already failed)
```

**Problem 2: Lock contention on SQLite.** SQLite uses a single-writer architecture. The job tries to write while the original transaction holds the lock, causing "database is locked" errors.

Use `after_create_commit`, `after_save_commit`, `after_update_commit`, `after_destroy_commit` for anything that enqueues jobs, calls external APIs, or broadcasts over Turbo. These fire **after** the transaction has committed -- the record is guaranteed to exist and the write lock is released.

```ruby
# Bad: job fires inside the transaction
after_create -> { RelayJob.perform_later(self) }

# Good: job fires after commit
after_create_commit -> { RelayJob.perform_later(self) }

# Fine: synchronous in-transaction work
before_create -> { self.number ||= generate_number }
before_save   -> { self.total = line_items.sum(:amount) }
```

### The `_later` / `_now` Pair

Name the job-enqueuing method `<verb>_later`; add `<verb>_now` for synchronous work. The job calls `_now`.

```ruby
class Message < ApplicationRecord
  after_create_commit -> { relay_later }

  def relay_later = RelayMessageJob.perform_later(self)

  def relay_now
    recipients.each { |r| Transport.send(self, to: r) }
    update!(relayed_at: Time.current)
  end
end

class RelayMessageJob < ApplicationJob
  def perform(message) = message.relay_now
end
```

**Benefits:** Jobs stay shallow (no business logic). Testing is simple (call `_now` directly). Rake tasks and console use work without job infrastructure. The async/sync split is obvious to readers.

This pattern works naturally inside concerns:

```ruby
# app/models/message/relaying.rb
module Message::Relaying
  extend ActiveSupport::Concern

  included do
    after_create_commit -> { relay_later }
  end

  def relay_later = Message::RelayJob.perform_later(self)

  def relay_now
    recipients.each { |r| Transport.send(self, to: r) }
    update!(relayed_at: Time.current)
  end
end
```

### Retry, Discard, and Recurring Tasks

```ruby
class ApplicationJob < ActiveJob::Base
  retry_on Net::OpenTimeout, wait: :polynomially_later, attempts: 5
  discard_on ActiveJob::DeserializationError
  discard_on ActiveRecord::RecordNotFound
end
```

Define scheduled jobs in `config/recurring.yml` (Solid Queue's native format):

```yaml
production:
  clean_expired_sessions:
    class: CleanExpiredSessionsJob
    schedule: every 1 hour
  send_digest_emails:
    class: SendDigestEmailsJob
    schedule: every day at 9am
```

---

## 7. Hotwire: Turbo

### Turbo Drive

Turbo Drive intercepts every link click and form submission, fetches via AJAX, and swaps the `<body>`. SPA-like speed with zero JavaScript. It is on by default. The `<head>` is merged (CSS/JS persists). Use `data-turbo="false"` to opt out individual links.

### Turbo Frames

Frames define independently refreshable regions. Only the frame's content updates on navigation.

```erb
<%= turbo_frame_tag "post_#{@post.id}_comments",
    loading: "lazy", src: post_comments_path(@post) do %>
  <p>Loading comments...</p>
<% end %>
```

- **`loading: "lazy"`** -- Defers fetch until viewport entry
- **`data-turbo-frame="_top"`** -- Breaks out of the frame, navigates the full page
- **Nesting** -- Frames can contain frames, each updating independently
- **Never style `<turbo-frame>` directly** -- Style content inside it

### Turbo Streams

Streams push DOM updates over ActionCable. Seven actions: `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`.

```erb
<%= turbo_stream_from Current.user, "messages" %>
<div id="messages"><%= render @messages %></div>
```

```ruby
class Message < ApplicationRecord
  after_create_commit  -> { broadcast_append_to  user, :messages, target: "messages" }
  after_update_commit  -> { broadcast_replace_to user, :messages }
  after_destroy_commit -> { broadcast_remove_to  user, :messages }
end
```

**Always scope broadcasts** (include user/account in stream name) to prevent cross-tenant leaks. Without scoping, every connected user receives every broadcast. Always use `_commit` callbacks for broadcasts -- broadcasting inside a transaction races with the commit.

### When to Use Turbo Stream Responses vs Broadcasts

Turbo Streams can also be returned as HTTP responses (not just broadcasts). Use **broadcast** for real-time pushes (chat, notifications). Use **response streams** when the server needs to update multiple page regions in response to a single form submission:

```ruby
# Controller returning stream response (multiple DOM updates)
def create
  @comment = @post.comments.create!(comment_params)
  respond_to do |format|
    format.turbo_stream  # renders create.turbo_stream.erb
    format.html { redirect_to @post }
  end
end
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", @comment %>
<%= turbo_stream.update "comments_count", "#{@post.comments.count} comments" %>
```

### Streams vs Frames

| Scenario | Use |
|---|---|
| User clicks/submits, one region updates | Turbo Frame |
| Server pushes without user action | Turbo Stream broadcast |
| Form validation errors | Turbo Drive (422 re-render) |
| Lazy-loading a page section | Turbo Frame (`loading: "lazy"`) |

### Form Submission Flow

Success: controller redirects (302), Turbo follows. Failure: controller renders with `status: :unprocessable_entity` (422), Turbo re-renders the form in place. This is why 422 is mandatory on failed submissions.

### Turbo Morphing

Rails 8's Turbo Morph uses DOM morphing instead of replacement, preserving scroll position, form state, and focus. Enable with `<meta name="turbo-refresh-method" content="morph">`.

---

## 8. Hotwire: Stimulus

### What Stimulus Is

A modest JavaScript framework for adding behavior to server-rendered HTML. Not React or Vue -- it does not manage state or render templates. It attaches behavior to existing DOM elements via data attributes.

### Controller Structure

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["form", "status"]
  static values  = { delay: { type: Number, default: 1000 } }
  static outlets = ["other-controller"]
  static classes = ["active"]

  #timer = null   // Private field for internal state

  // Lifecycle
  connect()    { /* element entered DOM */ }
  disconnect() { clearTimeout(this.#timer) }

  // Actions (called from data-action attributes)
  save() { this.formTarget.requestSubmit() }

  // Private
  #schedule = () => {
    clearTimeout(this.#timer)
    this.#timer = setTimeout(() => this.save(), this.delayValue)
  }
}
```

### Data Attributes

```html
<div data-controller="clipboard" data-clipboard-content-value="Hello">
  <button data-action="click->clipboard#copy">Copy</button>
  <span data-clipboard-target="status"></span>
</div>
```

- `data-controller` -- Attaches the controller
- `data-action` -- Binds events: `event->controller#method`
- `data-<controller>-target` -- Names a target element
- `data-<controller>-<name>-value` -- Sets a typed value

### When Stimulus vs Plain Turbo

**Plain Turbo** handles server-driven updates: navigation, form submission, lazy loading, broadcasts. **Stimulus** fills gaps that need client-side behavior: toggling, clipboard, keyboard shortcuts, timers, third-party library integration.

### Common Patterns

**Auto-save:** Debounce input, submit after delay:

```javascript
export default class extends Controller {
  static targets = ["form"]
  static values  = { delay: { type: Number, default: 1000 } }
  #timer = null

  disconnect() { clearTimeout(this.#timer) }

  schedule() {
    clearTimeout(this.#timer)
    this.#timer = setTimeout(() => this.formTarget.requestSubmit(), this.delayValue)
  }
}
```

```html
<form data-controller="auto-save" data-action="input->auto-save#schedule">
```

**Dialog management:** Open/close native `<dialog>`:

```javascript
export default class extends Controller {
  static targets = ["dialog"]
  static values  = { modal: { type: Boolean, default: false } }

  open() {
    this.modalValue ? this.dialogTarget.showModal() : this.dialogTarget.show()
    this.dialogTarget.setAttribute("aria-hidden", "false")
  }

  close() {
    this.dialogTarget.close()
    this.dialogTarget.setAttribute("aria-hidden", "true")
  }

  closeOnBackdrop({ target }) {
    if (target === this.dialogTarget) this.close()
  }
}
```

**Toggle:** Show/hide with a CSS class:

```javascript
export default class extends Controller {
  static targets = ["content"]
  static classes = ["hidden"]

  toggle() { this.contentTarget.classList.toggle(this.hiddenClass) }
}
```

### Naming

File: `auto_save_controller.js` (snake_case). HTML: `data-controller="auto-save"` (kebab-case). Stimulus auto-maps between them.

---

## 9. Views and ERB

### Partials and Collection Rendering

`render @posts` uses `to_partial_path` to find `_post.html.erb`. No loop needed. Add `cached: true` for collection caching:

```erb
<%= render partial: "posts/post", collection: @posts, cached: true %>
```

### `dom_id` and Tag Helpers

`dom_id(@post)` produces `"post_123"`, matching Turbo broadcast targets. Use `tag.div`, `tag.p` builders instead of raw HTML for proper escaping and class composition:

```erb
<%= tag.div id: dom_id(@post), class: ["post", ("featured" if @post.featured?)] do %>
  ...
<% end %>
```

### `link_to` vs `button_to`

`link_to` for navigation (GET). `button_to` for state changes (POST/PATCH/DELETE):

```erb
<%= link_to "View Post", @post %>

<%= button_to "Delete", @post, method: :delete,
    data: { turbo_confirm: "Are you sure?" } %>
```

Use `turbo_confirm` (not `confirm`) in Turbo-driven apps.

### HTML Safety

**Never `.html_safe` on user input.** It disables escaping and opens XSS:

```erb
<%# DANGEROUS: user can inject <script> tags %>
<%= comment.body.html_safe %>

<%# SAFE: escaped by default %>
<%= comment.body %>

<%# SAFE: sanitized subset when rich text is needed %>
<%= sanitize(comment.rich_body,
    tags: %w[p a em strong ul ol li],
    attributes: %w[href]) %>
```

### I18n, Content Regions, Modals

- `t(".key")` for i18n lazy lookup -- resolves against the view path automatically.
- `content_for :sidebar` / `yield :sidebar` for layout regions.
- Native `<dialog>` for modals -- built-in Escape handling, focus trapping, top-layer API.
- **No inline `<script>` blocks.** All behavior via Stimulus `data-controller`/`data-action`. Inline scripts bypass CSP, are not cacheable, and create parallel behavior systems.

### Server-Rendered HTML Only

Never generate HTML in JavaScript. If you need new markup on the client, fetch a rendered partial from the server:

```javascript
// Bad: HTML built in JS
const html = `<div class="comment"><p>${body}</p></div>`

// Good: fetch server-rendered partial
const response = await fetch("/comments", {
  method: "POST",
  headers: { "Accept": "text/html", "X-CSRF-Token": this.csrfToken },
  body: new URLSearchParams({ "comment[body]": body })
})
this.listTarget.insertAdjacentHTML("beforeend", await response.text())
```

This keeps views themeable from one place, keeps Rails helpers authoritative, and lets system tests exercise real markup.

---

## 10. Error Handling

### `rescue_from` in Controllers

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionController::ParameterMissing, with: :bad_request

  private
    def not_found
      render file: "public/404.html", status: :not_found, layout: false
    end

    def bad_request(exception)
      render json: { error: exception.message }, status: :bad_request
    end
end
```

Declare specific exceptions before general ones. Do not rescue `StandardError` in `rescue_from`.

### Exception Specificity

Never bare `rescue => e`. It catches everything including bugs you need to see. Be specific:

```ruby
begin
  external_api.fetch
rescue ExternalAPI::TimeoutError => e
  Rails.logger.warn "API timeout: #{e.message}"
  retry_with_backoff
rescue ExternalAPI::ClientError => e
  Rails.error.report(e, context: { endpoint: external_api.endpoint })
  nil
end
```

### `Rails.error.report`

Unified error reporting that routes to your configured service (Sentry, Honeybadger). Also: `Rails.error.handle(SomeError) { risky_operation }` for non-critical operations.

### Custom Error Classes

Define domain error hierarchies for specific handling:

```ruby
class PaymentError < StandardError; end
class PaymentDeclinedError < PaymentError; end
class PaymentTimeoutError < PaymentError; end

# Usage: rescue specific subclass or the whole hierarchy
rescue PaymentDeclinedError => e
  notify_user_of_declined_payment(e)
rescue PaymentTimeoutError => e
  retry_payment
end
```

### Job Error Handling

In background jobs, error handling is built into the retry/discard mechanism:

```ruby
class PaymentChargeJob < ApplicationJob
  retry_on PaymentTimeoutError, wait: :polynomially_longer, attempts: 5
  discard_on PaymentDeclinedError

  def perform(invoice) = invoice.charge_now
end
```

No explicit begin/rescue needed. Undeclared exceptions bubble to Solid Queue's failure recorder.

### `ActiveRecord::RecordNotFound`

When you scope queries through `Current.user`, `RecordNotFound` serves double duty -- "record does not exist" and "record is not yours" both correctly return 404:

```ruby
@post = Current.user.posts.find(params[:id])
# Raises RecordNotFound if post does not exist OR belongs to another user
```

---

## 11. Performance and Caching

### N+1 Detection

Repeated queries in the dev log (one per collection item) signal N+1:

```
Post Load (0.5ms)  SELECT * FROM posts LIMIT 10
User Load (0.3ms)  SELECT * FROM users WHERE id = 1
User Load (0.2ms)  SELECT * FROM users WHERE id = 2
User Load (0.3ms)  SELECT * FROM users WHERE id = 3
```

Fix with `includes` in the controller. The `Bullet` gem auto-detects N+1 in development:

```ruby
# Gemfile
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.alert = true
  Bullet.rails_logger = true
end
```

### Fragment Caching

```erb
<% cache @post do %>
  <article><h1><%= @post.title %></h1></article>
<% end %>
```

Cache key auto-expires on `@post.updated_at`. Pair with `touch: true` on child associations for Russian-doll caching: child update touches parent, parent cache invalidates, nested caches regenerate only what changed.

### Collection and Low-Level Caching

`render partial: "post", collection: @posts, cached: true` -- caches per item, fetches via `read_multi`.

`Rails.cache.fetch(key, expires_in: 1.hour) { expensive_work }` -- block executes on miss.

**Solid Cache** is Rails 8's default store. Database-backed, survives restarts, no Redis.

### Counter Caches

`counter_cache: true` on `belongs_to` keeps a denormalized count column. No `COUNT(*)` on render. See Chapter 5 for the full migration and model setup.

### Indexes

Indexes are the single most impactful performance tool in a relational database.

**Rules:** Index every foreign key. Index every column you filter or sort by. Add composite indexes for multi-column queries.

**Composite index column ordering:** The leftmost-prefix rule applies:

```ruby
add_index :posts, [:account_id, :status]
# Serves: WHERE account_id = ?                     (yes)
#         WHERE account_id = ? AND status = ?       (yes)
#         WHERE status = ?                          (NO -- not leftmost)
```

**Partial indexes** for common filters:

```ruby
add_index :posts, :created_at,
  where: "published_at IS NOT NULL",
  name: "index_published_posts_on_created_at"
```

Add indexes in the same migration as the column whenever possible.

### `size` vs `count` vs `length`

`size` does the right thing (in-memory count if loaded, `COUNT(*)` if not). `count` always hits DB. `length` always loads everything. Default to `size`.

---

## 12. Testing

### Minitest and Fixtures

DHH's rationale for **Minitest**: plain Ruby, no DSL (`describe`/`it`/`let`), immediately readable to anyone who knows Ruby. For **fixtures**: load once (fast), deterministic IDs, real schema, no factory inheritance chains to trace.

```yaml
# test/fixtures/users.yml
alice:
  email: alice@example.com
  password_digest: <%= BCrypt::Password.create("password", cost: 4) %>
```

```ruby
class PostTest < ActiveSupport::TestCase
  test "publishing sets published_at" do
    post = posts(:draft)
    post.publish
    assert_not_nil post.published_at
  end
end
```

**If your project uses RSpec/FactoryBot, use them.** Consistency with the codebase beats alignment with this guide.

### Key Assertions

```ruby
assert_difference -> { Post.count }, +1 do ... end
assert_enqueued_with(job: NotifyJob, args: [post]) do ... end
assert_changes -> { post.published_at }, from: nil do ... end
assert_no_difference -> { Post.count } do ... end
```

### System Tests

Capybara-driven browser tests in `test/system/`:

```ruby
require "application_system_test_case"

class PostsTest < ApplicationSystemTestCase
  test "authoring and publishing a post" do
    sign_in users(:alice)

    visit new_post_path
    fill_in "Title", with: "Hello Hotwire"
    fill_in "Body",  with: "Turbo Frames make this page feel instant."
    click_on "Create Post"

    assert_selector "h1", text: "Hello Hotwire"
    click_on "Publish"
    assert_text "Published"
  end
end
```

Use label-based selectors (`fill_in "Title"`, `click_on "Publish"`), not CSS selectors. Selectors like `find(".post-card > .header > button")` break when markup changes. Test the critical user path; system tests are slow, so be selective.

### Parallel Tests and VCR

`parallelize(workers: :number_of_processors)` in test_helper. Each worker gets its own database, so tests do not interfere.

Use `vcr` + `webmock` to record/replay external API responses:

```ruby
test "fetches profile from external API" do
  VCR.use_cassette("github_profile") do
    profile = GitHubClient.fetch_profile("dhh")
    assert_equal "David Heinemeier Hansson", profile.name
  end
end
```

### What NOT to Test

Rails framework behavior (callbacks fire, routes route). Trivial getters. Implementation details (private methods, specific SQL). Third-party gem internals.

---

## 13. Security and Authentication

### Rails 8 Authentication Generator

`bin/rails generate authentication` scaffolds User + Session models, SessionsController, PasswordsController, and an Authentication concern. ~200 lines you own. The session lives in a DB row; the browser carries a signed cookie with the session's `signed_id`.

**Why not Devise?** 200 lines you understand vs. 10,000+ lines you do not. Customize by editing your own code, not overriding gem internals.

### `Current` Attributes

`ActiveSupport::CurrentAttributes` provides per-request global state, scoped to the current thread:

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end
```

Set in authentication, read everywhere:

```ruby
# In models -- default associations
belongs_to :author, class_name: "User", default: -> { Current.user }

# In controllers -- scope queries
@posts = Current.user.posts

# In views
<%= Current.user.name %>
```

**Thread safety:** Stored in `Thread.current`, automatically isolated between requests. Reset at the end of each request by Rails middleware.

**Testing:**

```ruby
setup do
  Current.session = sessions(:alice_session)
end

teardown do
  Current.reset_all
end
```

**In jobs:** Jobs run in different threads. Explicitly set context:

```ruby
def perform(record)
  Current.set(session: record.user.sessions.last) do
    # work that needs Current.user
  end
end
```

### CSRF Protection

`protect_from_forgery` is on by default. Every form includes a hidden token; Turbo includes it automatically in AJAX requests. Do not disable it.

### Content Security Policy (CSP)

Configure in the initializer. CSP is your strongest defense against XSS:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
  policy.frame_ancestors :none
end

Rails.application.config.content_security_policy_nonce_generator =
  ->(request) { SecureRandom.base64(16) }
```

Key directives: `default_src :self` (same-origin only), `script_src :self` (blocks injected scripts), `frame_ancestors :none` (prevents clickjacking), `object_src :none` (blocks Flash/plugins).

### Encrypted Credentials

Secrets go in encrypted credentials, not environment variables:

```bash
bin/rails credentials:edit  # Opens in $EDITOR
```

```ruby
Rails.application.credentials.stripe.secret_key
```

Commit `credentials.yml.enc`. Never commit `master.key` -- add it to `.gitignore`.

### `allow_browser` and HTML Safety

`allow_browser versions: :modern` drops legacy browsers (pre-Chrome 119, Safari 17, Firefox 121).

**Never `.html_safe` on user input.** Use `sanitize` with an allowlist. The only safe uses of `.html_safe`: strings you built entirely from trusted components, or output from Rails helpers.

---

## 14. Migrations

### Reversibility

Use the `change` method for auto-reversible operations:

```ruby
class AddStatusToPosts < ActiveRecord::Migration[8.0]
  def change
    add_column :posts, :status, :integer, default: 0, null: false
    add_index  :posts, :status
  end
end
# Rollback automatically removes the index and column
```

Auto-reversible: `add_column`/`remove_column`, `add_index`/`remove_index`, `create_table`/`drop_table`, `add_reference`/`remove_reference`, `add_foreign_key`/`remove_foreign_key`, `change_column_default` (with `from:`/`to:`), `change_column_null`.

Use `up`/`down` for data migrations and type changes where Rails cannot infer the reverse.

### Zero-Downtime Patterns

**Adding a column:** Add with a default. Modern PostgreSQL (11+) and SQLite handle this without rewriting the table:

```ruby
add_column :posts, :published, :boolean, default: false, null: false
```

**Removing a column (two-step):**

Step 1 -- Stop using it and deploy:
```ruby
class Post < ApplicationRecord
  self.ignored_columns += ["legacy_status"]
end
```

Step 2 -- Remove the column and deploy:
```ruby
remove_column :posts, :legacy_status, :string
```

**Renaming a column (four-step):** Add new column, backfill, update code, drop old. Never `rename_column` in production -- it breaks running code referencing the old name.

**Index concurrently (PostgreSQL):**

```ruby
class AddIndexOnPostsAuthorId < ActiveRecord::Migration[8.0]
  disable_ddl_transaction!

  def change
    add_index :posts, :author_id, algorithm: :concurrently
  end
end
```

`algorithm: :concurrently` builds the index without locking the table for writes. Requires `disable_ddl_transaction!`.

### Foreign Keys and UUIDs

`foreign_key: true` on every reference. Database enforcement catches bugs ORMs miss.

UUIDv7 primary keys when IDs are URL-exposed, generated across systems, or needed offline. Configure in `config/application.rb`. Default integers are fine for most apps.

Consider `strong_migrations` gem for large production databases as a safety net against unsafe operations.

---

## 15. Tooling and DX

### `rubocop-rails-omakase`

Rails' official linting configuration. No custom overrides:

```yaml
# .rubocop.yml
inherit_gem:
  rubocop-rails-omakase: rubocop.yml
```

Enforces double quotes, indentation, method length, Rails-specific cops. The philosophy: the whole team uses the same rules. Consistency beats individual preference. Run: `bin/rubocop`. Auto-fix: `bin/rubocop -a`.

### `brakeman`

Static analysis security scanner. Catches SQL injection, XSS, mass assignment, CSRF issues, unsafe redirects, command injection:

```bash
bin/brakeman --no-pager
```

Brakeman is conservative -- it flags potential issues even when code is safe. Document genuine false positives in `config/brakeman.ignore` with justification.

### `bundler-audit` and `bin/importmap audit`

- `bundle exec bundler-audit --update` -- checks Gemfile.lock against known CVEs
- `bin/importmap audit` -- checks JavaScript dependencies (importmap-rails projects)

### CI and Development Scripts

**`bin/ci`** mirrors the CI pipeline locally:

```bash
#!/usr/bin/env bash
set -e
bin/rubocop
bundle exec bundler-audit --update
bin/importmap audit
bin/brakeman --no-pager
bin/rails db:test:prepare
bin/rails test
bin/rails test:system
```

**`bin/setup`** bootstraps a fresh machine (bundle install, db:prepare, db:seed). **`bin/dev`** starts Puma + Solid Queue via `foreman` and `Procfile.dev`.

---

## 16. Rails 8+ Features and Deployment

### Solid Queue

Database-backed job processing. No Redis.

**Puma plugin mode** (single-server, especially SQLite -- SQLite's single-writer architecture requires one process):

```ruby
# config/puma.rb
plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"] == "true"
```

**Standalone mode** (larger deployments with PostgreSQL): `bin/jobs` as a separate process. Scale web and job workers independently.

Configuration:

```yaml
# config/queue.yml
production:
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 0.1
```

Supports priorities, concurrency limits, recurring tasks. Job data survives restarts.

**Monitoring:**

```ruby
SolidQueue::Job.where(finished_at: nil).count  # Pending jobs
SolidQueue::FailedExecution.count               # Failed jobs
```

**Comparison to Sidekiq:**

| Aspect | Solid Queue | Sidekiq |
|---|---|---|
| Backend | Database (SQLite/PostgreSQL) | Redis |
| Infrastructure | No extra services | Requires Redis |
| Throughput | Hundreds/sec (most apps) | Thousands/sec |
| Durability | Survives restarts by default | Requires Redis persistence |
| Cost | Free | Pro/Enterprise features are paid |

Sidekiq only wins on very high throughput where Redis's in-memory speed matters.

### Solid Cache and Solid Cable

**Solid Cache:** Database-backed `Rails.cache`. Survives restarts. Can cache larger datasets than RAM-limited Redis. `config.cache_store = :solid_cache_store`.

**Solid Cable:** Database-backed ActionCable adapter. Replaces Redis for WebSocket pub/sub.

**Net effect:** A Rails 8 app runs with just Rails and a database. No Redis, no Memcached, no external queue. This dramatically simplifies:
- **Deployment:** One app server, one database. No Redis container.
- **Monitoring:** One system to watch.
- **Cost:** No Redis hosting fees.
- **Reliability:** Fewer moving parts, fewer failure modes.

### Importmap-rails and Propshaft

### Importmap-rails

Browser-native ES module imports. No bundler, no build step:

```bash
bin/importmap pin @hotwired/turbo
bin/importmap pin @hotwired/stimulus
```

```ruby
# config/importmap.rb
pin "application"
pin "@hotwired/turbo-rails", to: "turbo.min.js"
pin "@hotwired/stimulus", to: "stimulus.min.js"
pin_all_from "app/javascript/controllers", under: "controllers"
```

**How it works:** An import map tells the browser where to find each module. `import { Controller } from "@hotwired/stimulus"` looks up the URL in the import map and fetches directly. No bundling, no transpilation.

**Limitations:** No tree-shaking, no TypeScript compilation, not suitable for heavy npm dependency trees (React, Vue). For those, use `jsbundling-rails` with esbuild.

### Propshaft

Serves assets with a digest hash for cache busting (`application-abc123.css`). Does not compile, concatenate, or minify -- those are handled by Tailwind CLI, importmap-rails, or jsbundling-rails. Replaces Sprockets' complexity with focused simplicity.

### Kamal Deployment

Deploys to any SSH-accessible server via Docker:

```yaml
# config/deploy.yml
service: myapp
image: myregistry/myapp
servers:
  web: [192.168.0.1]
proxy:
  ssl: true
  host: app.example.com
registry:
  server: ghcr.io
  username: myuser
  password: [KAMAL_REGISTRY_PASSWORD]
env:
  secret: [RAILS_MASTER_KEY]
  clear:
    SOLID_QUEUE_IN_PUMA: "true"
```

**Flow:** Build image, push to registry, pull on server, start new container, healthcheck (`/up`), switch traffic, stop old container. Zero-downtime rolling deploys.

**Commands:** `kamal setup` (provision), `kamal deploy` (ship), `kamal rollback <version>` (revert).

**Secrets management:** Store in `.kamal/secrets` (git-ignored):

```bash
# .kamal/secrets
RAILS_MASTER_KEY=abc123...
KAMAL_REGISTRY_PASSWORD=ghp_...
```

**Health checks:** Rails 8 ships a `/up` endpoint returning 200 when healthy. Kamal uses it during deploys. Point your monitoring (UptimeRobot, Healthchecks.io) at it too.

**Rollback:** If a deploy introduces a bug, `kamal rollback <version>` reverts to the previous image instantly. No rebuild required.

---

## 17. Gotchas and Common Mistakes

### Symbol Callbacks Instead of Lambda

```ruby
# Wrong                               # Right
after_create_commit :relay_later       after_create_commit -> { relay_later }
```

The most commonly violated rule. See Chapter 6 for the rationale: clarity at call site, greppability, no indirection.

### `after_create` for Jobs Instead of `after_create_commit`

```ruby
# Wrong: job fires inside the transaction
after_create -> { NotifyJob.perform_later(self) }

# Right: job fires after commit
after_create_commit -> { NotifyJob.perform_later(self) }
```

Causes race conditions (job runs before record is committed) and lock contention on SQLite.

### Unscoped Queries

```ruby
# Wrong: authorization bypass
@post = Post.find(params[:id])

# Right: scoped to current user
@post = Current.user.posts.find(params[:id])
```

### `.html_safe` on User Input

XSS vulnerability. Use `sanitize` with an allowlist, or let Rails escape by default.

### Missing `inverse_of`

When association name differs from class name, specify explicitly: `belongs_to :author, class_name: "User", inverse_of: :posts`. Otherwise, stale data and unnecessary queries.

### N+1 in Views

The controller loads a collection without `includes`. Fix is always in the controller.

### Fat Controllers and God Models

Controllers over 10 lines: move logic to model methods. Models over ~100 lines: extract concerns.

### Testing Implementation Instead of Behavior

```ruby
# Wrong: testing how
assert_called(post, :broadcast_publication) { post.publish }

# Right: testing what
post.publish
assert post.published_at.present?
```

### Devise in New Projects

Rails 8 auth generator produces ~200 lines you own. Use it unless you need specific Devise features.

### Redis When Solid Trio Suffices

Solid Queue + Cache + Cable handle most apps. Redis adds operational complexity without proportional benefit for typical workloads.

### Missing 422 on Form Errors

```ruby
# Wrong: Turbo navigates away, losing errors
render :new

# Right: Turbo re-renders form in place
render :new, status: :unprocessable_entity
```

---

## 18. Recommended Stack

| Concern | Tool | Why |
|---|---|---|
| **Framework** | Rails 8+ | Convention over configuration |
| **Auth** | `bin/rails generate authentication` | ~200 lines you own |
| **Jobs** | Solid Queue | DB-backed, no Redis |
| **Cache** | Solid Cache | DB-backed, persistent |
| **Pub/Sub** | Solid Cable | DB-backed, no Redis |
| **JavaScript** | Importmap-rails + Stimulus | No build step |
| **Real-Time** | Turbo (Drive + Frames + Streams) | Server-rendered, minimal JS |
| **Assets** | Propshaft | Simple fingerprinting |
| **Deploy** | Kamal | Any server via SSH |
| **Testing** | Minitest + Fixtures | Ships with Rails |
| **Linting** | rubocop-rails-omakase | Official, no config |
| **Security** | Brakeman + bundler-audit | Static analysis + CVEs |
| **Database** | SQLite (small) or PostgreSQL (large) | Match your scale |
| **Scheduled** | `config/recurring.yml` | Native to Solid Queue |
| **Authorization** | Plain Ruby model methods | No gem needed |
| **Storage** | Active Storage | Built into Rails |
| **Email** | Action Mailer | Built into Rails |

---

## 19. Quick Reference Cheat Sheet

### Ruby Style
- `frozen_string_literal: true` at the top of every file
- Double quotes by default
- Expanded conditionals over guard clauses
- Class methods, then public, then private -- ordered by call sequence
- `private` with no blank line after, methods indented under it
- Bang methods only when a non-bang counterpart exists
- Plain Ruby over gems when the problem fits in 20 lines

### Controllers
- Every action maps to CRUD on a resource -- new resources over custom actions
- Under 10 lines per action; `params.expect` for strong params
- `status: :unprocessable_entity` on failed form renders
- `head :status` for bodyless responses

### Models
- Business logic on models, not controllers or services
- Extract concerns at ~100 lines; include alphabetically
- `normalizes`, `generates_token_for`, `encrypts`, `enum ... prefix: true`
- Lambda defaults: `default: -> { Current.user }`

### ActiveRecord
- `inverse_of` on non-standard names; `touch: true` for cache busting
- `counter_cache: true` for counts; `dependent:` on every has_many/has_one
- `includes` for N+1; `exists?` over `present?`; `pluck` for lean queries
- `find_each` for batching; `where.missing`/`where.associated`

### Callbacks and Jobs
- Lambda syntax always: `after_save -> { ... }`
- `_commit` callbacks for async: `after_create_commit -> { ... }`
- `_later`/`_now` pairs; jobs are shallow wrappers
- `retry_on` transient, `discard_on` permanent; `config/recurring.yml`

### Hotwire
- Turbo Drive: automatic. Frames: scoped updates. Streams: real-time broadcasts
- Stimulus: targets/values/outlets at top; lifecycle/actions/private sections
- `#privateFields` for internal state; native `<dialog>` for modals

### Views
- `render @collection`; `dom_id`; `tag.div`; `link_to` vs `button_to`
- Never `.html_safe` on user input; no inline `<script>`

### Security
- Auth generator over Devise; scope queries through `Current.user`
- CSP in initializer; credentials via `rails credentials:edit`
- `allow_browser versions: :modern`

### Migrations
- `change` for reversibility; add/deploy/backfill for new columns
- `ignored_columns` before dropping; `algorithm: :concurrently` for large indexes
- `foreign_key: true` on every reference; index every FK

### Testing
- Minitest + fixtures for new projects; match existing framework
- `assert_difference`, `assert_enqueued_with`, `assert_changes`
- System tests with label selectors; test behavior, not implementation

### Deployment
- Solid Trio replaces Redis; Importmap replaces Webpack; Propshaft replaces Sprockets
- Kamal deploys to any server; `/up` for healthchecks
- `bin/ci` locally mirrors CI; `bin/dev` starts all processes
