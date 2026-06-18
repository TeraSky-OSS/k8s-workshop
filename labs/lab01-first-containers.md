# Lab 1: Your First Containers

In this lab you pull images from Docker Hub, run containers interactively and in the background, and learn the core lifecycle commands: inspect, pause, unpause, logs, stats, and stop. By the end you will have a solid feel for how containers start, run, and get cleaned up.
---

## Prerequisites

- You are logged in to your Ubuntu 24.04 EC2 instance as your assigned user (e.g. `student01`).
- Docker is pre-installed and your user is already in the `docker` group, so no `sudo` is needed for any `docker` command.

Verify Docker is accessible:

```console
docker version
```

You should see both a Client and a Server section. If you see a permission error, log out and log back in so the group membership takes effect.

---

## Part 1: Pull images

Docker images are read-only templates stored in a registry. Pulling downloads them to your local machine so containers can be created from them.

### Step 1.1: Pull the nginx image

```console
docker pull nginx
```

Example output (versions may differ):

```console
Using default tag: latest
latest: Pulling from library/nginx
Digest: sha256:...
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
```

Docker pulled the `latest` tag of the official `nginx` image from Docker Hub. The image is now stored locally.

### Step 1.2: Pull the busybox image

```console
docker pull busybox
```

`busybox` is a minimal Linux image, around 5 MB. You will use it in the next part for interactive exploration.

### Step 1.3: Confirm both images are available locally

```console
docker images
```

Example output:

```console
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    <id>           <time ago>     192MB
busybox      latest    <id>           <time ago>     4.27MB
```

---

## Part 2: Run a container interactively

The `-it` flags attach your terminal to the container so you can type commands inside it. This is the most direct way to explore a container's filesystem and environment.

### Step 2.1: Start an interactive busybox shell

```console
docker run -it busybox sh
```

Your prompt changes to something like `/ #`. You are now inside the container.

### Step 2.2: Explore the container environment

Run a few commands to see what is available:

```console
hostname
```

```console
cat /etc/hostname
```

```console
ls /
```

```console
ps aux
```

Notice how minimal the process list is. Only your shell and `ps` are running. The container sees its own isolated filesystem and process namespace, completely separate from the host.

### Step 2.3: Exit the container

```console
exit
```

Your prompt returns to the host. Because you exited the only process in the container, the container stopped automatically. This is a fundamental Docker behavior: a container runs as long as its main process runs.

---

## Part 3: List running and stopped containers

### Step 3.1: List running containers

```console
docker ps
```

The busybox container you just exited is not in this list because it stopped.

### Step 3.2: List all containers, including stopped ones

```console
docker ps -a
```

Example output:

```console
CONTAINER ID   IMAGE     COMMAND   CREATED         STATUS                     PORTS     NAMES
<id>           busybox   "sh"      1 minute ago    Exited (0) 1 minute ago              <random_name>
```

The `Exited (0)` status means the container's main process ended with exit code 0 (success). The container still exists on disk. It is not running, but it has not been deleted.

---

## Part 4: Run a named nginx container in the background

The `-d` flag (detach) starts the container in the background. The `--name` flag gives it a human-readable name you can use in later commands instead of the container ID.

### Step 4.1: Start nginx detached

```console
docker run -d --name web -p 8080:80 nginx
```

The `-p 8080:80` flag maps port 8080 on your host to port 80 inside the container, so nginx is reachable from outside.

Docker prints a container ID and returns your prompt immediately.

### Step 4.2: Confirm it is running

```console
docker ps
```

Example output:

```console
CONTAINER ID   IMAGE   COMMAND                  CREATED         STATUS         PORTS                  NAMES
<id>           nginx   "/docker-entrypoint.…"   5 seconds ago   Up 5 seconds   0.0.0.0:8080->80/tcp   web
```

The `STATUS` column shows `Up`, and the `PORTS` column shows the port mapping.

### Step 4.3: Make a test request to nginx

```console
curl http://localhost:8080
```

You should see the nginx welcome page HTML. This confirms the container is serving traffic through the port mapping.

---

