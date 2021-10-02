The following guide is a work-in-progress. Some instructions may not be valid yet.

---

# Citadel App Framework

If you can code in any language, you already know how to develop an app for Citadel. There is no restriction on the kind of programming languages, frameworks or databases that you can use. Apps run inside isolated Docker containers, and the only requirement (for now) is that they should have a web-based UI.

> Some server apps might not have a UI at all. In that case, the app should serve a simple web page listing the connection details, QR codes, setup instructions, and anything else needed for the user to connect. Because Citadel is made for users without any technical knowledge as well as for users with higher level technical knowledge, it is important that the app is easy to use.

To keep this document short and easy, we won't go into the app development itself, and will instead focus on packaging an existing app.

Let's straightaway jump into action by packaging [BTC RPC Explorer](https://github.com/janoside/btc-rpc-explorer), a Node.js based blockchain explorer, for Citadel.

There are 4 steps:

1. [🛳 Containerizing the app using Docker](#1-containerizing-the-app-using-docker)
1. [🏰 Packaging the app](#2-packaging-the-app)
1. [🛠 Testing the app on Ctadel](#3-testing-the-app-on-citadel)
    <!--1. [Testing on development environment (Linux or macOS)](#31-testing-the-app-on-citadel-development-environment)-->
    1. [Testing on Citadel OS (Raspberry Pi 4)](#32-testing-on-citadel-os-raspberry-pi-4)
1. [🚀 Submitting the app](#4-submitting-the-app)

___

## 1. 🛳&nbsp;&nbsp;Containerizing the app using Docker

1\. Let's start by cloning BTC RPC Explorer on our system:

```sh
git clone --branch v2.0.2 https://github.com/janoside/btc-rpc-explorer.git
cd  btc-rpc-explorer
```

2\. Next, we'll create a `Dockerfile` in the app's directory:

```Dockerfile
FROM node:16-bullseye-slim AS builder

WORKDIR /build
COPY . .
RUN apt-get update
RUN apt-get install -y git python3 build-essential
RUN npm ci --production

FROM node:16-bullseye-slim

USER 1000
WORKDIR /build
COPY --from=builder /build .
EXPOSE 3002
CMD ["npm", "start"]
```

### A good Dockerfile:

- [x] Uses `debian:bullseye-slim` (or its derivatives, like `node:16-bullseye-slim`) as the base image — resulting in less storage consumption and faster app installs as the base image is already cached on the user's node.
- [x] Uses [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) for smaller image size.
- [x] Ensures development files are not included in the final image.
- [x] Has only one service per container.
- [x] Doesn't run the service as root.
- [x] Uses remote assets that are verified against a checksum.
- [x] Results in deterministic image builds.

3\. We're now ready to build the Docker image of BTC RPC Explorer. Citadel supports both 64-bit ARM and x86 architectures, so we'll use `docker buildx` to build, tag, and push multi-architecture Docker images of our app to Docker Hub.

```sh
docker buildx build --platform linux/arm64,linux/amd64 --tag runcitadel/btc-rpc-explorer:v3.2.0 --output "type=registry" .
```

> You need to enable ["experimental features"](https://docs.docker.com/engine/reference/commandline/cli/#experimental-features) in Docker to use `docker buildx`.

___

## 2. Packaging the app

1\. Let's fork the [runcitadel/compose-nonfree](https://github.com/runcitadel/compose-nonfree) repo on GitHub, clone our fork locally, create a new branch for our app, and then switch to it:

```sh
git clone https://github.com/<username>/compose-nonfree.git citadel
cd citadel
git checkout -b btc-rpc-explorer
```

2\. It's now time to decide an ID for our app. An app ID should only contain lowercase alphabetical characters and dashes, and should be humanly recognizable. For this app we'll go with `btc-rpc-explorer`.

We need to create a new subdirectory in the apps directory with same name as our app ID and move into it:

```sh
mkdir apps/btc-rpc-explorer
cd apps/btc-rpc-explorer
```

3\. We'll now create a `app.yml` file in this directory to define our application.

> New to Docker Compose? It's a simple tool for defining and running Docker applications that can have multiple containers. Follow along the tutorial, we promise it's not hard if you already understand the basics of Docker.

Let's copy-paste the following template `app.yml` file in a text editor and edit it according to our app.

```yml
version: 1

# First, define some properties for the app
metadata:
  category: Explorers
  name: BTC RPC Explorer
  version: 3.2.0
  # The tagline shouldn't have more than 50 characters.
  tagline: Simple, database-free blockchain explorer
  # 50 to 200 words description of the app
  description: >-
    BTC RPC Explorer is a full-featured, self-hosted explorer for the
    Bitcoin blockchain. With this explorer, you can explore not just the blockchain
    database, but also explore the functional capabilities of your node.


    It comes with a network summary dashboard, detailed view of blocks, transactions,
    addresses, along with analysis tools for viewing stats on miner activity, mempool
    summary, with fee, size, and age breakdowns. You can also search by transaction
    ID, block hash/height, and addresses.


    It's time to appreciate the "fullness" of your node.
  developer: Dan Janosik
  website: https://explorer.btc21.org
  dependencies:
  - electrum
  - bitcoind
  repo: https://github.com/janoside/btc-rpc-explorer
  support: https://github.com/janoside/btc-rpc-explorer/discussions
  port: 3002
  # This property is used by the dashboard later to display preview images.
  # You can ignore this property for now, we'll take care of this once you submit your PR.
  gallery:
  - 1.jpg
  - 2.jpg
  - 3.jpg

# Now, define the docker containers
containers:
  - name: web
    image: <docker-image>:<tag>
    restart: on-failure
    stop_grace_period: 1m
    # Define the permissions your app needs.
    permissions:
      - bitcoind
      - electrum
      - lnd
    port: 10000  # Replace this with the port your app uses
    data:
      # Uncomment to mount your data directories inside
      # the Docker container for storing persistent data
      # - foo:/foo
      # - bar:/bar

      # A few mounts will be automatically generated depending on the permissions.
      # LND's data directory will be available at /lnd
      # Bitcoind's data directory will be available at /bitcoin
    environment:
      # Pass any environment variables to your app for configuration in the form:
      # VARIABLE_NAME: value
      #
      # Here are all the provided variables that you can pass through to
      # your app to connect to Bitcoin Core, LND, Electrum and Tor:
      #
      # Bitcoin Core environment variables
      # $BITCOIN_NETWORK - Can be "mainnet", "testnet" or "regtest"
      # $BITCOIN_IP - Local IP of Bitcoin Core
      # $BITCOIN_P2P_PORT - P2P port
      # $BITCOIN_RPC_PORT - RPC port
      # $BITCOIN_RPC_USER - RPC username
      # $BITCOIN_RPC_PASS - RPC password
      # $BITCOIN_RPC_AUTH - RPC auth string
      #
      # LND environment variables
      # $LND_IP - Local IP of LND
      # $LND_GRPC_PORT - gRPC Port of LND
      # $LND_REST_PORT - REST Port of LND
      #
      # Electrum server environment variables
      # $ELECTRUM_IP - Local IP of Electrum server
      # $ELECTRUM_PORT - Port of Electrum server
      #
      # Tor proxy environment variables
      # $TOR_PROXY_IP - Local IP of Tor proxy
      # $TOR_PROXY_PORT - Port of Tor proxy
      #
      # App specific environment variables
      # $APP_HIDDEN_SERVICE - The address of the Tor hidden service your app will be exposed at
      # $APP_DOMAIN - Local domain name of the app ("citadel.local" on Citadel OS)
  # If your app has more conrtainers, like a database, you can define those
  # services below:
  # - name: db
  #   image: <docker-image>:<tag>
  #   ...

```

4\. For our app, we'll update `<docker-image>` with `runcitadel/btc-rpc-explorer`, `<tag>` with `v2.0.2`, and `<port>` with `3002`. Since BTC RPC Explorer doesn't need to store any persistent data and doesn't require access to Bitcoin Core's or LND's data directories, we can remove the entire `volumes` block.

BTC RPC Explorer is an application with a single Docker container, so we don't need to define any other additional services (like a database service, etc) in the compose file.

> If BTC RPC Explorer needed to persist some data we would have created a new `data` directory next to the `docker-compose.yml` file. We'd then mount the volume `- ${APP_DATA_DIR}/data:/data` in  `docker-compose.yml` to make the directory available at `/data` inside the container.

Updated `docker-compose.yml` file:

```yml
# First, define some properties for the app
metadata:
  category: Explorers
  name: BTC RPC Explorer
  version: 3.2.0
  # The tagline shouldn't have more than 50 characters.
  tagline: Simple, database-free blockchain explorer
  # 50 to 200 words description of the app
  description: >-
    BTC RPC Explorer is a full-featured, self-hosted explorer for the
    Bitcoin blockchain. With this explorer, you can explore not just the blockchain
    database, but also explore the functional capabilities of your node.


    It comes with a network summary dashboard, detailed view of blocks, transactions,
    addresses, along with analysis tools for viewing stats on miner activity, mempool
    summary, with fee, size, and age breakdowns. You can also search by transaction
    ID, block hash/height, and addresses.


    It's time to appreciate the "fullness" of your node.
  developer: Dan Janosik
  website: https://explorer.btc21.org
  dependencies:
  - electrum
  - bitcoind
  repo: https://github.com/janoside/btc-rpc-explorer
  support: https://github.com/janoside/btc-rpc-explorer/discussions
  port: 3002
  # This property is used by the dashboard later to display preview images.
  # You can ignore this property for now, we'll take care of this once you submit your PR.
  gallery:
  - 1.jpg
  - 2.jpg
  - 3.jpg

# Now, define the docker containers
containers:
  - name: web
    image: runcitadel/btc-rpc-explorer:v3.2.0
    restart: on-failure
    stop_grace_period: 1m
    # Define the permissions your app needs.
    permissions:
      - bitcoind
      - electrum
    port: 3002
```

5\. Next, let's set the environment variables required by our app to connect to Bitcoin Core, Electrum server, and for app-related configuration ([as required by the app](https://github.com/janoside/btc-rpc-explorer/blob/master/.env-sample)).

So the final version of `docker-compose.yml` would be:

```yml
# First, define some properties for the app
metadata:
  category: Explorers
  name: BTC RPC Explorer
  version: 3.2.0
  # The tagline shouldn't have more than 50 characters.
  tagline: Simple, database-free blockchain explorer
  # 50 to 200 words description of the app
  description: >-
    BTC RPC Explorer is a full-featured, self-hosted explorer for the
    Bitcoin blockchain. With this explorer, you can explore not just the blockchain
    database, but also explore the functional capabilities of your node.


    It comes with a network summary dashboard, detailed view of blocks, transactions,
    addresses, along with analysis tools for viewing stats on miner activity, mempool
    summary, with fee, size, and age breakdowns. You can also search by transaction
    ID, block hash/height, and addresses.


    It's time to appreciate the "fullness" of your node.
  developer: Dan Janosik
  website: https://explorer.btc21.org
  dependencies:
  - electrum
  - bitcoind
  repo: https://github.com/janoside/btc-rpc-explorer
  support: https://github.com/janoside/btc-rpc-explorer/discussions
  port: 3002
  # This property is used by the dashboard later to display preview images.
  # You can ignore this property for now, we'll take care of this once you submit your PR.
  gallery:
  - 1.jpg
  - 2.jpg
  - 3.jpg
  environment:
    # Bitcoin Core connection details
    BTCEXP_BITCOIND_HOST: $BITCOIN_IP
    BTCEXP_BITCOIND_PORT: $BITCOIN_RPC_PORT
    BTCEXP_BITCOIND_USER: $BITCOIN_RPC_USER
    BTCEXP_BITCOIND_PASS: $BITCOIN_RPC_PASS

    # Electrum connection details
    BTCEXP_ELECTRUMX_SERVERS: "tcp://$ELECTRUM_IP:$ELECTRUM_PORT"

    # App Config
    BTCEXP_HOST: 0.0.0.0
    DEBUG: "btcexp:*,electrumClient"
    BTCEXP_ADDRESS_API: electrumx
    BTCEXP_SLOW_DEVICE_MODE: "true"
    BTCEXP_NO_INMEMORY_RPC_CACHE: "true"
    BTCEXP_PRIVACY_MODE: "true"
    BTCEXP_NO_RATES: "true"
    BTCEXP_RPC_ALLOWALL: "false"
    BTCEXP_BASIC_AUTH_PASSWORD: ""      

```

6\. We're pretty much done here. The next step is to commit the changes, push it to our fork's branch, and test out the app on Citadel.

```sh
git add .
git commit -m "Add BTC RPC Explorer"
git push origin btc-rpc-explorer
```

___

## 3. 🛠&nbsp;&nbsp;Testing the app on Citadel

<!--
### 3.1 Testing the app on Citadel development environment

Citadel development environment ([`citadel-dev`](https://github.com/runcitadel/citadel-dev)) is a lightweight regtest instance of Citadel that runs inside a virtual machine on your system. It's currently only compatible with Linux or macOS, so if you're on Windows, you may skip this section and directly test your app on a Raspberry Pi 4 running [Citadel OS](https://github.com/runcitadel/os).

1\. First, we'll install the `citadel-dev` CLI and it's dependencies [Virtual Box](https://www.virtualbox.org) and [Vagrant](https://vagrantup.com) on our system. If you use [Homebrew](https://brew.sh) you can do that with just:

```sh
brew install lukechilds/tap/citadel-dev gnu-sed
brew install --cask virtualbox vagrant
```

2\. Now let's initialize our development environment and boot the VM:

```sh
mkdir citadel-dev
cd citadel-dev
citadel-dev init
citadel-dev boot
```

> The first `citadel-dev` boot usually takes a while due to the initial setup and configuration of the VM. Subsequent boots are much faster.

After the VM has booted, we can verify if the Citadel dashboard is accessible at http://citadel-dev.local in our browser to make sure everything is running fine.

3\. We need to switch the Citadel installation on `citadel-dev` to our fork and branch:

```sh
cd runcitadel/compose-nonfree
git remote add <username> git@github.com:<username>/compose-nonfree.git
git fetch <username> btc-rpc-explorer
git checkout <username>/btc-rpc-explorer
```

4\. And finally, it's time to install our app:

```sh
citadel-dev app install btc-rpc-explorer
```

That's it! Our BTC RPC Explorer app should now be accessible at http://citadel-dev.local:3002

5\. To make changes:

Edit your app files at `runcitadel/compose-nonfree/apps/<app-id>/` and then run:

```sh
citadel-dev reload
```

Once you're happy with your changes, just commit and push.

>Don't forget to shutdown the `citadel-dev` virtual machine after testing with `citadel-dev shutdown`!
 -->

### 3.2 Testing on Citadel OS (Raspberry Pi 4)

1\. We'll first install and run Citadel OS on a Raspberry Pi 4. After installation, we'll set it up on http://citadel.local, and then SSH into the Pi:

```sh
ssh citadel@citadel.local
```

(SSH password is the same as your Citadel's dashboard password)

2\. Next, we'll switch the Citadel installation to our fork and branch:

```sh
sudo ~/citadel/scripts/update/update --repo <username>/compose-nonfree#btc-rpc-explorer
```

3\. Once the installation has updated, it's time to test our app:

```sh
app/app-manager.py install btc-rpc-explorer
```

The app should now be accessible at http://citadel.local:3002

4\. To uninstall:

```sh
app/app-manager.py uninstall btc-rpc-explorer
```

> When testing your app, make sure to verify that any application state that needs to be persisted is in-fact being persisted in volumes.
>
> A good way to test this is to restart the app with `app/app-manager.py restart <app-id>`. If any state is lost, it means that state should be mapped to a persistent volume.
>
> When stopping/starting the app, all data in volumes will be persisted and anything else will be discarded. When uninstalling/installing an app, even persistent data will be discarded.

___

## 4. 🚀&nbsp;&nbsp;Submitting the app

We're now ready to open a pull request on the main [runcitadel/compose-nonfree](https://github.com/runcitadel/compose-nonfree) repo to submit our app. Let's copy-paste the following markdown for the pull request description, fill it up with the required details, and then open a pull request.

```
# App Submission

### App name
...

### Version
...

### One line description of the app
_(max 50 characters)_

...

### Summary of the app
_(50 to 200 words)_

...

### Developer name
...

### Developer website
...

### Source code link
_(Link to your app's source code repository.)_

...

### Support link
_(Link to your Telegram support channel, GitHub issues/discussions, support portal, or any other place where users could contact you for support.)_

...

### Requires
- [ ] Bitcoin Core
- [ ] Electrum server
- [ ] LND

### 256x256 SVG icon
_(Submit an icon with no rounded corners as it will be dynamically rounded with CSS. GitHub doesn't allow uploading SVGs directly, so please upload your icon to an alternate service, like https://svgur.com, and paste the link below.)_

...

### Gallery images
_(Upload 3 to 5 high-quality gallery images (1440x900px) of your app in PNG format, or just upload 3 to 5 screenshots of your app and we'll help you design the gallery images.)_

...


### I have tested my app on:
<!-- - [ ] [dev environment](https://github.com/runcitadel/compose-nonfree-dev)-->
- [ ] [Citadel OS on a Raspberry Pi 4](https://github.com/runcitadel/os)
- [ ] [Custom Citadel install on Linux](https://github.com/runcitadel/compose-nonfree#-installation)
```

This is where the above information is used when the app goes live in the App Store:

![App Store Labels](https://i.imgur.com/0CorPRK.png)


> After you've submitted your app, we'll review your pull request, create the required Tor hidden services for it, make some adjustments in the `docker-compose.yml` file, such as removing any port conflicts with other apps, pinning Docker images to their sha256 digests, assigning unique IP addresses to the containers, etc before merging.

🎉 Congratulations! That's all you need to do to package, test and submit your app to Citadel. We can't wait to have you onboard!


---

## FAQs

1. **How to push app updates?**

    Every time you release a new version of your app, you should build, tag and push the new Docker images to Docker Hub. Then open a new PR on our main repo (runcitadel/compose-nonfree) with your up-to-date docker image. Once we merge your PR, the app will be automatically rolled out to all users.

1. **How do users install apps?**

    Users install apps via the App Store. They do not use the `app/app-manager.py` CLI directly as it's only meant for development use.

1. **I need help with something else?**

    Join our [discord server](https://discord.gg/6U3kM2cjdB) to get help with development.
