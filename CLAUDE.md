# Professor's Coding Patterns & Feature Reference

This project follows specific coding conventions required by the professor for ENTR-451. All code must match these patterns exactly.

## Syntax & Notation Rules

### String Keys with Hash Rockets (NEVER symbol shorthand)
```ruby
# YES:
User.find_by({ "id" => session["user_id"] })
Like.where({ "post_id" => post["id"] })

# NO:
User.find_by(id: session[:user_id])
```

### Bracket Notation for Attributes (NEVER dot notation)
```ruby
# YES:
@user["first_name"]
@user["id"]
post["body"]

# NO:
@user.first_name
@user.id
```

### Bracket Notation for params, session, flash
```ruby
params["email"]
session["user_id"]
flash["notice"]
```

### Explicit Nil Checks (NEVER .nil? or .present?)
```ruby
# YES:
if @user != nil
if @user == nil

# NO:
if @user.present?
if @user.nil?
```

### Field-by-Field Assignment (NEVER mass assignment or strong params)
```ruby
@post = Post.new
@post["body"] = params["body"]
@post["user_id"] = @user["id"]
@post.save
```

### for..in Loops in Views (NEVER .each)
```ruby
<% for post in @posts %>
```

### Redirect with String Paths (NEVER named route helpers)
```ruby
redirect_to "/posts"
redirect_to "/login"
```

### Raw HTML Forms (NEVER form_with, form_for, form_tag)
```html
<form action="/posts" method="post">
  <div class="mb-3">
    <label for="body_input" class="form-label">Body</label>
    <input type="text" name="body" id="body_input" class="form-control">
  </div>
  <button class="btn btn-primary">Save</button>
</form>
```

### DELETE via Hidden Input
```html
<form action="/likes/<%= like["id"] %>" method="post">
  <input type="hidden" name="_method" value="delete">
  <button class="btn p-0">Delete</button>
</form>
```

### Routes
```ruby
resources "posts"           # string argument, not symbol
resources "users"

# custom routes with explicit hash
get("/login", { :controller => "sessions", :action => "new" })
get("/logout", { :controller => "sessions", :action => "destroy" })
get("/", { :controller => "posts", :action => "index" })
```

### Models are Minimal
No `has_many`, `belongs_to`, or validations. Associations are handled manually via queries in controllers/views. The one exception is Active Storage: `has_one_attached :uploaded_image`.

### Database Queries Directly in Views
```erb
<% user = User.find_by({ "id" => post["user_id"] }) %>
<%= Like.where({ "post_id" => post["id"] }).count %>
```

### JSON Responses use Hash Rocket
```ruby
render :json => @posts
```

---

## Feature 1: User Authentication

### ApplicationController (current_user)
```ruby
class ApplicationController < ActionController::Base
  before_action :current_user

  def current_user
    @current_user = User.find_by({ "id" => session["user_id"] })
  end
end
```

### Signup (UsersController)
```ruby
def new
end

def create
  @user = User.new
  @user["first_name"] = params["first_name"]
  @user["last_name"] = params["last_name"]
  @user["email"] = params["email"]
  @user["password"] = BCrypt::Password.create(params["password"])
  @user.save
  flash["notice"] = "Thanks for signing up. Now login."
  redirect_to "/login"
end
```
- User is NOT auto-logged-in after signup
- Password hashed with `BCrypt::Password.create()`

### Login (SessionsController)
```ruby
def new
end

def create
  @user = User.find_by({ "email" => params["email"] })
  if @user != nil
    if BCrypt::Password.new(@user["password"]) == params["password"]
      session["user_id"] = @user["id"]
      flash["notice"] = "Hello."
      redirect_to "/posts"
    else
      flash["notice"] = "Nope."
      redirect_to "/login"
    end
  else
    flash["notice"] = "Nope."
    redirect_to "/login"
  end
end
```
- Same "Nope." message for both wrong email and wrong password
- Password compared with `BCrypt::Password.new(stored_hash) == plaintext`

### Logout
```ruby
def destroy
  session["user_id"] = nil
  flash["notice"] = "Goodbye."
  redirect_to "/login"
end
```

### Login Form (sessions/new.html.erb)
```html
<h1>Login</h1>

<form action="/sessions" method="post">
  <div class="mb-3">
    <label for="email_input" class="form-label">Email</label>
    <input type="text" name="email" id="email_input" class="form-control">
  </div>
  <div class="mb-3">
    <label for="password_input" class="form-label">Password</label>
    <input type="password" name="password" id="password_input" class="form-control">
  </div>
  <button class="btn btn-primary">Login</button>
</form>
```

### Signup Form (users/new.html.erb)
```html
<h1>New User Account</h1>

<form action="/users" method="post">
  <div class="mb-3">
    <label for="first_name_input" class="form-label">First Name</label>
    <input type="text" name="first_name" id="first_name_input" class="form-control">
  </div>
  <div class="mb-3">
    <label for="last_name_input" class="form-label">Last Name</label>
    <input type="text" name="last_name" id="last_name_input" class="form-control">
  </div>
  <div class="mb-3">
    <label for="email_input" class="form-label">Email</label>
    <input type="text" name="email" id="email_input" class="form-control">
  </div>
  <div class="mb-3">
    <label for="password_input" class="form-label">Password</label>
    <input type="password" name="password" id="password_input" class="form-control">
  </div>
  <button class="btn btn-primary">Signup</button>
</form>
```

### Gemfile Requirement
```ruby
gem "bcrypt", "~> 3.1.7"
```

---

## Feature 2: User Authorization

