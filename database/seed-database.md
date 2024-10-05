# Seeding Databases with Ecto

Seeding is the process of inserting initial data into a database, typically for testing or development purposes. To seed a database using Ecto, you need to create a `.exs` file that serves as your seed script. The convention is to place this file inside the `priv/repo` directory.

> NOTE: For new Phoenix projects, a seed file is already created for you in the `priv/repo` directory.

In this file, you can write to the database using Ecto as you normally would. For example:

```elixir
MyApp.Repo.insert!(%MyApp.User{name: "John Doe"})
```

This creates an initial user in the database with the name "John Doe". All your application's models are available here.

To run the seed file, first ensure you are in a test or development environment, then execute this command:

```bash
mix run priv/repo/seed.exs # replace with the path to your seed file
```

This will run the seed file and populate the database with all the initial data the seed file inserts into it.

> NOTE: Phoenix projects come with aliases configured in the `mix.exs` file to run the seed file automatically when you run `mix ecto.setup`.

For writing operations, it's recommended to use the bang functions, such as `insert!`, `update!`, and `delete!`, because they will raise an error if something goes wrong. In a development environment, better error handling could be useful, but this is up to you.

You may also want to delete all prior data before inserting new data to ensure the state of the database before and after each seed is consistent. This can be done using the `Repo.delete_all/1` function:

```elixir
MyApp.Repo.delete_all(MyApp.User)
MyApp.Repo.insert!(%MyApp.User{name: "John Doe"})
```

This will delete all users in the database and then insert a new one, ensuring that after each seeding, the database will have a single user with the name "John Doe". In general, it's good practice to call the `Repo.delete_all/1` function for each model you have before creating new records of that model in the seed process.

Remember that the seed file is just a `.exs` file, so any type of logic you want could be included. For example, you can use `if` statements to insert data only if you're in a specific environment or use a `case` statement for error handling:

```elixir
# Only insert this data if we are in a test environment
if Mix.env() == :test do
  MyApp.Repo.insert!(%MyApp.Post{title: "Test Post"})
end

# Manage errors
case MyApp.Repo.insert(%MyApp.User{name: "John Doe"}) do
  {:ok, user} -> IO.puts("User created: #{user.name}")
  {:error, changeset} -> IO.puts("Error: #{inspect(changeset.errors)}")
end
```

You may have multiple seed files in the `priv/repo` directory and manage them as you see fit. For example, it could be useful for your project to have separate seed files for testing and development to avoid having a single file with many environment checks.

It's also possible to create modules to seed data. The benefit of this approach is the ability to seed data via IEx. The process is quite similar:

```elixir
defmodule MyApp.Seeder do
  alias MyApp.Repo
  alias MyApp.Post

  @post_amount 100

  def insert_posts do
    clear()
    Enum.each(1..@post_amount, fn number ->
      Repo.insert!(%Post{title: "Post #{number}"})
    end)
  end

  defp clear do
    Repo.delete_all(Post)
  end
end
```

Then, from IEx:

```bash
$ iex -S mix
iex(1)> MyApp.Seeder.insert_posts
```

Both approaches (seed files and seed modules) make it easy to have an initial data state for your database and, as we've seen, are very flexible to adapt to your needs.
