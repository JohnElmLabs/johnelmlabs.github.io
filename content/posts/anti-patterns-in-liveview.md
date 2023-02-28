---
title: "Phoenix LiveView Anti Patterns"
date: 2023-02-25T13:37:13-06:00
draft: true
---

Phoenix LiveView enables rapid development of interactive web apps. It's extremely powerful and an absolute pleasure to write every day. Below are some of the most common mistakes people make when writing LiveView applications and their solutions.

In short:

- Don't pass the socket as an argument to functions
- Function heads should pattern match only on what is necessary to determine code control flow
- Index lists for improved performance
- Preload LiveComponent assigns to avoid N + 1 queries

## Don't pass the socket as an argument to functions

Much in the same way HTTP request objects are not passed to business logic functions, never pass the socket struct to functions in LiveViews and LiveComponents. LiveView is built on top of Elixir GenServers and GenServers blur the line between client and server partially due to the fact that most, if not all, of the code lives in the same file. The classic `View -> Controller -> Context` separation of "regular" Phoenix apps is not available to us in LiveView. It can be difficult to know where to draw the boundary line. With the exception of the `socket.assigns`, the socket should be considered opaque. They are implementation details of LiveView that application logic does not need to concern itself with.

### Why is this a problem?
- Functions only needs the socket's assigns or a subset of them. Passing the entire socket violates the [You aren't gonna need it](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) principle
- Passing the socket as an argument violates proper separation of concerns. Live callbacks should be the only places the application calls LiveView methods that operate on the socket such as `assign/2`, `assign/3`, `push_navigate/2`, etc. All other functions should not accept a `socket` argument.
- Functions that take sockets as arguments are brittle, especially if they pattern matches on the `socket.assigns`. The function has an implicit dependency on the functions that were invoked before it, the state of the `socket.assigns`, and LiveView itself.
  - If the function is invoked out of order or the state of the socket is slightly off, a `(FunctionClauseError) no function clause matching` will be raised
    - A `FunctionClauseError`, especially with multiple function heads, is a headache to debug - That will be covered in more detail further on

In short:
- LiveView callbacks should handle the actual assignment of values to the socket (`assign/2`, `assign/3`)
- Application logic should calculate the assigns and returns them to the lifecycle callbacks. Business logic functions should not take the socket as one of its arguments.

### Example

In the code below, the socket is piped through some functions to retrieve data and assign it before being rendered:

```elixir
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

# This function does not actually need the socket. It needs the socket.assigns
# This function will fail if it's called before any other function in the pipeline in the `mount/3` callback
def do_widget_calculations(%Socket{} = socket) do
  %{widgets: widgets, users: users, departments: departments} = socket.assigns
  business_result = MyApp.get_business_result(widgets, users, departments)
  assign(socket, :business_result, business_result)
end
```

### What To Do Instead
Perform all data retrieval & business logic before assigning the results to the socket.
Store the results in intermediate variables if one function requires the result of another.

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
  MyApp.do_widget_calculations(departments, users, widgets)
end
```

Even better, `assign/2` accepts a map (or keyword list). Instead of a long pipeline of individual assigns, everything can be put into a map:

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

Now the LiveView is terse, expressive, and the responsibilities are in the right place -- The LiveView callbacks assigns data to the socket, and data retrieval and business logic is delegated to functions which receive only what they need to perform their duties.

#### Note
There are LiveView functions that annotate the socket for flash messages and redirects (e.g. [put_flash/3](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#put_flash/3) and [push_navigate/2](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#push_navigate/2)).
It's tempting to pass the socket to a function to determine, e.g., where to send redirects (say, after a user's first successful sign-in). Instead of passing the socket, determine the options in a business logic function and return them to LiveView callback:

```elixir
def handle_event("sign-in", params, socket) do
  %{to: redirect_url, replace: replace?} = business_logic_function(params)
  {:noreply, push_navigate(socket, to: redirect_url, replace: replace?)}
end
```

## Function Head Pattern Matching Abuse

Binding every variable in a `params` or `socket.assigns` on LiveView or LiveComponent callbacks is messy, difficult to read, and obscures the callback's intent.
Pattern matching in function heads should be used as control flow: Match on only the values needed to determine which function head to take. Function heads should not unwrap every value the function will use.

### Why is this a problem?
If a function head fails to match on the passed arguments, a cryptic `(FunctionClauseError) no function clause matching` error is raised. With more than one function head (common with LiveComponent's `update/2` callback), the stacktrace will always point to the first callback/function implementation - which is not necessarily the function head that failed to match. Determining where the actual problem lies requires reverse engineering not only which function head should have been taken, but which value was missing and/or incorrect. In LiveViews and LiveComponents with large assigns maps, this can be difficult to spot.

### Example

Imagine a new user form with several inputs. The form is simple, but different code paths must be taken depending on which authorization provider the user chooses: Username/password, Sign in with Github, Google, Facebook, Sign in with Apple, etc.

```html
<.form for={@form} phx-change="validate" phx-submit="create-user">
  <.input type="text" field={@form[:username]}/>
  <.input type="password" field={@form[:password]}/>
  <.input type="select" field={@form[:authorization_provider]}/>
  <.input type="select" field={@form[:permission_level]}/>
  <.input type="phone" field={@form[:phone]}/>
  <.input type="checkbox" field={@form[:likes_happy_hour]}/>
  <.input type="checkbox" field={@form[:is_admin]}/>
  <.input type="text" field={@form[:oauth_api_key]}/>
