# Vanilla Rails + Hotwire Patterns

Patterns from 37signals for building Rails applications with Hotwire.

## Routing: Resources as Nouns

### REST Purity

Model every action as CRUD on a resource. Custom actions become nested resources:

```ruby
# Bad: custom actions
resources :cards do
  post :close
  post :reopen
end

# Good: nested resource
resources :cards do
  resource :closure  # create = close, destroy = reopen
end
```

Apply consistently:

```ruby
resources :cards do
  resource :closure      # close/reopen
  resource :pin          # pin/unpin
  resource :watch        # watch/unwatch
  resource :publish      # publish/unpublish
  resource :reading      # mark as read
  resources :comments
end
```

### Scoped Modules

Use `scope module:` to namespace controllers without affecting URLs:

```ruby
resources :cards do
  scope module: :cards do
    resource :closure
    resources :comments do
      resources :reactions, module: :comments
    end
  end
end
```

### Route Resolvers

Simplify polymorphic routing:

```ruby
resolve "Comment" do |comment, options|
  options[:anchor] = ActionView::RecordIdentifier.dom_id(comment)
  route_for :card, comment.card, options
end

resolve "Notification" do |notification, options|
  polymorphic_url(notification.notifiable_target, options)
end
```

## Controllers

### Thin Controllers, Rich Models

Controllers call model methods. No service objects unless justified:

```ruby
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create
    @card.close(user: Current.user)
    render_card_replacement
  end

  def destroy
    @card.reopen(user: Current.user)
    render_card_replacement
  end
end
```

### Scoping via Concerns

Concerns handle common setup:

```ruby
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card, :set_board
  end

  private
    def set_card
      @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
    end

    def set_board
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(
        [@card, :card_container],
        partial: "cards/container",
        method: :morph,
        locals: { card: @card.reload }
      )
    end
end
```

### Parameter Expectations

Rails 8's `params.expect` replaces `require`/`permit`:

```ruby
def card_params
  params.expect(card: [:status, :title, :description, tag_ids: []])
end
```

### Simple Authorization

```ruby
before_action :ensure_permission, only: %i[destroy]

private
  def ensure_permission
    head :forbidden unless Current.user.can_administer?(@card)
  end
```

## Models

### Concern-Heavy Architecture

Models include small, focused concerns:

```ruby
class Card < ApplicationRecord
  include Closeable, Watchable, Pinnable, Taggable, Searchable
end
```

Each concern handles one behavior:

```ruby
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed?
    closure.present?
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure!(user: user)
        track_event :closed, creator: user
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure&.destroy
        track_event :reopened, creator: user
      end
    end
  end
end
```

### Concerns as Namespaced Files

```
app/models/card.rb
app/models/card/closeable.rb
app/models/card/watchable.rb
app/models/card/pinnable.rb
```

### Defaults via Lambdas

```ruby
belongs_to :account, default: -> { board.account }
belongs_to :creator, class_name: "User", default: -> { Current.user }
```

### Expressive Scopes

```ruby
scope :chronologically, -> { order(created_at: :asc, id: :asc) }
scope :latest, -> { order(last_active_at: :desc, id: :desc) }

scope :indexed_by, ->(index) do
  case index
  when "closed" then closed
  when "active" then active
  else all
  end
end
```

## Turbo Integration

### Turbo Streams from Controllers

```ruby
def render_card_replacement
  render turbo_stream: turbo_stream.replace(
    [@card, :card_container],
    partial: "cards/container",
    method: :morph,
    locals: { card: @card.reload }
  )
end
```

### Turbo Stream Templates

`create.turbo_stream.erb`:

```erb
<%= turbo_stream.before [@card, :new_comment],
    partial: "cards/comments/comment",
    locals: { comment: @comment } %>

<%= turbo_stream.update [@card, :new_comment],
    partial: "cards/comments/new",
    locals: { card: @card } %>
```

### Array-Based DOM IDs

Arrays create scoped identifiers:

```erb
<%= turbo_frame_tag [@card, :new_comment] do %>
  ...
<% end %>
```

Generates: `<turbo-frame id="card_123_new_comment">`

### Broadcasts from Models

```ruby
module Card::Broadcastable
  extend ActiveSupport::Concern

  included do
    broadcasts_refreshes
  end
end
```

Subscribe in views:

```erb
<%= turbo_stream_from @card %>
<%= turbo_stream_from @card, :activity %>
```

### Flash via Turbo Stream

