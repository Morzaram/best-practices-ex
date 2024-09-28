# Nesting Ecto Associations

This article assumes you are using a relational database.

Nesting Ecto Associations refers to the practice of working with nested or hierarchical data structures in Elixir's Ecto library. This often comes up when defining changesets of schemas with multiple associations or querying and inserting deeply related records. Let's break this down into each of those operations. For all the examples below, we will be using the following relationship model:

- A User has one Profile.
- A User has many Orders.
- A User has many Reviews.
- A User has many Addresses.
- A Product belongs to one Category.
- A Product has many Reviews.
- A Product has many OrderItems (meaning that a Product can be ordered multiple times).
- A Category can have one parent Category (self-referential).
- A Category can have many subcategories (self-referential).
- An Order has many OrderItems.
- An Order has one ShippingAddress.

## Changesets

When working with related schemas, we need to be able to handle associations inside their changesets. This includes casting data. For that, we use `cast_assoc/3` or `put_assoc/4` functions from `Ecto.Changeset`.

`cast_assoc/3` is used to cast and manage associations based on external data that comes from outer sources, such as function parameters or a raw CSV file. By using it, Ecto will validate the sent data against the existing data in the struct.

`put_assoc/4` is used when we have the associations as structs and the changesets in memory, in other words, when we want to manually insert the associations and the parent already exists. Ecto will use the data as is.

```elixir
defmodule MyApp.Order do
  use Ecto.Schema
  import Ecto.Changeset

  schema "orders" do
    # order fields

    belongs_to :user, MyApp.User
    has_many :order_items, MyApp.OrderItem

    timestamps()
  end

  def changeset(order, attrs) do
    order
    |> cast(attrs, [:status, :total])
    |> put_assoc(:user)
    |> cast_assoc(:order_items)
  end
end

defmodule MyApp.OrderItem do
  use Ecto.Schema
  import Ecto.Changeset
  
  schema "order_items" do
    # order_item fields

    belongs_to :order, MyApp.Order
    belongs_to :product, MyApp.Product
    
    timestamps()
  end

  def changeset(order_item, attrs) do
    order_item
    |> cast(attrs, [:quantity, :price])
    |> put_assoc(:order)
    |> put_assoc(:product)
  end
end
```

## Querying

Let's say we want to retrieve a user with their orders and addresses. We can do this by using the `Ecto.Query.preload/2` function. This is the most common and efficient way to query nested associations, although it does multiple queries to the database. This is because the final results are PARENT + CHILDREN. 

Note: The `preload/2` function comes from `Ecto.Query` and not from `Ecto.Repo`. The `Ecto.Repo.preload/2` function fetches related data too, but does not have the capabilities to make more complex queries with joins or aggregations that `Ecto.Query.preload/2` has.

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Repo
  import Ecto.Query
  
  def get_user_with_orders_and_addresses(user_id) do
    query = 
      from u in User,
      where: u.id == ^user_id,
      preload: [:orders, :addresses]
    
    Repo.one(query)
  end
end
```

In case we want to query data with more than one association, we can go through each level of nesting:

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Repo
  import Ecto.Query
  
  def get_user_orders_items(user_id) do
    query = 
      from u in User,
      where: u.id == ^user_id,
      preload: [
        orders: [:order_items],
      ]
    
    Repo.one(query)
  end
end
```

We could also use `preload/2` to query self-related data, such as categories in our example.

```elixir
defmodule MyApp.Categories do
  alias MyApp.Categories.Category
  alias MyApp.Repo
  import Ecto.Query
  
  def get_category_with_relations(category_id) do
    query =
      from c in Category,
      where: c.id == ^category_id,
      preload: [
        :parent,
        :subcategories,
      ]
    
    Repo.one(query)
  end
end
```

A good approach to query nested associations is to use `preload/2` with `assoc/3` and joins, to ensure that it results in a single query run against the database. In this pattern, we use a join using `assoc/3` to bring back the nested association and then use `preload/2` to preload the data. As this uses joins, the results will be PARENT * CHILDREN, which Ecto will take care of transforming into the proper structs with the associations.

```elixir 
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Repo
  import Ecto.Query
  
  # This will result in two queries,
  # one to bring back the user 
  # and another to bring back the orders related to that user.
  def get_user_orders(user_id) do
    query =
      from u in User,
      where: u.id == ^user_id,
      preload: [:orders]
    
    Repo.one(query)
  end

  # This will result in a single query,
  # and retrieves the same data as the previous example.
  def get_user_orders(user_id) do
    query =
      from u in User,
      where: u.id == ^user_id,
      join: o in assoc(u, :orders),
      preload: [orders: o]

    Repo.one(query)
  end
end
```

More complex queries could be accomplished by using the capabilities of `Ecto.Query`, especially joins and aggregations. Let's say we want to retrieve all the products that a user has ordered and that have not been delivered or cancelled yet. To do that, we need to get all the orders the user has placed and the order items that belong to those orders. Then, we need to get the product each order item belongs to. We can do all of this in a single database query, using the power of `Ecto.Query`:

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Accounts.Order
  alias MyApp.Accounts.OrderItem
  alias MyApp.Accounts.Product
  alias MyApp.Repo
  import Ecto.Query
  
  def get_user_products(user_id) do
    query =
      from u in User,
        where: u.id == ^user_id,
        join: o in Order, on: o.user_id == u.id,
        where: o.status == "active",
        join: oi in OrderItem, on: oi.order_id == o.id,
        join: p in Product, on: p.id == oi.product_id,
        select: p
    Repo.all(query)
  end