</.form>
```

To keep function bodies slim, all arguments are unwrapped in the function head. Which function head will be taken if "Sign in with Facebook" is the chosen authorization provider? It's not easy to tell in the function head soup:

```elixir
def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "authorization_provider" => "yubikey", "password" => password, "phone" => phone} = params, socket) do
  # validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "authorization_provider" => "saml", "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone, "authorization_provider" => "apple"} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"permission_level" => permission_level, "username" => username, "authorization_provider" => "facebook", "is_admin" => is_admin, "phone" => phone, "password" => password} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"permission_level" => permission_level, "authorization_provider" => "google", "is_admin" => is_admin, "username" => username, "password" => password, "phone" => phone, "likes_happy_hour" => likes_happy_hour} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"is_admin" => is_admin, "authorization_provider" => "oauth", "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end

def handle_event("validate", %{"authorization_provider" => "username_password", "is_admin" => is_admin, "permission_level" => permission_level, "username" => username, "password" => password, "phone" => phone} = params, socket) do
  # Validate and return
end
```

### What to do Instead

Only pattern match on what is absolutely necessary to determine which code path to take.
After that, (optionally) pattern match on the values needed within the function body.
In this way, the function:
- Clearly communicates why a code path will be taken 
- Has vastly improved readability
- Receives improved error messages
  - Raises a `(MatchError)` instead of a `(FunctionClauseError)`
  - The line number in the stacktrace is accurate
    - `MatchError` points to exactly the line where the match failed. `FunctionClauseError` always points to the first function/callback's line number
  - It's easier to figure out what assign is missing (Drop an `IO.inspect` on the assigns before the pattern match in the function body)

Re-writing the above example to follow the rules laid out above, how much easier is it now to spot the Facebook code path?

```elixir
def handle_event("validate", %{"authorization_provider" => "yubikey"} = params, socket) do
  %{
    "permission_level" => permission_level,
    "authorization_provider" => "google",
    "is_admin" => is_admin,
    "username" => username,
    "password" => password,
    "phone" => phone,
    "likes_happy_hour" => likes_happy_hour
  } = params
end

def handle_event("validate", %{"authorization_provider" => "saml"} = params, socket) do
  %{
    "permission_level" => permission_level,
    "authorization_provider" => "google",
    "is_admin" => is_admin,
    "username" => username,
    "password" => password,
    "phone" => phone,
    "likes_happy_hour" => likes_happy_hour
  } = params
end

