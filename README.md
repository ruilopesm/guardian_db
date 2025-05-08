# Guardian.DB

[![Hex.pm](https://img.shields.io/hexpm/v/guardian_db.svg)](https://hex.pm/packages/guardian_db)
![Build Status](https://github.com/ueberauth/guardian_db/workflows/Continuous%20Integration/badge.svg)
[![Codecov](https://codecov.io/gh/ueberauth/guardian_db/branch/master/graph/badge.svg)](https://codecov.io/gh/ueberauth/guardian_db)
[![Inline docs](https://inch-ci.org/github/ueberauth/guardian_db.svg)](https://inch-ci.org/github/ueberauth/guardian_db)

`Guardian.DB` is an extension to `Guardian` that tracks tokens in your
application's database to prevent playback.

## Installation

`Guardian.DB` assumes that you are using the Guardian framework for
authentication.

To install `Guardian.DB`, first add it to your `mix.exs` file:

```elixir
defp deps do
  [
    {:guardian_db, "~> 3.0"}
  ]
end
```

Then run `mix deps.get` on your terminal.

Configure your application as seen in the *Configuration* section below prior to attempting to generate the migration or you will get an `application was not loaded/started` error.

Following configuration add the Guardian migration:

run `mix guardian.db.gen.migration` to generate a migration.

**Do not run the migration yet,** we need to complete our setup first.

## Configuration

```elixir
config :guardian, Guardian.DB,
  adapter: Guardian.DB.EctoAdapter, # default
  repo: MyApp.Repo, # Add your repository module
  schema_name: "guardian_tokens", # default
  token_types: ["refresh_token"], # store all token types if not set
```

To use [ETS](https://hexdocs.pm/elixir/1.16/erlang-term-storage.html) instead
of Ecto for storing tokens, you can set `adapter` to `Guardian.DB.ETSAdapter`.

To sweep expired tokens from your db you should add
`Guardian.DB.Sweeper` to your supervision tree.

```elixir
children = [
  {Guardian.DB.Sweeper, [interval: 60 * 60 * 1000]} # 1 hour
]
```

`Guardian.DB` works by hooking into the lifecycle of your `Guardian` module.

You'll need to add it to:

* `after_encode_and_sign`
* `on_verify`
* `on_refresh`
* `on_revoke`

For example:

```elixir
defmodule MyApp.AuthTokens do
  use Guardian, otp_app: :my_app

  # snip...

  def after_encode_and_sign(resource, claims, token, _options) do
    with {:ok, _} <- Guardian.DB.after_encode_and_sign(resource, claims["typ"], claims, token) do
      {:ok, token}
    end
  end

  def on_verify(claims, token, _options) do
    with {:ok, _} <- Guardian.DB.on_verify(claims, token) do
      {:ok, claims}
    end
  end

  def on_refresh({old_token, old_claims}, {new_token, new_claims}, _options) do
    with {:ok, _, _} <- Guardian.DB.on_refresh({old_token, old_claims}, {new_token, new_claims}) do
      {:ok, {old_token, old_claims}, {new_token, new_claims}}
    end
  end

  def on_revoke(claims, token, _options) do
    with {:ok, _} <- Guardian.DB.on_revoke(claims, token) do
      {:ok, claims}
    end
  end
end
```

Now run the migration and you'll be good to go.

### Custom schema name

`Guardian.DB` allows custom schema name in migrations, based on following
configuration:

```elixir
config :guardian, Guardian.DB,
  schema_name: "my_custom_schema"
```

And when you run the migration, it'll generate the following migration:

```elixir
  create table(:my_custom_schema, primary_key: false) do
    add(:jti, :string, primary_key: true)
    add(:aud, :string, primary_key: true)
    add(:typ, :string)
    add(:iss, :string)
    add(:sub, :string)
    add(:exp, :bigint)
    add(:jwt, :text)
    add(:claims, :map)
    timestamps()
  end
```

Then, run the migration and you'll be good to go.

### Considerations

`Guardian` is already a very robust JWT solution. However, if your
application needs the ability to immediately revoke and invalidate tokens that
have already been generated, you need something like `Guardian.DB` to build upon
`Guardian`.

In `Guardian`, you as a systems administrator have no way of revoking
tokens that have already been generated, you can call `Guardian.revoke`, but in
`Guardian` **that function does not actually do anything** - it just provides
hooks for other libraries, such as this one, to define more specific behavior.
Discarding the token after something like a log out action is left up to the
client application. If the client application does not discard the token, or
does not log out, or the token gets stolen by a malicious script (because the
client application stores it in localStorage, for instance), the only thing you
can do is wait until the token expires. Depending on the scenario, this may not
be acceptable.

With `Guardian.DB`, records of all generated tokens are kept in your
application's database. During each request, the `Guardian.Plug.VerifyHeader`
and `Guardian.Plug.VerifySession` plugs check the database to make sure the
token is there. If it is not, the server returns a 401 Unauthorized response to
the client. Furthermore, `Guardian.revoke!` behavior becomes enhanced, as it
actually removes the token from the database. This means that if the user logs
out, or you revoke their token (e.g. after noticing suspicious activity on the
account), they will need to re-authenticate.

### Disadvantages

In `Guardian`, token verification is very light-weight. The only thing
`Guardian` does is decode incoming tokens and make sure they are valid. This can
make it much easier to horizontally scale your application, since there is no
need to centrally store sessions and make them available to load balancers or
other servers.

With `Guardian.DB`, every request requires a trip to the database, as `Guardian`
now needs to ensure that a record of the token exists. In large scale
applications this can be fairly costly, and can arguably eliminate the main
advantage of using a JWT authentication solution, which is statelessness.
Furthermore, session authentication already works this way, and in most cases
there isn't a good enough reason to reinvent that wheel using JWTs.

In other words, once you have reached a point where you think you need
`Guardian.DB`, it may be time to take a step back and reconsider your whole
approach to authentication!

### Create your own Repo

We created `Guardian.DB.Adapter` behaviour to allow creating other repositories for persisting JWT tokens.
You need to implement the `Guardian.DB.Adapter` behavior working with your preferred storage.

### Adapters

1. Redis adapter - [`guardian_redis`](https://github.com/alexfilatov/guardian_redis)

Feel free to create your adapters using `Guardian.DB.Adapter` behavior and you are welcome to add them here.
