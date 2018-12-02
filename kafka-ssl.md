########################################
## VM: kafka-client (vagrant ssh kafka1)
########################################

## fazendo o primeiro teste em plaintext
### na janela 1
```
cd ~/kafka-Producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextProducer
```

### na janela 2
```
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextConsumer
```

##################################
## VM: kafka1 (vagrant ssh kafka1)
##################################
```
export SRVPASS=serversecret
mkdir ssl
cd ssl
```
## criacao do certificado e keystore
```
keytool -genkey -keystore kafka.server.keystore.jks -validity 365 -storepass $SRVPASS -keypass $SRVPASS  -dname "CN=kafka1.infobarbosa.github.com" -storetype pkcs12

keytool -list -v -keystore kafka.server.keystore.jks
```

## criacao de um certification request file que serah assinado pela ca
```
keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass $SRVPASS -keypass $SRVPASS
```

## Copiar o arquivo para o diretorio /vagrant onde pode ser acessado pela CA.
### O diretorio /vagrant eh, na realizada, o diretorio raiz do projeto no
### host compartilhado com a vm
```
cp cert-file /vagrant/
```

##################################
## VM: ca (vagran ssh ca)
## Assinatura do certificado.
## Input: /vagrant/cert-file
## Output: /vagrant/cert-signed
##################################
```
export SRVPASS=serversecret

sudo openssl x509 -req -CA ~/ssl/ca-cert -CAkey ~/ssl/ca-key -in /vagrant/cert-file -out /vagrant/cert-signed -days 365 -CAcreateserial -passin pass:$SRVPASS
```

## copia a chave publica para /Vagrant
```
cp ~/ssl/ca-cert /vagrant/

```
############################################################################
## VM: kafka1 (vagrant ssh kafka1)
## Checa o certificado assinado e importa para a truststore e keystore.
############################################################################
```
keytool -printcert -v -file /vagrant/cert-signed
keytool -list -v -keystore /home/vagrant/ssl/kafka.server.keystore.jks
```

# Trust the CA by creating a truststore and importing the ca-cert
```
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file /vagrant/ca-cert -storepass $SRVPASS -keypass $SRVPASS -noprompt

```
# Import CA and the signed server certificate into the keystore
```
keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file /vagrant/ca-cert -storepass $SRVPASS -keypass $SRVPASS -noprompt
keytool -keystore kafka.server.keystore.jks -import -file /vagrant/cert-signed -storepass $SRVPASS -keypass $SRVPASS -noprompt
```

# Ajustar as seguintes configuracoes do broker (alterar arquivo /etc/server.properties)
```
listeners=PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093
advertised.listeners=PLAINTEXT://kafka1.infobarbosa.github.com:9092,SSL://kafka1.infobarbosa.github.com:9093

ssl.keystore.location=/home/vagrant/ssl/kafka.server.keystore.jks
ssl.keystore.password=serversecret
ssl.key.password=serversecret
ssl.truststore.location=/home/vagrant/ssl/kafka.server.truststore.jks
ssl.truststore.password=serversecret

```
# Reiniciar o Kafka
```
sudo systemctl restart kafka
sudo systemctl status kafka  
```
# Verify Broker startup
```
sudo grep "EndPoint" /var/log/kafka/server.log
```

# Verify SSL config from your local computer
```
openssl s_client -connect 192.168.56.12:9093
```

############################################################################
## VM: kafka-client (vagrant ssh kafka-client)
############################################################################

# Client configuration for using SSL

## grab CA certificate from remote server and add it to local CLIENT truststore

```
export CLIPASS=clientpass
cd ~
mkdir ssl
cd ssl
cp /vagrant/ca-cert .
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file /vagrant/ca-cert  -storepass $CLIPASS -keypass $CLIPASS -noprompt

keytool -list -v -keystore kafka.client.truststore.jks
```

## Abrir as classes SslProducer.java e SslConsumer.java e observar o uso das propriedades abaixo:
```
BOOTSTRAP_SERVERS_CONFIG=kafka1.infobarbosa.github.com:9093
security.protocol=SSL
ssl.truststore.location=/home/vagrant/ssl/kafka.client.truststore.jks
ssl.truststore.password=clientpass
```

## TEST
## fazendo um segundo teste, já apontando para o kafka1 na porta 9093
### na janela 1
```
cd ~/kafka-Producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslProducer
```

### na janela 2
```
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslConsumer
```
