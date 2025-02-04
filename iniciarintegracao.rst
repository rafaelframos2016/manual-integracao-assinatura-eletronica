﻿API de assinatura digital GovBR
================================

Este documento visa detalhar a estrutura da API REST para assinatura digital usando certificados avançados gov.br.

A API adota o uso do protocolo OAuth 2.0 para autorização de acesso e protocolo HTTP para acesso aos endpoints. Assim sendo, o uso da API envolve duas etapas:

1. Geração do token de acesso OAuth (Access Token)

2. Acesso ao serviço de assinatura

Geração do Access Token
+++++++++++++++++++++++

Para geração do Access Token é necessário redirecionar o navegador do usuário para o endereço de autorização do servidor OAuth, a fim de obter seu consentimento para o uso de seu certificado para assinatura. Nesse processo, a aplicação deve usar credenciais previamente autorizadas no servidor OAuth. As seguintes credencias podem ser usadas para testes:

.. code-block:: console

		Servidor OAuth = https://sistemas.homologacao.ufsc.br/govbr/oauth2.0
		Client ID= devLocal
		Secret = younIrtyij3
		URI de redirecionamento = http://127.0.0.1:*/**

As credenciais para Client ID “devLocal” estão configuradas no servidor OAuth para aceitar qualquer aplicação executando localmente (host 127.0.0.1, qualquer porta, qualquer caminho). Aplicações remotas não poderão usar essas credenciais de teste.

A URL usada para redirecionar o usuário para o formulário de autorização, conforme a especificação do OAuth 2.0, é a seguinte:

.. code-block:: console

		https://<Servidor OAuth>/authorize?response_type=code&redirect_uri=<URI de redirecionamento>&scope=sign&client_id=<clientId>

Nesse endereço, o servidor OAuth autentica o usuário e pede a autorização expressa do mesmo para acessar seu certificado para assinatura. Neste instante será pedido um código de autorização a ser enviado por SMS. **IMPORTANTE: EM HOMOLOGAÇÃO**, NÃO SERÁ ENVIADO SMS, DEVE-SE USAR O CÓDIGO **12345**.

Após a autorização ser dada pelo usuário, o servidor OAuth redireciona o mesmo para o endereço <URI de redirecionamento> especificado, e passa, como um parâmetro de query, o atributo Code. O <URI de redirecionamento> deve ser um endpoint da aplicação correspondente ao padrão autorizado no servidor OAuth, e capaz de receber e tratar o parâmetro “code”. Este atributo deve ser usado na fase seguinte do protocolo OAuth, pela aplicação, para pedir um Access Token ao servidor OAuth, com a seguinte requisição HTTP com método POST:

.. code-block:: console

		https://<Servidor OAuth>/token?code=<code>&client_id=<clientId>&grant_type=authorization_code&client_secret=<secret>&redirect_uri=<URI de redirecionamento>

O <URI de redirecionamento> deve ser exatamente o mesmo valor passado na requisição “authorize” anterior. O servidor OAuth retornará um objeto JSON contendo o Access Token, que deve ser usado nas requisições subsequentes aos endpoints do serviço.

**Importante**: O servidor OAuth está delegando a autenticação ao ambiente de **Staging** do gov.br

Obtenção do certificado do usuário
++++++++++++++++++++++++++++++++++

Para obtenção do certificado do usuário deve-se fazer uma requisição HTTP Get para o seguinte end-point:

.. code-block:: console

		https:// govbr-uws.homologacao.ufsc.br/CloudCertService/certificadoPublico 

Deve-se enviar o cabeçalho Authorization  com o tipo de autorização Bearer e o Access Token obtido anteriormente. Exemplo de requisição:

.. code-block:: console

		GET /CloudCertService/certificadoPublico HTTP/1.1
		Host: govbr-uws.homologacao.ufsc.br 
		Authorization: Bearer <Access token>

Será retornado o certificado digital em formato PEM na resposta.

Realização da assinatura digital Raw de um HASH SHA-256
+++++++++++++++++++++++++++++++++++++++++++++++++++++++

