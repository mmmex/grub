# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define "grub" do |grub|

    grub.vm.box = 'centos/7'

    grub.vm.provider "virtualbox" do |vb|
      vb.gui = true
      vb.memory = "1024"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end

  end

end
