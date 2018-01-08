---
layout: post
title:  "Atualizando a versão do Rails"
date:   2014-11-08 22:00
categories: [rails,upgrading,atualizar,gem,update,versao,framework,ruby]
description: "Nesse post conto um pouco da minha experiência atualizando a versão do rails"
permalink: /upgrading-rails
author: kelvin
---


Atualizar o rails em uma aplicação grande é sempre uma tarefa difícil, nesse post vou contar como foi minha experiência migrando o rails da versão 3.1.10 para a 3.2.18.

O primeiro passo é sempre olhar o que a documentação nos recomenda: 

<a href="http://edgeguides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-3-1-to-rails-3-2" target="_blank">Upgrading Rails</a>

* Atualizar o Gemfile;
* Duas recomendações de configuração para o ambiente de desenvolvimento e teste;
* Um alerta sobre o vendor/plugins;
* Alterar :dependent => :restrict por :dependent => :destroy.

Legal, parece simples, então vamos começar alterando o Gemfile:

{% highlight ruby %}
gem 'rails', '3.2.18'
 
group :assets do
  gem 'sass-rails',   '~> 3.2.6'
  gem 'coffee-rails', '~> 3.2.2'
  gem 'uglifier',     '>= 1.0.3'
end
{% endhighlight %}

Uma dica interessante aqui seria não rodar simplesmente um bundle update atualizando todas as gems, isso pode te trazer problemas que não estão relacionados com a atualização do rails, tornando a migração mais demorada e confusa. Aqui vamos atualizar gem por gem conforme for necessário, assim, temos maior controle do código e dos problemas que forem surgindo.

Atualizado, agora deveriamos executar os testes mas logo ao tentar subir o servidor de testes temos o primeiro erro:


~~~~~~
bundle exec spork
...
uninitialized constant Haml::Util::Sass (NameError)
/home/vagrant/.rvm/gems/ruby-1.9.3-p550@rails31/gems/haml-3.1.8/lib/haml/util.rb:348:in `try_sass'
…
~~~~~~~~

Esse foi simples, atualizamos o haml para a versão 4 e o haml-rails para a versão 0.4, mas antes, precisamos atualizar a gem maillcatcher, que na versão atual não suporta o haml maior que o 3.1.8.

Feito isso, entre muitos warnings, o spork roda, mas quando executamos os testes, mais uma falha:

~~~ ruby
Exception encountered:#<TypeError: superclass mismatch for class Mysql2Adapter>
~~~

Após algumas pesquisas descubro que o erro está relacionado com a gem database_cleaner. No <a href="https://github.com/DatabaseCleaner/database_cleaner/blob/8d7f72ffc99befd5ce67d333a99e0cf054c24f81/History.rdoc#110-2013-08-01" target="_blank">changelog</a> da gem diz que esse bug foi corrigo na versão 1.1.0, então basta atualizar e finalmente conseguimos rodar os testes.

![alt text; "Tests"](/images/upgrading-rails/tests.png)

O sistema tem uma cobertura de testes muito boa e ver todos os testes passando já nos da uma segurança muito boa até aqui. Mas ainda temos que fazer deploy em um ambiente de homologação para ver como está o frontend e fazer alguns testes manuais nas principais funcionalidades antes de por em produção.
Mas ao tentar fazer deploy no Heroku, obtenho o erro no qual passei a maior parte do tempo.

~~~~~~
-----> Writing config/database.yml to read from DATABASE_URL
-----> Preparing app for Rails asset pipeline
       Running: rake assets:precompile
       rake aborted!
       undefined method `[]' for nil:NilClass
       (See full trace by running task with --trace)
       (in /tmp/build_5cedb08201c76b6ff36094b3d746fe22)
 !
 !     Precompiling assets failed.
 !
 !     Push rejected, failed to compile Ruby app
~~~~~~~

O pior aqui é que não temos informações sobre o erro, poderia ser a atualização do sass, mas a versão do rails que estamos não suporta um downgrade para a versão que estava antes. Aparentemente ele está falhando ao tentar executar o precompile, então rodei local com a opção –trace, e obtive o seguinte erro:

