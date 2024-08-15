# The env-injector extension

The env-injector extension adds support for passing user-defined environment variables to snaps. 
By adding the extension to snapped apps, the user of the snap will be allowed to set environment variables via [snap configuration options](https://snapcraft.io/docs/configuration-in-snaps) or environment files and have them exported to the apps on startup.

## Using the extension

Add `extensions: [ env-injector ]` to the application definition in your `snapcraft.yaml` file.

Example, adding the extension to an app named `server`:
```yaml
apps:
  server:
    command: run.sh
    daemon: simple
    extensions: [ env-injector ]
```

With this, the user of the snap will be enabled to pass environment variables via snap options or env files.

### Pass environment variables via snap configuration options

For example, to set `HTTP_PORT=8080` for `web` app, the user can run:

```bash
sudo snap set <snap-name> env.http-port=8080
```

Internally, the extension converts `http-port` to `HTTP_PORT` and exports it before executing the app's command.

The above syntax is to set environment variables for all the apps that use this extension within the snap.
We refer to these as *global options*. 
With the given example, the snap only has one app which uses the extension so setting a global option is acceptable. 

If the snap has multiple apps that need support for environment variables with conflicting variable names, then the user can target the desired app by setting a *local option*.
To set an environment variable for a single app, the user needs to add an `apps.<app>` prefix to the snap config option.

Using the same example as above but targetting only the `server` app:
```bash
sudo snap set <snap-name> apps.server.env.http-port=8080
```

The app's name is taken from the definition in the snapcraft YAML file.
This can be overridden with an alias; see [env alias](#env-alias).

Refer [here](#syntax) for more details on the syntax.


### Pass environment variables via env files

A list of environment variables can be passed to the snap via one or more environment files.

> **Note**  
> For confined snaps, the files need to be accessible from within the snap, either by placing them in the [writable area](https://snapcraft.io/docs/data-locations) of the snap or via a file access interface, such as the [home](https://snapcraft.io/docs/home-interface) or [personal-files](https://snapcraft.io/docs/personal-files-interface).

For example, to set the path to the env file located at `/var/snap/my-snap/common/config.env`:
```bash
sudo snap set <my-snap> envfile=/var/snap/my-snap/common/config.env
```

The environment variables inside `config.env` get exported to all apps that use the extension. 

If the snap has multiple apps that use the extension, the user can avoid conflicts by targeting a single app.

For example, to export the content of `/var/snap/my-snap/common/server.env` to `server` app:
```bash
sudo snap set <my-snap> apps.server.envfile=/var/snap/my-snap/common/server.env
```

The app's name is taken from the definition in the snapcraft YAML file.
This can be overridden with an alias; see [env alias](#env-alias).

Refer [here](#syntax) for more details on the syntax.

## Syntax and Rules

To avoid conflicts with existing snap options, some rules are defined below, along with the allowed syntax accepted by the extension exporter program.

### Syntax

The snap configuration options expect the following syntax for the keys:

* **Global:** Environment variables are visible to all apps within the snap that use this extension.
* **Local:** Environment variables are visible only to the specific `app` defined during the snap set command.

Snap configuration options to environment variable mapping:

|        | Snap configuration option      | Environment variable |
|--------|--------------------------------|----------------------|
| Global | `env.<key>=<value>`            | `<KEY>=<value>`      |
| Local  | `apps.<app>.env.<key>=<value>` | `<KEY>=<value>`      |

The `key` is converted to environment variable name based on the following rules:

Snap configuration options for setting env file paths:

|        | Snap configuration option   |
|--------|-----------------------------|
| Global | `envfile=<path>`            |
| Local  | `apps.<app>.envfile=<path>` |


| Accepted characters in snap option keys                   | Mapped environment variable names |
|-----------------------------------------------------------|-----------------------------------|
| Lowercase letters                                         | Uppercase letters                 |
| Numbers (not at the beginning and with lowercase letters) | Numbers                           |
| Hyphens (surrounded by lowercase letters)                 | Underscores                       |

Any characters not listed in the first column of the above table are not allowed.

### Order precedence

The environment variables are processed in the following order:

1. Env vars extracted from the **global** env file.
2. Env vars extracted from the **local** env file.
3. Env vars from global snap config options.
4. Env vars from local snap config options.

### Env alias

The app's name is taken from the definition in the snapcraft YAML file. 
If necessary, this can be overriden by setting the `env_alias` variable. 

For example, overriding `server` with `web-server`:
```yaml
apps:
  server:
    command: run.sh
    daemon: simple
    extensions: [ env-injector ]
    environments:
      env_alias: web-server
```

Then, the local option can be set as:
```bash
sudo snap set <snap-name> apps.web-server.env.http-port=8080
```

Similarly, the local env file can be set as:
```bash
sudo snap set <my-snap> apps.web-server.envfile=/var/snap/my-snap/common/server.env
```

## Extension's internals

This section provides some high level context on how the extension works.

### During Build

<!--TODO: Put a link to exporter program repository -->
For each app that uses the extension:
1. The application responsible for processing environment variables (the exporter program) is added to its [command chain](https://snapcraft.io/docs/snapcraft-app-and-service-metadata#heading--command-chain).
2. The application's name is taken either from the app's name or `env_alias` set by the developer.

### At Runtime:

1. The exporter program runs before the snapped application.

2. It makes a request to the [snapd API](https://snapcraft.io/docs/using-the-api) through the snapd Unix socket.

3. It reads the available environment variables or paths to the env files.

4. It translates the snap options into environment variables.

5. If available, it loads the env files and sets the listed variables.

6. It sets all converted environment variables.

7. Executes the application's chained command.

