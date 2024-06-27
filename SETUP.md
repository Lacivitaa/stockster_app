[Back to main](../../README.md)

# Local AEM Setup

## Prerequisites

* Oracle Java 11

## Installation

### Author

1. Install a new AEM 6.5.18 author instance.
2. Bypass known [FELIX-6184](https://wttech.blog/blog/2020/aem-on-java-11-with-gradle/#3-bypass-known-felix-6184-issue) issue and install our [HMAC Key](https://start.1password.com/open/i?a=CTGAMKNWBZFNJFBH5VL7IA7MJU&v=lmu2o64u6oefjiizm7i6lcbu2u&i=7ypb2cb3ouapbiqgjp7dyt7fhu&h=ibm.ent.1password.com) ([a guide to do this](https://experienceleague.adobe.com/docs/experience-manager-65/administering/security/encapsulated-token.html?lang=en#replicating-the-hmac-key)).
3. Configure AEM to always start with run mode "local".
   > [!NOTE]  
   > The [start-author.sh](start-author.sh) script already defines the necessary run modes.
4. Install our code including 3rd party dependencies by running the following maven build command.
   ```shell
   mvn clean install -PautoInstallSinglePackage
   ```
   > [!WARNING]  
   > The first installation of our package usually doesn't fully complete automatically, so we have to manually do the following steps:
   > * Restart AEM to ensure ACS Commons is properly installed
   > * Install the package `ibm-dam.ui.content` if it hasn't benn installed automatically
5. Install the workflow package by running the following maven build command in the `ui.workflows` directory.
   ```shell
   cd ui.workflows
   mvn clean install -PautoInstallPackage
   ```
6. Install the initial content by running the following maven build command in the `ui.content.initial` directory.
   ```shell
   cd ui.content.initial
   mvn clean install -PautoInstallPackage
   ```
7. Follow the instruction for [Local Author Dispatcher Setup](/dispatcher.author/README.md#local-docker-setup).

### Publish

1. Install a new AEM 6.5.18 publish instance.
2. Bypass known [FELIX-6184](https://wttech.blog/blog/2020/aem-on-java-11-with-gradle/#3-bypass-known-felix-6184-issue) issue and install our [HMAC Key](https://start.1password.com/open/i?a=CTGAMKNWBZFNJFBH5VL7IA7MJU&v=lmu2o64u6oefjiizm7i6lcbu2u&i=7ypb2cb3ouapbiqgjp7dyt7fhu&h=ibm.ent.1password.com) ([a guide to do this](https://experienceleague.adobe.com/docs/experience-manager-65/administering/security/encapsulated-token.html?lang=en#replicating-the-hmac-key)).
3. Configure AEM to always start with run mode "local".
   > [!NOTE]  
   > The [start-publish.sh](start-publish.sh) script already defines the necessary run modes.
4. Install our code including 3rd party dependencies by running the following maven build command.
   ```shell
   mvn clean install -PautoInstallSinglePackagePublish
   ```
   > [!WARNING]  
   > The first installation of our package usually doesn't fully complete automatically, so we have to manually do the following steps:
   > * Restart AEM to ensure ACS Commons is properly installed
   > * Install the package `ibm-dam.ui.content` if it hasn't benn installed automatically
5. Install the initial content by running the following maven build command in the `ui.content.initial` directory.
   ```shell
   cd ui.content.initial
   mvn clean install -PautoInstallPackagePublish
   ```
6. Follow the instructions for [Local Publish Dispatcher Setup](/dispatcher.publish/README.md#local-docker-setup).

### Dynamic Media

1. Make sure to always start the author instance with run mode `dynamicmedia_scene7`.
   > [!NOTE]  
   > The [start-author.sh](start-author.sh) script already defines the necessary run modes.
2. Download & install the [Dynamic Media Classic app ](https://experienceleague.adobe.com/docs/dynamic-media-classic/using/intro/dynamic-media-classic-desktop-app.html?lang=en)
3. Login with your email as username & the temporary password provided in the email by adobe
4. You will be prompted to change the temporary password. After the change you can close dynamic media classic

#### Dynamic Media Configuration
1. On author make sure that you have enabled the "Dynamic Media Asset Activation (scene7)" replication agent
2. On author navigate to **Tools > Cloud Services > Dynamic Media Configuration**
3. On the Dynamic Media Configuration Browser page, in the left pane, select **global** (do not select the folder icon to the left of **global**), then select **Create**.
4. On the **Dynamic Media Configuration** page make following selections (make sure to enter your new password!)
   <img alt="dm_scene7_config.png" src="images/dm_scene7_config.png" width="1280"/>
   > [!WARNING]  
   > It's important that every developer has his own personal folder (e.g. `ibmdev/<your-name>/` <- make sure to include the trailing slash ), because otherwise we would overwrite the assets of other developers.<br>
   > The field "Company Root Folder Path" isn't editable by default, just inspect the field and make it editable.
5. On author create a sling mapping entry for your personal folder (note: replace `<your-name>` with the value created above)
   ```text
   /etc/map/_css
      +-- jcr:primaryType = "sling:Mapping"
      +-- sling:match = "^[^/]+/[^/]+/is/content/ibmdev/_CSS/(.*)"
      +-- sling:redirect = "/is/content/ibmdev/<your-name>/_CSS/$1"
   ```

#### Local Publish Server
1. Create or extend the following directory structure on your local machine
   ```
   nginx-docker/
   ├── ssl/
   │   ├── certs/
   │   │   └── ibm-assets.crt
   │   └── private/
   │       └── ibm-assets.key
   └── nginx.conf
   ```
   > [!NOTE]  
   > Certificates can be found in [1Password > DAM SSL Certificates](https://start.1password.com/open/i?a=CTGAMKNWBZFNJFBH5VL7IA7MJU&v=lmu2o64u6oefjiizm7i6lcbu2u&i=e22rixuggveidipsa4i4sfyjpi&h=ibm.ent.1password.com)
2. Add the following config to the `nginx.conf` file
   ```nginx
   server {
       listen 443 ssl;
       server_name assets.ibm.local;
   
       ssl_certificate /etc/nginx/certs/certs/ibm-assets.crt;
       ssl_certificate_key /etc/nginx/certs/private/ibm-assets.key;
   
       location / {
           proxy_pass https://ibm.scene7.com;
           proxy_set_header X-IBM-From-Akamai <scene7-security-header>;
           proxy_buffering off;
       }
   
       location ~ ^/is/content/ibmdev/_CSS/(.*)$ {
           proxy_pass https://ibm.scene7.com/is/content/ibmdev/<your-name>/_CSS/$1;
           proxy_set_header X-IBM-From-Akamai <scene7-security-header>;
           proxy_buffering off;
       }
   
       location /dam/content/ {
           proxy_pass https://host.docker.internal:443/dam/content/;
           proxy_set_header Host publish.dam.ibm.local;
       }
   }
   ```
   > [!NOTE]
   > * Replace `<scene7-security-header>` with [1Password > Scene7 security header](https://start.1password.com/open/i?a=CTGAMKNWBZFNJFBH5VL7IA7MJU&v=lmu2o64u6oefjiizm7i6lcbu2u&i=7i43zvtlkrmgakexs5dqguts7a&h=ibm.ent.1password.com)
   > * Replace `<your-name>` with you personal folder name created earlier
3. Add the following entry to your `/etc/hosts` file
   ```text
   127.0.0.1 assets.ibm.local
   ```
4. Start the docker container by running the following command in the `nginx-docker` directory
   ```shell
   docker run -dit --name ibm-nginx \
       --log-opt max-size=10m \
       -p 443:443 \
       -v "$PWD/ssl:/etc/nginx/certs" \
       -v "$PWD/nginx.conf:/etc/nginx/conf.d/default.conf" \
       nginx
   ```
5. On author navigate to **Tools > Assets > Dynamic Media General Settings** and set the Publish Server Name to "https://assets.ibm.local"


#### Initial Asset Sync

If you are enabling Dynamic media on an AEM instance tha already contains assets, perform the following actions to sync all existing assets to Dynamic Media.

1. Make sure that you have deployed the "Scene7 Initialization" workflow
4. Use [execute-workflows-in-bulk.groovy](/ui.content/src/main/content/jcr_root/var/groovyconsole/scripts/ibm-dam/dynamicmedia-migration/execute-workflows-in-bulk.groovy) to migrate your assets
   > [!NOTE]  
   > Use `tail -F error.log | grep "BulkWorkflowExecutionJob"` to monitor the execution.<br>
   > Use [stop-workflows-in-bulk.groovy](/ui.content/src/main/content/jcr_root/var/groovyconsole/scripts/ibm-dam/dynamicmedia-migration/stop-workflows-in-bulk.groovy) to stop the execution.
