---
title: "Phoenix LiveView Anti Patterns"
date: 2023-02-25T13:37:13-06:00
draft: true
---

Phoenix LiveView, built on top of Elixir GenServers, represents a new paradigm for building web applications. It is extremely powerful, but there are common mistakes many people make when writing LiveView. Let's take a look at some of the most common mistakes and how to remedy them

## Do not pass the socket as an argument to functions

Never pass the socket struct to functions in LiveViews and LiveComponents. With the exception of the `socket.assigns` field, the Socket should be considered opaque. They are implementation details of LiveView that your application logic does not need to concern itself with.

What problems does this cause?
- The function does not need the socket struct to compute its business logic (Violating the "You aren't gonna need it" principle)
- The function is brittle, especially if it pattern matches on the `socket.assigns`. The function has an implicit dependency on the functions that were invoked before it, as well as the state of the `socket.assigns`.
  - You will receive a `(FunctionClauseError) no function clause matching` if you try to invoke that method when the assigns do not match
    - These errors, especially with multiple function heads, are difficult to debug - We will cover that in more detail further down

// TODO: Link to YAGNI above

In the below example, we retrieve some data and perform a business calculation. We send the socket through the pipeline to get our data. We are breaking all of the rules laid out above:

```elixir
# Avoid:
def mount(_, _, socket) do
  socket =
    socket
    |> list_departments()
    |> list_users()
    |> list_widgets()
    |> do_widget_calculations()

  {:ok, socket}
end

def list_departments(socket) do
  assign(socket, :departments, Repo.all(Department))
end

def list_widgets(socket) do
  assign(socket, :widgets, Repo.all(Widget))
end

def list_users(socket) do
  assign(socket, :users, Repo.all(User))
end

# The method does not actually need the socket. It needs the socket.assigns
# This method will fail if it's called too early in the pipeline in the `mount/3` callback
def do_widget_calculations(%Socket{} = _socket) do
  %{widgets: widgets, users: users, departments: departments} = assigns
  # Apply business logic
  assign(socket, :business_result, business_result)
end
```

### What do I do instead?
Perform all business logic before assigning the results to the socket.
Store the results in intermediate variables if one method requires the result of another.

```elixir
def mount(_, _, socket) do
  departments = Repo.all(Department)
  users = Repo.all(User)
  widgets = Repo.all(Widget)
  business_result = do_widget_calculations(departments, users, widgets)
  socket =
    socket
    |> assign(:departments, departments)
    |> assign(:users, users)
    |> assign(:widgets, widgets)
    |> assign(:business_result, business_result)
  
  {:ok, socket}
end

# Function now receives only dependencies that are required to calculate the business logic
def do_widget_calculations(departments, users, widgets) do
  # Calculate and return business logic
end
```

Even better, `assign/2` accepts a map (or keyword list). Instead of a long pipeline with individual assigns, we can stuff everything into a map and pass that to `assign/2`.

Now our LiveView looks like this:

```elixir
def mount(_, _, socket) do
  {:ok, assign(socket, do_widget_calculations())}
end

def do_widget_calculations do
  departments = Repo.all(Department)
  users = Repo.all(User)
  widgets = Repo.all(Widget)
  business_result = MyApp.do_widget_calculations(departments, users, widgets)
  %{departments: departments, users: users, widgets: widgets, business_result: business_result}
end
```