Authorization is lightweight and done per-view/per-action (no global gate):

### In Views (show/hide content based on login)
```erb
<%# posts/new.html.erb %>
<% if @user != nil %>
  <%# show the form %>
<% else %>
  You must first <a href="/login">login</a> to post.
<% end %>

<%# posts/index.html.erb - like buttons %>
<% if session["user_id"] == nil %>
  <%# show static heart icon, no form %>
<% else %>
  <%# show like/unlike form %>
<% end %>
```

### In Controllers (protect create actions)
```ruby
def create
  @user = User.find_by({ "id" => session["user_id"] })
  if @user != nil
    # proceed with creating the record
  else
    flash["notice"] = "Login first."
  end
  redirect_to "/posts"
end
```

### Navbar Conditional (in layout)
```erb
<% @user = User.find_by({ "id" => session["user_id"] }) %>
<% if @user == nil %>
  <li class="nav-item"><a href="/users/new" class="nav-link">Sign Up</a></li>
  <li class="nav-item"><a href="/login" class="nav-link">Login</a></li>
<% else %>
  <li class="nav-text">Logged in as: <%= @user["first_name"] %></li>
  <li class="nav-item"><a href="/logout" class="nav-link">Logout</a></li>
<% end %>
```

---

## Feature 3: Frontend with Bootstrap

### Layout Head (CDN loading)
```html
<!-- Bootstrap CSS -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha3/dist/css/bootstrap.min.css" rel="stylesheet">

<!-- Bootstrap Icons -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css" rel="stylesheet">

<!-- Custom CSS -->
<link rel="stylesheet" href="/stylesheets/application.css">

<!-- Bootstrap JS -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha3/dist/js/bootstrap.bundle.min.js"></script>
```

### Navbar Structure
```html
<nav class="navbar navbar-expand-lg navbar-light bg-light">
  <div class="container">
    <a class="navbar-brand" href="/">App Name</a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse"
            data-bs-target="#navbarSupportedContent">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navbarSupportedContent">
      <ul class="navbar-nav me-auto mb-2 mb-lg-0">
        <!-- left-side links -->
      </ul>
      <ul class="navbar-nav mb-2 mb-lg-0">
        <!-- right-side auth links (conditional) -->
      </ul>
    </div>
  </div>
</nav>
```

### Flash Messages
```erb
<div class="container mt-3">
  <% if flash["notice"] != nil %>
    <div class="alert alert-primary">
      <%= flash["notice"] %>
    </div>
  <% end %>
  <%= yield %>
</div>
```

### Common Bootstrap Classes Used
- Layout: `container`, `row`, `col-sm-6 col-md-3`, `col`, `mt-3`
- Forms: `mb-3`, `form-label`, `form-control`
- Buttons: `btn btn-primary`, `btn btn-success`, `btn p-0`
- Text: `text-end`, `text-danger`, `small`, `fst-italic`
- Images: `img-fluid`
- Alerts: `alert alert-primary`
- Icons: `bi bi-heart`, `bi bi-heart-fill`

### Post Grid Layout
```erb
<div class="row">
  <% for post in @posts %>
    <div class="col-sm-6 col-md-3">
      <!-- post content -->
    </div>
  <% end %>
</div>
```

### Date Formatting
```erb
<%= post["created_at"].strftime("%-m/%d/%y at %-I:%M %p") %>
```

---

## Feature 4: File Attachment (Active Storage)

### Model
```ruby
class Post < ApplicationRecord
  has_one_attached :uploaded_image
end
```

### Form (with enctype for file upload)
```html
<form action="/posts" method="post" enctype="multipart/form-data">
  <div class="mb-3">
    <label for="uploaded_image_input" class="form-label">Upload Image</label>
    <input type="file" name="uploaded_image" id="uploaded_image_input" class="form-control">
  </div>
  <button class="btn btn-primary">Save</button>
</form>
```

### Controller (attaching the file)
```ruby
@post = Post.new
@post["body"] = params["body"]
@post.uploaded_image.attach(params["uploaded_image"])
@post["user_id"] = @user["id"]
@post.save
```
Note: Active Storage methods (`attach`, `attached?`, `url_for`) use dot notation — this is the one exception to the bracket notation rule.

### Displaying in Views
```erb
<% if post.uploaded_image.attached? %>
  <img src="<%= url_for(post.uploaded_image) %>" class="img-fluid">
<% else %>
  <img src="<%= post["image"] %>" class="img-fluid">
<% end %>
```

### Storage Config (config/storage.yml)
```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

### Production Config (config/environments/production.rb)
```ruby
config.active_storage.service = :local
```

### Gemfile Requirement
```ruby
gem "aws-sdk-s3", require: false
```

---

## Feature 5: Deployment (Render)

### Procfile
```
web: bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}
release: bundle exec rails db:migrate
```

### Database Split in Gemfile
```ruby
group :development, :test do
  gem "sqlite3", "~> 1.4"
end

group :production do
  gem "pg"
end
```
- SQLite for development/test
- PostgreSQL for production

### Seed Data Pattern
Seed data lives in `scripts/create_data.rb`, run via:
```bash
rails runner scripts/create_data.rb
```

Seed script uses the same field-by-field bracket notation:
```ruby
Post.destroy_all

post = Post.new
post["body"] = "Delish!"
post["image"] = "https://example.com/image.jpg"
post.save
```

### Key Production Settings
- `config.active_storage.service = :local` (local disk storage)
- Database URL configured via `DATABASE_URL` env var on the hosting platform
- `release` phase in Procfile runs migrations automatically on deploy
