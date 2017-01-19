# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
if File.file?('vagrant.yaml')
  vagrant_config = YAML.load_file('vagrant.yaml')
elsif File.file?('vagrant.yaml.dist')
  vagrant_config = YAML.load_file('vagrant.yaml.dist')
end

Vagrant.configure("2") do |config|
  ###############################################################################
  # Plugin management                                                           #
  ###############################################################################
  if vagrant_config['required_plugins']
    plugins = vagrant_config['required_plugins']
  else
    plugins = ['vagrant-hostmanager']
  end
  plugins.each do |plugin|
    unless Vagrant.has_plugin?(plugin)
      raise plugin << " has not been installed."
    end
  end

  ###############################################################################
  # Plugin settings                                                             #
  ###############################################################################
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :machine
  end

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.ignore_private_ip = false
    config.hostmanager.include_offline = true
  end

  ###############################################################################
  # Puppet settings                                                             #
  ###############################################################################
  if vagrant_config['puppet_environment']
    environment = vagrant_config['puppet_environment']
  else
    environment = 'production'
  end

  if vagrant_config['puppet_version']
    version = vagrant_config['puppet_version']
  else
    version = :latest
  end
  if Vagrant.has_plugin?("vagrant-puppet-install")
    config.puppet_install.puppet_version = version
  end

  if Vagrant.has_plugin?("vagrant-triggers")
    config.trigger.after [:destroy] do
      target = @machine.config.vm.hostname.to_s
      puppetmaster = "puppetmaster"
      if target != puppetmaster
        system("vagrant ssh #{puppetmaster} -c 'sudo /usr/bin/puppet cert clean #{target}'" )
      end
    end
  end

  ###############################################################################
  # VirtualBox settings                                                         #
  ###############################################################################
  if vagrant_config['virtualbox_group']
    config.vm.provider 'virtualbox' do |v|
      v.customize [
        'modifyvm', :id,
        '--groups', vagrant_config['virtualbox_group']
      ]
    end
  end


  ###############################################################################
  # VM definitions                                                              #
  ###############################################################################
  config.vm.synced_folder ".", "/vagrant", disabled: true
  vagrant_config['nodes'].each do |node|
    config.vm.define node["name"] do |srv|
      srv.vm.box                = node["box"]
      if node["box_version"]
        srv.vm.box_version      = node["box_version"]
      end
      if node["box_check_update"]
        srv.vm.box_check_update = node["box_check_update"]
      end
      srv.vm.hostname           = node["hostname"]
      if node["ports"]
        node["ports"].each do |port|
          srv.vm.network :forwarded_port, guest: port["guest"], host: port["host"]
        end
      end
      srv.vm.provider "virtualbox" do |v|
        if node["cpus"]
          v.cpus   = node["cpus"]
        end
        if node["memory"]
          v.memory = node["memory"]
        end
      end
      srv.vm.network :private_network, ip: node["ip"]
      if node["aliases"]
        srv.hostmanager.aliases = node["aliases"]
      end
      if node["synced_folders"]
        node["synced_folders"].each do |folder|
          srv.vm.synced_folder folder["src"], folder["dst"]
        end
      end
      srv.vm.synced_folder "#{environment}/hieradata", "/etc/puppetlabs/code/environments/#{environment}/hieradata"
      srv.vm.provision :puppet do |puppet|
        puppet.environment = "#{environment}"
        puppet.environment_path = "."
        puppet.hiera_config_path = "#{environment}/hiera.yaml"
      end
    end
  end
end
