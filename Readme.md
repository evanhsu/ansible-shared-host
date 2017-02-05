# DevOps Framework for Hosting Machines
* This is a collection of Ansible Roles and Playbooks that are used to provision web hosting machines.
* Currently, the playbooks are configured to provision a machine with a webserver, Redis cache, and a database rather
than using separate machines for each role.

## Prerequisites:

### The Control Machine:
_(Commands provided are specific to Ubuntu)_

  * Must have Git installed:
  
        apt-get update
        apt-get -y install git
        
        
  * Must have Ansible installed
  
        sudo apt-get install software-properties-common
        sudo apt-add-repository ppa:ansible/ansible
        sudo apt-get update
        sudo apt-get install ansible
  
  * Must have the 'passlib' Python module for generating passwords
  
        pip install passlib --user python
  
  * Must have the 'dopy' Python module installed (Digital Ocean API wrapper)
  
        pip install dopy --user python
  
  * Must have the 'sshpass' program (when connecting to a local Vagrant machine only - 
    All other environments should use private-keys instead of passwords)
  
        brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb

### The machine being managed (a 'managed node')

  * On the managed nodes, you need a way to communicate, which is normally ssh. 
    By default this uses sftp. 
    If that’s not available, you can switch to scp in ansible.cfg. 
  * You also need Python 2.4 or later. If you are running less than Python 2.5 on the remotes, you will also need:
  
        python-simplejson

## Editing the list of servers to be provisioned:

1. Each machine being managed by this codebase should have an entry in the 'inventory':

    * Open the `/inventory` directory and set the URLs and hostnames for the local, staging, 
    and production machines being managed.
    
    * To specify a server that should be provisioned by this script but not dynamically created,
    add the address (IP or domain) to the `[web]` section of an inventory file.
    
        * Example: To provision a static server for Staging, the `inventory/staging` file should contain:
                
                [web]
                staging.my-domain.com
    
    * To add a Digital Ocean droplet to the inventory, just add the name of the droplet beneath the `[droplets]`
    heading in an inventory file.  If the droplet doesn't already exist when the script runs, it will be dynamically
    created.  If it already exists, it will 
        
        * Example: To provision 2 identical Production droplets, the `inventory/production` file should contain:
        
                [droplets]
                my-production-droplet-1
                my-production-droplet-2
    
        * To change the type of Droplet that you want to use for a particular environment, set the 
        variables in the `[droplets:vars]` section of the inventory file (i.e. inventory/production):
        
                [droplets:vars]
                droplet_size_id=2gb
                droplet_region_id=sfo2
                droplet_image_id=ubuntu-16-04-x64
            

## Setting up vhosts, user accounts, and databases for a website

1. All of the vhosts ('server blocks') that should exist for the Nginx webserver are defined in the `group_vars`
   folder in a file named after their environment (i.e. `group_vars/production.yml` defines users and vhosts on
   production machines).  The structure of these files is as follows:
   
       ---
       GROUPS_LIST:
       
         - &developers
           name: "developers"
           id: 1000
         - &clients
           name: "clients"
           id: 1001
       
       DEVELOPERS_LIST:
       
         - &jeff
           name: "jeff"
           id: 2000
           groups:
             - "developers"
             - "sudo"
           ssh_keys:
             - "jeff.pub"
             - "jeff_at_macbook.pub"
       
       CLIENTS_LIST:
       
         - &sunvalleymag
           name: "sunvalleymag"
           id: 3017
           groups:
             - "clients"
             
       FTPUSERS_LIST:
       
         - &sunvalleymag-ftp
           name: "sunvalleymag-ftp"
           password: $1$SomeSalt$CyaZSTvM30xGoZ8PajwNA1
           root: "/home/sunvalleymag/sites"
           id: 4000
           groups:
             - "sunvalleymag"
             
       user_groups:
         - <<: *developers
         - <<: *clients
         - <<: *ftpusers
       
       users:
         - <<: *jeff
         - <<: *sunvalleymag
         
           databases:
             - name: "sunvalleymag"

           vhosts:
             - listen: "80"
               server_name: "sunvalleymag.com"
               server_aliases:
                 - "www.sunvalleymag.com"
               wordpress: true
               php: true
               root: "/home/sunvalleymag/sites/sunvalleymag.com/public"


