# coliseo-development-workspace-setup
Helper package to setup the coliseo workspace for local development

This packages has everything you need in order to locally develop `coliseo`. You should follow the instructions in order to  have a complete setup that will allow for fast local development. This uses docker and docker-compose so you should have those 2 installed and set up properly.

## Getting the packages

First of all, you should clone the packages you are working with. This setup expects the engine to be inside `coliseo` and the web app in `coliseo-web`, following the respective repositories' names.

Go ahead and clone them:

```
git clone git@github.com:<your git username here>/coliseo.git

git clone git@github.com:<your git username here>/coliseo-web.git
```

## Running the app, and development workflow

This setup leverages Docker and Docker Compose to build the components and serve the application. Each component has its own Docker Service (and container). You can check their definitions in `docker-compose.dev.yml`.

### Building the dependencies

Before running the main app --- likely `coliseo-web` --- you'll want to build the dependencies locally first. This is only needed if you are actually modifying them locally. Otherwise, just build the webapp, and it will fetch the dependencies from npm as usual

For this example, let's assume we are working on `coliseo-web`, and we want to make changes to the engine `coliseo` as well.

First, build the dependency, `coliseo`:

```
docker-compose -f docker-compose.dev.yml up --build coliseo-engine-dev
```

Then, you can run the web app, `coliseo-web`, with:

```
# No need for the --build flag here as the run command is the one that builds everyhting
docker-compose -f docker-compose.dev.yml up web-server-dev
```

Every time you make a change in the dependecy, you just need to rebuild it, then re-run the consumer and it should pick up your changes!
## Adding a local dependency
In order to add a new dependency that can be locally overriden (built and used without having to publish to `npm`), we need 2 things:

* a docker service to build it and locally publish it (using yalc)
* a shared volume that can be shared with other docker services that may want to consume it

Let's take the example of coliseo's engine:

```
coliseo-engine-dev:
    build:
        context: ./coliseo
    environment:
        - NODE_ENV=development
    volumes:
        - coliseo_engine_dependency:/root/.yalc/packages/@coliseo/engine:rw # important to use this path, as it is what yalc uses when publishing the local versions
```

It's important to note that the dependency build phase (as set up in the respective Dockerfile) must include `CMD yalc publish` so that the dependency is locally published.

Then, in the `volumes` section:
```
volumes:
    coliseo_engine_dependency:
```

Then, for any service that needs this dependency (such as the web app):

```
web-server-dev:
    ...
    volumes:
        # important to mount it in this path as it is what yalc expects, as well has having in :ro (read-only) mode 
        - coliseo_engine_dependency:/root/.yalc/packages/@coliseo/engine:ro 
```

## How this works

Internally, this system uses [yalc](https://github.com/wclr/yalc) in order to publish npm packages locally for local dependents to use. In essence the docker service of the dependency writes to the shared volume its local version, so that the dependent service can mount that local volume and consume the local override.