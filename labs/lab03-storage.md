# Lab 3: Docker Storage, Volumes, and Bind Mounts
## What this lab covers

Containers are ephemeral by design. When a container is removed, everything written to its writable layer disappears with it. That is fine when the container is stateless, but most real workloads need to persist data beyond a single container's lifetime: database files, upload directories, configuration that changes at runtime, logs that need to survive a crash.

Docker solves this with two mechanisms. **Volumes** are managed by Docker itself: Docker creates and owns the storage directory on the host, tracks it by name, and can attach it to any number of containers. **Bind mounts** let you mount a specific path from the host filesystem directly into a container, giving the container access to files that already exist on the host and letting both sides see changes in real time.

In this lab you will work through both mechanisms, observe that data survives across separate containers, and build intuition for when to reach for each one.

## Prerequisites

- You are logged in to your EC2 instance as your assigned user (for example, `student01`).
- Labs 1 and 2 are complete.
- You can run `docker info` without `sudo`. If it fails with a permission error, ask your instructor.

---

## Part 1: Named Volumes

### Step 1: Create a named volume

```console
docker volume create lab-data
```

Example output:

```console
lab-data
```

Docker created a managed storage area on the host. You do not need to know or care where it lives on disk. Docker tracks it by name.

Confirm it exists:

```console
docker volume ls
```

You should see `lab-data` in the list with driver `local`.

### Step 2: Run a container and mount the volume

Start a `busybox` container with `lab-data` mounted at `/app` inside the container:

```console
docker run --rm -v lab-data:/app busybox sh -c "for i in \$(seq 1 50); do echo \"Line \$i\" >> /app/lines; done"
```

This command runs a shell inside `busybox`, loops from 1 to 50, and appends each line to `/app/lines`. When the loop finishes the container exits and is removed (`--rm`). The file lives in the volume, not in the container layer.

No output is expected. If the command exits without error, it succeeded.

### Step 3: Verify the file from a second container

Run a brand new container, attach the same volume at `/app2`, and count the lines:

```console
docker run --rm -v lab-data:/app2 busybox wc -l /app2/lines
```

Example output:

```console
50 /app2/lines
```

This second container has never existed before. It has no relationship to the first container other than sharing the same volume. The 50 lines are there because the volume persisted them.

### Why did data survive?

When the first container wrote to `/app/lines`, it was writing to a directory on the host that Docker manages. Removing the container discarded the container's writable layer but left the volume untouched. The second container got a fresh writable layer of its own, but `/app2` was mounted from the same host directory. Docker volumes are independent of any individual container's lifecycle.

This is the core model: volumes outlive containers. You can remove every container and the volume still holds your data. You can attach the same volume to ten containers simultaneously if you need to.

---

## Part 2: Bind Mounts

### Step 4: Create a directory on the host

```console
mkdir -p /tmp/bind-test
```

Verify it exists and is empty:

```console
ls -la /tmp/bind-test
```

### Step 5: Write a file using a bind-mounted container

Run a `busybox` container with `/tmp/bind-test` on the host mounted to `/data` inside the container:

```console
docker run --rm -v /tmp/bind-test:/data busybox sh -c "for i in \$(seq 1 50); do echo \"Line \$i\" >> /data/lines; done"
```

### Step 6: Read the file directly from the host

Because this is a bind mount, you can read the file from the host without starting another container:

```console
wc -l /tmp/bind-test/lines
```

Example output:

```console
50 /tmp/bind-test/lines
```

```console
cat /tmp/bind-test/lines
```

You will see all 50 lines. The container wrote to the host filesystem directly through the bind mount.

### Step 7: Confirm access from a second container

Just as with the volume, you can attach the same host directory to a new container:

```console
docker run --rm -v /tmp/bind-test:/data2 busybox wc -l /data2/lines
```

Example output:

```console
50 /data2/lines
```

---

## Volumes vs. Bind Mounts: When to Use Each

| Concern | Named Volume | Bind Mount |
|---|---|---|
| Docker manages the storage location | Yes | No, you specify the host path |
| Works the same on any host / in CI | Yes | No, depends on host directory existing |
| Easy to inspect or edit from the host | Harder (need to know Docker's path) | Trivial, it is a normal directory |
| Share host config or source code into a container | Poor fit | Ideal (dev mode, config injection) |
| Production database or application data | Recommended | Acceptable but less portable |
| Back up with `docker volume` commands | Yes | No, use host backup tools |
| Permissions controlled by Docker | Yes | No, inherits host filesystem permissions |

**When to use a named volume:** persistent application data (databases, uploaded files, caches) that should survive container replacements and not depend on host directory layout. Volumes are the right default for anything you care about keeping.

**When to use a bind mount:** injecting files that already exist on the host into a container. Common cases are mounting source code during local development so the running container sees edits in real time, or injecting a config file (like `nginx.conf`) without baking it into the image.

---

## Cleanup

Remove the volume when you are done:

```console
docker volume rm lab-data
```

Remove the bind mount directory:

```console
rm -rf /tmp/bind-test
```

---