1. When a vhost is added that didn't previously exist on the server, the following actions are taken on the server:

    1. A new user is created with a home folder but no password (`/sbin/nologin`)
    2. A server block (vhost) is added to nginx.
    3. A new MariaDb database is created for each member of the `users.databases` list (defined in the `group_vars` YAML
       files). Connections to these db's have no password, but are only allowed from localhost.

1. When a vhost is removed (or renamed) in the vhosts.yml file, its symlink is deleted from 
   the `/etc/nginx/sites-enabled` folder, but no other changes are made. 
   
   All of the site data remains in the document root, and the vhost config file remains in `/etc/nginx/sites-available`.
   
1. Example vhost entry with ALL OPTIONS documented:

 Example vhost entry with all options:

    - <<: *client_username
      databases:
        - name: "client_database_1"
        - name: "client_database_2"
        
      vhosts:
        - listen: "80"
          server_name: "ansible-test.com"                         # REQUIRED
          server_aliases: 
              - "www.ansible-test.com www2.ansible-test.com"
          root: "/home/ansibletest/ansible-test.com/public"       # REQUIRED
            
          # The 'extra_parameters' will get copied directly
          # into your nginx server block and can be used to implement
          # a custom behavior.
          extra_parameters: >
              location ~ ^/(assets/|images/|img/|favicon.ico) {
                access_log off;
                expires max;
          }
              
          redis_port: "6379"
            
          # Install SSL certificates for this site - these files must exist in the `/ssl-certificates` folder of this repo.
          ssl:
              ssl_cert_filename: sunvalleymag.com.crt
              ssl_key_filename: sunvalleymag.com.key
            
          # Add cron jobs to this user's crontab by adding them each on their own line
          cron:
               - "* * * * * (cd /home/boise/sites/boise.guru && php artisan schedule:run)"
               - "* * * * * (cd /home/boise/sites/boise.guru && php artisan inspire)"
                  
          # Use the Wordpress variable to apply nginx snippets to this vhost's server config
          wordpress: true
            
          # Set `php` to `true` to create a php-fpm pool for this user
          php: true 
            
          php_fpm:
              listen_allowed_clients: 127.0.0.1
              pool_user:  www-data
              pool_group: www-data
              pm_start_servers: 5
              pm_max_children: 5
              pm_min_spare_servers: 5
              pm_max_spare_servers: 10
              pm_max_requests: 500
              php_admin_disable_functions: "exec,passthru,shell_exec,system"
              php_admin_allow_url_fopen: "off"
              
          nodejs:
              version: '6.6.0'
              packages:
                - name: eslint
                - name: stylelint
             
          pm2_apps:
              - run: "/home/oliverfinley/sites/oliverfinley.com/process.json"
                env:
                  NODE_ENV: staging
              
          env:
              - APP_DEBUG=false
              - APP_ENV=production
              - CACHE_DRIVER=redis
              - SESSION_DRIVER=file
              - QUEUE_DRIVER=sync
              # DON'T CREATE DATABASE ENTRIES (DB_HOST, DB_USERNAME, etc)
              # THESE ARE AUTOMATICALLY GENERATED FROM THE 'database' KEY ABOVE
       
      upstreams:
       
        - listen: "80"
          server_name: "oliverfinley.staging.portonefive.com"
          server_aliases:
            - "www.oliverfinley.staging.portonefive.com"
          name: ofa-frontend
          location: "/"
          strategy: "ip_hash"
          keepalive: 32
          servers: {
            "localhost:3000",
          }
       
          extra_parameters: >
            location /assets {
                root /home/oliverfinley/sites/admin.oliverfinley.staging.portonefive.com/public;
            }
        