def handle_event("validate", %{"authorization_provider" => "apple"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_provider" => "facebook"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_provider" => "google"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_provider" => "oauth"} = params, socket) do
  # Unwrap params, validate, and return
end

def handle_event("validate", %{"authorization_provider" => "username_password"} = params, socket) do
  # Unwrap params, validate, and return
end
```

The function heads can be scanned quickly and easily - saving time and frustration.

## Indexing: Not Just For the Database

Working with lists underpins functional programming and Elixir is no different. Inevitably, these lists are going to get large. The `List` & `Enum` modules have several methods for working with lists including, but not limited to:

* `Enum.find/3`
* `Enum.find_index/3`
* `Enum.find_value/3`
* `Enum.at/3`
* `Enum.fetch/2`
* `List.pop_at/3`
* `List.update_at/3`

### Why is this a problem?

Lists, unlike arrays, are not stored contiguously in memory. As such, they must be iterated on in order to retrieve values. This means performance degrades linearly as the list grows due to the `O(n)` time complexity.

Consider a scenario where a department is retrieved from a list of departments when a user selects a department from a dropdown. When a user selects a department from the dropdown, the `handle_event/3` callback receives its ID.

That's pretty easy to take care of, and in a dev environment with a relatively low number of departments, `Enum.find/3` does the job well:

```elixir
def handle_event("select-department", params, socket) do
  %{"department-id" => department_id} = params
  %{departments: departments} = socket.assigns
  department = Enum.find(departments, & &1.id == department_id)

  {:noreply, assign(socket, :selected_department, department)}
end
```

The problem is performance is going to suffer in a production environment when customers who have hundreds, or even thousands, of departments (or any list item) are using the application.

This is particularly problematic because the lack of an explicit C-style loop construct makes it easier to incidentally write inefficient "loops" in Elixir, especially for those new to the language.

### Example

To show a basic hierarchy of departments, one could build a map of `%{department => parent_department}`. A first pass might look like:

```elixir
Enum.reduce(departments, %{}, fn department, acc ->
  Map.put(acc, department, Enum.find(departments, & &1.id == department.parent_id))
end)
```

The above has a time complexity of `O(n^2)` algorithm in just 3 lines. The entire list of departments must be iterated in `Enum.reduce` and then again, for each department, in `Enum.find`. This code can hide deep in a LiveView or LiveComponent callstack and degrade application performance significantly.

### What to do instead

Enter `index_by/3`, the missing addition to the Elixir `Enum` module. Think of it like `Enum.group_by` where the key points to a single value instead of a list.

```elixir
def index_by(collection, key_fun, value_fun \\ fn v -> v end) do
  collection
  |> Enum.map(&{key_fun.(&1), value_fun.(&1)})
  |> Enum.into(%{})
end
```

It is extraordinarily helpful (and performant) when working with any collection that has a unique attribute (e.g. Ecto schemas with a primary key):

```elixir
users = [%User{}, %User{}, %User{}]
users_by_id = index_by(users, & &1.id)
# %{
#   1 => %User{id: 1},
#   2 => %User{id: 2},
#   3 => %User{id: 3}
# }
```

This small but extremely powerful change makes enables the use of the functions in the `Map` module (and its `O(n log n)`) performance when accessing elements in a collection.

Continuing on from the department example above, the `update/2` callback of the LiveComponent can build the indexed list and assign it to the socket. Any and all callbacks which would otherwise need to traverse the list repeatedly can now have near constant-time access to any member of the list provided it has department IDs.

The implementation might now look something like:

```elixir
def update(assigns, socket) do
  %{departments: departments} = assigns
  departments_by_id = index_by(departments, & &1.id)
  {:ok, assign(socket, :departments_by_id, departments_by_id)}
end

def handle_event("build-hierarchy", params, socket) do
  %{"department_id" => department_id} = params
  %{departments_by_id: departments_by_id} = socket.assigns
  Enum.reduce(departments_by_id, %{}, fn {_dept_id, department}, acc ->
    Map.put(acc, department, departments_by_id[department.parent_id])
  end)
end
```

`index_by/3` is useful in a whole host of situations. Consider a couple of other examples:

```elixir
# Index a list of users
users_by_id = index_by(users, & &1.id)

# Find a user given an ID:
# Was O(n), is now O(n log n)
user = users_by_id[user_id]

# Retrieve a list of users given a list of IDs:
# Was O(n), is now O(n log n) (when compared to Enum.filter)
user_ids = [1, 2, 5]
Map.take(users_by_id, user_ids)

# Get the list of users back
Map.values(users_by_id)

# Get list of all user IDs
Map.keys(users_by_id)

# Filter out users
non_admin_user_ids = [10, 11, 12]
Map.drop(non_admin_users)
```

In situations where a large list does not have a unique property to index on, `Enum.group_by/3` can be used in its place.

Incorporating `index_by/3` into a codebase is one of the quickest and simplest ways to get vastly improved performance - especially where large lists are involved.

## You Have N + 1 Queries In Your LiveComponents

This is probably the easiest and most detrimental performance trap to fall into. As soon as a LiveComponent that performs database access is rendered more than once, it becomes an N + 1 query problem. Eliminating N + 1 queries is one of the best ways to improve poor web app performance.

### Example

Consider a LiveComponent that displays information about a user with a form to make changes to said user.

The responsibility of individual user changesets and their forms is encapsulated in a `LiveComponent` - so far so good.

The LiveView and LiveComponent look something like this:

```elixir
# UsersLive.ex
<.live_component
  :for={user <- @users}
  module={UserDetailComponent}
  id={"user-detail-#{user.id}"}
  user={user}
/>

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

Because the LiveComponent is rendered as part of a list comprehension (`for={user <- @users}`), the `Repo.all` operation is an N + 1 query. Every `UserDetailComponent` rendered issues a DB query. Showing 10 users issues 10 queries, plus 1 for the LiveView to retrieve all Users. 25 users issues 26 queries. 50 users issues 51 queries, and so on.

### What to do Instead

The [preload/1](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveComponent.html#c:preload/1) callback was designed to solve exactly this problem.

Preload takes, as its only argument, a list of each component's assigns. It returns an updated list of assigns (in the same order in which it was received) which the component receives in its `update/2` callback

With `preload/1` implemented, `UserDetailComponent` now looks like:

```elixir
def preload(list_of_assigns) do
  departments = Repo.all(Department)
  departments_by_id = index_by(departments, & &1.id)

  Enum.map(list_of_assigns, fn assigns ->
    %{user: %{department_ids: department_ids}} = assigns
    Map.put(assigns, :departments, Map.take(departments_by_id, department_ids))
  end)
end

def update(assigns, socket) do
  # The user's departments are available in assigns from the preload
  %{user: user, departments: departments} = assigns
  {:noreply, assign(socket, user: user, departments: departments)}
end
```

The N + 1 problem is eliminated with `preload/1`. Now, no matter how many `UserDetailComponent`s are rendered on the page, only two queries are issued (One for all users and one for all departments). Notice, too, that `index_by/3` is used in order to avoid iterating over the list of departments for each user.


## Conclusion

