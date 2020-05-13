# Better OAF development setup

A set of configuration patches and customizations (for the Weblogic-based infrastruture which EBS / OAF is runinng on) to make OAF development less annoying and faster. It:

- Eliminates the need to restart the whole Application Server instance and all of its OA cores during development
- Allows to assign individual OA cores to specific IP addresses
- Multiple developers can work on single Application Server instance, all working on dedicated OA cores assigned to their IP and not colliding with each other
- Supports remote debugging
- Brings debugging to even higher level with classes Hot Reloading by using custom JDK with Dynamic Code Evolution Virtual Machine patches (DCEVM)

## General assumptions

By default we have 4 OA core servers, all included in load balancing setup. The goal is to add two new cores and then leave only 3 of them in load balancing configuration, while the other 3 will be assigned to specific IP addresses for use by developer and his testers. 

The more free cores you have (and by free I mean excluded from load balancing setup), the more OAF developers can work on one EBS instance without colliding with each other.

In all code samples below I assume the following:
- Port numbers used for OA Cores are 7231, 7232, 7233, 7234, 7235, 7246
- Internal hostname of the OAF instance is `develop.example.com`
- Port number of the Weblogic Admin webserver is 7031
- User directory on the filesystem is `/home/user/`, and our custom mapping config file `ohs-proxy-map` will be created in `/home/user/.config`

You can find some of these things in your existing config files mentioned further in this guide (**mod_wl_ohs.conf**, **apps.conf**)

# Setup