end
```

Another example of a complex query: this time we want to retrieve all the products that a user has ordered in their entire activity, not only the products for active orders. For each product, we want to know the categories of the product, the reviews that user has made, and the total times the user has ordered that product. Again, we accomplish this in a single database query.

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Accounts.Order
  alias MyApp.Accounts.OrderItem
  alias MyApp.Accounts.Product
  alias MyApp.Accounts.Category
  alias MyApp.Accounts.Review
  alias MyApp.Repo
  import Ecto.Query
  
  # function name shortened for simplicity
  def get_user_products(user_id) do
    from u in User,
      where: u.id == ^user_id,
      join: o in assoc(u, :orders),
      join: oi in assoc(o, :order_items),
      join: p in assoc(oi, :product),
      join: c in assoc(p, :category),
      left_join: r in Review, on: r.product_id == p.id and r.user_id == ^user_id,
      
      # Group the results by product ID, category ID, and review ID.
      # This ensures that the results are aggregated based on these unique identifiers,
      # allowing us to count the total number of orders for each combination
      # of product and category
      group_by: [p.id, c.id, r.id],
      
      select: %{
        product: p,
        category: c,
        reviews: r,
        # Count the number of orders associated with the product
        total_times_ordered: count(o.id)
      }
      
    Repo.all(query)
  end
end
```

`Ecto.Query` also has functions for any other type of joins and a lot of useful aggregations. When dealing with complex queries that retrieve deeply nested data, it is a good idea to think first about the query in SQL terms and then translate it to Ecto's query syntax.

As `Ecto.Query`'s functions are composable, we can use them along with `Repo` via the pipe operator. This allows us to extend queries in a declarative way. In particular, we could write a function like this, which takes the associations we want to preload as an argument:

```elixir	
def get(user_id, associations) do
  User 
  |> preload(associations)
  |> Repo.get(user_id)
end

User.get(id, [:orders])
```

However, this will not result in a single database query because we are not using joins, so take care of that.

## Inserting

Ecto allows us to insert new records along with their associations by passing a single map containing the data for both the parent record and associated records.

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Accounts.Profile
  alias MyApp.Repo

  def create_user(attrs) do
    %User{} 
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  def create_user_with_profile(%{
    name: "John Doe",
    profile: %{
      bio: "Software Engineer"
    }
  })
end
```

But in most scenarios, we need to break the insertion process into multiple steps to work properly on complex data creation. In those cases, it is crucial to have flexibility to manage related data accordingly. There are two main approaches to take:

1. Use changesets to build the data we want to insert and its associations. Consider the following example:

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Accounts.Profile
  alias MyApp.Repo

  def create_user_with_profile(attrs) do
    user = User.changeset(%User{}, attrs.user)
    profile = Profile.changeset(%Profile{}, attrs.profile)

    user_with_profile = Ecto.Changeset.put_assoc(user, :profile, profile)
    Repo.insert(user_with_profile)
  end
end
```

The `put_assoc/4` function from `Ecto.Changeset` creates or replaces an association within the changeset. The changeset will be validated by Ecto.

2. Run a transaction to create the parent record and then create the related record referencing the parent. This is how we would do it:

```elixir
defmodule MyApp.Accounts.Users do
  alias MyApp.Accounts.User
  alias MyApp.Accounts.Profile
  alias MyApp.Repo

  def create_user_with_profile(attrs) do
    Repo.transaction(fn ->
      user = Repo.insert!(User.changeset(%User{}, attrs.user))
      profile = Ecto.build_assoc(user, :profile, attrs.profile)
      Repo.insert!(Profile.changeset(profile))
    end)
  end
end
```

Here, the `build_assoc/3` function, also from `Ecto.Changeset`, creates a struct for an associated record and automatically sets the foreign key according to the association. Its last argument is the rest of the associated record's data.

Both approaches in combination with `put_assoc/4` and `build_assoc/3` allow us to insert deeply nested data in an efficient way. Let's take a look at a more complex, real-world example, where we want to create an order with its order items and shipping address.

```elixir
defmodule MyApp.Orders do
  alias MyApp.Order
  alias MyApp.OrderItem
  alias MyApp.ShippingAddress
  alias MyApp.Repo

  def create_order(order_attrs, order_items, shipping_address_attrs) do
    Repo.transaction(fn ->
      case Repo.insert(Order.changeset(%Order{}, Map.put(order_attrs, :status, "active"))) do
        {:ok, order} ->
          Enum.map(order_items, fn %{order_item_attrs: attrs, product: product} ->
            order_item_changeset =
              %OrderItem{}
              |> OrderItem.changeset(attrs)
              |> put_assoc(:order, order)
              |> put_assoc(:product, product)

            Repo.insert!(order_item_changeset)
          end)

          shipping_address_changeset =
            %ShippingAddress{}
            |> Ecto.build_assoc(order, :shipping_address)
            |> ShippingAddress.changeset(shipping_address_attrs)

          Repo.insert!(shipping_address_changeset)

          {:ok, %{order: order, order_items: order_items, shipping_address: shipping_address}}

        {:error, changeset} -> 
          Repo.rollback(changeset)
      end
    end)
  end
end
```
