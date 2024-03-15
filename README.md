# Docker Compose
![image info](container.png)

### Docker compose files for use in my Homelab, deployed and auto-updated using Portainer stacks.

Below I have detailed configuration options for all of my running services and containers.

---
## Reverse-Proxy & Authentication Services:
[NGINX Proxy Manager](https://github.com/adam-beckett-1999/Docker-Compose?tab=readme-ov-file#nginx-proxy-manager "NGINX Proxy Manager Configuration")
- [Cloudflare API Key](https://github.com/adam-beckett-1999/Docker-Compose?tab=readme-ov-file#cloudflare-api-key "NGINX Proxy Manager: Cloudflare API Key")
- [SSL Certificates](https://github.com/adam-beckett-1999/Docker-Compose?tab=readme-ov-file#ssl-certificate "NGINX Proxy Manager: SSL Certificates")
- [Creating a Proxy Host](https://github.com/adam-beckett-1999/Docker-Compose?tab=readme-ov-file#creating-a-proxy-host "NGINX Proxy Manager: Creating a Proxy Host")

[Authentik](https://github.com/adam-beckett-1999/Docker-Compose?tab=readme-ov-file#authentik "Authentik Configuration")

### **NGINX Proxy Manager**

Once the container is started, navigate to http://{ip_of_your_docker_host}:81/ where you will need to login for the first time with the default credentials:

  ``` default-login
  Email: admin@example.com
  Password: changeme
  ```

When you have logged in you will be prompted to change the default login credentials, you can then begin adding services to be reverse proxied.

#### Cloudflare API Key
In order to create SSL certificates from within NPM, you need to do some setup to ensure DNS challenges will work. I use cloudflare as my registrar and DNS provider, so will need an API key in order to successfully generate certs with Letsencrypt.

1. Login to your Cloudflare account and navigate to the section for the domain you wish to use, in the sidebar to the right should be a section titled 'API'.
2. From there, click the 'Get your API token' link and create a new token.
3. Use the 'Edit zone DNS' template, you don't need to change any of the options, and continue to summary where you will get a copy of your API key which you need to copy into NPM.

#### SSL Certificate
To setup an SSL certificate to use with your services, I prefer to assign a wildcard certificate to make the process easier. 
1. To setup, go to the 'SSL Certificates' tab and click 'Add SSL Certificate'.
2. Enter your domain, your email address, and click the 'Use a DNS Challenge' toggle.
3. You need to select your DNS provider (in my case it's Cloudflare) which will show a box where you will need to enter a Cloudflare API key (explained in the section above).
4. Once you have the API key copied, click the 'I Agree to the Let's Encrypt Terms of Service' toggle and save. It could take a minute or two for the SSL cert to be registered and go through.

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
This process is similar to that of the Proxy Provider. However it will require configuring authentik as an SSO provider in your service once the Application/Provider is created.

1. Under the 'Applications' dropdown in the side-bar, click on the 'Applications' sub-menu and click the 'Create With Wizard' button at the top of the page to simplify the process. You can create the application and provider seperately, but this isn't necessary.
2. Specify a name for your application, this can just be the name of the service you are adding authentication to. The 'slug' field should auto populate. Under 'UI Settings' you can also specify the external URL for the service for the link that will be created on the Authentik dashboard, which would be: https://{service}.domain.tld/. Click 'next'.
3. On the 'Provider Type' page, select 'OAuth2/OIDC' and click next.
4. The name will auto populate, so move on to selecting an authorization flow. I typically use the 'authorization-explicit-consent' which upon logging into the service for the first time, will prompt the user to allow the service access to the data provided by Authentik for the authentication process. If you don't want this prompt then select the 'implicit-consent' option.
5. Under 'Protocol settings', select either 'Confidential' or 'Public' as your 'client type' depending on what the service asks for in its configuration. Most services want a Client ID and Client Secret from their OAuth/OpenID provider, so use 'Confidential'. If you're unsure of what you should be configuring your service with, check its documentation. You will need to save this Client ID and Secret for later. You shouldn't need to change any of the other options. Click 'Next' and 'Finish'.
6. With the Application and Provider created, you will want to grab the rest of the details to use in your service configuration. These can be found in the provider you just created. Under 'Providers', navigate to the new provider and you will see that Authentik has created some URLs for you to copy during the setup in your service. The 'OpenID Configuration URL' will be used in some services that are able to auto-discover the relevant information from the SSO provider. Others will require you to specify the specific URLs for each role.

As an example, i'll include how to setup OAuth SSO in Portainer:
1. With a provider already created, go to the Portainer webUI and go to 'Settings' in the side-bar, and click 'Authentication'.
2. Under 'Authentication Method', select OAuth and click the slider for 'Use SSO' under 'Single Sign-On'. You should also click the slider for 'Automatic user provisioning' to ensure that Portainer creates users automatically for anyone that signs in using Authentik. (I've also found that Portainer doesn't like to match Authentik usernames to Portainer ones when logging in, which leads to errors. So enabling this option will allow users to be created using email addresses that it gets from Authentik.)
3. Under the 'Provider' section, select 'Custom' and here you will see what details Portainer needs to correctly configure your SSO provider. Copy and paste in your Client ID, Client Secret, and URLs for 'Authorization', 'Access', 'Resource' (which is userinfo) and 'Logout'. The Redirect URL will be the URL you use to access the service through the reverse proxy, so https://portainer.domain.tld/ or something similar.
4. The user identifier will need to be 'email', and the scopes need to be set as 'email openid profile'. This is the only combination of options that has worked for me. Ensure that when putting the scopes in, you don't follow the formatting shown as an example in the box, copy as exactly as specified above.
5. Now you can save and test. You should get a 'Login with OAuth' option on the Portainer login screen.

#### Access restrictions with Groups and Policies
Its possible to restrict access to certain services using Groups and Policies defined in Authentik.

**Group and Expression Policy:**
1. Under 'Directory' in the side-bar, click on 'Groups' and 'Create' a new group from here.
2. Specify a name. In my case, I have groups for access to my media-server utilities (sonarr, radarr etc), and a privileged access group for services that only I need access to (useful in my case as I have authentik users who are setup just for access to certain services).
3. In the 'Attributes', you can specify a set of custom attributes which can be passed through to services using HTTP basic authentication. For example, I have sonarr configured to use basic auth with a preset username and password, and in the attributes section of my group, I have 'media_management_user' and 'media_management_password' with the credentials. This is then configured through my provider for Sonarr, under 'Authentication settings', enable 'Send HTTP-Basic Authentication' and put the variables you added to the group in 'HTTP-Basic Username Key' and 'HTTP-Basic Password Key'. If using multiple services, you can configure basic auth with the same credentials in all of them and add the Basic_auth attributes in their Providers.
4. For the next step, go to 'Customisation' and 'Policies' in the side-bar. Create a new policy, and select 'Expression Policy'.
5. Give the policy a name, and in the 'Expression' code block, you can use something similar to this to restrict access to the services based on group membership:

  ```
  is_permitted = ak_is_group_member(request.user, name="{name_of_group}")
  if not is_permitted:
    ak_message("Access Denied")
    return False
  if is_permitted:
    return True
  ```

6. Now all that is needed is to add the policy to the services you want restricted. Go to 'Applications' and select your service, in the top menu go to 'Policy/Group/User Bindings' and click 'Bind existing policy' on this page. In the create binding window, select your policy and leave all other settings as default. Click 'create' and now you should be able to test whether your policy is working correctly.

For users in the group, they should be automatically authenticated into the service. Anyone not in the group will be taken to an Access Denied page.

**Using service based matching:**
1. Create a group like above. Ignore the attributes section and instead leave it empty.
2. If your service supports it, you can use its built in groups and match them against Authentik groups. An example would be Grafana. By specifying a custom attribute path (using the GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH= variable), you can tell it to assign permissions to users based on the groups they are in in Authentik. I have my Grafana instance configured using Authentik OAuth, and the following variable configured in the docker environment variables:
  ```
  GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH=contains(groups, 'Grafana Admins') && 'Admin' || contains(groups, 'Grafana Editors') && 'Editor' || 'Viewer'
  ```
3. When a user logs into the service, as long as they are in the correct group and the service is configured to allow new users to be added from Authentik, then they should be assigned the correct roles and permissions on first login.

### **Goaccess for NPM**

Once GoAccess is installed there is no extra configuration required, logs and data may take a while to populate as it relies on there being log files available from NGINX Proxy Manager to populate the graphs and metrics correctly.

- The 'EXCLUDE_IPS' environment variable will force GoAccess to ignore log data corresponding to internal requests, instead only showing service access from external IP's. This can be set to ignore any range of IPs you choose, in this case, I have it set to exclude 127.0.0.1 which is equivalent to 'localhost' and my internal IP ranges (192.168.0.0/24) etc.

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
