# Test Locally With Red Hat Developer Hub

Welcome to RHDH Local, the simplest way to test your software catalogs, techdocs, plugins, and more!

RHDH local is the ideal proving ground for trying out the basic features of RHDH (like Software Catalogs or TechDocs) but, it's also great for testing dynamic plugins and their configuration settings. To use RHDH Local, all you really need is a basic knowledge of tools like Docker or Podman, a PC, and a web browser. You can run it on your laptop, desktop, or homelab server. Better still, when you're done working it's easy to remove.

>**RHDH Local is NOT a substitute for Red Hat Developer Hub**. Do not use RHDH Local as a production system. RHDH Local is designed to help individual developers test various RHDH features. It's not designed to scale to allow use by multiple people and it's not suitable for use by teams (there is no RBAC for example). There's also currently no support for RHDH Local. You use RHDH Local at your own risk. Contributions are welcome.

## What You'll Need Before You Get Started

To use RHDH Local you'll need a few things:

1. A PC based on an x86 64Bit (amd64) architecture
1. Docker or Podman installed with adequate resources available
1. An internet connection for downloading container images, plugins, etc.
1. (Optional) The `git` command line client for cloning this repository (or you can download and extract the Zip from GitHub)
1. (Optional) A GitHub account if you want to integrate GitHub
1. (Optional) The node `npx` tool if you intend to use GitHub authentication 
1. (Optional) A [Red Hat account](https://access.redhat.com/RegistryAuthentication#getting-a-red-hat-login-2) if you want to use PostgreSQL database

## Getting Started With RHDH Local

1. Clone this repository to a location on your PC

   ```sh
   git clone https://github.com/redhat-developer/rhdh-local.git
   ```

1. Move to the `rhdh-local` folder.

   ```sh
   cd rhdh-local
   ```

1. Create your own local `.env` file by using a copy of the `env.sample` provided.

   ```sh
   cp env.sample .env
   ```

   In most cases, when you don't need GitHub Auth or testing different releases you
   can leave it as it is, and it should work.

1. (Optional) Update `configs/app-config.local.yaml`.
   If you need fetching files form from GitHub you should configure `integrations.github`.
   The recommended way is to use GitHub Apps. You can find hints on how to configure it in [github-app-credentials.example.yaml](configs/github-app-credentials.example.yaml) or mode detailed instruction in [Backstage documentation](https://backstage.io/docs/integrations/github/github-apps).

1. Start RHDH Local.
   This repository should work with either `docker compose` using Docker Engine or `podman-compose` using Podman. When using Podman there are some exceptions. Check [Known Issues when using Podman Compose](#known-issues-when-using-podman-compose) for more info.

   ```sh
   podman-compose up -d
   ```

   If you prefer `docker compose` you can just replace `podman-compose` with `docker compose`

   ```sh
   docker compose up -d
   ```

## Changing Your Configuration

When you change `app-config.local.yaml` you can restart `rhdh` to load RHDH with new configuration.

```sh
podman-compose stop rhdh && podman-compose start rhdh
```

When you change `dynamic-plugins.yaml` you need to re-run `install-dynamic-plugins` container and than restart RHDH instance.

```sh
podman-compose run install-dynamic-plugins
podman-compose stop rhdh && podman-compose start rhdh
```

## Loading dynamic plugins from a local directory

During boot the `install-dynamic-plugins` container reads the contents of the `configs/dynamic-plugins.yaml` file and activates, configures, or downloads any plugins contained in that file. In addition, the `local-plugins` directory is mounted into the `install-dynamic-plugins` container on the path `/opt/app-root/src/local-plugins`. Any plugins in that location can also be activated and configured in the same way (without downloading).

You can use the `local-plugins` folder install dynamic plugins directly from your local machine using the following steps:

1. Copy the dynamic plugin binary file into the `local-plugins` directory.
2. Make sure that the permissions are set to allow container to read files (quick and dirty solution is `chmod -R 777 local-plugins`)
3. Configure your dynamic plugin in `dynamic-plugins.yaml`. See commented out examples in that file for examples.
4. See [Changing Your Configuration](#changing-your-configuration) section for more information about how to change and load new configuration.

## Changing The Container Image

You can switch between RHDH and Janus-IDP by changing the container image name hold by the `RHDH_IMAGE` environment variable in your `.env` file.

To use nightly build of Janus-IDP, set the variable as follows:

```sh
RHDH_IMAGE=quay.io/janus-idp/backstage-showcase:next
```

To use the official release of RHDH 1.3, set the variable as follows:

```sh
RHDH_IMAGE=quay.io/rhdh/rhdh-hub-rhel9:1.3
```

## Testing RHDH in a simulated corporate proxy setup

If you want to test how RHDH would behave if deployed in a corporate proxy environment,
you can run `podman-compose` or `docker-compose` by merging both the [`compose.yaml`](./compose.yaml) and [`compose-with-corporate-proxy.yaml`](./compose-with-corporate-proxy.yaml) files.

Example with `podman-compose` (note that the order of the YAML files is important):

```sh
podman-compose \
   -f compose.yaml \
   -f compose-with-corporate-proxy.yaml \
   up -d
```

The [`compose-with-corporate-proxy.yaml`](compose-with-corporate-proxy.yaml) file includes a specific [Squid](https://www.squid-cache.org/)-based proxy container as well as an isolated network, such that:

1. only the proxy container has access to the outside
2. all containers part of the internal network need to communicate through the proxy container to reach the outside. This can be done with the `HTTP(S)_PROXY` and `NO_PROXY` environment variables.

## Cleanup

To reset RHDH Local you can use the following command. This will clean up any attached volumes, but your configuration changes will remain.

```sh
podman-compose down --volumes
```

To reset everything in the cloned rhdh-local repository, including any configuration changes you've made try:

```sh
git reset --hard
```

To remove the RHDH containers completely from your system (after you have run a `compose down`):

```sh
docker system prune --volumes # For rhdh-local running on docker
podman system prune --volumes # For rhdh-local running on podman
```


### Known Issues when using Podman Compose

Works with `podman-compose` only with image that include this following fix https://github.com/janus-idp/backstage-showcase/pull/1585

Older images don't work in combination with `podman-compose`.
This is due to https://issues.redhat.com/browse/RHIDP-3939. RHDH images currently populate dynamic-plugins-root directory with all plugins that are packaged inside the image.
Before podman mounts volume over `dynamic-plugins-root` directory it copies all existing files into the volume. When the plugins are installed using `install-dynamic-plugins.sh` script it create duplicate installations of some plugins, this situation than prevents Backstage to start.

This also doesn't work with `podman compose` when using `docker-compose` as external compose provider on macOS.

It fails with

```
install-dynamic-plugins-1  | Traceback (most recent call last):
install-dynamic-plugins-1  |   File "/opt/app-root/src/install-dynamic-plugins.py", line 429, in <module>
install-dynamic-plugins-1  |     main()
install-dynamic-plugins-1  |   File "/opt/app-root/src/install-dynamic-plugins.py", line 206, in main
install-dynamic-plugins-1  |     with open(dynamicPluginsFile, 'r') as file:
install-dynamic-plugins-1  | PermissionError: [Errno 13] Permission denied: 'dynamic-plugins.yaml'
```

It looks like `docker-compose` when used with podman doesn't correctly propagate `Z` SElinux label.

## Using PostgreSQL database

By default, in-memory db is used.
If you want to use PostgreSQL with RHDH, here are the steps:

> **NOTE**: You must have [Red Hat Login](https://access.redhat.com/RegistryAuthentication#getting-a-red-hat-login-2) to use `postgresql` image.

1. Login to container registry with *Red Hat Login* credentials to use `postgresql` image

   ```sh
   podman login registry.redhat.io
   ```

   If you prefer `docker` you can just replace `podman` with `docker`

   ```sh
   docker login registry.redhat.io
   ```

2. Uncomment the `db` service block in [compose.yaml](compose.yaml) file

   ```yaml
   db:
     image: "registry.redhat.io/rhel8/postgresql-16:latest"
     volumes:
       - "/var/lib/pgsql/data"
     env_file:
       - path: "./.env"
         required: true
     environment:
       - POSTGRESQL_ADMIN_PASSWORD=${POSTGRES_PASSWORD}
     healthcheck:
       test: ["CMD", "pg_isready", "-U", "postgres"]
       interval: 5s
       timeout: 5s
       retries: 5
   ```

3. Uncomment the `db` section in the `depends_on` section of `rhdh` service in [compose.yaml](compose.yaml)

   ```yaml
   depends_on:
     install-dynamic-plugins:
       condition: service_completed_successfully
     db:
       condition: service_healthy
   ```

4. Comment out the SQLite in-memory configuration in [`app-config.local.yaml`](configs/app-config.local.yaml)

   ```yaml
   # database:
   #   client: better-sqlite3
   #   connection: ':memory:'
   ```

## Using VSCode to debug backend plugins

You can use RHDH-local to debug your backend plugins using VSCode. Here is how you can do it:

1. Start RHDH-local with debug compose file

   ```sh
   # in rhdh-local directory
   podman-compose up -f compose.yaml -f compose-debug.yaml
   ```

2. Open your plugin source code in VSCode
3. Export plugin as dynamic plugin

   ```sh
   # in plugin source code directory
   npx @janus-idp/cli@latest package export-dynamic-plugin
   ```

4. Copy exported derived plugin package to `dynamic-plugins-root` directory in RHDH container.

   ```sh
   # in plugin source code directory
   podman cp dist-dynamic rhdh:/opt/app-root/src/dynamic-plugins-root/<your-plugin-name>
   ```

5. If your plugin requires configuration add it to `app-config.local.yaml` of RHDH-local

6. Restart rhdh container

   ```sh
   # in rhdh-local directory
   podman-compose stop rhdh
   podman-compose start rhdh
   ```

7. Configure VSCode debugger to attach to RHDH container.

   `.vscode/launch.json` example:

   ```json
   {
      "version": "0.2.0",
      "configurations": [
         {
            "name": "Attach to Process",
            "type": "node",
            "request": "attach",
            "port": 9229,
            "localRoot": "${workspaceFolder}",
            "remoteRoot": "/opt/app-root/src/dynamic-plugins-root/<your-plugin-name>",
         }
      ]
   }
   ```

8. Now, you can start debugging your plugin using VSCode debugger.
   Source mapping should work, and you should be able to put breakpoints to your TypeScript files.
   If it doesn't work, most likely you need to adjust `localRoot` and `remoteRoot` paths in `launch.json`.

   Every time you make changes to your plugin source code, you need to repeat steps 3-6.