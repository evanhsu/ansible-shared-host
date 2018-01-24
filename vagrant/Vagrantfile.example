# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'


vagrantConfig = YAML.load_file(File.join(File.dirname(__FILE__), 'vagrant.yml'))

Vagrant.configure("2") do |config|

  server = Hash.new
  server["hostname"] = "firecrew.us.local"
  server["box"] =  "bento/ubuntu-16.04"
  server["cpus"] = 1
  server["memory"] = 1024

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = false
  end

  config.ssh.forward_agent = true

  config.vm.define server["hostname"] do |srv|

       srv.vm.box = server["box"]

       # Network Settings
       srv.hostmanager.enabled = true
       srv.hostmanager.manage_host = true
       srv.hostmanager.ignore_private_ip = false
       srv.hostmanager.include_offline = false
       srv.vm.network "private_network", :ip => "0.0.0.0", :auto_network => true

       # VirtualBox settings
       srv.vm.provider :virtualbox do |vb|
         vb.name = server["hostname"]
         vb.customize ["modifyvm", :id, "--cpus", server['cpus']]
         vb.customize ["modifyvm", :id, "--memory", server['memory']]
         vb.customize ["guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 60000]
         vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
         vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
       end

       server_aliases = Array.new
       username = "none"
       vagrantConfig["users"].each do |user|

         if user["vhosts"]
           username = user["name"]
           user["vhosts"].each do |vhost|
             server_aliases.push(vhost["server_name"]);
             if vhost["server_aliases"]
                 server_aliases += vhost["server_aliases"]
             end

             # Mount local folders to each site root and grant ownership to the site-user
             config.vm.synced_folder ".", "/home/#{username}/sites/#{vhost["server_name"]}",
               id: "approot",
               owner: user['id'],
               group: user['id']
           end
         end

         if user["upstreams"]
           user["upstreams"].each do |upstream|
             server_aliases.push(upstream["server_name"]);
             if upstream["server_aliases"]
                 server_aliases += upstream["server_aliases"]
             end
           end
         end

         if user["mounts"]
           user["mounts"].each do |mount|
             config.vm.synced_folder mount["source"], mount["destination"],
               id: mount["source"],
               owner: user['id'],
               group: user['id']
           end
         end
       end

       srv.hostmanager.aliases = server_aliases



       # Run the Ansible playbook to provision the box
       config.vm.provision "ansible-playbook", type: "ansible" do |ansible|
         ansible.vault_password_file = "./ansible_vault_password"
         ansible.extra_vars = vagrantConfig
         ansible.become = true
         ansible.playbook = "./ansible/sharedhost.yml"
         ansible.groups = {
            "core"              => [server['hostname']],
            "http"              => [server['hostname']],
            "sql"               => [server['hostname']],
            "dev"               => [server['hostname']],
            "postgresql"        => [server['hostname']],
            "mail"              => [server['hostname']],
            "vagrant:children"  => ["core", "http", "sql", "dev", "postgresql", "mail"]
         }
       end
  end

end
