# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure('2') do |config|
  config.vm.box = 'bento/ubuntu-16.04'
  config.vm.provider 'virtualbox' do |vb|
    vb.gui = false
    vb.memory = '4096'
  end
  config.vm.provision 'shell', inline: 'apt-get update'
  {
    'ansible-awx'   => '192.168.33.11',
    'ansible-lab'   => '192.168.33.12'
  }.each do |short_name, ip|
    config.vm.define short_name do |host|
      host.vm.network 'private_network', ip: ip
      host.vm.hostname = "#{short_name}.dev"
    end
  end
end