## Deploying your changes
This codebase is meant to reflect the current state of live hosting machines. Previous configurations
can be inspected by using version control.

1. Changes to this repository should follow a similar Git workflow as other projects:
    1. New changes should be made on the `develop` branch.  Testing these new changes should be done locally 
    (on a Vagrant machine, for example)
    
    2. Applying your changes to the staging server is done by merging into the `staging` branch. This
       triggers a Bamboo build that runs:
    
            $ ansible-playbook portonefive.yml -i inventory/staging.yml
        
       which executes the script on each server in the `inventory/staging.yml` file.
    
    3. Applying your changes to the production server is done by merging into the `master` branch. This
       triggers a Bamboo build that runs:
       
            $ ansible-playbook hosting-box.yml -i inventory/production.yml
        
       which executes the script on each server in the `inventory/production.yml` file.

  
 ## Changing the nginx configuration
 
 All of the nginx configuration variables can be set in the default vars file for the nginx role:
 
* Global vars
    
        roles/geerlingguy.nginx/defaults/main.yml
    
* Ubuntu-specific vars:
    
        roles/geerlingguy.nginx/vars/Debian.yml
  
## Troubleshooting

### Errors during the Ansible playbook execution

* If the Ansible playbook fails because of unreachable hosts, it could be caused by Ansible attempting to provision a new Digital Ocean droplet before the droplet has finished booting up.  Ansible is supposed to wait for the droplet to become available on port 22 before beginning to provision, but sometimes it doesn't work.  This issue resolves itself if the playbook is executed again after the droplet has finished booting up (usually within 10 seconds or so).  This issue will only arise the first time the script runs after adding a new droplet to an inventory file.

* If the playbook fails on the `restart nginx` task:

    RUNNING HANDLER [geerlingguy.nginx : reload nginx] *****************************
    fatal: [138.68.13.234]: FAILED! => {"changed": false, "failed": true, "msg": "Job for nginx.service failed because the control process exited with error code. See \"systemctl status nginx.service\" and \"journalctl -xe\" for details.\n"

  there is most likely a problem with your vhost configuration.  Make sure that any changes to the group_vars files follow the examples in this Readme.

### Problems with the server after provisioning
  
* 404 Not Found

  * Try adding an entry to your `hosts` file to connect your Digital Ocean droplet's IP with the hostname that you're trying to visit.  This will solve issues where there is either no DNS record configured (no domain has been linked to the droplet's IP) or the DNS hasn't taken effect yet (if you've just recently created the DNS records).
  * Make sure to check Cloudflare's DNS records if the domain you're trying to connect to is cached by Cloudflare!

* 502 Bad Gateway 
  
  * SSH into the droplet and run `systemctl status nginx.service`
    This will tell you if there's something wrong with the nginx service.

  * SSH into the droplet and run `sudo nginx -t`
    This will check the nginx vhost config files and tell you if there's any syntax errors.  Make sure to run this command with `sudo` or else the test will report lots of permissions errors because it's not able to access the files that you've asked it to test.

  * SSH into the droplet and run `sudo vim /home/VHOST-USER/logs/DOMAIN-error.log` making sure to replace `vhost-user` and `DOMAIN` with your actual values.  For example, the nginx error log for `portonefive.com` (owned by the `portonefive` user) is located at `/home/portonefive/logs/portonefive.com-error.log`.  If you see this error:
  `connect() failed (111: Connection refused) while connecting to upstream...` then you have an invalid vhost configuration.  This problem arises when you have a `proxy_pass` statement inside a `location` block in your vhost config file that points to a hostname that isn't defined by an `upstream` statement at the top of that same file.

  * SSH into the droplet and run `sudo vim /etc/nginx/sites-available/yourdomain.com`
    Make sure to replace `yourdomain.com` with the actual domain name that you entered into the vhost settings in the ansible script.  This will open up the vhost configuration file for your site.

    