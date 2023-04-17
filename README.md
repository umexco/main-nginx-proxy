# main-nginx-proxy
This setup uses a global nginx proxy server to distribute incoming requests to different services based on their domain name. 
It eliminates the need for a local nginx server on the host and automates Let's Encrypt provisioning.

## Setup
```sh
git clone --depth=1 --branch=main https://github.com/umexco/main-nginx-proxy.git \
&& rm -rf ./main-nginx-proxy/.git
```

### 1. Create global proxy network
Before starting any docker containers, run 
```sh
docker network create main-nginx-proxy
```

All containers that need to communicate with the proxy server must join this network.


### 2. Config
Customize the email address in `docker-compose.yaml` at `DEFAULT_EMAIL=` to obtain certificates

### 3. Start the proxy server
```sh
docker compose up -d
```


### 4. Connect other services
To make a service accessible through the proxy server, 
set the following environment variables in its `docker-compose.yaml` file:

- `LETSENCRYPT_HOST`: the target domain for the service
- `VIRTUAL_HOST`: the target domain for the service
- `VIRTUAL_PORT`: the service port where the service is accessible

To run a service behind a sub-route, also add the following environment variables:
- `VIRTUAL_PATH`: path, e.g. `VIRTUAL_PATH=/myapp`
- `VIRTUAL_DEST=/` as rewrite base

The `VIRTUAL_PORT` may vary depending on the configuration. 
For example, if you started your application web server using `php artisan serve --port=8765`, the variable should be set to `VIRTUAL_PORT=8765` accordingly.

```yaml
version: "3"

services:
   helloworld:
      image: nginxdemos/hello
      container_name: hello-world
      environment:
         - LETSENCRYPT_HOST=testing.domain.com
         - VIRTUAL_HOST=testing.domain.com
         - VIRTUAL_PORT=80
      networks:
         - main-nginx-proxy
      restart: unless-stopped

networks:
   main-nginx-proxy:
      external: true
```

Finally start the service
```sh
docker compose up -d
```

After a few seconds, the new service should be available at the configured domain. The nginx proxy automatically detects the new service after launch and
obtains a certificate for the domain if required.
