# Lab 4: Multi-Stage Docker Builds

## Overview

Every compiled application needs a toolchain to build it. That toolchain (compilers, linkers, SDKs) can be hundreds of megabytes. None of it belongs in the image you ship.

Multi-stage builds solve this. A single Dockerfile can describe multiple stages. Each stage starts fresh from a base image. A later stage copies specific files from an earlier one and leaves everything else behind.

In this lab you will build a minimal Go HTTP server twice: once naively (the entire Go toolchain ends up in the image) and once with a multi-stage build (only the compiled binary). The size difference makes the point better than any explanation.

In this lab you will:

- Write a tiny Go HTTP server
- Build a naive single-stage image to see the problem
- Build a two-stage image that ships only the binary
- Compare the sizes side-by-side and verify the app works

---

## Prerequisites

You completed Labs 1 through 3. Docker is already installed and your user is in the `docker` group.

---

## Step 1: Create a working directory

```console
mkdir /tmp/ms
cd /tmp/ms
```

---

## Step 2: Write the application

```console
cat > main.go <<'EOF'
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello from Lab 4")
    })
    http.ListenAndServe(":8080", nil)
}
EOF
```

```console
cat > go.mod <<'EOF'
module lab4

go 1.24
EOF
```

That is the entire application: a single handler that responds with a line of text, and a server that listens on port 8080.

---

## Step 3: Build a naive single-stage image

Write a Dockerfile that takes the obvious approach: start from the official Go image and build inside it.

```console
cat > Dockerfile.fat <<'EOF'
FROM golang:1.24
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["/app/server"]
EOF
```

Build it:

```console
docker build -f Dockerfile.fat -t go-fat .
```

This will pull `golang:1.24` if you don't have it locally. That download also caches the image for the next step.

Example output:

```console
[+] Building 58.8s (9/9) FINISHED
=> [internal] load build definition from Dockerfile.fat
=> => transferring dockerfile: 125B
=> [internal] load metadata for docker.io/library/golang:1.24
=> [internal] load .dockerignore
=> => transferring context: 2B
=> [1/4] FROM docker.io/library/golang:1.24@sha256:d2d2bc1c84f7e60d7d2438a3836ae7d0c847f4888464e7ec9ba3a1339a1ee804
=> => resolve docker.io/library/golang:1.24@sha256:d2d2bc1c84f7e60d7d2438a3836ae7d0c847f4888464e7ec9ba3a1339a1ee804
=> => sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1 32B / 32B
=> => sha256:50a27cd32f8983e7e43777ec65195fb6594ea877f31993fd555513c261ffc054 126B / 126B
=> => sha256:f7bdfd728ac2ad72d43b82689890dc698260d3a1049845f48fb3fb942df6c581 79.13MB / 79.13MB
=> => sha256:b2b04fcbed4bf6e5373e2607d2705704ec5b220f1d1306e06ab8fe9471b2f86a 102.14MB / 102.14MB
=> => sha256:b5e2021c4c8bd1a46b34d9608a9381afdc333600ee1ef3c94306ecf7373e1956 67.79MB / 67.79MB
=> => sha256:ef235bf1a09a237b896b69935c8c8d917c9c6a78b538724911414afc0a96763c 49.29MB / 49.29MB
=> => extracting sha256:ef235bf1a09a237b896b69935c8c8d917c9c6a78b538724911414afc0a96763c
=> => sha256:954d6059ca7bdbb9ceb566ca2239e01ef312165659d656753d7dbace7771a591 25.61MB / 25.61MB
=> => extracting sha256:954d6059ca7bdbb9ceb566ca2239e01ef312165659d656753d7dbace7771a591
=> => extracting sha256:b5e2021c4c8bd1a46b34d9608a9381afdc333600ee1ef3c94306ecf7373e1956
=> => extracting sha256:b2b04fcbed4bf6e5373e2607d2705704ec5b220f1d1306e06ab8fe9471b2f86a
=> => extracting sha256:f7bdfd728ac2ad72d43b82689890dc698260d3a1049845f48fb3fb942df6c581
=> => extracting sha256:50a27cd32f8983e7e43777ec65195fb6594ea877f31993fd555513c261ffc054
=> => extracting sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1
=> [internal] load build context
=> => transferring context: 447B
=> [2/4] WORKDIR /app
=> [3/4] COPY . .
=> [4/4] RUN go build -o server .
=> exporting to image
=> => exporting layers
=> => exporting manifest sha256:92547293536599645db1994e5b8cae10fd1c1f792c25e2ac65d2f6ae95f52a10
=> => exporting config sha256:72e6e02fbc449a19bc9604dac3769dd35b658d85252e93f81cf55832559c3727
=> => exporting attestation manifest sha256:3ce023b929ec3d21d05d544b551f8fea00ba72beb3f5f37215fd595cc18c73e6
=> => exporting manifest list sha256:9fe252d120694739488b10a3500fc1c1acca20da264817f7dd8559f217b762bc
=> => naming to docker.io/library/go-fat:latest
=> => unpacking to docker.io/library/go-fat:latest
```

---

## Step 4: Build the multi-stage image

Now write the same build, but with a second stage that takes only the compiled binary:

```console
cat > Dockerfile <<'EOF'
FROM golang:1.24 AS build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM scratch
COPY --from=build /app/server /server
CMD ["/server"]
EOF
```

Two things to note before building:

- `CGO_ENABLED=0` tells the Go compiler to produce a fully static binary with no C library dependency.
- `FROM scratch` is a special empty base image. No OS, no shell, no package manager. Just what you copy into it. A static binary is the only thing that can run here.

Build it:

```console
docker build -t multi-stage .
```

Example output:

