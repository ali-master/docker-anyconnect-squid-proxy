# openconnect + squid proxy

This Docker image contains an [openconnect client](http://www.infradead.org/openconnect/) (version 8.04 with pulse/juniper support) and the [squid proxy server](https://ubuntu.com/server/docs/proxy-servers-squid) for http/https connections.

# Requirements

If you don't want to set the environment variables on the command line
set the environment variables in a `.env` file:
```env
OPENCONNECT_URL=<Gateway URL>
OPENCONNECT_USER=<Username>
OPENCONNECT_PASSWORD=<Password>
OPENCONNECT_OPTIONS=--authgroup <VPN Group> \
	--servercert <VPN Server Certificate> --protocol=<Protocol> \
	--reconnect-timeout 86400
```

_Don't use quotes around the values!_

See the [openconnect documentation](https://www.infradead.org/openconnect/manual.html) for available options. 

Either set the password in the `.env` file or leave the variable `OPENCONNECT_PASSWORD` unset, so you get prompted when starting up the container.

Optionally set a multi factor authentication code:
```env
OPENCONNECT_MFA_CODE=<Multi factor authentication code>
```

# Run container in foreground

To start the container in foreground run:
```bash
docker run -it --rm --privileged --env-file=.env \
	-p 3306:3306 -p 8889:8889 alimaster/openconnect-squid-proxy:latest
```

The proxies are listening on ports 3306 (http/https). Either use `--net host` or `-p <local port>:3306 -p <local port>:8889` to make the proxy ports available on the host.

Without using a `.env` file set the environment variables on the command line with the docker run option `-e`:
```bash
docker run … -e OPENCONNECT_URL=vpn.gateway.com/example \
-e OPENCONNECT_OPTIONS='<Openconnect Options>' \
-e OPENCONNECT_USER=<Username> …
```

# Run container in background

To start the container in daemon mode (background) set the `-d` option:
```bash
docker run -d -it --rm …
```

In daemon mode you can view the stderr log with `docker logs`:
```bash
docker logs `docker ps | grep "alimaster/openconnect-squid-proxy" | awk -F' ' '{print $1}'`
```

# Use container with docker-compose
```yml
version: '3.2'

services:
  vpn:
    container_name: vpn
    image: alimaster/openconnect-squid-proxy:latest
    ports:
      - 0.0.0.0:3306:3306
    privileged: true
    env_file:
      - .env
    cap_add:
    - NET_ADMIN
```

Set the environment variables for _openconnect_ in the `.env` file again (or specify another file) and 
map the configured ports in the container to your local ports if you want to access the VPN 
on the host too when running your containers. Otherwise only the docker containers in the same
network have access to the proxy ports.

# Route traffic through VPN container

Let's say you have a `vpn` container defined as above, then add `network_mode` option to your other containers:
```yml
	depends_on:
	  - vpn
	network_mode: "service:vpn"
```

Keep in mind that `networks`, `extra_hosts`, etc. and `network_mode` are mutually exclusive!

# Configure proxy

The container is connected via _openconnect_ and now you can configure your browser
and other software to use one of the proxies (3306 for http/https).

For example FoxyProxy (available for Firefox, Chrome) is a suitable browser extension.

You may also set environment variables:
```bash
export http_proxy="http://127.0.0.1:3306/"
export https_proxy="http://127.0.0.1:3306/"
```
composer, git (if you don't use the git+ssh protocol, see below) and others use these.

# Build

You can build the container yourself with
```bash
docker build -f build/Dockerfile -t alimaster/openconnect-squid-proxy:custom ./build
```