#### Note
There are LiveView methods that annotate the socket (e.g. [push_navigate/2](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#push_navigate/2).
You might be tempted to pass the socket inside of your business logic function to determine, e.g., where to send redirects (say, after a user's first successful sign-in). Instead of passing the socket, calculate the options inside of your business logic and pass them to the function that operates on the socket. For instance:

```elixir
def handle_event("sign-in", params, socket) do
  %{to: url, replace: replace?} = business_logic_function(params)
  {:noreply, push_navigate(socket, to: url, replace: replace?)}
end
```

As a rule of thumb: Phoenix LiveView can operate on the socket, your application should not

## Function Head Pattern Matching Abuse

Elixir's pattern matching is extremely expressive and powerful. With great power comes great responsibility.

Pattern matching should be used as control flow - not an unwrap every value to be used party
Binding every variable in a `params` or `socket.assigns` on LiveView or LiveComponent callbacks is messy and obscures intent
Further, if a function head fails to bind, you will get a cryptic "No Function Clause Matching" error and you get to play "dig through the stacktrace" and painstakingly reverse engineer not only which function head should have been taken, but which pattern-matched value was missing

Imagine you have a new user form with several inputs. This form is fairly simple, but there are many different code paths and side effects to do - because you have many different ways your user could authenticate. Username/password, OAuth, Google, Facebook, Sign in with Apple, etc.

```html
<.form for={@form} phx-change="validate" phx-submit="create-user">
  <.input type="text" field={@form[:username]}/>
  <.input type="password" field={@form[:password]}/>
  <.input type="phone" field={@form[:phone]}/>
  <.input type="select" field={@form[:authorization_options]}/>
  <.input type="checkbox" field={@form[:likes_happy_hour]}/>
  <.input type="select" field={@form[:permission_level]}/>
  <.input type="checkbox" field={@form[:is_admin]}/>
  <.input type="text" field={@form[:oauth_api_key]}/>
</.form>
```

In this example, there are slightly different validation rules based on which authorization type people are using to sign in.

You like to keep your method bodies slim, and you want to unwrap the user's attributes in the function head when validating their attributes:

Can you tell which function head will be taken to sign in with facebook?

```elixir
def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "authorization_options" => "yubikey", "password" => password, "phone" => phone} = params, socket) do
  # validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "authorization_options" => "saml", "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone, "authorization_options" => "apple"} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"permission_level" => permission_level, "username" => username, "authorization_options" => "facebook", "is_admin" => is_admin, "phone" => phone, "password" => password} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"permission_level" => permission_level, "authorization_options" => "google", "is_admin" => is_admin, "username" => username, "password" => password, "phone" => phone, "likes_happy_hour" => likes_happy_hour} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "authorization_options" => "oauth", "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"authorization_options" => "username_password", "is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end
```

The above example violates the principle of "pattern matching as flow control". 

### What to do Instead

Only pattern-match what is necessary to determine which code path to take.
After that, pattern-match within the function body
In this way, you:
- Clearly communicate intent on the code path to be taken 
- Receive improved error messages
  - Raises a `(MatchError)` instead of a `(FunctionClauseError) no function clause matching`. Now you know exactly which function head failed to match, because you get an accurate line number in the stacktrace

Avoid "function head soup" and only pattern match on what you need to determine which code path to take. Our re-worked example is now:

```elixir
def handle_event("validate", %{"authorization_options" => "yubikey"} = params, socket) do
  %{
    "permission_level" => permission_level,
    "authorization_options" => "google",
    "is_admin" => is_admin,
    "username" => username,
    "password" => password,
    "phone" => phone,
    "likes_happy_hour" => likes_happy_hour
  } = params
end

def handle_event("validate", %{"authorization_options" => "saml"} = params, socket) do
  %{
    "permission_level" => permission_level,
    "authorization_options" => "google",
    "is_admin" => is_admin,
    "username" => username,
    "password" => password,
    "phone" => phone,
    "likes_happy_hour" => likes_happy_hour
  } = params
end

def handle_event("validate", %{"authorization_options" => "apple"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_options" => "facebook"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_options" => "google"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_options" => "oauth"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_options" => "username_password"} = params, socket) do
  # Unwrap params, validate, and return
end
```

You can now quickly scan the function heads and you know exactly which one to jump into - your function heads and teammates will thank you.

## You Have N + 1 Queries In Your LiveComponents

Consider a LiveComponent that displays information (and allows you to change) a user - It accepts the user as an assign
You'd like to encapsulate not have the LiveView responsible for user changesets and handling form actions, so you make that the responsibility of the LiveComponent.
Your component looks something like this:

```elixir
# UsersLive.ex
<.live_component :for={user <- @users} id={"user-detail-#{user.id}"} module={UserDetailComponent} user={user} />

# UserDetailComponent.ex
def update(assigns, socket) do
  %{user: user} = assigns

  # List departments the user belongs to
  departments =
    Department
    |> where([d], d.id in ^user.department_ids)
    |> Repo.all()

  socket =
    socket
    |> assign(:user, user)
    |> assign(:departments, departments)

  {:ok, socket}
end
```

Did you spot the problem? It's subtle, and it's killing your app's performance

The `Repo` operation is an N + 1 Query (Normally that logic would be wrapped in a Context. It's illustrative)

Every User rendered on the page issues a DB query. 10 users? That's 10 queries, plus 1 for the LiveView to get all Users (`:for={user <- @users}`) 25 users? That's 26 queries. You get the picture.

### What to do Instead

// TODO: Link to preload/1
The preload/1 callback was designed exactly for this purpose.

Preload takes as its only argument a list of each components assigns. It returns an updated list of assigns which the component receives in its update/2 callback

From our above example, `UserDetailComponent` should now look like this:

```elixir
def preload(list_of_assigns) do
  user_ids = Enum.map(list_of_assigns, & &1.user.id)
  departments = Repo.all(Department)
  Enum.map(list_of_assigns, fn assigns ->
    %{user: user} = assigns
    Map.put(assigns, :departments, Enum.map(departments, & &1.id in user.department_ids))
  end)
end

def update(assigns, socket) do
  # The user's departments are available in the assigns!
  %{user: user, departments: departments} = assigns
  {:noreply, assign(socket, user: user, departments: departments)}
end
```

## Indexing: It's not just for databases
// TODO: Put a note about use the index, Luke
// TODO: Put a link about The Art of PostgreSQL

Imagine you're working on BusinessAppWeb and you want to add a button that alters a boolean field in an widget. There are lots of widgets, and each one has its own button. These widgets are stored server-side in a list, `@widgets`. Your users love these buttons, and submit hundreds of click events every second.

Working with lists underpins the heart and soul of Elixir. Inevitably, these lists are going to get large. You undoubtedly have code like this eating up your CPU cycles:

A fairly common scenario you will find in Elixir is retrieving an element from a list of elements based on some attribute (say a database ID).

That's pretty easy to take care of:

```elixir
def handle_event(socket, params) do
  %{"user-id" => user_id} = params
  user = Enum.find(users, & &1.id == user_id)
end
```

Did you spot the problem?

Every `Enum.find` is an `O(n)` operation -- When your lists get large (and they will), you'll be burning CPU cycles for nothing.
The lack of an explicit loop construct causes people, especially those new to Elixir, to accidentally write `O(n^2)` (or even `O(n^3)`!) algorithms.

For instance, say we want to build a map of our users and who referred them to our decentralized blockchain NFT mastodon instance metaverse trading platform algorithm:

```elixir
Enum.reduce(users, %{}, fn user, acc ->
  Map.put(acc, user, Enum.find(users, & &1.id == user.referred_user_id))
end)
```

We have now unwittingly created an `O(n^2)` algorithm in just 3 lines. Thankfully, because your social media platform is decentralized, only your instance is suffering awful performance. The rest are humming along just fine. The resiliency of decentralization has been proven and you may rest easy knowing that you've accomplished your mission.


### What to do instead

Enter `index_by`, 5 lines of code that will change your life. It is the missing addition to the Elixir `Enum` module. Think of it like `Enum.group_by` where the key points to a single value instead of a list.

```elixir
def index_by(collection, key_fun, value_fun \\ fn v -> v end) do
  collection
  |> Enum.map(&{key_fun.(&1), value_fun.(&1)})
  |> Enum.into(%())
end
```

It is extraordinarily helpful when working with any collection that has a unique attribute (e.g. Ecto schemas with a primary key):

```elixir
users_by_id = index_by(& &1.id)
```
// TODO: Reference time complexities of Elixir maps

This small but simple change makes lets us use all of the `Map` operations (and its `O(n log n)`) performance when accessing elements in a collection.

Consider our re-written examples from above and a couple of others:

```elixir
# Find a user given an ID:
# Was O(n), is now O(n log n)
user = users_by_id[user_id]

# Create a mapping for our DBNFTNMI (Decentralized blockchain NFT mastadon metaverse instance):
# Was O(n^2), is now O(n)
Enum.reduce(users, %{}, fn user -> Map.put(acc, :user, users[user.referred_user_id]) end)

# Retrieve a list of users given a list of IDs:
# O(n log n) versus Enum.filter - O(n)
user_ids = [1, 2, 5]
Map.take(users_by_id, user_ids)

# I want my list of users back:
Map.values(users_by_id)

# I want all user IDs:
Map.keys(users_by_id)

# Filter out users
non_admin_user_ids = [10, 11, 12]
Map.drop(non_admin_users)
```

`index_by/2` is ridiculously helpful. I am willing to wager there are some optimizations you can make in your codebase with `index_by/2` at your side.


## Conclusion
