# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.provision "shell", inline: <<-SHELL
  apt-get update 
  apt-get upgrade -y
  apt-get install -y bind9 dnsutils
  SHELL

  config.vm.define "tierra" do |master|
    master.vm.network "private_network", ip: "192.168.57.10"

     master.vm.provision "shell", name: "master-dns", inline: <<-SHELL
      cp -v /vagrant/named /etc/default
      cp -v /vagrant/named.conf.options /etc/bind
      cp -v /vagrant/tierra/named.conf.local /etc/bind
      cp -v /vagrant/dns.sri /var/lib/bind
      systemctl restart named
      SHELL
  end
  config.vm.define "venus" do |slave|
    slave.vm.network "private_network", ip: "192.168.57.11"
    slave.vm.provision "shell", name: "slave-dns", inline: <<-SHELL
    cp -v /vagrant/named /etc/default/
    cp -v /vagrant/named.conf.options /etc/bind
    cp -v /vagrant/venus/named.conf.local /etc/bind
    systemctl restart named
    SHELL
  end
end
