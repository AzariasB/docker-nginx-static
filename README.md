# Custom Nginx Image

This is a fork of the excellent [https://github.com/docker-nginx-static/docker-nginx-static](https://github.com/docker-nginx-static/docker-nginx-static). With a few changes :

 - The image is distroless, meaning it only contains the nginx executables and necessary files for it to run. Thus it is not possible to connect to the container with a shell.
 - The gzip_static and gunzip modules are not installed. The reverse proxy should be in charge of doing the compression
 - The http_realip module is installed, to show the actual client ip in the logs instead of the proxy ip. The module needs to be configured with the `set_real_ip_from` config
 - Access logs and error logs are enabled by default
 - A custom "/healthz" route is added to have an easy healthcheck endpoint. This route has logs disabled for obvious reasons
 - The server tokens are completely disabled. This obviously doesn't stop from attacks, but at leat gives less informations about the server

```shell
docker run -v /absolute/path/to/serve:/static -p 8080:80 azariasb/nginx-static`
```

This command exposes an nginx server on port 80 which serves the folder `/absolute/path/to/serve` from the host.

The image can only be used for static file serving but has with **less than 4 MB** roughly 1/10 the size of the official nginx image. The running container needs **~1 MB RAM**.

### nginx-static via HTTPS

To serve your static files over HTTPS you must use another reverse proxy. We recommend [træfik](https://traefik.io/) as a lightweight reverse proxy with docker integration. Do not even try to get HTTPS working with this image only, as it does not contain the nginx ssl module.

## nginx-static with docker-compose
This is an example entry for a `docker-compose.yaml`
```yaml
version: '3'
services:
  example.org:
    image: azariasb/nginx-static
    container_name: example.org
    ports:
      - 8080:80
    volumes: 
      - /path/to/serve:/static
```


## nginx-static with træfik 3.x

To use nginx-static with træfik 3.x add an entry to your services in a docker-compose.yaml. To set up traefik look at this [simple example](https://docs.traefik.io/user-guides/docker-compose/basic-example/). 

In the following example, replace everything contained in \<angle brackets\> and the domain with your values.

```yaml
services:
  traefik:
    image: traefik:3.3 # check if there is a newer version
  # Your traefik config.
    ...
  example.org:
    image: azariasb/nginx-static
    container_name: example.org
    expose:
      - 80
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.<router>.rule=Host(`example.org`)"
      - "traefik.http.routers.<router>.entrypoints=<entrypoint>"
# If you want to enable SSL, uncomment the following line.
#      - "traefik.http.routers.<router>.tls.certresolver=<certresolver>"
    volumes: 
      - /host/path/to/serve:/static
```

If traefik and the nginx-static are in distinct docker-compose.yml files, please make sure that they are in the [same network](https://doc.traefik.io/traefik/routing/providers/docker/#traefikdockernetwork).

For a traefik 1.7 example look [at an old version of the readme](https://github.com/flashspys/docker-nginx-static/blob/bb46250b032d187cab6029a84335099cc9b4cb0e/README.md)

## nginx-static for multi-stage builds

nginx-static is also suitable for multi-stage builds. This is an example Dockerfile for a static node.js application:

```dockerfile
FROM node:alpine AS build
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm install && npm run build

FROM azariasb/nginx-static
COPY --from=build /usr/src/app/dist /static
```

### Custom nginx config

In the case you already have your own Dockerfile you can easily adjust the nginx config by adding the following command in your Dockerfile. In case you don't want to create an own Dockerfile you can also add the configuration via volumes, e.g. appending `-v /absolute/path/to/custom.conf:/etc/nginx/conf.d/default.conf` in the command line or adding the volume in the docker-compose.yaml respectively. This can be used for advanced rewriting rules or adding specific headers and handlers. See the default config [here](nginx.vh.default.conf).

```dockerfile
…
FROM azariasb/nginx-static
COPY your-custom-nginx.conf /etc/nginx/conf.d/default.conf
```
