The way to prevent dangling database dependencies with Ecto is to use the `on_delete` option for references (also knows as foreing keys) in migrations. This option allows you to specify how the database should handle associated records when a parent record is deleted. The options available are:

1. `:nothing`: Does nothing. This is the default behavior.
2. `:delete_all`: Deletes all associated records when the parent record is deleted.
3. `:nilify_all`: Sets the foreign key to `nil` in all associated records when the parent record is deleted.
4. `{:nilify, columns}`: Expects a list of atoms. Sets the foreign key to `nil` in the specified columns in all associated records when the parent record is deleted. It is not supported by all databases.
5. `:restrict`: Raises an error if you attempt to delete a parent record that has associated records.

Here's an example of how to use the `on_delete` option in a migration:

```elixir
def change do
  create table(:posts) do
    # other post fields...
    add :user_id, references(:users, on_delete: :delete_all)
  end
end
```

In this example, when a user is deleted, all of their associated posts will be automatically deleted as well.

You can also specify the `on_delete` behavior in your schema definitions:

```elixir
schema "posts" do
  belongs_to :user, User, on_delete: :restrict
end
```

This configuration will raise an error if you attempt to delete a user who has associated posts.

Using the `on_delete` option in schemas is DISCOURAGED on relational databases according to the Ecto documentation. If you are working with this type of databases, you always should use the `on_delete` option in migrations.

When dealing with deeply nested relational models, careful consideration of foreign key behaviors is essential. Applying `:delete_all` indiscriminately can lead to unintended cascading deletions. Consider the following example, extracted from this [post](https://doriankarter.com/avoiding-data-loss-understanding-the-ondelete-option-in-elixir-migrations/):

```elixir
defmodule BookStore.Repo.Migrations.CreateCustomersOrdersAndMailingAddresses do
  use Ecto.Migration

  def change do
    create table(:customers) do
      add :name, :string, null: false
    end

    create table(:mailing_addresses) do
      add :customer_id, references(:customers, on_delete: :delete_all), null: false
      add :nickname, :string, null: false
      add :address_1, :string, null: false
      add :address_2, :string
      add :city, :string, null: false
      add :province_code, :string, null: false
      add :zipcode, :string, null: false
      add :country_code, :string, null: false
    end

    create table(:orders) do
      add :name, :string, null: false
      add :customer_id, references(:customers, on_delete: :delete_all), null: false
      add :mailing_address_id, references(:mailing_address, on_delete: :delete_all), null: false

      timestamps()
    end

    create index(:mailing_addresses, [:customer_id])
    create index(:orders, [:customer_id])
    create index(:orders, [:mailing_address_id])
  end
end
```

In the case above, if a customer deletes a mailing address, any associated orders to that mailing address will be also deleted, which might not be the desired behavior. To address this, the author of the post proposes the two following solutions:

1. **Use `:nilify_all` instead `delete_all`**: 
   ```elixir
   # removing the `null: false` from the `mailing_address_id` 
   # and change `delete_all` to `nilify_all`.
   add :mailing_address_id, references(:mailing_addresses, on_delete: :nilify_all)
   ```

2. **Implement Soft Deletes**: 
   Add a `deleted_at` timestamp to tables where you want to preserve data:
   ```elixir
   add :deleted_at, :utc_datetime
   ```
   Then, instead of actually deleting records, update this field to mark them as deleted. 

   Also, soft deletes could be implemented in more complex ways depending on the use case.


Here are some best practices for avoiding dangling database dependencies:

1. **Be Explicit**: Avoid relying on the default `on_delete` behavior (which is `:nothing` in Ecto). Always specify the desired behavior explicitly to prevent confusion and potential data integrity issues.

2. **Consider Data Importance**: Use `:delete_all` for dependent records that don't make sense without their parent (e.g., a user's profile picture). Use `:nilify_all` or `:restrict` for more independent data.

3. **Use Soft Deletes for Critical Data**: For important data that you might need to reference later, consider using soft deletes instead of hard deletes.

4. **Test Deletion Scenarios**: Always test various deletion scenarios to ensure your database behaves as expected in different situations.

5. **Document Your Choices**: Clearly document your decisions regarding deletion behaviors, especially for complex relationships.