## Part 5: Inspect, pause, and unpause the container

### Step 5.1: Inspect the container

`docker inspect` returns a detailed JSON document with everything Docker knows about the container: network settings, mounts, environment variables, restart policy, and more.

```console
docker inspect web
```

The output is long. To extract a specific field, use the `--format` flag with Go template syntax:

```console
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
```

This prints the container's internal IP address on the Docker bridge network.

### Step 5.2: Pause the container

Pausing freezes all processes in the container using Linux's `cgroup freezer`. The container is still alive, but no code is executing.

```console
docker pause web
```

Check the status:

```console
docker ps
```

The `STATUS` column now shows `Up X seconds (Paused)`.

Make a request while paused:

```console
curl --max-time 3 http://localhost:8080
```

The request times out because nginx is frozen and cannot respond.

### Step 5.3: Unpause the container

```console
docker unpause web
```

Check the status again:

```console
docker ps
```

Status is back to `Up`. Make another request:

```console
curl http://localhost:8080
```

nginx responds immediately. Pausing and unpausing is useful for freezing a container to take a consistent snapshot or to throttle activity temporarily.

---

## Part 6: Logs, stats, and the process list

### Step 6.1: View container logs

```console
docker logs web
```

You see the nginx access log entries from the `curl` requests you made. Docker captures everything the container writes to stdout and stderr and stores it as the container's log.

To follow the log in real time (similar to `tail -f`), use:

```console
docker logs -f web
```

Press `Ctrl+C` to stop following.

### Step 6.2: View live resource stats

```console
docker stats web
```

Example output (refreshed every second):

```console
CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O       BLOCK I/O   PIDS
<id>           web    0.00%   7.5MiB / 7.7GiB     0.09%   1.2kB / 0B    0B / 0B     5
```

This shows real-time CPU, memory, network I/O, and disk I/O for the container. Press `Ctrl+C` to exit.

### Step 6.3: List the processes running inside the container

```console
docker top web
```

Example output:

```console
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                77617               77594               0                   13:10               ?                   00:00:00            nginx: master process nginx -g daemon off;
message+            77715               77617               0                   13:10               ?                   00:00:00            nginx: worker process
message+            77716               77617               0                   13:10               ?                   00:00:00            nginx: worker process
message+            77718               77617               0                   13:10               ?                   00:00:00            nginx: worker process
message+            77719               77617               0                   13:10               ?                   00:00:00            nginx: worker process
```

`docker top` shows the processes inside the container from the host's perspective. Notice that nginx runs as both a master process and one or more worker processes. These processes exist in the host's PID namespace but in a separate, namespaced view inside the container.

---

## Part 7: Stop and clean up

### Step 7.1: Stop the nginx container

```console
docker stop web
```

`docker stop` sends `SIGTERM` to the container's main process and gives it 10 seconds to shut down gracefully. If it does not exit in time, Docker sends `SIGKILL`.

### Step 7.2: Confirm it is stopped

```console
docker ps -a
```

The `web` container shows `Exited (0)`.

### Step 7.3: Remove the stopped containers

Remove the nginx container by name:

```console
docker rm web
```

List all containers again:

```console
docker ps -a
```

You still see the busybox container from Part 2. Remove it by its container ID (use the actual ID from your output):

```console
docker rm <container_id>
```

Or remove all stopped containers at once:

```console
docker container prune
```

Docker asks for confirmation. Type `y` and press Enter.

### Step 7.4: Verify all containers are gone

```console
docker ps -a
```

The output should be empty (only the header row).

---

## Summary

You have completed the following:

1. Pulled `nginx` and `busybox` images from Docker Hub.
2. Ran an interactive shell inside a `busybox` container and observed its isolated environment.
3. Listed running and stopped containers with `docker ps` and `docker ps -a`.
4. Started a named `nginx` container in detached mode with a port mapping.
5. Inspected the container with `docker inspect`, paused and unpaused it.
6. Viewed logs with `docker logs`, live stats with `docker stats`, and the process list with `docker top`.
7. Stopped the container and removed all stopped containers.

---
