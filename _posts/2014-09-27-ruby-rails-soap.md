---
layout: post
title:  "Trabalhando com SOAP em aplicações Ruby on Rails"
date:   2014-09-27 22:00
categories: [rails,soap,ruby]
permalink: /rails-soap
description: "Como fazer uma consulta utilizando SOAP em aplicações Ruby on Rails de forma simples e rápida"
author: kelvin
---

SOAP (Simple Object Access Protocol) é um protocolo baseado em XML para troca de informações em aplicações distribuídas, rodando diferentes sistemas operacionais, com diferentes tecnologias e linguagens.

> Why SOAP? \\
> A better way to communicate between applications is over HTTP,\\
> because HTTP is supported by all Internet browsers and servers. \\
> SOAP was created to accomplish this.

<a href="http://www.w3schools.com/webservices/ws_soap_intro.asp" target="_blank">http://www.w3schools.com/webservices/ws_soap_intro.asp</a>


A estrutura do SOAP consiste em três partes

![alt text; "SOAP Structure Version"](/images/soap-rails/soap-structure.png)

**Envelope**: Basicamente define o inicio e o fim da mensagem \\
**Header**: É opcional, contém informações como dados para autenticação \\
**Body**: Contém as informações da mensagem. Dentro do Body ainda existe um elemento opcional que é o **Fault**, utilizado para reportar erros.

Para facilitar nosso entendimento de como funciona essa comunicação com o webservice, vamos criar um cenário bem simples onde consultamos o status de processamento de um produto identificado pelo lote_id. Vamos também utilizar a gem <a href="http://savonrb.com" target="_blank">Savon</a>.

Para criar a conexão utilizei os seguintes parâmetros:

**WSDL**: Web Service Description Language, fornece todas as informações do web service, como acessá-lo e os métodos disponíveis;\\
**endpoint**: link para o servidor;\\
**wsse_auth**: autenticação;\\
**proxy**: Por motivos de segurança, o servidor só aceita requisições vindas de um determinado IP e porta;\\
**namespaces**: São os namespaces da mensagem, quando um namespace diferente é usado a mensagem é discartada.

~~~ ruby
class Product < ActiveRecord::Base

  WSDL_PATH_TO_CONSULT = "#{Rails.root}/public/ConsultarLoteAcumulov1.wsdl"
  PROXY_URL = "http://myserver.com.br:3068"

  def check_status!
    client = Savon.client(
      wsdl: WSDL_PATH_TO_CONSULT,
      endpoint: self.check_status_endpoint,
      wsse_auth: [ ENV["USER"], ENV["PASS"] ],
      proxy: PROXY_URL,
      log: true,
      namespaces: {"xmlns:ebs"=>"http://ebs.SERVER.com.br/v1",
                  "xmlns:ebo"=>"http://ebo.SERVER.com.br/v1"}
      )

    # para saber quais metodos estao disponiveis
    # pry(main)> client.operations
    # => [:consultar_lote_acumulo]
    begin
      response = client.call(:consultar_lote_acumulo, message: check_status_hash)
      # ...
    rescue Savon::SOAPFault => error
      #...
    end
  end

  # É comum existir um endpoint para homologação e um para produção
  def check_status_endpoint
    if Rails.env.production?
      "https://prod.b2b.SERVER.com.br:8443/API/ConsultarLoteAcumuloAPISv1"
    else
      "https://qa.b2b.SERVER.com.br:8443/API/ConsultarLoteAcumuloAPISv1"
    end
  end

  # hash da mensagem que será passada no Body
  def check_status_hash
    {
      "ebs:lote-acumulo" => {
        "ebo:lote" => { "ebo:id-lote" => self.id },
        "ebo:parceiro" => {
          "ebo:dados-cadastrais-pj" => { "ebo:cnpj" => "111111111111111" }
        }
      },
      "ebs:indicador-detalhar-itens" => "false",
      "ebs:propriedades-execucao" => { "ebo:id-interface" => "1" }
    }
  end
end
~~~

No Controller teremos algo do tipo

~~~ ruby
product = Product.find(params[:id])
response = product.check_status!
~~~

Para checar a resposta usamos o método _body_ que retorna o SOAP Body como um hash:
![alt text; "SOAP Response"](/images/soap-rails/soap-response-body.png)

Um software legal para usar enquanto estamos testando é o <a href="http://www.soapui.org" target="_blank">SoapUI</a>.
Basta clicar em Novo Projeto, inserir o arquivo WSDL e ele monta a estrutura do resquest:
![alt text; "SOAPUI Resquest"](/images/soap-rails/SOAPUI-request.png)

Exemplo do response:
![alt text; "SOAPUI Response"](/images/soap-rails/SOAPUI-response.png)


Até a próxima :D