# Kafka SSL Encryption Lab

Esse laboratorio tem por objetivo exercitar a feature de encriptação do Kafka.
Nele vamos primeiramente testar as nossas aplicações clientes e constatar quanto inseguro é o tráfego de dados via texto puro.

## Fazendo o primeiro teste em plaintext

#### Janela 1
```
vagrant ssh kafka-client
cd ~/kafka-producer
java -cp target/kafka-producer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextProducer
[Control + c]
```

#### Janela 2
```
vagrant ssh kafka-client
cd ~/kafka-consumer
java -cp target/kafka-consumer-1.0-SNAPSHOT-jar-with-dependencies.jar com.github.infobarbosa.kafka.PlaintextConsumer
```

#### Janela 3

Executar um "tcpdump" para "escutar" o conteúdo trafegado.<br/>

Esse comando pode ser executado tanto na máquina do Kafka (kafka1) como na aplicação cliente (kafka-client)
```
vagrant ssh kafka-client
sudo tcpdump -v -XX  -i enp0s8 -c 10
sudo tcpdump -v -XX  -i enp0s8 -c 10 -w dump.txt
```
Agora que vimos o quanto expostos estão nossos dados, é hora de fazer o setup pra resolver isso com encriptação. <br/>

Abra outra janela e entre na máquina do Kafka:
```
vagrant ssh kafka1
```

### Primeiro uma limpeza...
Assim como eu, você pode querer executar esse lab muitas e muitas vezes então é legal manter o diretório raiz limpo.
```
rm /vagrant/cert-file
rm /vagrant/cert-signed
rm /vagrant/dump.txt
```

### Gerando o certificado e a keystore
```
export SRVPASS=serversecret
mkdir ssl
cd ssl

keytool -genkey -keystore kafka.server.keystore.jks -validity 365 -storepass $SRVPASS -keypass $SRVPASS  -dname "CN=kafka1.infobarbosa.github.com" -storetype pkcs12

ls -latrh
```
Vamos checar o conteúdo da keystore
```
keytool -list -v -keystore kafka.server.keystore.jks -storepass $SRVPASS
```

## Certification request

É hora de criar o certification request file. Esse arquivo vamos enviar para a **autoridade certificadora (CA)** pra que seja assinado.
```
keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass $SRVPASS -keypass $SRVPASS
```

Pra simular o envio do arquivo para a CA, vamos copiar o arquivo **cert-file** para o diretorio **/vagrant** onde pode ser acessado pela CA.
Observação: O diretório **/vagrant** é, na verdade, o diretório raiz do projeto no host. Apenas foi compartilhado com a vm.
```
cp ~/ssl/cert-file /vagrant/
```

## Assinatura do certificado

Abra outro terminal e entre na máquina da autoridade certificadora:
```
vagrant ssh ca
```
Verifique se o arquivo está acessível:
```
ls -ltrh /vagrant/cert-file
```
Tudo pronto! É hora de efetivamente assinar o certificado:

```
export SRVPASS=serversecret

sudo openssl x509 -req -CA ~/ssl/ca-cert -CAkey ~/ssl/ca-key -in /vagrant/cert-file -out /vagrant/cert-signed -days 365 -CAcreateserial -passin pass:$SRVPASS
```
Veja que o comando **openssl** recebe cert-file (**-in /vagrant/cert-file**) e devolve cert-signed (**/vagrant/cert-signed**) no mesmo diretório /vagrant.

## A chave pública da Autoridade Certificadora

Para este setup também vamos precisar da chave pública divulgada pela CA, então aproveita que está na máquina da CA e copie este certificado para o diretório /vagrant de forma a ficar acessível para as máquinas do Kafka (kafka1) e aplicação cliente (kafka-client).
```
cp ~/ssl/ca-cert /vagrant/
```

## VM: kafka1 (vagrant ssh kafka1)

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