1. Add two new OA Cores (oacore_server5 and oacore_server6), see: [Adding new OA Core](#Adding-new-OA-Core)
1. Configure the HTTP server, see: [Configuring Oracle HTTP Server](#Configuring-Oracle-HTTP-Server)
1. Prepare the IP Address to OA Core mapping config file, see: [Sample `ohs-proxy-map` config](#Sample-ohs-proxy-map-config)
1. Configure your development OA cores, see: [Configuring OA Core startup parameters for remote debugging](#configuring-oa-core-startup-parameters-for-remote-debugging)
1. Prepare custom JDK build with DCEVM patches (TBC)
1. Configure development OA cores to use custom DCEVM JDK instead of the HotSpot shipped by default (TBC)

> **Note:** For a quick setup without the convenience of full-blown hot reloading of the classes, you can skip steps 5 and 6. 

# Working with the customized environment

## Restarting the specific development OA Core

```bash
export wlspass="wlspass"
(echo $wlspass | admanagedsrvctl.sh stop oacore_server1) ; (echo $wlspass | admanagedsrvctl.sh start oacore_server1)
```

## Assigning OA Cores to user's machine

See: [Sample `ohs-proxy-map` config](#Sample-ohs-proxy-map-config)

## Low-level troubleshooting

These might be useful sometimes:

- `-DAFLOG_ENABLED=TRUE -DAFLOG_LEVEL=DEBUG` flags (OA Core configuration > Server Start > JVM Arguments)
- `tail -f ${EBS_DOMAIN_HOME}/servers/oacore_server*/logs/oacore*.log`
- `tail -f ${EBS_DOMAIN_HOME}/servers/oacore_server*/logs/access.log`

# Sub-guides

## Adding new OA Core

This is all done in terminal session via SSH. `$wlspass` variable must contain your Weblogic admin password, and `$apps`  is the password for `apps` user in your Databsae.

```
export wlspass="wlspass"
export apps="appspass"
```

1. Stop everything

   ```bash
   echo $wlspass | adstpall.sh apps/$apps
   ```

2. Start WebLogic Admin server only

   ```bash
   ${INST_TOP}/admin/scripts/adadminsrvctl.sh start
   ```

3. Put the name of the new core, the port it should listen on and the hostname of the server instance into variables for future commands

   ```bash
   export NEW_OA_CORE_NAME=oacore_server5
   export NEW_OA_CORE_PORT=7235
   export SERVER_HOSTNAME=develop.example.com
   ```

   > **Note:** Make sure that the name and port is not already in use by another server or service. Also make sure that the port does not overlap with servers created during online patching procedure.

4. Create the OACORE

   ```bash
   perl ${AD_TOP}/patch/115/bin/adProvisionEBS.pl \
   ebs-create-managedserver -contextfile=${CONTEXT_FILE} \
   -managedsrvname=${NEW_OA_CORE_NAME} -servicetype=oacore \
   -managedsrvport=${NEW_OA_CORE_PORT} -logfile=addMS_${NEW_OA_CORE_NAME}.log
   ```

5. Sync apache configuration?

   ``` bash
   perl ${FND_TOP}/patch/115/bin/txkSetAppsConf.pl -contextfile=${CONTEXT_FILE} \
   -configoption=addMS -oacore=${SERVER_HOSTNAME}:${NEW_OA_CORE_PORT}
   ```

6. Stop WebLogic Admin Server

   ``` bash
   ${INST_TOP}/admin/scripts/adadminsrvctl.sh stop
   ```

7. (Re)start everything

   ``` bash
   restart_appl.sh
   ```

## Configuring Oracle HTTP Server

1. Login to Enterprise Manager 11g Fusion Middleware Control panel (i.e. at `http://develop.example.com:7031/em`)
1. In the left sidebar, expand the **Web Tier** node and then click on your web instance

   ![Enterprise Manager menu screenshot](/images/oaf-dev-setup-1.png)
   
1. In **Oracle HTTP Server** menu choose **Administration > Advanced Configuration**

   ![Enterprise Manager menu screenshot](/images/oaf-dev-setup-2.png)
   
1. On the next page select **mod_wl_ohs.conf** from the list and press **Go**

   ![Enterprise Manager menu screenshot](/images/oaf-dev-setup-3.png)

1. Backup this file somewhere, then make the following modifications:

   1. Take the code below and modify: 
      - the path to IP address mapping config file (`txt:/home/user/.config/ohs-proxy-map`)
      - the hostname and port number in all `http://` URLs to match your instance setup (7 occurences; 6 for OA Cores and one for Weblogic Cluster)
      
      After you're done with modifications, insert the whole code block just below the line starting with `<IfModule mod_weblogic.c>` in **mod_wl_ohs.conf**.
      
      ```apache
      # START custom config 2020-05-11 https://github.com/4O4/oaf-development-config
      RewriteEngine On
      RewriteCond %{HTTP:X-Dev-OHS-Proxy}  ^oacore_server\d+$
      RewriteRule ^/OA_HTML(.*)$ balancer://%{HTTP:X-Dev-OHS-Proxy}/OA_HTML$1  [P,L,QSA]
      RewriteCond %{HTTP:X-Dev-OHS-Proxy}  ^weblogic-cluster$
      RewriteRule ^/OA_HTML(.*)$ balancer://%{HTTP:X-Dev-OHS-Proxy}/OA_HTML$1  [P,L,QSA]
      RewriteMap ohs_proxy_map "txt:/home/user/.config/ohs-proxy-map"
      RewriteCond %{HTTP:X-Forwarded-For}  ^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$
      RewriteCond %{HTTP:X-Forwarded-For}  !^127\.0\.0\.1$
      RewriteCond ${ohs_proxy_map:%{HTTP:X-Forwarded-For}}  ^.+$
      RewriteRule ^/OA_HTML(.*)$ balancer://${ohs_proxy_map:%{HTTP:X-Forwarded-For}}/OA_HTML$1  [P,L,QSA]
      ProxyPreserveHost On
      <Proxy balancer://oacore_server1>
      RequestHeader set X-Dev-OHS-Proxy "" 
      BalancerMember http://develop.example.com:7231
      </Proxy>
      <Proxy balancer://oacore_server2>
      RequestHeader set X-Dev-OHS-Proxy "" 
      BalancerMember http://develop.example.com:7232
      </Proxy>
      <Proxy balancer://oacore_server3>
      RequestHeader set X-Dev-OHS-Proxy "" 
      BalancerMember http://develop.example.com:7233
      </Proxy>
      <Proxy balancer://oacore_server4>
      RequestHeader set X-Dev-OHS-Proxy "" 
      BalancerMember http://develop.example.com:7234
      </Proxy>
      <Proxy balancer://oacore_server5>
      RequestHeader set X-Dev-OHS-Proxy "" 
      BalancerMember http://develop.example.com:7235
      </Proxy>
      <Proxy balancer://oacore_server6>
      RequestHeader set X-Dev-OHS-Proxy ""
      BalancerMember http://develop.example.com:7246
      </Proxy>
      <Proxy balancer://weblogic-cluster>
      RequestHeader set X-Dev-OHS-Proxy ""   
      BalancerMember http://develop.example.com:8030
      </Proxy>
      # END custom config 2020-05-11 https://github.com/4O4/oaf-development-config
      ```
   
   1. Locate the following lines:
   
      ```apache
      <Location /OA_HTML>
      SetHandler weblogic-handler
      WebLogicCluster ...
      ```
      We're going to modify the last line starting with `WebLogicCluster`. Duplicate it and comment one of them (for backup and reference). Modify the other uncommented one by leaving only 3 out of 6 of these URLs. The cores you specify here will be used for load balancing of standard traffic (the traffic from users whose IPs are not assigned to any specific free unbalanced core). In the example below I'm leaving cores 4, 5 and 6 in load balancing setup.
   
      Example of initial value:
      ```apache
      WebLogicCluster develop.example.com:7232,develop.example.com:7231,develop.example.com:7234,develop.example.com:7233,develop.example.com:7235,develop.example.com:7246
      ```
      
      Example after changes:
      ```apache
      # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
      # WebLogicCluster develop.example.com:7232,develop.example.com:7231,develop.example.com:7234,develop.example.com:7233,develop.example.com:7235,develop.example.com:7246
      WebLogicCluster develop.example.com:7234,develop.example.com:7235,develop.example.com:7246
      # END edit 2020-05-11 https://github.com/4O4/oaf-development-config
      ```
      
1. We're done with **mod_wl_ohs.conf**. Before you click **Apply** button, **make sure to backup all of your modifications made in the text field (i.e. copy it to clipboard)** to avoid losing your data just in case, becasue your control panel session might expire in the meantime.
1. We're going to modify one more file. Select **apps.conf** from the list and press **Go**

   ![Enterprise Manager menu screenshot](/images/oaf-dev-setup-3.png)

1. Backup this file somewhere, then make the following modifications:
   1. Find the line starting with `ProxyRequests Off` and insert these lines below it:
      ```apache
      # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
      Alias /OA_JAVA /ORACLE/appl/fs2/EBSapps/comn/java/classes
      Alias /OA_MEDIA /ORACLE/appl/fs2/EBSapps/comn/java/classes/oracle/apps/media
      # END edit 2020-05-11 https://github.com/4O4/oaf-development-config
      ```
      
      > **Note:** The path might be different depending on which filesystem is currently used by your applications server. Remember to change `fs2` to `fs1` if needed. The rest of the path will most likely be the same forever until some major upgrade of the Oracle systems.
      
   1. Find a code block which looks like this and comment out all lines by adding `#` at the beginning
      ```apache
       ######################
       # for oacore
       ######################
       <Location /OA_MEDIA>
         ProxyPass balancer://oacorecluster_oamedia
         ProxyPassReverse balancer://oacorecluster_oamedia
       </Location>
       <Proxy balancer://oacorecluster_oamedia>
         BalancerMember http://develop.example.com:7232/OA_HTML/media
         BalancerMember http://develop.example.com:7231/OA_HTML/media
         BalancerMember http://develop.example.com:7234/OA_HTML/media
         BalancerMember http://develop.example.com:7233/OA_HTML/media
         BalancerMember http://develop.example.com:7235/OA_HTML/media
         BalancerMember http://develop.example.com:7246/OA_HTML/media
       </Proxy>


       <Location /OA_JAVA>
         ProxyPass balancer://oacorecluster_oajava
         ProxyPassReverse balancer://oacorecluster_oajava
       </Location>
       <Proxy balancer://oacorecluster_oajava>
         BalancerMember http://develop.example.com:7232/OA_HTML/classes
         BalancerMember http://develop.example.com:7231/OA_HTML/classes
         BalancerMember http://develop.example.com:7234/OA_HTML/classes
         BalancerMember http://develop.example.com:7233/OA_HTML/classes
         BalancerMember http://develop.example.com:7235/OA_HTML/classes
         BalancerMember http://develop.example.com:7246/OA_HTML/classes
       </Proxy>
      ```
      
      Example after changes:
      ```apache
      ######################
      # for oacore
      ######################
      # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
      # <Location /OA_MEDIA>
      #   ProxyPass balancer://oacorecluster_oamedia
      #   ProxyPassReverse balancer://oacorecluster_oamedia
      # </Location>
      # <Proxy balancer://oacorecluster_oamedia>
      #   BalancerMember http://develop.example.com:7232/OA_HTML/media
      #   BalancerMember http://develop.example.com:7231/OA_HTML/media
      #   BalancerMember http://develop.example.com:7234/OA_HTML/media
      #   BalancerMember http://develop.example.com:7233/OA_HTML/media
      #   BalancerMember http://develop.example.com:7235/OA_HTML/media
      #   BalancerMember http://develop.example.com:7246/OA_HTML/media
      # </Proxy>


      # <Location /OA_JAVA>
      #   ProxyPass balancer://oacorecluster_oajava
      #   ProxyPassReverse balancer://oacorecluster_oajava
      # </Location>
      # <Proxy balancer://oacorecluster_oajava>
      #   BalancerMember http://develop.example.com:7232/OA_HTML/classes
      #   BalancerMember http://develop.example.com:7231/OA_HTML/classes
      #   BalancerMember http://develop.example.com:7234/OA_HTML/classes
      #   BalancerMember http://develop.example.com:7233/OA_HTML/classes
      #   BalancerMember http://develop.example.com:7235/OA_HTML/classes
      #   BalancerMember http://develop.example.com:7246/OA_HTML/classes
      # </Proxy>
      # END edit 2020-05-11 https://github.com/4O4/oaf-development-config
      ```
      
   1. Locate a code block which looks like this and comment it out too:
      ```apache
       ######################
       # for forms
       ######################
       <Location /OA_MEDIA>
        ProxyPass balancer://formscluster_oamedia
        ProxyPassReverse balancer://formscluster_oamedia
       </Location>
       <Proxy balancer://formscluster_oamedia>
         BalancerMember http://develop.example.com:7431/OA_HTML/media
       </Proxy>


       <Location /OA_JAVA>
        ProxyPass balancer://formscluster_oajava
        ProxyPassReverse balancer://formscluster_oajava
       </Location>
       <Proxy balancer://formscluster_oajava>
         BalancerMember http://develop.example.com:7431/OA_HTML/classes
       </Proxy>
       ```
       
       Example after changes:
       ```apache
       ######################
       # for forms
       ######################
       # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
       # <Location /OA_MEDIA>
       #  ProxyPass balancer://formscluster_oamedia
       #  ProxyPassReverse balancer://formscluster_oamedia
       # </Location>
       # <Proxy balancer://formscluster_oamedia>
       #   BalancerMember http://develop.example.com:7431/OA_HTML/media
       # </Proxy>


       # <Location /OA_JAVA>
       #  ProxyPass balancer://formscluster_oajava
       #  ProxyPassReverse balancer://formscluster_oajava
       # </Location>
       # <Proxy balancer://formscluster_oajava>
       #   BalancerMember http://develop.example.com:7431/OA_HTML/classes
       # </Proxy>
       # END edit 2020-05-11 https://github.com/4O4/oaf-pment-config
       ```
   1. Locate a code block which looks like this:
      ```apache
       <Location /OA_CGI/FNDWRR.exe>
        ProxyPass balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl 
        ProxyPassReverse balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl 
       </Location>
       <Proxy balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl>
         BalancerMember http://develop.example.com:7232
         BalancerMember http://develop.example.com:7231
         BalancerMember http://develop.example.com:7234
         BalancerMember http://develop.example.com:7233
         BalancerMember http://develop.example.com:7235
         BalancerMember http://develop.example.com:7246
       </Proxy>
       ```
       
       Comment out individual `BalancerMember` lines with the addresses of your development cores. The only uncommented cores should be the same as you've specified in `WebLogicCluster` in the **mod_wl_ohs.conf** previously.
       
       Example after changes:
       ```apache
       <Location /OA_CGI/FNDWRR.exe>
        ProxyPass balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl 
        ProxyPassReverse balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl 
       </Location>
       <Proxy balancer://oacorecluster_oacgi/OA_HTML/txkFNDWRR.pl>
       # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
         # BalancerMember http://develop.example.com:7232
         # BalancerMember http://develop.example.com:7231
       # END edit 2020-05-11 https://github.com/4O4/oaf-development-config
         BalancerMember http://develop.example.com:7234
       # START edit 2020-05-11 https://github.com/4O4/oaf-development-config
         # BalancerMember http://develop.example.com:7233
       # END edit 2020-05-11 https://github.com/4O4/oaf-development-config
         BalancerMember http://develop.example.com:7235
         BalancerMember http://develop.example.com:7246
       </Proxy>
       ```
       
   
   By making these modifications we're completely disabling load balancing on `/OA_JAVA` and `/OA_MEDIA` paths. Media files and classes are served directly from the file system instead, without any cache layers which disrupt the development process. We've also ensured that `FNDWR` program doesn't use our development cores.
   
   To apply these changes, you can restart Weblogic Admin server with:
   ```
   ${INST_TOP}/admin/scripts/adadminsrvctl.sh start
   ```
   
   You don't need to do it though if you're continuing with further configuration steps and you're going to restart the whole Application Server anyway.

## Configuring OA Core startup parameters for remote debugging

   1. Login to Weblogic Server 11g Administration Console (i.e. at http://develop.example.com:7031/console)
   1. In the left sidebar choose **Environment > Servers**

      ![Weblogic console menu screenshot](/images/oaf-dev-setup-5.png)

   1. Make sure that everything essential on the list is green, up and running

      ![Weblogic console servers table screenshot](/images/oaf-dev-setup-7.png)

   1. Before proceeding with any configuration changes, we must lock it first
      
      ![Weblogic console lock screenshot](/images/oaf-dev-setup-4.png)

   1. Open **Environment > Servers** table again

   1. Click on the development OA core name (development cores are number 1, 2 and 3 if you were following this guide strictly) to open the configuration page. 
      
      ![Weblogic console lock screenshot](/images/oaf-dev-setup-6.png)

   1. Open **Configuration > Server Start** page

      ![Weblogic console lock screenshot](/images/oaf-dev-setup-8.png)

   1. Add the following arguments to the JVM Arguments text field and press **Save**.
      ```
      -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=2721
      ```

      Port 2721 is our remote debugging port. Change it to an open port number of your choice. Use unique port number for each development core.

      Remember to add an extra space between the existing Arguments and the `-agentlib...` string you're appending. 

      ![Weblogic console lock screenshot](/images/oaf-dev-setup-9.png)

   1. Repeat steps 6-8 for each of the development cores.

   1. Press **Activate  Changes** in the top left corner.
      
      ![Weblogic console lock screenshot](/images/oaf-dev-setup-11.png)

   1. When everything is configured correctly, you should see a message like this:

      ![Weblogic console lock screenshot](/images/oaf-dev-setup-12.png)

   1. Now restart the whole Application Server for changes to take effect.

      ``` bash
      restart_appl.sh
      ```

You should now be able to connect your IDE's debugger to the remote OA Core JVM. In the debugger connection settings put the server IP and one of the port numbers that you have configured above.

> **Note:** Beware the company's firewalls

## Sample `ohs-proxy-map` config

For full reference see [`RewriteMap` in Apache HTTPD docs](https://httpd.apache.org/docs/2.4/rewrite/rewritemap.html).

Short explanation: Anything after the pound symbol `#` is a comment. Add mapping lines in the form of `<user machine IP address> <OA core ID>` for all users and their network interfaces that you want to assign to a specific OA Core.

```ini
# user1
172.20.0.1   oacore_server3 # wifi
172.99.0.1  oacore_server3 # eth

# user2
172.20.0.2 oacore_server3 # wifi
172.99.0.2 oacore_server3 # eth

# user3
172.20.0.3   oacore_server1 # wifi
```

In this example, `user1` and `user2` are assigned to the third OA Core server. They use both wireless and cable connection, so both interfaces are specified here. If the users are connected via VPN, you might need to add their VPN interface IP address to the list too.

`user3` is assigned alone to the first OA Core, but only when they use Wireless connection.

All other users / network interfaces configuration not specified here will be load-balanced between Cores 4, 5 and 6 (if using the exact same HTTP server configuration as described earlier in this document). This means that `user3` connected via ethernet will also be load balanced in this example.
  
# Warranty and Legal Disclaimer

I give absolutely no warranty for features and customizations described in this guide. I am not a lawyer and I have no idea of legal implications of all of the modifications I've proposed here. Contact your lawyer for legal advices regarding usage of this work when in doubt.

This guide is intended for use **ONLY ON DEVELOPMENT INSTANCES** of OAF / EBS application servers, to make developer's lifes easier.

Oracle® is registered trademark of Oracle and/or its affiliates. I am not affiliated with Oracle in any way and this guide is my own work on top of their technologies.

# License

_Please report feedback and contribute._

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br /><span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">Better OAF development setup</span> by <a xmlns:cc="http://creativecommons.org/ns#" href="https://github.com/4O4" property="cc:attributionName" rel="cc:attributionURL">Paweł Kierzkowski</a> is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.
