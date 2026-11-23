

#### Build and publish this Compose image

This repository's Compose configuration builds and tags the Juice Shop image as
`fredndabikunze01/fred-ndabikunze-juice:v6.0`.

1. Install Docker with the Compose plugin and sign in to Docker Hub:

   ```sh
   docker login -u fredndabikunze01
   ```

   Enter your Docker Hub password or access token when prompted. Do not put credentials in this file or in shell
   commands.

2. From the repository directory, build and push the image:

   ```sh
   docker compose build juice-shop
   docker compose push juice-shop
   ```

   To build and push without Compose, use:

   ```sh
   docker build -t fredndabikunze01/fred-ndabikunze-juice:v6.0 .
   docker push fredndabikunze01/fred-ndabikunze-juice:v6.0
   ```

#### Run the image from Docker Hub with Compose

On the machine that will run the app, make the repository's `compose.yaml` and `nginx.conf` available in the same
directory, then run:

```sh
docker compose pull juice-shop
docker compose up -d --no-build
```

Compose starts Juice Shop together with NGINX and MongoDB. Open <http://localhost:8085> through NGINX or
<http://localhost:3001> directly. Check status and logs with `docker compose ps` and
`docker compose logs -f juice-shop`; stop the stack with `docker compose down`. The MongoDB data volume is preserved
when the stack is stopped.

> **Warning:** Juice Shop is intentionally vulnerable and is designed for local security training. Do not expose it
> or the published MongoDB port to the public internet.
