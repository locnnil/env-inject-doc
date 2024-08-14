# The env-injector extension

This extension is designed to help users export snap options configured with the [snap set command](https://snapcraft.io/docs/configuration-in-snaps) into environment variables.

Additionally, this extension supports environment files, which we refer to as `envfile` for brevity.

To prevent conflicts with other snap configurations and to maintain some level of isolation between apps, the following rules apply:

* **Global:** Environment variables are visible to all apps within the snap that use this extension.
* **Local:** Environment variables are visible only to the specific `app` defined during the snap set command.

|        | Snap Configuration Option      | Environment Variable |
|--------|--------------------------------|----------------------|
| Global | env.<key>=<value>              | <KEY>=<value>        |
| Local  | apps.<app>.env.<key>=<value>   | <KEY>=<value>        |

**Examples**

Assuming `my-snap` is the name of the snap that uses the env-injector extension:

* `sudo snap set <my-snap> env.default_port=8080` will be mapped to `DEFAULT_PORT=8080` globally.

* `sudo snap set <my-snap> apps.server.env.endpoint="/v2/store"` will be mapped to `ENDPOINT="/v2/store"` and will be visible only to the `server` app.

For environment files, where `path` refers to the `envfile` path:

|        | Snap Configuration Option    |
|--------|------------------------------|
| Global | envfile=<path>               |
| Local  | apps.<app>.envfile=<path>    |

**Examples**

* `sudo snap set <my-snap> envfile=$HOME/envfile.env` makes all the environment variables globally visible.

* `sudo snap set <my-snap> apps.server.envfile=$HOME/envfile.env` makes the environment variables visible only to the `server` app.

## Syntax and Rules

The precedence of environment variables follows this order:

1. Env vars from local snap config options.
2. Env vars from global snap config options.
3. Env vars extracted from the **local** env file.
4. Env vars extracted from the **global** env file.

| Accepted Characters in Snap Option Keys                   | Mapped Environment Variable Names |
|-----------------------------------------------------------|-----------------------------------|
| Lowercase letters                                         | Uppercase letters                 |
| Numbers (not at the beginning and with lowercase letters) | Numbers                           |
| Hyphens (surrounded by lowercase letters)                 | Underscores                       |

Any characters not listed in the first column of the above table are not allowed.

## Exporter Program Workflow

The process works as follows:

### During Build:

- An application responsible for dynamically exporting environment variables is added to the needed apps's [command chain](https://snapcraft.io/docs/snapcraft-app-and-service-metadata#heading--command-chain).

### At Runtime:

- The exporter program runs before the snapped application.

- It makes a request to the [snapd API](https://snapcraft.io/docs/using-the-api) through the snapd Unix socket.

- It reads the available environment variables or paths to the envfiles.

- It translates the snap options into environment variables.

- If available, it sources the envfiles.

- It sources all envfiles.

- It sources all variables.

- It then executes the snapped application.
