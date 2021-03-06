+++
title = "Deploy Elixir Phoenix Application"
[taxonomies]
topics = [ "Phoenix", "Elixir" ]
+++

Phoenix created a config/prod.secret.exs file with your project. This is production connectivity information and thus ignored by git for security. We need to create this file on the target server. We're also going to store our applications in ~/apps/ format which is the equivalent of /home/deploy/apps/.

You can generate a new secret key by using `mix phx.gen.secret`.

```elixir
use Mix.Config

config :new_application_name, NewApplicationNameWeb.Endpoint,
  secret_key_base: "notreal4eG460CU9/2x68+hGBjJ5DIJ8YSxNeb6/S/uY8bm0x5VlpN4VsYtSwR3sqmdXt"

config :new_application_name, NewApplicationName.Repo,
  username: "phx",
  password: "the_password_for_the_phx_database_user",
  database: "new_application_database_prod",
  pool_size: 15
```

## Distillery & edeliver

Distillery compiles our Phoenix application into releases, and edeliver uses ssh and scp to build and deploy the releases to our production server. These are needed in our local project that we plan to deploy.

On the local machine in `mix.exs`:

```elixir
{:edeliver, ">= 1.6.0"},
{:distillery, "~> 2.0", warn_missing: false},
```

Add :edeliver to extra_applications in the application block

```elixir
def application do
   [
     mod: {NewApplicationName.Application, []},
     extra_applications: [:logger, :runtime_tools, :edeliver]
   ]
end
```

```
mix deps.get
```

## Initialize Distillery

Distillery requires a build configuration file that is not generated by default.

```
mix distillery.init
```

This generates configuration files for Distillery in the `rel` directory.

On the local machine, create the `.deliver` directory with the `config` file inside:

```bash
#!/bin/bash

APP="new_application_name"

BUILD_HOST="new_application_name.slrs.io"
BUILD_USER="deploy"
BUILD_AT="/tmp/edeliver/$APP/builds"

START_DEPLOY=true
CLEAN_DEPLOY=true

# prevent re-installing node modules; this defaults to "."
GIT_CLEAN_PATHS="_build rel priv/static"

PRODUCTION_HOSTS="new_application_name.slrs.io"
PRODUCTION_USER="deploy"
DELIVER_TO="/home/deploy/apps"
AUTO_VERSION=revision

# For Phoenix projects, symlink prod.secret.exs to our tmp source
pre_erlang_get_and_update_deps() {
  local _prod_secret_path="/home/deploy/apps/$APP/secret/prod.secret.exs"
  if [ "$TARGET_MIX_ENV" = "prod" ]; then
    status "Linking '$_prod_secret_path'"
    __sync_remote "
      [ -f ~/.profile ] && source ~/.profile
      mkdir -p '$BUILD_AT'
      ln -sfn '$_prod_secret_path' '$BUILD_AT/config/prod.secret.exs'
    "
  fi
}

pre_erlang_clean_compile() {
  status "Running npm install"
    __sync_remote "
      [ -f ~/.profile ] && source ~/.profile
      set -e
      cd '$BUILD_AT'/assets
      npm install
    "

  status "Compiling assets"
    __sync_remote "
      [ -f ~/.profile ] && source ~/.profile
      set -e
      cd '$BUILD_AT'/assets
      node_modules/.bin/webpack --mode production --silent
    "

  status "Running phoenix.digest" # log output prepended with "----->"
  __sync_remote " # runs the commands on the build host
    [ -f ~/.profile ] && source ~/.profile # load profile (optional)
    set -e # fail if any command fails (recommended)
    cd '$BUILD_AT' # enter the build directory on the build host (required)
    # prepare something
    mkdir -p priv/static # required by the phoenix.digest task
    # run your custom task
    APP='$APP' MIX_ENV='$TARGET_MIX_ENV' $MIX_CMD phx.digest $SILENCE
    APP='$APP' MIX_ENV='$TARGET_MIX_ENV' $MIX_CMD phx.digest.clean $SILENCE
  "
}
```

Note on host name, we tend to deploy projects on a subdomain of our slrs.io domain, but this may differ depending on your particular project. Confirm the target url, configure it on CloudFlare ( https://www.cloudflare.com/ ), and put the correct url in this config file, or use the Server IP instead.

## SystemD service

```
sudo touch /etc/systemd/system/my_app.service
```

```
[Unit]
Description=example_phoenix
After=network.target

[Service]
User=deploy
Group=deploy
Restart=on-failure

Type=forking
Environment=HOME=/home/deploy/example_phoenix
EnvironmentFile=/home/deploy/example_phoenix.env # For environment variables like REPLACE_OS_VARS=true
ExecStart= /home/deploy/example_phoenix/bin/example_phoenix start
ExecStop= /home/deploy/example_phoenix/bin/example_phoenix stop

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable my_app.service
```

## Config Prod

```elixir
use Mix.Config

# For production, don't forget to configure the url host
# to something meaningful, Phoenix uses this information
# when generating URLs.
#
# Note we also include the path to a cache manifest
# containing the digested version of static files. This
# manifest is generated by the `mix phx.digest` task,
# which you should run after static files are built and
# before starting your production server.
config :new_application_name, NewApplicationNameWeb.Endpoint,
  http: [port: {:system, "PORT"}],
  url: [host: "new_application_name.slrs.io", port: 80],
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  code_reloader: false

# Do not print debug messages in production
config :logger, level: :info

config :phoenix, :serve_endpoints, true
import_config "prod.secret.exs"
```

## Target server

```
export MIX_ENV=prod
export PORT=4000
```

## Deployment

On your local machine, we need to build the production release.

```
mix edeliver build release production
```

This builds the release on the target server, and stores the archive in your local .edeliver/releases directory. If everything went well with the build, lets deploy:

```
mix edeliver deploy release to production
```

This uploads the release archive to the specified directory and extracts it, and starts the production server.

Next, lets migrate our database schema with the command:

```
mix edeliver migrate production
mix edeliver ping production # shows which nodes are up and running
mix edeliver version production # shows the release version running on the nodes
mix edeliver show migrations on production # shows pending database migrations
mix edeliver migrate production # run database migrations
mix edeliver restart production # or start or stop
```

## Nginx

```
upstream phoenix {
    server 127.0.0.1:4000;
}

server {
  listen 80 default_server;
  listen [::]:80 default_server;
  server_name _;
  location / {
    allow all;

    # Proxy Headers
    proxy_pass http://phoenix;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Cluster-Client-Ip $remote_addr;

    # WebSockets
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";

  }
}
```