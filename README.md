# Docker Compose
![image info](container.png)

### Docker compose files for use in my Homelab, deployed and auto-updated using Portainer stacks.

Below I have detailed configuration options for all of my running services and containers.

---
## Reverse-Proxy & Authentication Services:

### **NGINX Proxy Manager**

Once the container is started, navigate to http://{ip_of_your_docker_host}:81/ where you will need to login for the first time with the default credentials:

``` default-login
Email: admin@example.com
Password: changeme
```

When you have logged in you will be prompted to change the default login credentials, you can then begin adding services to be reverse proxied.

#### SSL Certificate
To setup an SSL certificate to use with your services, I prefer to assign a wildcard cert to make the process easier. 
1. To setup, go to the 'SSL Certificates' tab and click 'Add SSL Certificate'.
2. Enter your domain, your email address, and click the 'Use a DNS Challenge' toggle.
3. You need to select your DNS provider (in my case it's Cloudflare) which will show a box where you will need to enter a Cloudflare API key (instructions below).
4. Once you have the API key copied, click the 'I Agree to the Let's Encrypt Terms of Service' toggle and save. It could take a minute or two for the SSL cert to be registered and go through.

#### Cloudflare API Key
1. Login to your Cloudflare account and navigate to the section for the domain, in the sidebar to the right should be a section titled 'API'.
2. From there, click the 'Get your API token' link and create a new token.
3. Use the 'Edit zone DNS' template, you don't need to change any of the options, and continue to summary where you will get a copy of your API key which you need to enter into the box within NPM.

#### Creating a Proxy Host
1. Navigate to the 'Hosts' tab and click 'proxy hosts' from the dropdown.
2. Click 'Add Proxy Host' in the top right corner and enter the domain you wish to use for the service (e.g. plex.domain.tld), the hostname/ip where the service is running from and the port of the service. It is usually worth clicking the toggles below to 'Cache Assets', 'Block Common Exploits' and enable 'Websockets Support' to ensure sites load correctly once proxied.
3. The last step is to click on the SSL tab within the new proxy host window and click the SSL certificate dropdown and select the cert you created earlier. You can also click on the toggles below here as well (if your site experiences any errors once setup, try disabling some of these options to see if that resolves it). Once done, click save.
4. The service should now be available to access through the domain you have assigned it, if NPM detects that there is an error in the proxy configuration it will indicate it on the right under the 'status' column as 'offline'. This isn't always an indicator that the site works however, as there could be a configuration error within the service itself to allow forwarding through a reverse-proxy.

### **Authentik**

Once Authentik is installed, navigate to the link below to configure your install:
http://{ip_of_your_docker_host}:9000/if/flow/initial-setup/ 

#### Initial Configuration
1. Create the admin user, and you will be redirected to the homepage showing 'My Applications'. Go to the 'Admin Interface' and you can start to customise your install.

#### Branding & Customisation
To change the branding of Authentik, there are two areas which need to be modified.

1. Under 'System', navigate to 'Brands' and click the 'Edit' button under the 'Actions' header for the authentik-default domain. From there you can change the title, logo and favicon for your authentication page, as well as set your domain URL. The files used here are stored in the '/media' directory within the container, which is mapped to '/your_path/media' on the docker host. Make sure your files are in there so they are loaded correctly. If you go to the 'Other Global Settings' dropdown, you can also use custom attributes to change how Authentik behaves. I like to add the below attribute to force dark mode on every device regardless of the local settings of the user.

``` title:custom-attributes
settings:
  theme:
    base: dark
```
2. To customise the layout and background of authentication pages, you can do that individually for each type of auth-flow. Under 'Flows & Stages', click on 'Flows' and from there, click 'edit' under the actions column for the flow you wish to customise. Here you can change the text that appears on auth pages, and under the 'Appearance Settings' dropdown, you can change the layout of the auth page (which changes the appearance of the elements on the page to be centred, left or right, and with different styles) as well as set a background image.

#### Authentication for services without OAuth & OpenID support (Proxy Provider)
I typically use Authentiks 'Transparent Reverse Proxy' provider for applications that don't have OAuth and OpenID SSO support. This also makes configuring in NPM much easier. To secure an application using this provider, you need to do the following:

1. Under the 'Applications' dropdown in the side-bar, click on the 'Applications' sub-menu and click the 'Create With Wizard' button at the top of the page to simplify the process. You can create the application and provider seperately, but this isn't necessary.
2. Specify a name for your application, this can just be the name of the service you are adding authentication to. The 'slug' field should auto populate. Under 'UI Settings' you can also specify the external URL for the service for the link that will be created on the Authentik dashboard, which would be: https://{service}.domain.tld/. Click 'next'.
3. On the 'Provider Type' page, select 'Transparent Reverse Proxy' and click next.
4. Now we need to configure out Proxy Provider. The name will auto populate, so move on to selecting an authorization flow. I typically use the 'authorization-explicit-consent' which upon logging into the service for the first time, will prompt the user to allow the service access to the data provided by Authentik for the authentication process. If you don't want this prompt then select the 'implicit-consent' option.
5. Specify your external host URL, again this will be the same as before: https://{service}.domain.tld/
6. Specify your internal host URL, this will be the IP/hostname and port of the service that you access internally, eg. http://{ip_or_hostname}:8181/
7. Click next, and if Authentik is happy it will show a green tick and allow you to click 'Finish'. If you get an error, you will have to go back and make changes to your configuration.
8. The last step within Authentik is to go to assign the provider to your Proxy outpost. Under 'Applications' in the side-bar, click 'Outposts' and click the 'edit' button under the 'Actions' column on the Embedded Authentik outpost. From here, click on the service you just added under 'Available Applications' and use the arrow to move it to the 'Selected Applications' group, and click 'update'.
9. Now that the authentication is setup in Authentik, you can follow the instructions for adding the Proxy Host in NPM. However, when adding the host, the destination/forward address will be the URL for your Authentik server, as Authentik will be handling the forwarding to the service internally after the authentication flow has run. The external domain name must also match what you have specified in Authentik, so that the forwarding rule matches correctly.

As an extra configuration option, you can add an icon for the service in the Authentik dashboard by going to 'Applications', clicking the 'edit' button under the 'actions' column, and opening the 'UI settings' dropdown in the menu. From here you can upload an icon and specify a service publisher and description.

#### Authentication for services with OAuth & OpenID support

### **Goaccess for NPM**

Once GoAccess is installed there is no extra configuration required, logs and data may take a while to populate as it relies on there being log file available from NGINX Proxy Manager to populate the graphs and metrics correctly.

- The 'EXCLUDE_IPS' environment variables will force GoAccess to ignore log data corresponding to internal requests, instead only showing service access from external IP's. This can be set to ignore any range of IPs you choose, in this case, I have it set to exclude 127.0.0.1 which is equivalent to 'localhost' and my internal IP ranges (192.168.0.0/24) etc.

---
### Monitoring & Management Services:
* File-Browser
* Dozzle
* Apache Guacamole
* Upsnap
* Homepage
* Speedtest-Tracker
* Uptime-Kuma
---
### User Services:
* Bookstack
* Excalidraw
* Paperless-NGX
* Planka
---
### Media Services:
* Overseerr
* Tautulli
* Plex-Meta-Manager
* Prowlarr
* Radarr
* SABnzbd
* Sonarr
* qBittorrent
* YoutubeDL
---
### Docker Specific Services:
* Watchtower
* Docker-Socket-Proxy
* Traefik Kop
* CAdvisor
* Node-Exporter
* Portainer Agent
---
### Considerations/Alternatives:
* Outline - https://github.com/outline/outline
* Vikunja - https://vikunja.io/
* Pingvin - https://github.com/stonith404/pingvin-share
* Microbin - https://microbin.eu/
* Graylog - https://graylog.org/products/source-available/
* Nextcloud - https://nextcloud.com/
* Photoprism - https://www.photoprism.app/features