```ruby
module TurboFlash
  extend ActiveSupport::Concern

  included do
    helper_method :turbo_stream_flash
  end

  private
    def turbo_stream_flash(**flash_options)
      turbo_stream.replace(:flash, partial: "layouts/shared/flash", locals: { flash: flash_options })
    end
end
```

### Permanent Elements

Persist UI state across navigation:

```erb
<div id="footer_frames" data-turbo-permanent="true">
  <%= render "notifications/tray" %>
</div>
```

## Stimulus Patterns

### Naming

Descriptive, behavior-based names:

- `auto_save_controller.js`
- `dialog_controller.js`
- `fetch_on_visible_controller.js`
- `drag_and_drop_controller.js`

### Structure

```javascript
export default class extends Controller {
  static targets = ["dialog"]
  static values = {
    modal: { type: Boolean, default: false }
  }
  static classes = ["active"]
}
```

### Private Methods

ES2022 private fields:

```javascript
export default class extends Controller {
  #timer

  disconnect() {
    this.submit()
  }

  change(event) {
    if (!this.#dirty) {
      this.#scheduleSave()
    }
  }

  #scheduleSave() {
    this.#timer = setTimeout(() => this.#save(), 3000)
  }

  async #save() {
    this.#resetTimer()
    await submitForm(this.element)
  }

  get #dirty() {
    return !!this.#timer
  }
}
```

### Event Dispatching

Coordinate controllers via events:

```javascript
open() {
  this.dialogTarget.showModal()
  this.dispatch("show")
}

close() {
  this.dialogTarget.close()
  this.dispatch("close")
}
```

### Turbo Event Integration

```javascript
connect() {
  this.element.addEventListener("turbo:submit-end", this.#handleSubmitEnd.bind(this), { once: true })
}

#handleSubmitEnd(event) {
  if (event.detail.success) {
    this.element.remove()
  }
}
```

### Lazy Loading with IntersectionObserver

```javascript
export default class extends Controller {
  static values = { url: String }

  connect() {
    this.#observe()
  }

  #observe() {
    const observer = new IntersectionObserver((entries) => {
      if (entries.find(entry => entry.isIntersecting)) {
        this.#fetch()
      }
    })
    observer.observe(this.element)
  }

  #fetch() {
    get(this.urlValue, { responseKind: "turbo-stream" })
  }
}
```

### Using @rails/request.js

```javascript
import { post, get } from "@rails/request.js"

async #submitRequest(url) {
  const body = new FormData()
  return post(url, { body, headers: { Accept: "text/vnd.turbo-stream.html" } })
}
```

## View Patterns

### Layout Structure

```erb
<body data-controller="local-time turbo-navigation"
      data-action="turbo:morph@window->local-time#refreshAll">
  <header>
    <%= yield :header %>
  </header>

  <%= render "layouts/shared/flash" %>

  <main id="main">
    <%= yield %>
  </main>

  <footer>
    <div id="footer_frames" data-turbo-permanent="true">
      <%= render "notifications/tray" %>
    </div>
  </footer>
</body>
```

### Content Blocks

```erb
<% content_for :header do %>
  <div class="header__actions">
    <%= link_back_to(@board) %>
  </div>
<% end %>
```

### Helper Methods for Components

```ruby
def card_article_tag(card, **options, &block)
  classes = [
    options.delete(:class),
    ("card--active" if card.active?)
  ].compact.join(" ")

  tag.article(
    id: dom_id(card, :article),
    style: "--card-color: #{card.color}",
    class: classes,
    **options,
    &block
  )
end
```

## Code Style

### Method Ordering

1. Class methods
2. Public methods (initialize first)
3. Private methods

### Invocation Order

Methods appear in call order:

```ruby
def process
  validate
  transform
end

private
  def validate; end
  def transform; end
```

### Visibility Modifiers

No newline after `private`; indent content:

```ruby
class Example
  def public_method
  end

  private
    def private_method
    end
end
```

### Job Naming

Use `_later` and `_now` suffixes:

```ruby
module Event::Relaying
  def relay_later
    Event::RelayJob.perform_later(self)
  end

  def relay_now
    # actual work
  end
end
```

## Summary

1. **REST purity** - Model actions as resources
2. **Thin controllers** - Controllers call models
3. **Small concerns** - One behavior per concern
4. **Array DOM IDs** - `[@card, :container]` for scoped targets
5. **Private Stimulus methods** - `#method` for internal state
6. **Event coordination** - Controllers dispatch, others listen
7. **Morph updates** - `method: :morph` for smooth DOM updates
8. **Permanent elements** - `data-turbo-permanent` preserves state
9. **Current attributes** - `Current.user` everywhere
