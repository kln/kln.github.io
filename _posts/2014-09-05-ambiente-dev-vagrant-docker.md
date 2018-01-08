---
layout: post
title:  "Criando um ambiente de desenvolvimento com Vagrant e Docker"
date:   2014-09-05 22:00
categories: [vagrant,docker,linux]
permalink: /vagrant-docker-linux
description: "Como criar seu ambiente de desenvolvimento com vagrant e docker de forma simples e rápida"
author: kelvin
---


Atualmente meu ambiente de desenvolvimento conta com três box, duas para diferentes aplicações e uma para serviços como mysql, elasticsearch, redis etc. Além de se comunicarem diretamente, as aplicações também utilizam os mesmos serviços.

Manter as três VMs rodando consome muitos recursos da máquina, foi ai que decidi unir a performance dos containers utilizando o <a href="https://www.docker.com/whatisdocker/" target="_blank">Docker</a> e a praticidade de uma box utilizando o <a href="https://vagrantcloud.com/" target="_blank">Vagrant</a>.

Como estou começando a estudar sobre docker agora, me propus algo um pouco mais simples, um ambiente com uma app rails e um banco de dados mysql.

Primeiro vamos criar a maquina virtual com vagrant que manterá o ambiente totalmente isolado do host. Uma lista de base boxes está disponível <a href="https://www.vagrantbox.es" target="_blank">aqui</a>. Nesse exemplo vou utilizar um Ubuntu 14.04 64 bits, vale lembrar também que o docker só é compatível com sistemas 64 bits. O Docker vai manter tudo que diz respeito à aplicação isolado do banco de dados. A imagem abaixo esclarece bem essa diferença:

![alt text; "Docker vs Vagrant"](/images/vagrant-docker1/dockervsvagrant.png)

Criando a VM:

~~~ ruby
vagrant box add devhost https://cloud-images.ubuntu.com/vagrant/trusty/current/trusty-server-cloudimg-amd64-vagrant-disk1.box
vagrant init devhost
vagrant up
~~~

Acesse a VM (```vagrant ssh```) e vamos instalar o docker. <a href="https://docs.docker.com/installation/#installatio" target="_blank">Clique aqui</a> para ver um manual mais completo de instalação.

```
curl -sSL https://get.docker.io/ubuntu/ | sudo sh
```

![alt text; "Docker Version"](/images/vagrant-docker1/docker-version.png)


_Todos os comandos do docker devem ser executados como root_

Existem algumas imagens já prontas, mas eu decidi criar a minha, para isso vamos criar um arquivo chamado **Dockerfile** onde iremos listar todos os comandos para a criação da imagem.

Vou começar criando o container para o banco de dados:

~~~
mkdir mysql-docker
cd mysql-docker
vim Dockerfile
~~~

E adicionar os seguintes comandos:

~~~
### MYSQL DOCKERFILE
FROM ubuntu:14.04
MAINTAINER Maintaner Kelvin
RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get -y install mysql-client mysql-server
RUN sed -i -e"s/^bind-address\s*=\s*127.0.0.1/bind-address = 0.0.0.0/" /etc/mysql/my.cnf
RUN /usr/sbin/mysqld & \
    sleep 10s &&\
    echo "GRANT ALL ON *.* TO myuser@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION; FLUSH PRIVILEGES" | mysql
EXPOSE 3306
CMD ["/usr/bin/mysqld_safe"]
~~~

Para criar a imagem rodamos o seguinte comando:

```
docker build -t mysql_server:0.1 .
```

Para iniciar o processo:

```
docker run --name mysql_server -d -p 3306:3306 mysql_server:0.1
```

Antes de criar a proxima imagem, vou criar na minha máquina um diretório para a aplicação. Esse diretório será compartilhado com a box do vagrant.

~~~
mkdir -p projects/app
cd projects/app
git clone git@github.com/my/app .
~~~

Agora edito meu Vagrantfile:

~~~
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "devhost"
  config.vm.synced_folder "~/projects/app", "/home/vagrant/app"
  config.vm.network :forwarded_port, guest: 3000, host: 3000
end
~~~

Na minha box do vagrant crio um diretório no home chamado app. Reiniciando a VM (```vagrant reload```) eu vou conseguir enxergar os arquivos da minha aplicação nesse diretório.
Agora vamos criar uma imagem para a aplicação:

~~~
mkdir app-docker
cd app-docker
vim Dockerfile
~~~

E adicionar os seguintes comandos:

~~~
### APP DOCKERFILE
FROM ubuntu:14.04
MAINTAINER Maintaner Kelvin
RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get -y install build-essential ruby2.0 ruby2.0-dev nodejs libmysqlclient-dev libv8-dev bundler vim
RUN mkdir /app
WORKDIR /app
EXPOSE 3000
~~~

Para criar a imagem rodamos o seguinte comando:

```
docker build -t app:0.1 .
```

Agora vamos executar a images que criamos para a aplicação:

```
docker run -it --name rails-app -v /home/vagrant/app:/app --link mysql_server:mysql -p 3000:3000 app:0.1 /bin/bash
```

**-v** é para montar o diretório da aplicação dentro do container\\
**--link** vai que permitir que tenhamos acesso ao IP do container do banco de dados através de uma variável de ambiente\\
**-p** é a porta\\
**app:0.1** é o nome da imagem e a tag\\
**/bin/bash** é o comando executado no container


Agora vou editar meu arquivo para conectar com banco de dados, como é uma aplicação em rails, edito o ```database.yml``` passando como host a variável de ambiente gerada pelo comandos **--link**

~~~
defaults: &defaults
  adapter: mysql2
  encoding: utf8
  reconnect: false
  pool: 5
  username: 'myuser'
  password: 'mypassword'
  host: <%= ENV['MYSQL_PORT_3306_TCP_ADDR'] %>
  port: 3306

development:
  <<: *defaults
  database: simpleappdb_dev

test:
  <<: *defaults
  database: simpleappdb_test
~~~

feito isso, vou instalar as gems, criar o banco de dados, rodar as migrações, e finalmente subir o server:

~~~
bundle install
rake db:create
rake db:migrate
rails s
~~~

agora basta acessar localhost:3000

![alt text; "Running"](/images/vagrant-docker1/running.png)


Você pode verificar o status de cada container com o comando ```docker ps -a``` . Para iniciar ou parar um container você pode utilizar o comando ```docker start|stop name```:

![alt text; "Docker PS"](/images/vagrant-docker1/dockerps.png)

Se você tem alguma sugestão para melhorar esse processo ou alguma dica, por favor não deixe de comentar.

Até a pŕoxima :D

