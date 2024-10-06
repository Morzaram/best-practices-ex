Custom model validators are used to validate data in your models in ways not provided by Ecto's built-in validation functions. To create a custom model validator with Ecto, you need to write a function that takes the changeset as its first argument and may take additional arguments. These extra arguments are typically an atom or a list of atoms representing the field(s) to be validated. Then, add it to the model's validation pipeline.

```elixir
defmodule MyApp.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :name, :string
    field :email, :string
    field :password, :string
    field :age, :integer

    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :password, :age])
    |> validate_required([:name, :email, :password, :age])
    |> validate_length(:name, min: 3)
    |> validate_format(:email, ~r/@/)
    |> validate_min_age(:age, min_age: 18)
  end

  # Custom model validator to check if the user meets the minimum age requirement
  defp validate_min_age(changeset, age_field, opts) when is_atom(age_field) do
    with {:is_valid, true} <- {:is_valid, changeset.valid?},
         {:age, age} <- {:age, get_field(changeset, age_field)},
         {:min_age_valid, true} <- {:min_age_valid, age >= opts[:min_age]} do
      changeset
    else 
      {:is_valid, false} ->
        changeset
      
      {:min_age_valid, false} -> 
        add_error(changeset, :age, "must be at least #{opts[:min_age]} years old to register.")
    end
  end
end
```

A custom model validator should return the changeset in order to work properly on the validation pipeline.

If any of the other preciding validations fail, the custom validator will recieve a changeset with errors, and it should be able to handle it. This is why we call `changeset.valid?/0` in order to check if the changeset has errors or not before proceeding with the validations. If the changeset alredy has errors, we don't need to add perform any validation, just return it. Otherwise, we perform the validations and return the changeset with the errors we found if any.

In the above example, the function `get_field/3` will raise an error if the `age_field` is not present in the changeset. To avoid this, we can use the Ecto `validate_change/3` function. This takes the changeset, the field to be validated, and a function (referred to as the `validator`) as arguments. The `validator` is called only if the field is present in the changeset and is not `nil`. The `validator` must return a list of errors (each error would be appended to the changeset and must be a tuple with the field name and the error message). An empty list means the validation passed and no errors were found.

Here's a rewritten version of our example using `validate_change/3`:

```elixir
defmodule MyApp.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :name, :string
    field :email, :string
    field :password, :string
    field :age, :integer

    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :email, :password, :age])
    |> validate_required([:name, :email, :password, :age])
    |> validate_length(:name, min: 3)
    |> validate_format(:email, ~r/@/)
    |> validate_min_age(:age, min_age: 18)
  end

  # Custom model validator to check if the user meets the minimum age requirement
  defp validate_min_age(changeset, age_field, opts) when is_atom(age_field) do
    validate_change(changeset, age_field, fn field, value -> 
      case value >= opts[:min_age] do 
        true -> 
          []
        
        false -> 
          [{field, "must be at least #{opts[:min_age]} years old to register."}]
      end
    end)
  end
end
```


Another way to perform custom validations is to use the `Ecto.Changeset.prepare_changes/2` function. This takes a changeset and a function as arguments, then runs the provided function before the changes are emitted to the repository. The purpose of this is to perform adjustments on the changeset before insert/update/delete operations. The function passed to `prepare_changes/2` should return the changeset.

The following example from the [Ecto documentation](https://hexdocs.pm/ecto/Ecto.Changeset.html#prepare_changes/2) shows how to use `prepare_changes/2` to update a counter cache (a post's comments count) when a comment is created:

```elixir
def create_comment(comment, params) do
  comment
  |> cast(params, [:body, :post_id])
  |> prepare_changes(fn changeset ->
       if post_id = get_change(changeset, :post_id) do
         query = from Post, where: [id: ^post_id]
         changeset.repo.update_all(query, inc: [comment_count: 1])
       end
       changeset
     end)
end
```

> We retrieve the repo from the comment changeset itself and use update_all to update the 
> counter cache in one query. Finally, the original changeset must be returned.

Ideally, validations should not rely on database interactions or validate against the data in the database. Validations that depend on the database are "inherently unsafe" according to the Ecto documentation. This is why Ecto's built-in validations that needs a database to be executed are prefixed with `unsafe_`.

However, if you need to perform this kind of validation, consider moving this logic to another module related to the business logic, such as a context. A really good approach is to use `Ecto.Multi` and transactions. If you need to validate something before a database operation and that validation depends on the database, you could add this validation logic as the first step in a multi instead of adding it to the `changeset/2` function in the model.

Here's an example of how to use this approach:

```elixir
defmodule MyApp.Accounts do
  @moduledoc """
  Accounts context.
  """

  alias Ecto.Multi
  alias MyApp.Accounts.User
  alias MyApp.Repo

  defp validate_unique_username_multi(multi, username) do
    Multi.run(multi, :username, fn _repo, _changes -> 
      case Repo.get_by(User, username: username) do
        nil -> 
          {:ok, username}
        _ -> 
          {:error, "Username already taken."}
      end
    end)
  end

  def register_user(attrs) do
    changeset = User.changeset(%User{}, attrs)

    Multi.new()
    |> validate_unique_username_multi(attrs["username"])
    |> Multi.insert(:user, changeset)
    |> Repo.transaction()
    |> case do
      {:ok, %{user: user}} ->
        {:ok, user}
        
      {:error, _failed_operation, reason, _changes} ->
        # handle the error
    end
  end
end
```

Custom model validators in Ecto allow you to extend validation logic beyond the default capabilities provided by the library, giving you control over how specific data should be validated within your application. However, it's essential to avoid database-dependent validations within your changeset to maintain efficiency and safety. 






