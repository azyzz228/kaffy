# Kaffy

Extremely simple yet powerful admin interface for phoenix applications

## Installation

#### Add `kaffy` as a dependency
```elixir
def deps do
  [
    {:kaffy, "~> 0.2.0"}
  ]
end
```

#### These are the minimum configurations required

```elixir
# in your router.ex
use Kaffy.Routes, scope: "/admin", pipe_through: [:some_plug, :authenticate]
# :scope defaults to "/admin"
# :pipe_through defaults to kaffy's [:kaffy_browser]

# in your endpoint.ex
plug Plug.Static,
  at: "/kaffy",
  from: :kaffy,
  gzip: false,
  only: ~w(css img js scss vendor)

# in your config/config.exs
config :kaffy,
  otp_app: :my_app,
  ecto_repo: Bloggy.Repo,
  router: BloggyWeb.Router
```


## What You Get

![Post list page](demos/post_index.png)

## Customizations

If you don't specify a `resources` option in your configs, Kaffy will try to auto-detect your schemas and your admin modules. Admin modules should be in the same namespace as their respective schemas. For exmaple, if you have a schema `MyApp.Products.Product`, its admin module should be `MyApp.Products.ProductAdmin`.

If you'd like to explicitly specify your schemas and their admin modules, you can do like the following:

```elixir
# config.exs
config :kaffy,
  admin_title: "My Awesome App",
  ecto_repo: MyApp.Repo,
  router: MyAppWeb.Router,
  resources: [
    blog: [
      name: "My Blog", # a custom name for this context/section.
      schemas: [
        post: [schema: MyApp.Blog.Post, admin: MyApp.Blog.PostAdmin],
        comment: [schema: MyApp.Blog.Comment],
        tag: [schema: MyApp.Blog.Tag]
      ]
    ],
    inventory: [
      name: "Inventory",
      schemas: [
        category: [schema: MyApp.Products.Category, admin: MyApp.Products.CategoryAdmin],
        product: [schema: MyApp.Products.Product, admin: MyApp.Products.ProductAdmin]
      ]
    ]
  ]
```

The following admin module is what the screenshot above is showing:

### Customizing the index page

The `index/1` function takes a schema and must return a keyword list of fields and their options.

If the options are `nil`, Kaffy will use default values for that field.

If this function is not defined, Kaffy will return all fields with their respective values.

```elixir
defmodule MyApp.Blog.PostAdmin do
  def index(_) do
    [
      title: nil,
      views: %{name: "Hits"},
      date: %{name: "Date Added", value: fn p -> p.inserted_at end},
    ]
  end
end
```

Result

![Customized index page](demos/post_index_custom.png)

Note that the keyword list keys don't necessarily have to be schema fields as long as you provide a `:value` option.


