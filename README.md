# The env-injector extension

The env-injector extension allows developers to export snap options as environment variables within the snap sandbox.
These options can be configured using the [snap set command](https://snapcraft.io/docs/configuration-in-snaps) or via the [the snap API](https://snapcraft.io/docs/using-the-api).
This extension dynamically converts snap configuration options into environment variables, which are then accessible to the app within the snap sandbox.
For this to work, the snap options must follow a specific pattern that the extension can match.
The extension also supports environment files, where the snap configuration options specify the path to the environment file.


## How to use it

Add `extensions: [ env-injector ]` to the application definition in your `snapcraft.yaml` file.
By default the extension defines an app alias in the format of an env variable to be used in the snap config with the same name as defined in your apps section of the `yaml`.
But if you wish, you can avoid to use this default by creating an environment Variable yourself with named as `env_alias` with the name that you wish to use to set the environment variables to your app.

Add `extensions: [ env-injector ]` to the application definition in your `snapcraft.yaml` file.

By default, this extension creates an environment variable with the same name as the app defined in the yaml, which can be used in the snap configuration.
However, if you prefer, you can define this environment variable yourself by using `env_alias` with the name you want to use for setting the environment variables in your app.

- Default usage example:
```yaml
apps:
  rabbitmq:
    command: bin/run.sh
    daemon: simple
    extensions: [ env-injector ]
```

This way you can define environment variables to your application by doing:

```bash
sudo snap set <snap-name> apps.rabbitmq.env.rabbitmq-default-pass="password"
```

- Rewriting `env_alias` usage example:

```yaml
apps:
  rabbitmq:
    command: bin/run.sh
    daemon: simple
    environment:
      env_alias: broker
    extensions: [ env-injector ]
```

This way you can define environment variables to your application by doing:

```bash
sudo snap set <snap-name> apps.broker.env.rabbitmq-default-pass="password"
```

On both cases, for the `rabbitmq` app, the snap option will mapped into the environment variable: `RABBITMQ_DEFAULT_PASS=password`

To prevent conflicts with other snap configurations and to maintain some level of isolation between apps, the following rules apply:

* **Global:** Environment variables are visible to all apps within the snap that use this extension.
* **Local:** Environment variables are visible only to the specific `app` defined during the snap set command.

|        | Snap Configuration Option      | Environment Variable |
|--------|--------------------------------|----------------------|
| Global | `env.<key>=<value>`            | `<KEY>=<value>`      |
| Local  | `apps.<app>.env.<key>=<value>` | `<KEY>=<value>`      |

**Examples**

Assuming `my-snap` is the name of the snap that uses the env-injector extension:

* `sudo snap set <my-snap> env.default-port=8080` will be mapped to `DEFAULT_PORT=8080` and exported globally.

* `sudo snap set <my-snap> apps.server.env.endpoint="/v2/store"` will be mapped to `ENDPOINT="/v2/store"` and exported only to the `server` app.

For environment files, where `path` refers to the `envfile` path:

|        | Snap Configuration Option   |
|--------|-----------------------------|
| Global | `envfile=<path>`            |
| Local  | `apps.<app>.envfile=<path>` |

**Examples**

* `sudo snap set <my-snap> envfile=$HOME/envfile.env` makes all the environment variables globally visible.

* `sudo snap set <my-snap> apps.server.envfile=$HOME/envfile.env` makes the environment variables visible only to the `server` app.

> [!NOTE]
> Depending on the [snap confinement](https://snapcraft.io/docs/snap-confinement), it's necessary for the snap to have access to the location of the environment file.
> It can be done in different ways:
> - Having a [classic confined](https://snapcraft.io/docs/classic-confinement) snap.
> - Using the [home interface](https://snapcraft.io/docs/home-interface) and using the $HOME directory.
> - Using the [system-files interface](https://snapcraft.io/docs/system-files-interface).
> - Using the [snap common directory](https://snapcraft.io/docs/data-locations#heading--system).

## Syntax and Rules


### Order precedence

The environment variables are processed in the following order:

1. Env vars extracted from the **global** env file.
2. Env vars extracted from the **local** env file.
3. Env vars from global snap config options.
4. Env vars from local snap config options.

### Accepted Syntax

| Accepted Characters in Snap Option Keys                   | Mapped Environment Variable Names |
|-----------------------------------------------------------|-----------------------------------|
| Lowercase letters                                         | Uppercase letters                 |
| Numbers (not at the beginning and with lowercase letters) | Numbers                           |
| Hyphens (surrounded by lowercase letters)                 | Underscores                       |

Any characters not listed in the first column of the above table are not allowed.


## env-injector extension Workflow

This section brings some context on how the extension behaves.
The process is divided in two sections, on what happening during the build of the snap and during the snap runtime.
The process works as follows:

### During Build:

<!--TODO: Put a link to exporter program repository -->
- An application responsible for dynamically exporting environment variables (the exporter program) is added to the needed apps's [command chain](https://snapcraft.io/docs/snapcraft-app-and-service-metadata#heading--command-chain).

### At Runtime:

1. The exporter program runs before the snapped application.

2. It makes a request to the [snapd API](https://snapcraft.io/docs/using-the-api) through the snapd Unix socket.

3. It reads the available environment variables or paths to the env files.

4. It translates the snap options into environment variables.

5. If available, it sources the env files.

6. It sources all converted environment variables.

7. It then executes the snapped application.

