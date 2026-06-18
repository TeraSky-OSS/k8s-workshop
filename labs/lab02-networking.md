# Lab 2: Docker Networking
In this lab you will create an isolated Docker network, connect containers to it, and observe how network isolation works in practice. Understanding container networking is essential because it is the foundation of service-to-service communication in any containerized system.

---

## Prerequisites

- You are logged in to your assigned EC2 instance as your student user (for example, `student01`).
- You completed Lab 1 and are comfortable running and stopping containers.
- Docker CE is pre-installed and your user is in the `docker` group. No `sudo` is needed.

---

## Part 1: Create a Custom Network

### Step 1: List existing networks

Before creating anything, check what networks Docker already provides.

```console
docker network ls
```

Example output:

```console
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local
def456abc789   host      host      local
fedcba987654   none      null      local
```

Docker ships with three built-in networks. The `bridge` network is the default: any container you start without specifying a network lands here. The `host` network removes isolation entirely and shares the host's network stack. The `none` network disables networking altogether.

### Step 2: Create a custom bridge network

```console
docker network create lab-network
```

Example output:

```console
7f3c2a1b9d4e5f6789abc123def4567890abcdef1234567890abcdef12345678
```

Docker returns the full ID of the newly created network. A custom bridge network is functionally similar to the default `bridge`, but with one important difference: containers on the same custom network can resolve each other by container name. Containers on the default `bridge` cannot.

### Step 3: Confirm the network exists

```console
docker network ls
```

You should now see `lab-network` in the list:

```console
NETWORK ID     NAME          DRIVER    SCOPE
abc123def456   bridge        bridge    local
def456abc789   host          host      local
7f3c2a1b9d4e   lab-network   bridge    local
fedcba987654   none          null      local
```

---

## Part 2: Run nginx on the Custom Network

### Step 4: Start an nginx container on lab-network

```console
docker run -d --name web --network lab-network nginx:alpine
```

Example output:

```console
Unable to find image 'nginx:alpine' locally
alpine: Pulling from library/nginx
...
Status: Downloaded newer image for nginx:alpine
c9a1b2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1
```

The flags do the following:

- `-d` runs the container in the background (detached mode).
- `--name web` gives the container a predictable name so you can reference it by name instead of ID.
- `--network lab-network` attaches it to your custom network instead of the default `bridge`.

### Step 5: Verify nginx is running

```console
docker ps
```

You should see the `web` container with status `Up`:

```console
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
c9a1b2d3e4f5   nginx:alpine   "/docker-entrypoint.…"   5 seconds ago   Up 4 seconds   80/tcp    web
```

Notice that no port is published to the host (`PORTS` is empty). The container listens on port 80 inside the Docker network, but nothing on the host can reach it yet. That is fine: the goal of this lab is container-to-container communication, not host-to-container access.

---

## Part 3: Run busybox on the Default Bridge

### Step 6: Start a busybox container on the default bridge

```console
docker run -d --name tester busybox sleep 3600
```

Example output:

```console
a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2
```

This container uses the default `bridge` network because you did not specify `--network`. The `sleep 3600` command keeps it alive for one hour so you can run commands inside it.

### Step 7: Inspect both containers' networks

Check which network the `tester` container is on:

```console
docker inspect tester --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

You will see `bridge` (the default) listed as the only network. Now check `web`:

```console
docker inspect web --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

You will see `lab-network` listed. The two containers are on different networks.

---

## Part 4: Confirm Isolation

### Step 8: Try to reach nginx from tester

Run `wget` inside the `tester` container, targeting the `web` container by name:

```console
docker exec tester wget -qO- --timeout=3 http://web
```

Example output:

```console
wget: bad address 'web'
```

The name `web` does not resolve from the default `bridge` network. DNS-based container discovery only works within a custom network.

### Step 9: Try by IP address

Find nginx's IP address on `lab-network`:

```console
docker inspect web --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Example output (your IP will differ):

```console
172.18.0.2
```

Now try to reach nginx by IP from `tester`:

```console
docker exec tester wget -qO- --timeout=3 http://172.18.0.2
```

Example output:

```console
wget: download timed out
```

The request times out. IP address routing across different bridge networks is blocked by Docker's default iptables rules. The two networks are isolated at the network layer, not just at the DNS layer.

This is the core lesson: containers on different networks cannot communicate by default, regardless of whether you use a name or an IP address.

---

## Part 5: Connect tester to lab-network

### Step 10: Attach tester to lab-network

A container can belong to more than one network at the same time. Connect `tester` to `lab-network` without stopping it:

```console
docker network connect lab-network tester
```

This command produces no output on success.

### Step 11: Verify tester is now on both networks

```console
docker inspect tester --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
```

You should see two entries: `bridge` and `lab-network`. The container now has a network interface on each network.

---

## Part 6: Confirm Communication

### Step 12: Reach nginx by container name

```console
docker exec tester wget -qO- http://web
```

Example output (the nginx welcome page HTML):

```console
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

The request succeeds this time. Because both containers are now on the same custom network (`lab-network`), Docker's embedded DNS server can resolve the name `web` to nginx's IP address on that network.

### Step 13: Reach nginx by IP address

You can also confirm that the IP address route now works:

```console
docker exec tester wget -qO- http://172.18.0.2
```

The same HTML page appears.

---

## Cleanup

Remove the containers and network when you are done:

```console
docker rm -f web tester
docker network rm lab-network
```

Example output:

```console
web
tester
lab-network
```

---

## What You Did

1. Created a custom bridge network.
2. Started nginx attached to that network.
3. Started a busybox container on the default bridge (a different network).
4. Confirmed that isolation is enforced: the default bridge cannot reach `lab-network` by name or by IP.
5. Connected busybox to `lab-network` at runtime.
6. Confirmed that name-based DNS resolution works once both containers share a custom network.

The key takeaway: the **default bridge** network does not support container name resolution. Custom networks do. In production systems, you create an explicit network for each logical group of services so they can find each other by name and remain isolated from everything else.

---
