# config-faas-cluster

## Overview

These are the instructions for deploying a DiSARM OpenFaas cluster from scratch.

The main components are split into 2 stacks:

1. `rp` - the basic reverse proxy and management (`docker-compose.yml`)
   - [Portainer](https://www.portainer.io/): container management GUI
   - [Traefik](https://docs.traefik.io/): reverse proxy
   - [CORS-Anywhere](https://github.com/Rob--W/cors-anywhere): enable easy function-running from browsers
2. `func` - the OpenFaaS installation (`openfaas-docker-compose.yml`)
   - [OpenFaaS](https://www.openfaas.com/): serverless functions
   - [Squid](http://www.squid-cache.org/): caching proxy

If you're using the Portainer agent, this might setup a 3rd stack.

## Setup

### Prepare server

1. Create new server(s)! Needs to be running Docker. Everything below is to be run on the _manager_ node.
1. Ensure firewall is open for required ports: see [Firewall](#Firewall) below
1. Clone and `cd` into this repo: `git clone https://github.com/superuser-chi/my_stack`
1. Ensure `docker-compose.yml`, `openfaas-docker-compose.yml` and `traefik.toml` contain references to the correct domain/subdomains.

### Configure Docker Swarm mode

1. Get the internal IP of the machine (e.g. from Google VM dashboard) or external (ideally is static)
1. Create a swarm with address from previous step: `docker swarm init --advertise-addr XXX.XXX.XXX.XXX`
1. Note down the 'join a swarm' command from step above - will be needed to add more nodes to the cluster

### Create Docker prerequisites

1. Create network `traefik-net` with overlay driver: `docker network create -d overlay --attachable traefik-net`
1. Add secrets for openfaas:
   ```sh
   echo "admin" | docker secret create basic-auth-user -
   echo "verysecretpassword" | docker secret create basic-auth-password -
   ```

### Start (or redeploy) the stacks

1. Start the `rp` proxy stack: `docker stack deploy -c docker-compose.yml rp`
1. Start the `func` _OpenFaas_ stack: `docker stack deploy -c faas.yml func`
1. Add Squid secret: add a _secret_ called `ssl-cert`, to allow Squid's SSL certificate to be accesssed by functions: `docker secret create ssl-cert ./squid/cert/private.pem`.
   - **NOTE** This Must be run after certificate is created in `func` (OpenFaas) stack.
   - If this fails, give `squid` a little time to startup.
   - If redeploying the `func` stack, you **MUST** re-create the Squid secret

### Set default Docker logging to GCP

This is to get all container logs showing in GCP log viewer.

Need to repeat this for _every node in a cluster_.

[Source](https://cloud.google.com/community/tutorials/docker-gcplogs-driver)

1. Create /etc/docker/daemon.json.
   ```sh
   echo '{"log-driver":"gcplogs"}' | sudo tee /etc/docker/daemon.json
   ```
1. Restart the docker service.
   ```sh
   sudo systemctl restart docker
   ```

### Configure custom GCP logging for `squid`

To ingest custom logs with Google Cloud Platform logging.

1. Install the [Stackdriver logging agent ](https://cloud.google.com/monitoring/agent/install-agent) by running the commands:

   ```sh
   curl -sSO https://dl.google.com/cloudagents/install-logging-agent.sh
   sudo bash install-logging-agent.sh
   ```

1. Create a symbolic link from `squid_fluentd.conf` to the logging agent directory by running
   ```bash
   sudo ln -s /home/disarm/config-faas-cluster/squid_fluentd.conf /etc/google-fluentd/config.d/squid_fluentd.conf
   ```
1. Restart the agent with
   ```bash
   sudo service google-fluentd restart
   ```

### Confirm is alive

1. Confirm https://traefik.srv.disarm.io is live and reachable, with a couple of _Frontends_ and a couple of _Backends_
1. Visit https://port.srv.disarm.io to create initial username and password

## (Simple approach) Re-deploy functions from one installation to another

For example, to deploy all functions in `server1` to `server2`:

```
curl 'https://faas.server1.disarm.io/system/functions' -H 'authority: faas.server1.disarm.io' -H 'authorization: Basic dG9wc2VjcmV0dXNlcjphbmV3c2VjcmV0cGFzc3dvcmQ=' -H 'accept: application/json' > functions.json

# Deploy each with scaling
< functions.json | jq -r '.[] | .name,.image' | parallel -n 2 --dry-run 'faas deploy --image={2} --name={1} --gateway=https://faas.server2.disarm.io -l com.openfaas.scale.zero=true'

# Invoke each, to trigger the scaling-to-zero
< functions.json | jq -r '.[] | .name,.image' | parallel -n 2 --dry-run 'echo "" | faas invoke {1} --gateway=https://faas.server2.disarm.io'

```

### !! Limitations with this approach

It ignores any function-specific config which is contained in the `stack.yml` for that function. For example, if the function needs to use the Squid cache, or has a unique timeout.

Useful additional params might be `-e combine-output=false` and `-e write_timeout=300 -e read_timeout=300 -e exec_timeout=300`

## Development on another server

Useful to run something like `sed -i '' 's/srv/srv4/g' *.{yml,toml}` to quickly make changes on a remote server.

## Squid caching proxy

We use Squid to optionally cache any HTTP/HTTPS requests from deployed functions.

Any function wanting to make use of this will need to include the following properties in the `stack.yml`:

```yaml
provider: ...
functions:
  function-name:
    ...
    secrets:
      - ssl-cert
    environment:
      http_proxy: squid:3128
      https_proxy: squid:4128
      CURL_CA_BUNDLE: /var/openfaas/secrets/ssl-cert
```

## Firewall

```
22/tcp
2376/tcp
2377/tcp
7946/tcp
7946/udp
4789/udp
```

At least 2377, 4789, 7946 (from [here](https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-centos-7)) and also maybe list [here](https://gist.github.com/BretFisher/7233b7ecf14bc49eb47715bbeb2a2769). Also possibly 9001 for [Portainer Agent](https://portainer.readthedocs.io/en/stable/agent.html#connecting-an-existing-portainer-instance-to-an-agent)

## Cleanup

1. Once confirmed stack is up and Traefik is running fine, remove the port entry from `docker-compose.yml` to avoid leaving traefik dashboard exposed, and redeploy stack as above(`docker stack deploy...`).

## Monitoring

1. Optionally, add the Portainer Agent stack from the _app templates_, to help Portainer see what's happening on all nodes in the cluster.
