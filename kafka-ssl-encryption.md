###############################################
## VM: kafka-client (vagrant ssh kafka-client)
###############################################

## fazendo o primeiro teste em plaintext
### Janela 1
```
vagrant ssh kafka-client
cd ~/kafka-producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextProducer
[Control + c]
```

### Janela 2
```
vagrant ssh kafka-client
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextConsumer
```

### Janela 3. Opcional. tcpdump na porta do servico para "escutar" o conteudo trafegado.
### esse comando pode ser executado tanto na maquina do Kafka (kafka1) como na aplicacao cliente (kafka-client)
```
vagrant ssh kafka-client
sudo tcpdump -v -XX  -i enp0s8 -c 10
sudo tcpdump -v -XX  -i enp0s8 -c 10 -w dump.txt
```

##################################
## VM: kafka1 (vagrant ssh kafka1)
##################################
```
vagrant ssh kafka1
```

### primeiro uma limpeza...
```
rm /vagrant/cert-file
rm /vagrant/cert-signed
```

### agora a geracao do certificado e da keystore
```
export SRVPASS=serversecret
mkdir ssl
cd ssl

keytool -genkey -keystore kafka.server.keystore.jks -validity 365 -storepass $SRVPASS -keypass $SRVPASS  -dname "CN=kafka1.infobarbosa.github.com" -storetype pkcs12

ls -latrh

keytool -list -v -keystore kafka.server.keystore.jks -storepass $SRVPASS
```

## criacao de um certification request file que serah assinado pela ca
```
keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass $SRVPASS -keypass $SRVPASS
```

## Copiar o arquivo para o diretorio /vagrant onde pode ser acessado pela CA.
### O diretorio /vagrant eh, na realidade, o diretorio raiz do projeto no host. Apenas foi compartilhado com a vm.
### O que estamos fazendo aqui é emulando o envio da requisicao para o time de seguranca assinar o certificado.
```
cp ~/ssl/cert-file /vagrant/
```

##################################
## VM: ca (vagran ssh ca)
## Assinatura do certificado.
## Input: /vagrant/cert-file
## Output: /vagrant/cert-signed
##################################
```
vagrant ssh ca
export SRVPASS=serversecret

sudo openssl x509 -req -CA ~/ssl/ca-cert -CAkey ~/ssl/ca-key -in /vagrant/cert-file -out /vagrant/cert-signed -days 365 -CAcreateserial -passin pass:$SRVPASS
```

## copia a chave publica para /Vagrant
```
cp ~/ssl/ca-cert /vagrant/

```
############################################################################
## VM: kafka1 (vagrant ssh kafka1)
############################################################################

## Checa o certificado assinado e importa para a truststore e keystore.
```
keytool -printcert -v -file /vagrant/cert-signed
keytool -list -v -keystore /home/vagrant/ssl/kafka.server.keystore.jks -storepass $SRVPASS
```

# Criando a relacao de confianca com a CA.
## Cria a trustore e importa o certificado da CA (ca-cert) nele.
```
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file /vagrant/ca-cert -storepass $SRVPASS -keypass $SRVPASS -noprompt

```
# Import CA and the signed server certificate into the keystore
```
keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file /vagrant/ca-cert -storepass $SRVPASS -keypass $SRVPASS -noprompt
keytool -keystore kafka.server.keystore.jks -import -file /vagrant/cert-signed -storepass $SRVPASS -keypass $SRVPASS -noprompt
```

# Ajustar as seguintes configuracoes do broker (alterar arquivo /etc/kafka/server.properties)
```
vi /etc/kafka/server.properties

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

# Configuracao SSL na aplicacao cliente

## geracao da truststore importando a chave publica da autoridade certificadora (CA)

```
export CLIPASS=clientpass
cd ~
mkdir ssl
cd ssl
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file /vagrant/ca-cert  -storepass $CLIPASS -keypass $CLIPASS -noprompt

keytool -list -v -keystore kafka.client.truststore.jks -storepass $CLIPASS
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
### Janela 1
```
cd ~/kafka-producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslProducer
```

### Janela 2
```
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.SslConsumer
```

### Janela 3. Opcional. tcpdump na porta do servico para "escutar" o conteudo trafegado.
###     Esse comando pode ser executado tanto na maquina do Kafka (kafka1) como na aplicacao cliente (kafka-client)
###     Atencao! enp0s8 eh a minha interface de rede utilizada para host-only.
###     Se o comando nao funcionar entao teste quais interfaces estao funcionando via ifconfig ou tcpdump --list-interfaces
```
sudo tcpdump -v -XX  -i enp0s8
```
### caso queira enviar o log para um arquivo para analisar melhor
```
sudo -i
tcpdump -v -XX  -i enp0s8 -w dump.txt -c 10
```
