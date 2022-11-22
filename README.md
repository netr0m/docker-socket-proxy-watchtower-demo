# Automated container updates using `docker-socket-proxy` + `watchtower`

This repository provides an example of using [`Watchtower`](https://containrrr.dev/watchtower/) to automate the process of automating containers when new base images are released, proxying requests to the Docker socket via a [`docker-socket-proxy`](https://github.com/Tecnativa/docker-socket-proxy) container to limit the scope of available APIs.

## Prerequisites

> :warning: Note: If the host you're running on has `SELinux` or `AppArmor`, you may need to run the `docker-socket-proxy` container with the `--privileged` flag.

### Environment variables

If the container you wish to automatically update is stored in a private container registry, the following environment variables are required:
```env
REGISTRY_USER=<username>
REGISTRY_PASS=<password or token>
```

### Dependencies

#### OS dependencies
- [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [docker](https://docs.docker.com/get-docker/)

#### Python dependencies
##### [Optional] Create a virtual environment
```bash
# Install the virtualenv package
pip3 install virtualenv
# Create a virtual environment
virtualenv .venv
# Activate the virtual environment
source .venv
```
##### Dependencies
```bash
pip3 install -r requirements.txt
```

#### Ansible dependencies
##### Collections

```bash
ansible-galaxy collection install -r ansible-requirements.yml
```

## Usage

The repository provides an `Ansible Playbook` - see [`autoupdates.yml`](./autoupdates.yml)

To run the playbook, issue the following command:
```bash
ansible-playbook -i localhost autoupdates.yml
```

### Playbook description
The repository provides an Ansible Playbook which handles
1. Setting the necessary `variables`
2. Pulls the required Docker `images`
3. Creates a dedicated Docker `network` for the `socket-proxy` and `watchtower` `containers`
4. Starts the `socket-proxy` and `watchtower` `containers`
5. Pulls an example Docker `image`, `alpine:3.13`, and creates a new `tag` of this image as `latest`
    - i.e. `docker image tag alpine:3.13 alpine:latest`
    - This is done to ensure that our local image, `alpine:latest`, has a different `hash` from the remote image (on `DockerHub`), which will trigger an update
6. Starts the example Docker `image` `alpine:latest`, which will be automatically updated once `Watchtower` is triggered.

#### Manual run (without Ansible)
```bash
# Create environment variable files
cp socket-example.env socket.env
cp tower-example.env tower.env

# Create a Docker network
docker network create \
    -d bridge \
    --internal \
    socket-proxy-net

# Start the docker-socket-proxy container
docker run -d \
    --name socket-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --env-file socket.env \
    --network socket-proxy-net \
    ghcr.io/tecnativa/docker-socket-proxy:0.1.1

# Start the watchtower container
docker run -d \
    --name watchtower \
    --env-file tower.env \
    --network socket-proxy-net \
    --link socket-proxy \
    ghcr.io/containrrr/watchtower:1.5.1

# Pull the alpine:3.13 image (or any image, really)
docker pull alpine:3.13
docker image tag alpine:3.13 alpine:latest

# Start the example container to automatically update
docker run -d \
    --name api \
    --restart unless-stopped \
    --label com.centurylinklabs.watchtower.enable=true \
    alpine:latest /bin/sh -c 'while true; do sleep 1; done'

# Watch the logs of `watchtower`
docker logs -f watchtower
```