~~~ ruby
…
AssetSync: using /home/vagrant/site/config/initializers/asset_sync.rb
** Invoke tmp:cache:clear (first_time)
** Execute tmp:cache:clear
** Execute assets:precompile:nondigest
rake aborted!
Fog provider não pode ficar em branco, Fog directory não pode ficar em branco
…
~~~

<a href="https://github.com/fog/fog" target="_blank">Fog</a> é uma gem que auxilia na utilização de serviços em nuvem e é uma dependencia da gem <a href="https://github.com/rumblelabs/asset_sync" target="_blank">asset_sync</a>, gem utilizada para sincronizar nossos assets com o storage na amazon. Então parece que  não está reconhecendo a configuração da gem que é feita por meio de variáveis de ambiente por motivos de segurança. Para testar, faço essa configuração hardcoded e agora meu erro passa a ser um forbidden.

~~~ ruby
…
** Execute assets:sync
rake aborted!
Expected(200) <=> Actual(403 Forbidden)
...
 <Error><Code>SignatureDoesNotMatch</Code>
 <Message>The request signature we calculated does not match the signature you provided. Check your key and signing method.</Message>
…
~~~

A própria <a href="https://github.com/rumblelabs/asset_sync/blob/master/docs/heroku.md" target="_blank">documentação</a> da gem alerta sobre problemas conhecidos no Heroku, além disso, encontro em <a href="https://github.com/rumblelabs/asset_sync/issues/41" target="_blank">outros lugares</a> pessoas com problemas parecidos, para não ficar muito parado em um só problema, decido subir o diretório public/assets, assim, não entra na fase de compilação e completa o deploy até que eu possa entender melhor o problema.

Ao tentar rodar as migrations, o erro supreendentemente se repete, mas dessa vez fica bem mais claro.

~~~~~~
$ heroku run rake db:migrate --trace -a app-qa
Running `rake db:migrate --trace` attached to terminal... up, run.7806
# mil warnings ...
rake aborted!
undefined method `[]' for nil:NilClass
/app/config/initializers/connection_fix.rb:12:in `<top (required)>'
…
~~~~~~

Esse arquivo foi criado alguns anos atrás para solucionar o erro <a href="http://coderrr.wordpress.com/2009/01/08/activerecord-threading-issues-and-resolutions/" target="_blank">"Server has gone away"</a> com o mysql, e resolveu muito bem até essa migração.

**Mas o que mudou?** \\
Na linha 12 temos o seguinte

~~~ ruby
adapter = ActiveRecord::Base.configurations[Rails.env]['adapter']
~~~

Esse código pega a configuração do banco de dados baseado no database.yml, porém, o Heroku estabelece a conexão utilizando o DATABASE_URL, sendo assim,  ActiveRecord::Base.configurations retorna um array vazio.
 
![alt text; "AR Config - Heroku"](/images/upgrading-rails/activerecord.png)

* <a href="https://github.com/rails/rails/issues/6951" target="_blank">Rails issues</a>

Olhando o <a href="https://github.com/rails/rails/commit/a7c3c9025009cef98cd91ff27707f678cb8fb3c3#diff-31e798bb03e4140196a88e242f488880" target="_blank">diff</a> no rails, vemos que a correção foi implementada a partir da versão 4.0

Por hora, poderiamos corrigir esse problema da seguinte maneira:

~~~ ruby
Rails.application.config.after_initialize do 
  config  = ActiveRecord::Base.configurations[Rails.env] || 
              Rails.application.config.database_configuration[Rails.env]
  adapter = config['adapter']
~~~

Mas acabamos optando por retirar esse arquivo por se tratar de um problema muito antigo.

**Voltando ao deploy...**

Ao fazer deploy obtive a seguinte mensagem de erro, que impedia o sync de funcionar quebrando o style do sistema.

~~~
Warning. Error encountered while saving cache /path/to/rails/root/tmp/cache/sass/be5cee8cb77bd6548431cadb96c7cc283fde192f/application.css.scss.erbc: can't dump anonymous class #<Class:0x007fc5e352d6f8>
~~~

