This is a guide on how to configure NGINX through a Cloudflared/Argo tunnel, along with TinyAuth. This guide assumes you already have your own domain and docker installed, but will walk you through the steps for everything else.

We will also be combining everything into one compose.yml file, but you are able to separate them by adding them into their own compose.yml by creating an additional compose.yml that includes them. We can also add network creation into this yml file. E.G.:

```yaml
include:
  - path: ./cloudflared.yml
  - path: ./nginx.yml
  - path: ./tinyauth.yml
# Network Start
networks:
  tunnel-net:
    name: tunnel-net
    external: false
      driver: bridge
      config:
        - subnet: 172.20.0.0/16 # Can be set to your preferred IP address.
# Network End
```

# Cloudflared Tunnel configuration:
---

1. Create a cloudflared tunnel
    1. In the `Zero Trust` menu, navigate to Networks > Connectors
    2. Select `Create a tunnel`
    3. Select `Select Cloudflared`
        1. Name does not matter. Select Save tunnel.
        2. Under install and run a connector, set the device operating system to `Docker`
        3. Copy the code and convert it to a [compose.yml](https://www.composerize.com)
        4. Select Next and then create your first application route.
    4. Create application routes for HTTP and HTTPS
        1. Set subdomain as *
        2. Select the correct domain from the drop down
        3. Set Service Type as HTTP
        4. Set the URL as localhost:80
    5. Select complete setup
    5. Select the newly create tunnel
    6. Select `Published Application Routes`
    7. Create a new hostname the same as previously, except set the url to localhost:443

We are done with the cloudflared website. We will be making a running compose file from here on, but feel free to bring it up and test it. When you perform `docker compose up` you should see the tunnel show healthy in the cloudflare dashboard.

Your compose.yml file should look like the following:
  
```yaml
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped    
```
  10. Create a `.env` file in the same root location as the compose.yml by using `sudo nano .env` in a CLI
  
```env
CLOUDFLARE_TUNNEL_TOKEN=your_tunnel_token
```

# NPM Configuration:
---

We are now going to add NGINX Proxy Manager onto the previously create compose file. This is the example compose file from the [NPM GITHUB](https://github.com/NginxProxyManager/nginx-proxy-manager)

```yaml
services:
  NGINX:
    container_name: NGINX
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80' # HTTP Port
      - '81:81' # NPM Admin
      - '443:443' # HTTPS Port
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

However, this isn't enough to set up our tunnel. We have to tunnel the traffic through the cloudflared route. We will take the easy way out by adding it to our running compose file. It should look like the following:

```yaml
# Cloudflare Start
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped
# Cloudflare End
# NPM Start
  NGINX:
    container_name: NGINX
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80' # HTTP Port
      - '81:81' # NPM Admin
      - '443:443' # HTTPS Port
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
# NPM End
```

But even this isn't good enough. We need to make sure the NGINX traffic is only routed through cloudflared. This means cloudflared needs to handle all of the port publishing.

```yaml
# Cloudflared Start
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped
    ports:
      - '80:80' # HTTP Port
      - '81:81' # NPM Admin
      - '443:443' # HTTPS Port
# Cloudflared End
# NPM Start
  NGINX:
    container_name: NGINX
    network_mode: service:cloudflared # Ensures public traffic is routed through cloudflared. Port publishing is handled by cloudflare.
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
# NPM End
```
Okay, awesome. We can bring this up and everything should work. Let's start adding some services. Navigate to `localhost:81` and create an account.

## ⚪ Optional: Network Creation 
---
This step is not necessary, but it will assign a static IP to all of the services that you want to have NGINX route traffic to.

We need to ammend network creation to this `compose.yml` file.

```yaml
networks:
  tunnel-net:
    name: tunnel-net
    external: false # Ensures that this compose file will bring the network up and down. It will not be taken down if containers are still using the network.
    ipam: # IP Address Management
      driver: bridge
      config:
        - subnet: 172.20.0.0/16 # Can be set to your preferred IP address.
```

Or if your prefer, create a network by running the following command:

`docker network create --driver=bridge --subnet=172.20.0.0/16 tunnel-net`

And then we set `external:true` in the YML file.

```yaml
# Cloudflared Start
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    restart: unless-stopped
    ports:
      - '80:80' # HTTP Port
      - '81:81' # NPM Admin
      - '443:443' # HTTPS Port
    networks:
      - tunnel-net
# Cloudflared End
# NPM Start
  NGINX:
    container_name: NGINX
    network_mode: service:cloudflared # Ensures public traffic is routed through cloudflared. Port publishing is handled by cloudflare.
    depends_on:
     - cloudflared # Ensures that cloudflared is brought up before the NGINX Proxy Manager
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
# NPM End
# Network Start
networks:
  tunnel-net:
    name: tunnel-net
    external: false
      driver: bridge
      config:
        - subnet: 172.20.0.0/16 # Can be set to your preferred IP address.
# Network End
```

# Proxy Host Creation
---
Lets create our first proxy host. This guide assumes you have a container running on port 8080.

1. In the NGINX Proxy Manager Dashboard, select `Proxy Hosts.`
2. Select `Add Proxy Host`
  1. Set your preferred `Domain Name`. E.G. `test.example.com`
  2. Set the appropriate `scheme`. In most cases it will be `http.`
  3. Set `Forward Hostname/IP` as the gateway for the container. If you created the network in the previous step, it will be 172.20.0.1. Otherwise, you can inspect the container for the gateway by running the following command:
  `docker inspect container-name | grep Gateway`
  4. Select the `SSL` tab
  5. `SSL Certificate` type should be set to `Request a new Certificate` and it will automatically pull a new certificate.
  6. Select Save and wait for the certificate to generate. If you get internal error, that means that the container is not able to talk through port 443. Ensure that your firewall is allowing traffic through that port.
  
Good work! We now have our first website configured for `test.example.com`. But we don't really want to keep this network available to the public. Let's lock it down.

# Tinyauth Configuration
---

Lets pull the latest [Tinyauth](https://tinyauth.app/docs/guides/nginx-proxy-manager/) image. We don't need everything included in their example, so let's steal just the tinyauth portion:

```yaml
services:
  tinyauth:
    container_name: tinyauth
    image: ghcr.io/steveiliop56/tinyauth:v5
    restart: unless-stopped
    environment:
      - TINYAUTH_APPURL=http://tinyauth.example.com
      - TINYAUTH_AUTH_USERS=user:$$2a$$10$$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u # user:password
```

You can generate a username by running the following command:
`docker run -i -t --rm ghcr.io/steveiliop56/tinyauth:v5 user create --interactive`

Alternatively, we can link this with either [GITHUB OAuth](https://tinyauth.app/docs/guides/github-oauth/), [Google OAuth](https://tinyauth.app/docs/guides/google-oauth/), or some other OIDC. I personally use [PocketID](https://tinyauth.app/docs/guides/pocket-id/) in another container and it works fantastically.

Lets go ahead and add this to the running `compose.yml` file we have, and add variables to change the TinyAuth background.

```yaml
services:
  tinyauth:
    container_name: tinyauth
    image: ghcr.io/steveiliop56/tinyauth:v5
    restart: unless-stopped
    environment:
      - TINYAUTH_APPURL=http://tinyauth.example.com
      - TINYAUTH_AUTH_USERS=user:$$2a$$10$$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u # user:password
      - TINYAUTH_DATABASE_PATH=/config/tinyauth.db # We are going to bind mount the tinyauth DB to the config folder
      - TINYAUTH_UI_BACKGROUNDIMAGE=/resources/background.jpg # Background Images need to be included in the TinyAuth Resources folder, otherwise they will not be pulled.
    volumes:
      - ./tinyauth:/config
      - ./tinyauth:/data/resources/background.jpg # Required for background images
```

Okay perfect, now let's add it to the running compose.yml. We'll also add healthchecks just in case we want Autoheal integration. If for some reason the cloudflared service restarts, NGINX will not work until it is also restarted. Our `compose.yml` should now look like this:

```yaml
# Cloudflared Start
services:
  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
      - TUNNEL_METRICS=127.0.0.1:60123 # Needed for healthchecks.
    restart: unless-stopped
    ports:
      - '80:80' # HTTP Port
      - '81:81' # NPM Admin
      - '443:443' # HTTPS Port
    networks:
      - tunnel-net
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "--metrics", "localhost:60123", "ready"]
      interval: 30s
      timeout: 30s
      retries: 3
# Cloudflared End
# NPM Start
  NGINX:
    container_name: NGINX
    network_mode: service:cloudflared # Ensures public traffic is routed through cloudflared. Port publishing is handled by cloudflare.
    depends_on:
     - cloudflared # Ensures that cloudflared is brought up before the NGINX Proxy Manager
    image: 'docker.io/jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    healthcheck:
      test: ["CMD", "curl", "-f", "http://172.20.0.1:81"] 
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
# NPM End
# Tinyauth Start
  tinyauth:
    container_name: tinyauth
    image: ghcr.io/steveiliop56/tinyauth:v5
    restart: unless-stopped
    environment:
      - TINYAUTH_APPURL=http://tinyauth.example.com
      - TINYAUTH_AUTH_USERS=user:$$2a$$10$$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u # user:password
      - TINYAUTH_DATABASE_PATH=/config/tinyauth.db # We are going to bind mount the tinyauth DB to the config folder
      - TINYAUTH_UI_BACKGROUNDIMAGE=/resources/background.jpg # Background Images need to be included in the TinyAuth Resources folder, otherwise they will not be pulled.
    volumes:
      - ./tinyauth:/config
      - ./tinyauth:/data/resources/background.jpg # Required for background images
    ports:
      - '3000:3000' # Tinyauth Port
    networks:
      - tunnel-net
    healthcheck:
      test: wget --no-verbose --tries=1 --spider 172.20.0.1:3001 || exit 1
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
# Tinyauth End
# Network Start
networks:
  tunnel-net:
    name: tunnel-net
    external: false
      driver: bridge
      config:
        - subnet: 172.20.0.0/16 # Can be set to your preferred IP address.
# Network End
```
Okay! Now our `compose.yml` should be good to go. Good job getting this far, let's take a quick break....


# Tinyauth redirect
---

Okay! Now let's make sure our websites get redirected to Tinyauth. We can check the [Tinyauth](https://tinyauth.app/docs/guides/nginx-proxy-manager/) website again for the correct NPM Advanced Configuration. However, you need to set return 302 to https, otherwise cloudflare will redirect to https, and Tinyauth will see http://tinyauth.example.com as the redirect URI.

```
# Root location
location / {
  # Pass the request to the app
  proxy_pass          $forward_scheme://$server:$port;

  # Add other app-specific config here

  # Tinyauth auth request
  auth_request /tinyauth;
  error_page 401 = @tinyauth_login;
}

# Tinyauth auth request
location /tinyauth {
  # Pass request to Tinyauth
  proxy_pass http://172.20.0.1:3000/api/auth/nginx; # If you used the previous tunnel-net, you don't need to change this

  # Pass the request headers
  proxy_set_header x-forwarded-proto $scheme;
  proxy_set_header x-forwarded-host $http_host;
  proxy_set_header x-forwarded-uri $request_uri;
}

# Tinyauth login redirect
location @tinyauth_login {
  return 302 https://tinyauth.example.com/login?redirect_uri=$scheme://$http_host$request_uri; # Replace with your app URL
}
```

To add this, go into the `Proxy Host` and select the `Gear` Icon on the top right. Paste this into the `Custom Nginx Configuration` box. The only thing you need to change is the `return 302` to the correct Tinyauth website.

And we should be good to go! All we have to do is create a new service and include the Custom Nginx Configuration and our services will be up and protected by Tinyauth!

