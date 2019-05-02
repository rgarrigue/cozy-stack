[Table of contents](README.md#table-of-contents)

# Docker

## Supported tags

- `development`. Running `cozy-stack` compiled against `master` branch in `development` mode, a.k.a without enforcing HTTPS and other security features, so it can be run localy without certificates. Plus `couchdb` and `mailhog`, so it's a all in one environment for developers.

- `latest`. Running `cozy-stack` only compiled against `master` branch in `production` mode. *Use this one*. 

- `<version>` tag. Each one match a source code tag and are compiled in production mode, enforcing HTTPS and so on. Use this one.
- `latest` tag. It matches the `master` branch of the source code, also compiled in production mode. Can be unstable, but you'll have the latest features here.
- `development`. It also matches the `master` branch of the source code, but it's compiled in development mode, hence not enforcing HTTPS so it can be used localy without settings up

## Production

Production image for [Cozy Stack](https://cozy.io), Dockerfile docker-compose.yml & co lives [there](https://github.com/cozy/cozy-stack/tree/master/scripts)

### Requirements

You need docker and using docker-compose is a good idea. Here are the versions used for develoment

```
➜  docker git:(master) docker -v
Docker version 18.09.5, build e8ff056
➜  docker git:(master) docker-compose version
docker-compose version 1.24.0, build 0aa5906
docker-py version: 3.7.2
CPython version: 3.6.7
OpenSSL version: OpenSSL 1.1.1  11 Sep 2018
```

Cozy Stack needs at least a CouchDB 2.3 to run.

A reverse proxy is strongly recommended for HTTPS, Caddy being a good pick since it's able to provide on-demand TLS. Though any other can do.

Redis for caching is optional.

### Local tests or development, running Cozy Stack without HTTPS

Start a CouchDB

    docker run -d -e COUCHDB_USER=cozy -e COUCHDB_PASSWORD=cozy --name couchdb couchdb

Start your stack

    docker run -d -p 80:8080 -p 6060:6060 --link couchdb --name stack cozy/cozy-stack

Create your first cozy test instance

    docker exec -ti stack cozy-stack instances add --passphrase test test.localhost --apps home,drive,settings,store,photos

Then heads to http://test.localhost

Whenever done, remove the containers

    docker rm -f couchdb stack

To persist data you've to use volumes, meaning mounting relevant containers folders to local folders, something like

    docker run -d -e COUCHDB_USER=cozy -e COUCHDB_PASSWORD=cozy --name couchdb --volume $(pwd)/volumes/couchdb:/opt/couchdb/data couchdb
    docker run -d -p 80:8080 -p 6060:6060 --link couchdb --name stack -e LOCAL_USER_ID=$(id -u) -e LOCAL_GROUP_ID=$(id -g) --volume $(pwd)/volumes/stack:/var/lib/cozy/data cozy/cozy-stack

Removing `volumes/couchdb` afterward might requires root privileges, since CouchDB container creates files with UID:GID 5984:5984

### Running on a server and a domain, with HTTPS

If exposed on the Internet, it's very strongly recommended to run your Cozy Stack behind a reverse proxy providing HTTPS.

Provided you already got a server running with Docker installed, a reverse proxy running, a domain set up, and a wildcard certificate for `*.cozy.your-domain.com`, and set up your reverse proxy to redirect all incoming `*.cozy.your-domain.com` traffic to `localhost:8080`, you just have to

    docker run -d -e COUCHDB_USER=cozy -e COUCHDB_PASSWORD=cozy --name couch --volume $(pwd)/volumes/couchdb:/opt/couchdb/data couchdb
    docker run -d -p 127.0.0.1:8080:8080 -p 6060:6060 --link couch --name stack -e LOCAL_USER_ID=$(id -u) -e LOCAL_GROUP_ID=$(id -g) --volume $(pwd)/volumes/stack:/var/lib/cozy/data cozy/cozy-stack

An alternative to an expensive wildcard certificate can be Caddy Server as a reverse proxy. It's able to generate on demand certificates via the ACME protocol and LetsEncrypt. Meaning the first time you open an url, something like https://my-drive.cozy.your-domain.com, there'll be a few second lags for Caddy to grab the certificate, and you'll be good to go.

As it can be difficult to set up, we provide a [docker-compose.yml](https://raw.githubusercontent.com/cozy/cozy-stack/master/docker/docker-compose.yml)  plus [.env file sample](https://raw.githubusercontent.com/rgarrigue/cozy-stack/master/docker/env.sample) to get a Cozy Stack + CouchDB + Redis + Caddy started. Here's a quick how to use it

```bash
# Get the files
git clone https://github.com/cozy/cozy-stack
cd cozy-stack/scripts

cp env.sample .env
# Edit the .env file to specify COUCHDB_PASSWORD and COZY_ADMIN_PASSPHRASE parameters at least

# Start the environment
docker-compose up -d

# Create a first instance
docker-compose exec -T stack bash -c "cozy-stack instances add --email YourEmail@Somewhere.com --passphrase AVerySecretPasswordHere YourFirstInstanceName.stack.$DOMAIN"
```

Afterward if you wan't to clean up

```bash
docker-compose down
sudo rm -rf volumes/
```





## Development

This page list various operations that can be automated _via_ Docker.

### Running a CouchDB instance

This will run a new instance of CouchDB in `single` mode (no cluster) and in
`admin-party-mode` (no user). This command exposes couchdb on the port `5984`.

```bash
$ docker run -d \
    --name cozy-stack-couch \
    -p 5984:5984 \
    -v $HOME/.cozy-stack-couch:/opt/couchdb/data \
    apache/couchdb:2.3
$ curl -X PUT http://127.0.0.1:5984/{_users,_replicator}
```

Verify your installation at: http://127.0.0.1:5984/_utils/#verifyinstall.

Note: for running some unit tests, you will need to use `--net=host` instead of
`-p 5984:5984` as we are using CouchDB replications and CouchDB will need to be
able to open a connexion to the stack.

### Building a cozy-stack _via_ Docker

Warning, this command will build a linux binary. Use
[`GOOS` and `GOARCH`](https://golang.org/doc/install/source#environment) to
adapt to your own system.

```bash
# From your cozy-stack developement folder
docker run -it --rm --name cozy-stack \
    -v $(pwd):/go/src/github.com/cozy/cozy-stack \
    -v $(pwd):/go/bin \
    golang:1.12 \
    go get -v github.com/cozy/cozy-stack
```

### Publishing a new cozy-app-dev image

We publish the cozy-app-dev image when we release a new version of the stack.
See `scripts/release.sh` for details.


### Docker run and url name

A precision for the app name :

docker run --rm -it -p 8080:8080 -v "$(pwd)/build":/data/cozy-app/***my-app*** cozy/cozy-app-dev

***my-app*** will be the first part of : ***my-app***.cozy.tools:8080