Esse é um bug conhecido nas issues do Sass, nesse caso basta fixar a versão da gem na 3.2.15.

Deploy feito, todos os testes passando, agora é hora de deixar a aplicação sem nenhum warning :) 

**Warnings**

1 - <span class="error-color">DEPRECATION WARNING: ActiveSupport::Memoizable is deprecated and will be removed in future releases,simply use Ruby memoization pattern instead. (called from <top (required)> at /home/vagrant/site/config/environment.rb:5)</span>

Encontrei nessa <a href="https://github.com/rails/rails/issues/4678" target="_blank">discussão</a> a solução, atualizar a gem carrierwave.

2 - <span class="error-color">DEPRECATION WARNING: The InstanceMethods module inside ActiveSupport::Concern will be no longer included automatically. Please define instance methods directly in Myapp::Controller::Concerns::CommonMethods instead. (called from include at /home/vagrant/site/app/controllers/application_controller.rb:2)</span>

Esse alerta diz que agora os InstanceMethods são incluídos automaticamente, isso porque em nossa aplicação temos:

~~~ ruby
module Myapp::Controller::Concerns::CommonMethods
  extend ActiveSupport::Concern
  module InstanceMethods
    def foo 
	...
    end
  end
end
~~~

Agora deve ficar assim:

~~~ ruby
module Myapp::Controller::Concerns::CommonMethods
  extend ActiveSupport::Concern
  def foo
    ...
  end
end
~~~

3 - <span class="error-color">DEPRECATION WARNING: Calling set_table_name is deprecated. Please use self.table_name = 'the_name' instead. (called from <class:CampaignUser> at /home/vagrant/site/app/models/campaign_user.rb:4)</span>

Substituir em nossa aplicação o set_table_name por self.table_name = "tablename"

4 -<span class="error-color"> WARNING: Sinatra 1.2.x has reached its EOL. Please upgrade.</span> \\
Atualizar o Sinatra. 

5 - <span class="error-color">:public is no longer used to avoid overloading Module#public, use :public_folder instead
	from /home/vagrant/.rvm/gems/ruby-1.9.3-p550@rails31/gems/resque-1.19.0/lib/resque/server.rb:12:in `<class:Server>'</span>

Como podemos ver nesse <a href="https://github.com/resque/resque/pull/420" target="_blank">Pull Request</a> do resque, esse problema deriva da atualização do sinatra, e como vemos no <a href="https://github.com/resque/resque/commit/96bb1b2797132db4bf8464c8a1e85650c738092d" target="_blank">changelog</a>,  a correção já foi implementada na versão seguinte, 1.20.0.

6 - <span class="error-color">DEPRECATION WARNING: You have Rails 2.3-style plugins in vendor/plugins! Support for these plugins will be removed in Rails 4.0. Move them out and bundle them in your Gemfile, or fold them in to your app as lib/myplugin/* and config/initializers/myplugin.rb. See the release notes for more on this: http://weblog.rubyonrails.org/2012/1/4/rails-3-2-0-rc2-has-been-released. (called from <top (required)> at /home/vagrant/site/Rakefile:7)</span> \\
Lá no início, quando lemos a documentação já sabiamos desse problema, então basta mover seus plugins para lib.

7 - <span class="error-color">[deprecated] I18n.enforce_available_locales will default to true in the future. If you really want to skip validation of your locale you can set I18n.enforce_available_locales = false to avoid this message.</span>

Para não ver mais esse warning configure o enforce_available_locales para true ou false, no config/application.

8 -<span class="error-color">/home/vagrant/.rvm/gems/ruby-1.9.3-p550@rails31/gems/rake-0.8.7/lib/rake/alt_system.rb:32:
Use RbConfig instead of obsolete and deprecated Config.</span> \\
Atualizar o Rake.

Sem mais nenhum erro ou warnings, vamos rodar os testes novamente, e com tudo passando essa tarefa está concluída.

![alt text; "Tests"](/images/upgrading-rails/tests2.png)


\\
Até a próxima :D