# Proyecto DNS con Vagrant

Este proyecto configura un servidor DNS maestro y un servidor esclavo usando **Vagrant** y **Debian**. La configuración incluye la resolución directa e inversa para la zona `sistema.test`. A continuación, se detalla cada paso del proceso y los archivos utilizados.

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
```
### Explicacion de cada provision
- Instalación de Bind9: Ambos servidores (tierra y venus) instalan Bind9 y las herramientas necesarias.
- Configuración del maestro: Se copian los archivos de configuración para tierra y se reinicia el servicio.
- Configuración del esclavo: Se copian los archivos de configuración para venus y se reinicia su servicio DNS

## Archivos de configuracion
### `named`
En este archivo definimos las opciones basicas del servicio Bind9
```bash
# named
RESOLVCONF=no
OPTIONS="-u bind -4"
```

### `named.conf.options`
Configuramos las opciones del servidor DNS, incluyendo las ACLs y los reenviadores.
```bash
acl trusted {
  192.168.57.0/24;
  127.0.0.0/8;
};

acl listen-addresses {
  192.168.57.103;
  192.168.57.102;
  127.0.0.1;
};

options {
  directory "/var/cache/bind";
  allow-query { trusted; };
  
  forwarders {
    208.67.222.222;
  };
  
  listen-on port 53 { listen-addresses; };
  recursion yes;
  allow-recursion { trusted; };
  dnssec-validation yes;
}

```

### `sistema.test.dns`
Define la zona directa del dominio `sistema.test`
```bash
$TTL    86400
$ORIGIN sistema.test.
@   IN  SOA tierra root.sistema.test. (
          2       ; Serial
         604800   ; Refresh
          86400   ; Retry
        2419200   ; Expire
          7200 )  ; Negative Cache TTL

@       IN  NS  tierra
@       IN  NS  venus

ns1     IN  CNAME tierra
ns2     IN  CNAME venus
@       IN  MX 10 marte

mercurio  IN  A  192.168.57.101
venus     IN  A  192.168.57.102
tierra    IN  A  192.168.57.103
marte     IN  A  192.168.57.104
mail      IN  CNAME marte

```

### `sistema.test.rev`
Archivo de la zona inversa para `192.168.57.0/24`
```bash
$TTL    86400
$ORIGIN 57.168.192.in-addr.arpa.
@   IN  SOA tierra.sistema.test root.sistema.test. (
          1       ; Serial
         604800   ; Refresh
          86400   ; Retry
        2419200   ; Expire
          7200 )  ; Negative Cache TTL

@   IN  NS  tierra.sistema.test.
@   IN  NS  venus.sistema.test.

101 IN  PTR mercurio.sistema.test.
102 IN  PTR venus.sistema.test.
103 IN  PTR tierra.sistema.test.
104 IN  PTR marte.sistema.test.

```

### `named.conf.local` (tierra)
Define las zonas para el servidor maestro.
```bash
zone "sistema.test" {
    type master;
    file "/var/lib/bind/sistema.test.dns";
    allow-transfer { 192.168.57.102; };
};

zone "57.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/sistema.test.rev";
    allow-transfer { 192.168.57.102; };
};

```

### `named.conf.local` (venus)
Define las zonas para el servidor esclavo.
```bash
zone "sistema.test" {
    type slave;
    file "/var/lib/bind/sistema.test.dns";
    masters { 192.168.57.103; };
};

zone "57.168.192.in-addr.arpa" {
    type slave;
    file "/var/lib/bind/sistema.test.rev";
    masters { 192.168.57.103; };
};

```

## Comprobacion del servidor DNS
Ejecutamos el script que nos ha proporcionado el profe para comprobar que este todo en orden
### Antes de ejecutar el script
Debemos de darle permisos al archivo test.sh que es el donde se encuentra:
```bash
$ chmod +x test/test.sh
$ cd test/
$ ./test.sh 192.168.57.103
```
```bash
#!/bin/bash -x
#
# USAGE: ./test.sh <nameserver-ip>
#

# Salir si algún comando falla
set -euo pipefail

function resolver () {
    dig $nameserver +short $@
}

nameserver=@$1

resolver mercurio.sistema.test
resolver venus.sistema.test
resolver tierra.sistema.test
resolver marte.sistema.test

resolver ns1.sistema.test
resolver ns2.sistema.test

resolver sistema.test mx

resolver sistema.test ns

resolver -x 192.168.57.101
resolver -x 192.168.57.102
resolver -x 192.168.57.103
resolver -x 192.168.57.104
```
