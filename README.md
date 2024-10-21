# Proyecto DNS con Vagrant

Este proyecto configura un servidor DNS maestro y un servidor esclavo usando **Vagrant** y **Debian**. La configuración incluye la resolución directa e inversa para la zona `sistema.test`. A continuación, se detalla cada paso del proceso y los archivos utilizados.

## Requisitos previos

- [Vagrant](https://www.vagrantup.com/) y [VirtualBox](https://www.virtualbox.org/) instalados.
- Archivos de configuración DNS proporcionados en el directorio del proyecto.

## Estructura del Proyecto

- `Vagrantfile`: Configuración de Vagrant para levantar las máquinas.
- `named`: Configuración básica de Bind9.
- `named.conf.options`: Opciones de configuración de Bind9.
- `sistema.test.dns`: Zona directa para `sistema.test`.
- `sistema.test.rev`: Zona inversa para `192.168.57.0/24`.
- `named.conf.local`: Configuración local del servidor maestro y esclavo.

## Configuración de la Red

La red utilizada es **192.168.57.0/24**. Las máquinas configuradas son las siguientes:

| Máquina       | FQDN               | IP              |
|---------------|--------------------|-----------------|
| `mercurio`    | `mercurio.sistema.test` | `192.168.57.101` |
| `venus`       | `venus.sistema.test`    | `192.168.57.102` |
| `tierra`      | `tierra.sistema.test`   | `192.168.57.103` |
| `marte`       | `marte.sistema.test`    | `192.168.57.104` |

## Vagrantfile

El archivo `Vagrantfile` configura dos máquinas: `tierra` (servidor maestro) y `venus` (servidor esclavo). A continuación, el código:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  
  # Provisión básica
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update 
    apt-get upgrade -y
    apt-get install -y bind9 dnsutils
  SHELL

  # Configuración de la máquina 'tierra' (DNS Maestro)
  config.vm.define "tierra" do |master|
    master.vm.network "private_network", ip: "192.168.57.103"
    master.vm.hostname = "tierra.sistema.test"
    
    # Provisión del DNS Maestro
    master.vm.provision "shell", name: "master-dns", inline: <<-SHELL
      cp -v /vagrant/named /etc/default
      cp -v /vagrant/named.conf.options /etc/bind
      cp -v /vagrant/tierra/named.conf.local /etc/bind
      cp -v /vagrant/sistema.test.rev /var/lib/bind/
      cp -v /vagrant/sistema.test.dns /var/lib/bind/
      chown :bind /var/lib/bind/*
      systemctl restart named
    SHELL
  end

  # Configuración de la máquina 'venus' (DNS Esclavo)
  config.vm.define "venus" do |slave|
    slave.vm.network "private_network", ip: "192.168.57.102"
    slave.vm.hostname = "venus.sistema.test"
    
    # Provisión del DNS Esclavo
    slave.vm.provision "shell", name: "slave-dns", inline: <<-SHELL
      cp -v /vagrant/named /etc/default/
      cp -v /vagrant/named.conf.options /etc/bind
      cp -v /vagrant/venus/named.conf.local /etc/bind
      systemctl restart named
    SHELL
  end
end