Para assinar digitalmente um HASH SHA-256 usando a chave privada do usuário, deve-se fazer uma requisição HTTP POST para o seguinte end-point:

.. code-block:: console

		https:// govbr-uws.homologacao.ufsc.br/CloudCertService/assinarRaw

Deve-se enviar o cabeçalho Authorization com o tipo de autorização Bearer e o Access Token obtido anteriormente. Exemplo de requisição:

.. code-block:: console

		POST /CloudCertService/assinarRaw HTTP/1.1
		Host: govbr-uws.homologacao.ufsc.br
		Content-Type: application/json	
		Authorization: Bearer <Access token>
		Content-Type: application/json

		{"hashBase64":"<Hash SHA256 codificado em Base64>"}


Será retornada a assinatura digital SHA256-RSA codificada em Base64 na resposta.

Realização da assinatura digital de um HASH SHA-256 em PKCS#7
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Para gerar um pacote PKCS#7 contendo a assinatura digital de um HASH SHA-256 usando a chave privada do usuário, deve-se fazer uma requisição HTTP POST para o seguinte end-point:

.. code-block:: console

		https:// govbr-uws.homologacao.ufsc.br/CloudCertService/assinarPKCS7

Deve-se enviar o cabeçalho Authorization com o tipo de autorização Bearer e o Access Token obtido anteriormente. Exemplo de requisição:

.. code-block:: console

		POST /CloudCertService/assinarPKCS7 HTTP/1.1
		Host: govbr-uws.homologacao.ufsc.br
		Content-Type: application/json	
		Authorization: Bearer <Access token>
		Content-Type: application/json

		{"hashBase64":"<Hash SHA256 codificado em Base64>"}

Será retornado um arquivo contendo o pacote PKCS#7 com a assinatura digital do hash SHA256-RSA e com o certificado público do usuário. O arquivo retornado pode ser validado em https://govbr-verifier.homologacao.ufsc.br.

Exemplo de aplicação
++++++++++++++++++++

Logo abaixo, encontra-se um pequeno exemplo PHP para prova de conceito.

`Download Exemplo PHP <https://github.com/servicosgovbr/manual-integracao-assinatura-eletronica/raw/main/downloadFiles/exemploApiPhp.zip>`_

Este exemplo é composto por 3 arquivos:

1. index.php -  Formulário para upload de um arquivo
2. upload.php - Script para recepção de arquivo e cálculo de seu hash SHA256. O Resultado do SHA256 é armazenado na sessão do usuário.
3. assinar.php - Implementação do handshake OAuth, assim como a utilização dos dois endpoints acima. Como resultado, uma página conforme a figura abaixo será apresentada, mostrando o certificado emitido para o usuário autenticado e a assinatura.


.. image:: images/image.png


Para executar o exemplo, é possível utilizar Docker com o comando abaixo:

.. code-block:: console
	
		docker-compose up -d

e acessar o endereço http://127.0.0.1:8080

.. |site externo| image:: _images/site-ext.gif
.. _`codificador para Base64`: https://www.base64decode.org/
.. _`Plano de Integração`: arquivos/Modelo_PlanodeIntegracao_LOGINUNICO_final.doc
.. _`OpenID Connect`: https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse
.. _`auth 2.0 Redirection Endpoint`: https://tools.ietf.org/html/rfc6749#section-3.1.2
.. _`Exemplos de Integração`: exemplointegracao.html
.. _`Design System do Governo Federal`: http://dsgov.estaleiro.serpro.gov.br/ds/componentes/button
.. _`Resultado Esperado do Acesso ao Serviço de Confiabilidade Cadastral (Selos)`: iniciarintegracao.html#resultado-esperado-do-acesso-ao-servico-de-confiabilidade-cadastral-selos
.. _`Resultado Esperado do Acesso ao Serviço de Confiabilidade Cadastral (Categorias)` : iniciarintegracao.html#resultado-esperado-do-acesso-ao-servico-de-confiabilidade-cadastral-categorias
.. _`Documento verificar Código de Compensação dos Bancos` : arquivos/TabelaBacen.pdf