```elixir
# all the functions are optional

defmodule MyApp.Blog.PostAdmin do
  def index(_schema) do
    # index/1 should return a keyword list of fields and
    # their options.
    # Supported options are :name and :value.
    # Both options can be a string or an anonymous function.
    # If a function is provided, the current entry is passed to it.
    # If this function is not defined, Kaffy will return all the fields of the schema and their default values
    [
      id: %{name: "ID", value: fn post -> post.id + 100 end},
      title: nil, # this will render the default name for this field (Title) and its default value (post.title)
      views: %{name: "Hits", value: fn post -> "<strong>#{post.views}</strong>" end},
      published: %{name: "Published?", value: fn post -> published?(post) end},
      comment_count: %{name: "Comments", value: fn post -> comment_count(post) end}
    ]
  end

  def form_fields(_schema) do
    # Supported options are:
    # :label, :type, :choices, :permission
    # :type can be any ecto type in addition to :file and :textarea
    # If :choices is provided, it must be a keyword list and
    # the field will be rendered as a <select> element regardless of the actual field type.
    # Setting :permission to :read will make the field non-editable. It is :write by default.
    # If you want to remove a field from being rendered, just remove it from the list.
    # If this function is not defined, Kaffy will return all the fields with
    # their default types based on the schema.
    [
      title: %{label: "Subject"},
      slug: nil,
      image: %{type: :file},
      status: %{choices: [{"Pending", "pending"}, {"Published", "published"}]},
      body: %{type: :textarea, rows: 3},
      views: %{permission: :read}
    ]
  end

  def search_fields(_schema) do
    # Must return a list of :string fields to search against when typing in the search box.
    # If this function is not defined, Kaffy will return all the :string fields of the schema.
    [:title, :slug, :body]
  end

  def ordering(_schema) do
    # This returns how the entries should be ordered
    # if this function is not defined, Kaffy will return [desc: :id]
    [desc: :id]
  end

  def authorized?(_schema, _conn) do
    # authorized? is passed the schema and the Plug.Conn struct and
    # should return a boolean value.
    # returning false will prevent the access of this resource for the current user/request
    # if this function is not defined, Kaffy will return true.
    true
  end

  def create_changeset(schema, attrs) do
    # this function should return a changeset for creating a new record
    # if this function is not defined, Kaffy will try to call:
    # schema.changeset/2
    # and if that's not defined, Ecto.Changeset.change/2 will be called.
    Bloggy.Blog.Post.create_changeset(schema, attrs)
  end

  def update_changeset(entry, attrs) do
    # this function should return a changeset for updating an existing record.
    # if this function is not defined, Kaffy will try to call:
    # schema.changeset/2
    # and if that's not defined, Ecto.Changeset.change/2 will be called.
    Bloggy.Blog.Post.update_changeset(entry, attrs)
  end

  def singular_name(_schema) do
    # if this function is not defined, Kaffy will use the name of
    # the last part of the schema module (e.g. Post)
    "Post"
  end

  def plural_name(_schema) do
    # if this function is not defined, Kaffy will use the singular
    # name and add a "s" to it (e.g. Posts)
    "Posts"
  end

  def published?(post) do
    if post.status == "published",
      do: ~s(<span class="badge badge-success"><i class="fas fa-check"></i>),
      else: ~s(<span class="badge badge-light"><i class="fas fa-times"></i></span>)
  end

  defp comment_count(post) do
    post = Barbican.Repo.preload(post, :comments)
    length(post.comments)
  end
end
```

## Schema Form

![Post change page](demos/post_form.png)

The form is constructed from the `form_fields/1` function if it exists in the admin module.
Notice that even though the `status` field is of type `:string`, it is rendered as a `select` element.
Also notice that the `views` field is in "readonly" mode since we gave it the `:read` permission.


## Why another admin interface

Kaffy was created out of a need to have a minimum, flexible, and customizable admin interface 
without the need to touch the current codebase. It should work out of the box just by adding some
configs in your `config.exs` file (with the exception of adding a one liner to your `router.ex` file).

A few points that encouraged the creation of Kaffy:

- Taking contexts into account.
  - Supporting contexts makes the admin interface better organized.
- Can handle as many schemas as necessary.
  - Whether we have 1 schema or 1000 schemas, the admin interface should adapt well.
- Have a visually pleasant user interface.
  - This might be subjective.
- No generators or generated templates.
  - I believe the less files there are the better. This also means it's easier to upgrade for users when releasing new versions. This might mean some flexibility and customizations will be lost, but it's a trade-off.
- Existing schemas/contexts shouldn't have to be modified.
  - I shouldn't have to change my code in order to adapt to the package, the package should adapt to my code.
- Should be easy to use whether with a new project or with existing projects with a lot of schemas.
  - Adding kaffy should be as easy for existing projects as it is for new ones.
- Highly flexible and customizable.
  - Provide as many configurable options as possible.
- As few dependencies as possible.
  - Currently kaffy only depends on phoenix and ecto.
- Simple authorization.
  - I need to limit access for some admins to some schemas.
- Minimum assumptions.
  - Need to modify a schema's primary key? Need to hide a certain field? No problem.