```console
[+] Building 19.7s (10/10) FINISHED
=> [internal] load build definition from Dockerfile
=> => transferring dockerfile: 194B
=> [internal] load metadata for docker.io/library/golang:1.24
=> [internal] load .dockerignore
=> => transferring context: 2B
=> [internal] load build context
=> => transferring context: 279B
=> [build 1/4] FROM docker.io/library/golang:1.24@sha256:d2d2bc1c84f7e60d7d2438a3836ae7d0c847f4888464e7ec9ba3a1339a1ee804
=> => resolve docker.io/library/golang:1.24@sha256:d2d2bc1c84f7e60d7d2438a3836ae7d0c847f4888464e7ec9ba3a1339a1ee804
=> CACHED [build 2/4] WORKDIR /app
=> [build 3/4] COPY . .
=> [build 4/4] RUN CGO_ENABLED=0 go build -o server .
=> [stage-1 1/1] COPY --from=build /app/server /server
=> exporting to image
=> => exporting layers
=> => exporting manifest sha256:8ef944d22b5878a99cbdb825904dfe432843158eadf076ad07cbb1e0a28c98f3
=> => exporting config sha256:f25340daade95ae6633e6b0a0467a18b20ff53a6f0fabc1ddf2060bb31609fe1
=> => exporting attestation manifest sha256:b22aa5381ebb278d9990030da50b585c2ce42815b844bfe51a155cbfabc4dc6e
=> => exporting manifest list sha256:6aba91fb5ec958bd6f57153d1f2186722778ab34011ee3a05c3ac3cb5672a3a1
=> => naming to docker.io/library/multi-stage:latest
=> => unpacking to docker.io/library/multi-stage:latest
```

---

## Step 5: Compare the sizes

```console
docker image ls multi-stage
docker image ls go-fat
```

Example output:

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
multi-stage:latest   6aba91fb5ec9       12.8MB         4.62MB

IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
go-fat:latest   9fe252d12069       1.44GB          348MB
```

The same source code. Dramatically smaller (e.g. `4.62MB` vs `348MB`). The entire Go toolchain is gone.

This is the core idea: the build stage exists only to produce the binary. Once that binary is copied into the final stage, everything in the build stage is discarded. It never becomes part of the image you ship.

---

## Step 6: Run the multi-stage container

```console
docker run -d --name lab4 -p 8080:8080 multi-stage
```

Verify it is running:

```console
docker ps
```

Curl it:

```console
curl http://localhost:8080
```

Expected output:

```
Hello from Lab 4
```

---

## Step 7: Inspect the layer history

```console
docker image history multi-stage
```

Example output:

```
IMAGE          CREATED         CREATED BY                            SIZE      COMMENT
6aba91fb5ec9   3 minutes ago   CMD ["/server"]                       0B        buildkit.dockerfile.v0
<missing>      3 minutes ago   COPY /app/server /server # buildkit   8.16MB    buildkit.dockerfile.v0
```

Two layers. The binary and the CMD instruction. Nothing from the build stage appears here because `COPY --from=build` reaches across stages and pulls only the file you named.

Now compare:

```console
docker image history go-fat
```

You will see many more layers from the Go base image, totaling hundreds of megabytes.

Example output:

```console
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
9fe252d12069   6 minutes ago   CMD ["/app/server"]                             0B        buildkit.dockerfile.v0
<missing>      6 minutes ago   RUN /bin/sh -c go build -o server . # buildk…   98.3MB    buildkit.dockerfile.v0
<missing>      6 minutes ago   COPY . . # buildkit                             20.5kB    buildkit.dockerfile.v0
<missing>      6 minutes ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      4 months ago    WORKDIR /go                                     4.1kB     buildkit.dockerfile.v0
<missing>      4 months ago    RUN /bin/sh -c mkdir -p "$GOPATH/src" "$GOPA…   16.4kB    buildkit.dockerfile.v0
<missing>      4 months ago    COPY /target/ / # buildkit                      300MB     buildkit.dockerfile.v0
<missing>      4 months ago    ENV PATH=/go/bin:/usr/local/go/bin:/usr/loca…   0B        buildkit.dockerfile.v0
<missing>      4 months ago    ENV GOPATH=/go                                  0B        buildkit.dockerfile.v0
<missing>      4 months ago    ENV GOTOOLCHAIN=local                           0B        buildkit.dockerfile.v0
<missing>      4 months ago    ENV GOLANG_VERSION=1.24.13                      0B        buildkit.dockerfile.v0
<missing>      4 months ago    RUN /bin/sh -c set -eux;  apt-get update;  a…   287MB     buildkit.dockerfile.v0
<missing>      4 months ago    RUN /bin/sh -c set -eux;  apt-get update;  a…   202MB     buildkit.dockerfile.v0
<missing>      4 months ago    RUN /bin/sh -c set -eux;  apt-get update;  a…   65MB      buildkit.dockerfile.v0
<missing>      4 months ago    # debian.sh --arch 'amd64' out/ 'trixie' '@1…   134MB     debuerreotype 0.17
```

---

## Step 8: Clean up

```console
docker rm -f lab4
docker rmi multi-stage go-fat
cd
```

---

## Key takeaways

- Multi-stage builds let you use a full build environment and ship only the output. The build tools never appear in the final image.
- `COPY --from=<stage>` copies specific files from an earlier stage. Everything else in that stage is discarded.
- `FROM scratch` is the smallest possible base: an empty image. Combine it with a static binary (`CGO_ENABLED=0`) and you ship only what your application actually needs to run.
- A smaller image means a smaller attack surface: no shell, no package manager, no unused OS libraries means fewer things to patch and fewer tools available to an attacker who gains access to a container.

---
