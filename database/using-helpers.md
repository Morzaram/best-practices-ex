# Ecto Queries and Helper Functions

It is good practice to define helper functions that encapsulate common operations related to your Ecto models. This approach makes your code more reusable and easier to maintain.

Ecto queries are composable, allowing you to create helper functions that return reusable query fragments. These fragments are useful for filtering, ordering, or aggregating data. Let's explore this concept with an example from an e-commerce application.

Suppose we want to retrieve order products by their "trending score," which is calculated based on various factors. We aim to use this ordering logic in multiple queries across the entire application. We could create a helper function like this:

```elixir
defmodule MyApp.Products.Queries do
  import Ecto.Query, warn: false

  @doc """
  Orders products by a dynamic "trending" score.
  The trending score is calculated based on recent orders, ratings, and views.
  """
  def order_by_trending(query, days) do
    cut_off_date = Date.add(Date.utc_today(), -days)

    from p in query,
      left_join: o in assoc(p, :order_items),
      left_join: ord in assoc(o, :order),
      left_join: v in assoc(p, :product_views),
      where: ord.inserted_at >= ^cut_off_date or v.viewed_at >= ^cut_off_date,
      group_by: p.id,
      select_merge: %{
        trending_score: fragment(
          "SUM(CASE WHEN ? >= ? THEN 5 ELSE 0 END) + COUNT(DISTINCT ?) * 3 + COUNT(DISTINCT ?) * 1",
          p.average_rating,
          4.0,
          ord.id,
          v.id
        )
      },
      order_by: [desc: :trending_score]
  end
end
```

We can then use this function in our `MyApp.Products` module:

```elixir
defmodule MyApp.Products do
  alias MyApp.Products.{Product, Queries}
  alias MyApp.Repo
  import Ecto.Query, warn: false

  def list_by_trending(days) do
    query = from(p in Product, where: p.status == "active")

    query
    |> Queries.order_by_trending(days)
    |> Repo.all()
  end

  def is_top_10_trending?(id, days) when is_integer(id) and is_integer(days) and days > 0 do
    query = from(p in Product, where: p.status == "active")

    query
    |> Queries.order_by_trending(days)
    |> limit(10)
    |> where([p], p.id == ^id)
    |> Repo.exists?()
  end
end
```

Another example of a helper function in our app could be a query that aggregates sales data by a specified time period (day, week, month, or year). This could be used when generating reports or charts:

```elixir
defmodule MyApp.Sales.Queries do
  import Ecto.Query, warn: false

  @doc """
  Reusable query fragment to aggregate sales data by time period.
  Supports daily, weekly, monthly, and yearly aggregations.
  """
  def sales_by_period(query, period) when period in ~w(day week month year)a do
    from o in query,
      group_by: fragment("date_trunc(?, ?)", ^period, o.inserted_at),
      order_by: fragment("date_trunc(?, ?)", ^period, o.inserted_at),
      select: %{
        period: fragment("date_trunc(?, ?)", ^period, o.inserted_at),
        total_sales: sum(o.total_amount),
        order_count: count(o.id)
      }
  end
end

defmodule MyApp.Sales do
  alias MyApp.Sales.{Queries, Sale}
  alias MyApp.Repo

  def list_by_day do
    Sale
    |> Queries.sales_by_period("day")
    |> Repo.all()
  end

  def list_by_week do
    Sale
    |> Queries.sales_by_period("week")
    |> Repo.all()
  end

  # Other time periods...
end
```

By creating functions that take a query as an argument and return an extended query, you can easily reuse query fragments throughout your application. This approach enhances code readability, maintainability, and reusability.
