
1- Remoção do Index.html (Solução do item 1 de NFE_CONTINGENCY_PROD DRAFT REPORT)  
Removemos ele para ninguém ter acesso a tela inicial do tomcat, abrindo a possibilidade de acessar a lista de aplicações e outras configurações.

Para isso, basta comentar as seguintes linhas do arquivo conf/web.xml

``` xml
  <!-- <welcome-file-list>  
        <welcome-file>index.html</welcome-file>  
        <welcome-file>index.htm</welcome-file>  
        <welcome-file>index.jsp</welcome-file>  
    </welcome-file-list> -->
```
  

Também é possível aumentar ainda mais a segurança ocultando informações do servidor de aplicação que aparecem quando uma página está indisponível. Ao criar o diretório apache-tomcat-9.0.94\lib\org\apache\catalina\util podemos criar o arquivo ServerInfo.properties e adicionar os seguintes parâmetros a ele: 

```
server.info=  
server.number=  
server.built=
```

Assim o resultado final ao acessar diretamente localhost:8080/ (por exemplo) o resultado será esse:  
![](https://lh7-rt.googleusercontent.com/docsz/AD_4nXftYLvfvuCGhQP1FK2hCO4DSs3pjN2ZrN-mtU8BJI5CqNYODLl24DAnpEgBdVJkFWNV89r0XR6yLTf4YqEggYc2pPPiaSLYOM7tpUrk70V5VawxoFznHsOwYqly4qb9Q5ubGyDuw-ymu1V8CNK08Rf7M6U?key=7cqG7SJZjPiQI61exZA8Yg)

  

2- Desabilitando métodos TRACE e OPTIONS (Inseguros) (solução de “Cross-Site Scripting (XSS)” informado pelo Discord):  
desabilitando esses métodos HTTP que podem ser usados para ataque XSS. O código a seguir deve ser adicionado ao web.xml mas ele não desabilita o método em si, ele só exige uma autenticação, mas como o  `auth-constraint` não possui regra, ninguém terá permissão para usar os métodos.

``` xml
<security-constraint> 
<web-resource-collection> 
<web-resource-name>Disable HTTP Methods</web-resource-name> <url-pattern>/*</url-pattern> 
<http-method>TRACE</http-method> 
<http-method>OPTIONS</http-method> 
</web-resource-collection> 
<auth-constraint/> 
</security-constraint>
```
  

3- Adicionando cabeçalhos de resposta HTTP (Solução do item 4 de NFE_CONTINGENCY_PROD DRAFT REPORT):

Usados para mitigar os tipos mais comuns de ataques, como MIME e ClickJacking. basta adicionar ao arquivo web.xml

``` xml
  <filter>  
  <filter-name>SecurityHeadersFilter</filter-name>  
  <filter-class>org.apache.catalina.filters.HttpHeaderSecurityFilter</filter-class>  
  <init-param>  
    <param-name>antiClickJackingEnabled</param-name>  
    <param-value>true</param-value>  
  </init-param>  
  <init-param>  
    <param-name>antiClickJackingOption</param-name>  
    <param-value>SAMEORIGIN</param-value>  
  </init-param>  
  <init-param>  
    <param-name>blockContentTypeSniffingEnabled</param-name>  
    <param-value>true</param-value>  
  </init-param>  
</filter>  
<filter-mapping>  
  <filter-name>SecurityHeadersFilter</filter-name>  
  <url-pattern>/*</url-pattern>  
</filter-mapping>
```

4-(PERFORMANCE) Deve ser avaliado o uso do GZIP:

GZIP é um sistema de compressão de arquivos usado para aumentar a velocidade da resposta das requisições HTTP, beneficiando clientes com conexões mais lentas.

Alerta: A compressão economiza na largura de banda e também na velocidade da resposta a custo de aumentar o uso de CPU (ainda a ser calculado), recomendado não habilitar agora antes de ser averiguado se o Tomcat reduzirá o uso de CPU do servidor.

Para habilitar o GZIP é necessario adicionar o atributo compression=”on” no Conector presente no arquivo web.xml, como a seguir:  
  
``` xml
  <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" compression="on" />
```

5- Desabilitando o SSLv3 (Solução do item 5, 6 e 7 de NFE_CONTINGENCY_PROD DRAFT REPORT):

Desabilitando o protocolo pois ele possui uma falha que é utilizada para ataques Poodle, para isso devemos alterar o arquivo server. xml, adicionando no Connector as propriedades sslEnabledProtocols, passando os protocolos que podem ser usados e ciphers (opcional) que são conjuntos de criptografia utilizados para aumentar a segurança.

``` xml
  <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"  
              maxThreads="150" SSLEnabled="true"  
              maxParameterCount="1000"  
              sslEnabledProtocols="TLSv1.2,TLSv1.3"  
              ciphers="TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,  
            TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,  
            TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,  
            TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384"  
          >
```