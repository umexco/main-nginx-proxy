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
to create a global proxy network.<br>
All containers that need to communicate with the proxy server must join this network.


### 2. Config
Customize the email address in `docker-compose.yaml` at `DEFAULT_EMAIL=` to obtain certificates

### 3. Start the proxy server
```sh
docker compose up -d
```


### 4. Connect other Docker services
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
         - LETSENCRYPT_HOST=domain.com
         - VIRTUAL_HOST=domain.com
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

Finished

## Special configurations
### 5. Host services
To make services that run on the host machine available through the proxy service, use the following steps.
We assume, the service should use the domain `example.com` and is running on port `8765`

#### 1. Create a `example.com.conf` in your `main-nginx-proxy/docker/nginx/vhost/` directory
```perl
server {
    listen 80;
    server_name test123.com;

    location / {
        proxy_pass http://host.docker.internal:8765;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    listen 443 ssl;
    ssl_certificate /etc/nginx/certs/test123.com.crt;
    ssl_certificate_key /etc/nginx/certs/test123.com.key;
}
```
#### 2. Mount the config file into the proxy container by adding the volume to the `docker-compose.yaml`.
```yaml
volumes:
    - ./docker/nginx/vhost/example.com.conf:/etc/nginx/vhost.d/example.com.conf
```

#### 3. Apply the new config
Probably recreate the container if your proxy service was already running.
- `docker compose up -d --force-recreate nginx`


### 6. Forward `www` subdomains
**IMPORTANT:** Before creating this vhost, make sure you have the cert for the TLD with `LETSENCRYPT_HOST=domain.com,www.domain.com` up and running!

```yaml
environment:
    - LETSENCRYPT_HOST=domain.com,www.domain.com
    - VIRTUAL_HOST=domain.com
```

First restart to obtain the www included cert:
```sh
docker compose up -d --force-recreate nginx
```


Then create a config `www.domain.com.conf` with this content. Using variables in the SSL path is unfortunately not possible due to permissions.
```perl
server {
   listen 80;
   listen [::]:80;
   server_name www.domain.com;

   return 301 https://domain.com$request_uri;
}

server {
   listen 443 ssl http2;
   listen [::]:443 ssl http2;
   server_name www.domain.com;

   ssl_certificate /etc/nginx/certs/domain.com.crt;
   ssl_certificate_key /etc/nginx/certs/domain.com.key;

   return 301 https://domain.com$request_uri;
}
```

Mount it
```yaml
volumes:
    - ./docker/nginx/vhost/www.domain.com.conf:/etc/nginx/conf.d/www.domain.com.conf:ro
```

Finally restart the nginx-proxy
```sh
docker compose up -d --force-recreate nginx
```


### 7. Temporary extra configuration
If you need temporary adjustments on a specific host, e.g. to increase the file upload size to import a backup in phpmyadmin, you can use the following steps.
1. `docker compose exec nginx bash`
2. `vi /etc/nginx/conf.d/default.conf`
3. Make your adjustments, e.g. add `client_max_body_size 1G;`
5. `/usr/sbin/nginx -s reload`

#### Important vi commands:
1. `i` - insert mode (to start typing)
2. `Esc` - command mode (to stop typing and enter command mode)
3. `:w` - save the file
4. `:q` - quit vi
5. `:q!` - force quit vi (discard changes)
6. `:wq` - save and quit vi
7. `yy` - copy the current line
8. `dd` - delete the current line
9. `p` - paste the copied or deleted text
10. `u` - undo the last change
11. `Ctrl-r` - redo the last undo
