# Lab 5: Building, Tagging, and Pushing Images to a Private Registry
## What this lab is about

A **container registry** is a service that stores and distributes container images. It plays the same role for images that GitHub plays for source code: a central place where images are versioned, shared, and pulled from. Without a registry, images exist only on the machine that built them.

In this lab you will build a custom nginx-based image, tag it with a fully qualified name that includes the registry address, push it to a shared **Harbor** registry, pull the instructor's image to confirm the shared project is accessible, and inspect the layer history of an image. By the end, you have touched every part of the image lifecycle that matters in practice: build, tag, distribute, pull, and inspect.

This lab ties together everything from Labs 1-4. You wrote Dockerfiles, built images, ran containers, and worked with volumes. Now you distribute work the way real teams do.

---

## Before you start

Open a terminal and SSH into your assigned instance. Confirm the environment variables you will use throughout the lab:

```console
echo "Harbor host : $HARBOR_HOST"
echo "Username    : $USER"
```

Both must print a non-empty value. If `$HARBOR_HOST` is empty, ask your instructor. `$USER` is your OS login (e.g. `student01`).

---

## Step 1: Create the application files

Create a working directory for this lab:

```console
mkdir -p ~/lab05 && cd ~/lab05
```

Create `index.html`. Replace "Your Name" with your actual name and "Your Username" with your actual username:

```console
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My App</title>
</head>
<body>
  <h1>Hello from Your Name</h1>
  <p>Built and pushed by student: Your Username</p>
</body>
</html>
EOF
```

Edit the file to put your real name and username in:

```console
nano index.html
```

Change "Your Name" to your full name and "student01" to your actual username (`$USER`). Save with Ctrl+O, Enter, then Ctrl+X.

---

## Step 2: Write the Dockerfile

Create the Dockerfile in the same directory:

```console
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
CMD ["nginx", "-g", "daemon off;"]
EOF
```

Confirm the directory now contains both files:

```console
ls -l ~/lab05
```

Example output:

```
total 8
-rw-rw-r-- 1 student01 student01 xxx Dockerfile
-rw-rw-r-- 1 student01 student01 xxx index.html
```

---

## Step 3: Build and tag the image

Build the image and tag it with the full registry path in a single command:

```console
docker build -t $HARBOR_HOST/workshop/$USER/myapp:v1 ~/lab05
```

Docker resolves the tag format as `<registry>/<project>/<repository>:<tag>`. Harbor expects this structure: the registry hostname (`$HARBOR_HOST`), the project name (`workshop`), and then the repository name you choose (`$USER/myapp`).

Example output (abbreviated):

```
[+] Building 3.2s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/library/nginx:alpine
 => [1/2] FROM docker.io/library/nginx:alpine
 => [2/2] COPY index.html /usr/share/nginx/html/index.html
 => exporting to image
 => => naming to harbor.example.com/workshop/student01/myapp:v1
```

Verify the image is in your local image store:

```console
docker images $HARBOR_HOST/workshop/$USER/myapp:v1
```

---

## Step 4: Log in to the Harbor registry

Authenticate with your Harbor account. Your credentials are your OS username and the same password you use to log in to the instance:

```console
docker login $HARBOR_HOST
```

Docker prompts for your username and password:

```
Username: <your-username>
Password:
WARNING! Your password will be stored unencrypted in /home/student01/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

The "Login Succeeded" line confirms the credentials were accepted and a token was saved in `~/.docker/config.json`. All subsequent `docker push` and `docker pull` commands to this registry will use that token automatically.

The warning about unencrypted storage is expected in lab environments. In production you would use a credential helper or a CI secrets manager instead.

---

## Step 5: Push the image to Harbor

Push the image:

```console
docker push $HARBOR_HOST/workshop/$USER/myapp:v1
```

Example output:

```
The push refers to repository [harbor.example.com/workshop/student01/myapp]
a1b2c3d4e5f6: Pushed
f7e8d9c0b1a2: Pushed
...
v1: digest: sha256:abc123... size: 1234
```

Each line that says "Pushed" represents one image layer being uploaded. Layers that already exist on the registry (because another student pushed the same base image) will instead say "Layer already exists" and be skipped. This is the **content-addressable** deduplication that makes registries storage-efficient.

---

## Step 6: View the image in the Harbor web UI

Open a browser and go to `https://$HARBOR_HOST` (replace `$HARBOR_HOST` with the actual hostname your instructor gave you).

1. Click "Sign In" and log in with your credentials.
2. Click the "workshop" project.
3. Find the repository named `<your-username>/myapp`.
4. Click it to see the tag list. You should see `v1` with its digest, size, and push timestamp.

This is how operations teams audit what is deployed and when. The digest is a cryptographic fingerprint of the image content. If the digest matches on both ends, the image is identical.

---

## Step 7: Pull the instructor's image

The `workshop` Harbor project is shared across the classroom. Any authenticated student can pull from it. Your instructor pre-built and pushed an image at the start of the workshop. Pull it:

```console
docker pull $HARBOR_HOST/workshop/instructor/myapp:v1
```

Example output:

```
v1: Pulling from workshop/instructor/myapp
Digest: sha256:def456...
Status: Downloaded newer image for harbor.example.com/workshop/instructor/myapp:v1
```

Run it to see what's inside:

```console
docker run --rm -d -p 8081:80 --name instructor-demo $HARBOR_HOST/workshop/instructor/myapp:v1
sleep 2
curl http://localhost:8081
docker stop instructor-demo
```

You should see the instructor's page in the HTML output. This confirms the registry is accessible to all students and that images are genuinely portable.

---

## Step 8: Inspect image layers with docker image history

Inspect the layer history of your own image:

```console
docker image history $HARBOR_HOST/workshop/$USER/myapp:v1
```

Example output (columns may wrap):

```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
a1b2c3d4e5f6   2 minutes ago  CMD ["nginx" "-g" "daemon off;"]               0B        buildkit.dockerfile.v0
<missing>      2 minutes ago  COPY index.html /usr/share/nginx/html/index…   312B      buildkit.dockerfile.v0
<missing>      2 weeks ago    /bin/sh -c #(nop)  EXPOSE 80                   0B
<missing>      2 weeks ago    /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemo…   0B
<missing>      2 weeks ago    /bin/sh -c set -x ...                          5.63MB
...
```

What you are reading:

- Each row is one layer, listed newest-first.
- The top two rows come from your Dockerfile: the `CMD` instruction and the `COPY` instruction. The `COPY` row shows the size of `index.html`.
- The rows below are from the `nginx:alpine` base image. They are marked `<missing>` for the image ID because their layer metadata was not pulled to your machine, only the content.
- The `SIZE` column shows how much each layer adds to the total image size. Your two layers add almost nothing because the base image does the heavy lifting.

This view is useful for auditing: you can see exactly which instructions produced which layers, and spot unexpectedly large layers caused by, for example, leaving build tools or caches inside the image.

Now inspect the instructor's image:

```console
docker image history $HARBOR_HOST/workshop/instructor/myapp:v1
```

The base layers will be identical. Only the `COPY` layers differ. This is deduplication in action: those shared layers exist once on your disk and once on the registry, regardless of how many students pushed the same base.

---

## Cleanup

Remove the images from your local machine:

```console
docker rmi $HARBOR_HOST/workshop/$USER/myapp:v1
docker rmi $HARBOR_HOST/workshop/instructor/myapp:v1
cd
```

The images remain on the Harbor registry. You can pull them again any time.

---
