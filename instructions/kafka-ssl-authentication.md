# Kafka SSL Encryption and Authentication Lab

Esta é a segunda parte de um laboratório que visa exercitar features de segurança do Kafka.
Na [primeira parte](https://github.com/infobarbosa/kafka-security-base-box/blob/master/instructions/kafka-ssl-encryption.md) trabalhamos a **encriptação** dos dados. Agora vamos exercitar a **encriptação** combinada com **autenticação** utilizando TSL (o antigo SSL), modelo também conhecido como 2-way Authentication.<br/>
No exercício a seguir estou assumindo que você executou a primeira parte. Caso contrário, comece por [aqui](https://github.com/infobarbosa/kafka-security-base-box).<br/>

## Let's go!

Se ainda não tiver subido o laboratório, essa é a hora:
```
vagrant up
```

## Aplicação Cliente

Conecte-se na máquina da aplicação cliente:
```
vagrant ssh kafka-client
```

Crie de um certificado para a aplicação cliente:
```
export CLIPASS=clientpass

keytool -genkey -keystore ~/ssl/kafka.client.keystore.jks -validity 365 -storepass $CLIPASS -keypass $CLIPASS  -dname "CN=kafka-client.infobarbosa.github.com" -alias kafka-client -storetype pkcs12

keytool -list -v -keystore ~/ssl/kafka.client.keystore.jks -storepass $CLIPASS
```

## Criação do request Vagrantfile

Crie o request file que será assinado pela CA
```
keytool -keystore ~/ssl/kafka.client.keystore.jks -certreq -file /vagrant/client-cert-sign-request -alias kafka-client -storepass $CLIPASS -keypass $CLIPASS

```
## Enviando para a CA

Vamos simular o envio da requisição para a CA. Assim como no último lab, basta copiar para o diretório /vagrant, diretório raiz no host, compartilhado pelas vms deste laboratório:
```
cp ~/ssl/client-cert-sign-request /vagrant/
```

## Assinatura do certificado

Abra outra janela e entre na máquina da autoridade certificadora:
```
vagrant ssh ca
```

Assine o certificado utilizando a CA:

```
export SRVPASS=serversecret
openssl x509 -req -CA ~/ssl/ca-cert -CAkey ~/ssl/ca-key -in /vagrant/client-cert-sign-request -out /vagrant/client-cert-signed -days 365 -CAcreateserial -passin pass:$SRVPASS
```

## Kafka Cliente, setup da keystore

Se fechou a sessão com a aplicação cliente, vai precisar disto:
```
export CLIPASS=clientpass
```

Importe o certificado assinado para a keystore:
```
keytool -keystore ~/ssl/kafka.client.keystore.jks -import -file /vagrant/client-cert-signed -alias kafka-client -storepass $CLIPASS -keypass $CLIPASS -noprompt
```

Crie a relação de confiaça importanto a chave pública da CA para a Keystore:
```
keytool -keystore ~/kafka.client.keystore.jks -alias CARoot -import -file /vagrant/ca-cert -storepass $CLIPASS -keypass $CLIPASS -noprompt
```

Verifique se está tudo lá:
```
keytool -list -v -keystore kafka.client.keystore.jks -storepass $CLIPASS
```

Dê uma olhada no códido da aplicação cliente. Perceba a presença das linhas abaixo:
```
properties.put("ssl.keystore.location", "/home/vagrant/ssl/kafka.client.keystore.jks");
properties.put("ssl.keystore.password", "clientpass");
properties.put("ssl.key.password", "clientpass");
```

## Kafka Broker

É necessário fazer um pequeno ajuste no broker, habilitar a propriedade **ssl.client.auth**. <br/>
Entre na máquina do Kafka:
```
vagrant ssh kafka1
```
Abra o arquivo server.properties no seu editor de texto:
```
vi /etc/kafka/server.properties
```
Inclua a nova propriedade **ssl.client.auth**:
```
ssl.client.auth=required
```
Reinicie o Kafka:
```
sudo systemctl restart kafka
```
Verifique se o serviço subiu:
```
sudo systemctl status kafka
```

## Hora do teste!

#### Janela 1
```
cd ~/kafka-producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslAuthenticationProducer
```

#### Janela 2
```
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslAuthenticationConsumer
```
