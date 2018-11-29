##################################
## VM: kafka1 (vagrant ssh kafka1)
##################################
```
export SRVPASS=serversecret
mkdir ssl
cd ssl

keytool -genkey -keystore kafka.server.keystore.jks -validity 365 -storepass $SRVPASS -keypass $SRVPASS  -dname "CN=kafka1.infobarbosa.github.com" -storetype pkcs12

keytool -list -v -keystore kafka.server.keystore.jks
```

#######################################################################
## VM: kafka1 (vagrant ssh kafka1)
## criacao de um certification request file que serah assinado pela ca
#######################################################################
```
keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass $SRVPASS -keypass $SRVPASS
#> ll
```

############################################################################
## VM: kafka1 (vagrant ssh kafka1)
## Copiar o arquivo para o diretorio /vagrant onde pode ser acessado pela CA.
## O diretorio /vagrant eh, na realizada, o diretorio raiz do projeto no
## host compartilhado com a vm
############################################################################

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
## Setup da aplicacao cliente para acesso ao Kafka via porta 9093
############################################################################

# Client configuration for using SSL

## grab CA certificate from remote server and add it to local CLIENT truststore

```
export CLIPASS=clientpass
cd ~
mkdir ssl
cd ssl
cp /vagrant/ca-cert .
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca-cert  -storepass $CLIPASS -keypass $CLIPASS -noprompt

keytool -list -v -keystore kafka.client.truststore.jks
```

## Abrir a classe ProducerTutorial.java e ajustar a classe de propriedades com as seguintes informacoes
```
security.protocol=SSL
ssl.truststore.location=/home/vagrant/ssl/kafka.client.truststore.jks
ssl.truststore.password=clientpass

## 1. abrir no vi
vi src/main/java/com/github/infobarbosa/kafka/ProducerTutorial.java
## 2. acrescentar as linhas abaixo
properties.put("security.protocol", "SSL");
properties.put("ssl.truststore.location", "/home/vagrant/ssl/kafka.client.truststore.jks");
properties.put("ssl.truststore.password", "clientpass");

## 3. salvar e fechar
## 4. empacotar o projeto
mvn clean package

## 5. exportar o listener do kafka na porta 9093
export BOOTSTRAP_SERVERS_CONFIG=kafka1.infobarbosa.github.com:9093

```

## TEST
test using the console-consumer/-producer and the [client.properties](./client.properties)
### Producer
```
~/kafka/bin/kafka-console-producer.sh --broker-list <<your-public-DNS>>:9093 --topic kafka-security-topic --producer.config ~/ssl/client.properties

~/kafka/bin/kafka-console-producer.sh --broker-list <<your-public-DNS>>:9093 --topic kafka-security-topic


```
### Consumer
```
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server <<your-public-DNS>>:9093 --topic kafka-security-topic --consumer.config ~/ssl/client.properties
```
