![](https://github.com/chain-partners/definject/blob/master/brand/logo.png?raw=true)

[![Hex version badge](https://img.shields.io/hexpm/v/definject.svg)](https://hex.pm/packages/definject)
[![License badge](https://img.shields.io/hexpm/l/definject.svg)](https://github.com/chain-partners/definject/blob/master/LICENSE.md)
[![Elixir CI](https://github.com/definject/definject/workflows/Build%20and%20mix%20test/badge.svg)](https://github.com/chain-partners/definject/actions)

Unobtrusive Dependency Injector for Elixir

## Why?

Let's say we want to test following function.

```elixir
def send_welcome_email(user_id) do
  %{email: email} = Repo.get(User, user_id)

  welcome_email(to: email)
  |> Mailer.send()
end
```

Here's one possible solution to replace `Repo.get/2` and `Mailer.send/1` with mocks:

```elixir
def send_welcome_email(user_id, repo \\ Repo, mailer \\ Mailer) do
  %{email: email} = repo.get(User, user_id)

  welcome_email(to: email)
  |> mailer.send()
end
```

First, I believe that this approach is too obtrusive as it requires modifying the function body to make it testable. Second, with `Mailer` replaced with `mailer`, the compiler no longer check the existence of `Mailer.send/1`.

`definject` does not require you to modify function arguments or body. Instead, you just need to replace `def` with `definject`. It also allows injecting different mocks to each function. It also does not limit using `:async` option as mocks are contained in each test function.

## Installation

The package can be installed by adding `definject` to your list of dependencies
in `mix.exs`:

```elixir
def deps do
  [{:definject, "~> 1.1"}]
end
```

By default, `definject` is replaced with `def` in all but the test environment. Add the below configuration to enable in other environments.

```elixir
config :definject, :enable, true
```

To format `definject` like `def`, add following to your `.formatter.exs`

```elixir
locals_without_parens: [definject: 1, definject: 2]
```

## Documentation

API documentation is available at [https://hexdocs.pm/definject](https://hexdocs.pm/definject)

## Usage

### definject

`definject` transforms a function to accept a map where dependent functions and modules can be injected.

```elixir
import Definject

definject send_welcome_email(user_id) do
  %{email: email} = Repo.get(User, user_id)

  welcome_email(to: email)
  |> Mailer.send()
end
```

is expanded into (simplified to understand)

```elixir
def send_welcome_email(user_id, deps \\ %{}) do
  %{email: email} =
    Map.get(deps, &Repo.get/2,
      :erlang.make_fun(Map.get(deps, Repo, Repo), :get, 2)
    ).(User, user_id)

  welcome_email(to: email)
  |> Map.get(deps, &Mailer.send/1,
       :erlang.make_fun(Map.get(deps, Mailer, Mailer), :send, 1)
     ).()
end
```

Note that local function calls like `welcome_email(to: email)` are not expanded unless it is prepended with `__MODULE__`.

Now, you can inject mock functions and modules in tests.

```elixir
test "send_welcome_email" do
  Accounts.send_welcome_email(100, %{
    Repo => MockRepo,
    &Mailer.send/1 => fn %Email{to: "user100@gmail.com", subject: "Welcome"} ->
      Process.send(self(), :email_sent)
    end
  })

  assert_receive :email_sent
end
```

`definject` raises if the passed map includes a function or a module that's not used within the injected function.
You can disable this by adding `strict: false` option.

```elixir
test "send_welcome_email with strict: false" do
  Accounts.send_welcome_email(100, %{
    &Repo.get/2 => fn User, 100 -> %User{email: "user100@gmail.com"} end,
    &Repo.all/1 => fn _ -> [%User{email: "user100@gmail.com"}] end, # Unused
    strict: false
  })
end
```

### mock

If you don't need pattern matching in mock function, `mock/1` can be used to reduce boilerplates.

```elixir
import Definject

test "send_welcome_email with mock/1" do
  Accounts.send_welcome_email(
    100,
    mock(%{
      Repo => MockRepo,
      &Mailer.send/1 => Process.send(self(), :email_sent)
    })
  )

  assert_receive :email_sent
end
```

Note that `Process.send(self(), :email_sent)` is surrounded by `fn _ -> end` when expanded.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE.md) file for details
