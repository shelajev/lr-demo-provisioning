# -*- mode: ruby -*-
# vi: set ft=ruby :

@lr_subnet = "10.127.128"
@lr_ip_host = "#{@lr_subnet}.1"
@lr_ip_tomcatcluster = "#{@lr_subnet}.2"
@lr_ip_tomcat1 = "#{@lr_subnet}.3"
@lr_ip_tomcat2 = "#{@lr_subnet}.4"
@lr_ip_phpcluster = "#{@lr_subnet}.5"
@lr_ip_php1 = "#{@lr_subnet}.6"
@lr_ip_php2 = "#{@lr_subnet}.7"

@apt_repository = "http://ftp.estpak.ee/ubuntu/"
@selenium_base_url = "http://selenium.googlecode.com/files/"

Vagrant.configure("2") do |config|
  config.vm.box = "precise32"

  config.vm.provider :vmware_fusion do |vb|
    vb.gui = false
    vb.customize ["modifyvm", :id, "--memory", 384]
  end

  config.vm.provider :virtualbox do |v|
    v.gui = false
    v.vmx["memsize"] = "512"
  end

  config.vm.define :tomcatcluster do |config|
    config.vm.network :private_network, ip: @lr_ip_tomcatcluster
    config.vm.provision :chef_solo do |chef|
      chef_config(chef)
      chef_apt_config(chef)
      chef_hosts_config(chef)
      chef_cluster_config(chef, @lr_ip_tomcatcluster)
      chef.json.deep_merge!({
        :cluster => {
          :sessionid => "JSESSIONID|jsessionid",
          :nodeport => 8080,
          :scolonpathdelim => true,
          :nodes => [@lr_ip_tomcat1, @lr_ip_tomcat2]
        }
      })
    end
  end

  config.vm.define :tomcat1 do |config|
    chef_tomcat(config, @lr_ip_tomcat1, 1)
  end

  config.vm.define :tomcat2 do |config|
    chef_tomcat(config, @lr_ip_tomcat2, 2)
  end

  config.vm.define :phpcluster do |config|
    config.vm.network :private_network, ip: @lr_ip_phpcluster
    config.vm.provision :chef_solo do |chef|
      chef_config(chef)
      chef_apt_config(chef)
      chef_hosts_config(chef)
      chef_cluster_config(chef, @lr_ip_phpcluster)
      chef.json.deep_merge!({
        :cluster => {
          :sessionid => "BALANCEID",
          :nodeport => 9090,
          :nodes => [@lr_ip_php1, @lr_ip_php2]
        }
      })
    end
  end

  config.vm.define :php1 do |config|
    chef_php(config, @lr_ip_php1, 1)
  end

  config.vm.define :php2 do |config|
    chef_php(config, @lr_ip_php2, 2)
  end
end

def chef_config(chef)
  chef.cookbooks_path = "cookbooks"
  chef.data_bags_path = "data_bags"
  chef.roles_path = "roles"
end

def chef_hosts_config(chef)
  chef.add_recipe "liverebel-hosts"
  chef.json.deep_merge!({
    :hosts => {
      :host => @lr_ip_host,
      :java => @lr_ip_tomcatcluster,
      :java1 => @lr_ip_tomcat1,
      :java2 => @lr_ip_tomcat2,
      :php => @lr_ip_phpcluster,
      :php1 => @lr_ip_php1,
      :php2 => @lr_ip_php2
    }
  })
end

def chef_apt_config(chef)
  chef.json.deep_merge!({
    :apt => {
      :repository => @apt_repository
    }
  })
end

def chef_cluster_config(chef, ipAddress)
  chef.add_recipe "liverebel-cluster-node"
  chef.json.deep_merge!({
    :liverebel => {
      :hostip => @lr_ip_host,
      :agentip => ipAddress,
      :agent => {
        :user => 'lragent',
        :type => 'database-agent'
      }
    },
    :mysql => {
      :bind_address => ipAddress,
      :allow_remote_root => true,
      :server_user_password => "change_me",
      :server_root_password => "change_me",
      :server_repl_password => "change_me",
      :server_debian_password => "change_me"
    }
  })
end

def chef_tomcat_config(chef, ipAddress, identifier)
  chef.add_recipe "liverebel-tomcat-node"
  chef.json.deep_merge!({
    :liverebel => {
      :hostip => @lr_ip_host,
      :agentip => ipAddress,
      :tomcat_tunnelport => 18080+identifier
    },
    :tomcat => {
      :jvm_route => identifier
    },
    :selenium => {
      :base_url => @selenium_base_url
    }
  })
end

def chef_php_config(chef, ipAddress, identifier)
  chef.add_recipe "liverebel-php-node"
  chef.json.deep_merge!({
    :liverebel => {
      :hostip => @lr_ip_host,
      :agentip => ipAddress,
      :php_tunnelport => 19080+identifier,
      :agent => {
        :user => 'lragent',
        :group => 'www-data',
        :type => 'proxy'
      }
    },
    :php => {
      :server_route => identifier
    },
    :phpunit => {
      :install_method => "pear",
      :version => "3.7.14"
    }
  })
end

def chef_tomcat(config, ipAddress, identifier)
  config.vm.network :private_network, ip: ipAddress
  config.vm.provision :chef_solo do |chef|
    chef_config(chef)
    chef_apt_config(chef)
    chef_hosts_config(chef)
    chef_tomcat_config(chef, ipAddress, identifier)
  end
end

def chef_php(config, ipAddress, identifier)
  config.vm.network :private_network, ip: ipAddress
  config.vm.provision :chef_solo do |chef|
    chef_config(chef)
    chef_apt_config(chef)
    chef_hosts_config(chef)
    chef_php_config(chef, ipAddress, identifier)
  end
end

class Hash
  def deep_merge(other_hash, &block)
    dup.deep_merge!(other_hash, &block)
  end

  def deep_merge!(other_hash, &block)
    other_hash.each_pair do |k,v|
      tv = self[k]
      if tv.is_a?(Hash) && v.is_a?(Hash)
        self[k] = tv.deep_merge(v, &block)
      else
        self[k] = block && tv ? block.call(k, tv, v) : v
      end
    end
    self
  end
end
