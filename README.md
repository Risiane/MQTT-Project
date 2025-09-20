# Projeto MQTT (Broker + Sensor + Subscriber)

## Descrição
Implementação de um broker MQTT com autenticação, autorização e criptografia TLS, corrigindo falhas comuns de segurança.

## Serviços

- mqtt-broker (Mosquitto 2.0.20)
- temperature-sensor-1 (Python + Paho MQTT)
- mqtt-subscriber (mosquitto_sub)

## Vulnerabilidades e Soluções

## 1 - Acesso Anônimo
- Problema: Broker aceitava conexões sem login.
- Solução: Desativado `allow_anonymous false` e criados usuários com senha.


## 2 - Falta de Autorização

- Problema: Qualquer usuário podia acessar qualquer tópico.
- Solução: Implementada ACL com permissões específicas por usuário.

## 3 - Comunicação Não Criptografada

- Problema: Tráfego MQTT em texto plano.
- Solução: Habilitado TLS/SSL (porta 8883) com certificados.

## 4 - Clientes Sem Autenticação

- Problema: Sensor e subscriber conectavam sem credenciais.
- Solução: Código atualizado para usar usuário e senha.

## Implementação e Teste

## 1. Criar usuários
`mkdir -p mosquitto/config/auth`

`docker run --rm eclipse-mosquitto:2.0.20 sh -c "mosquitto_passwd -c -b /tmp/passwd mqttsensoruser sensor@mqtt && mosquitto_passwd -b /tmp/passwd mqttsubscriberuser subscriber@mqtt && mosquitto_passwd -b /tmp/passwd mqttadminuser admin@mqtt && cat /tmp/passwd"`

## 2. Criar ACL

echo user mqttsensoruser  
topic write sensor/+  
user mqttsubscriberuser  
topic read sensor/+  
user mqttadminuser  
topic readwrite #' > mosquitto/config/auth/acl

## 3. Gerar certificados
Criar pasta onde ficará os certificados:  
`mkdir -p mosquitto/config/certs`

Comandos para gerar os certificados:  
`openssl req -new -x509 -days 365 -nodes -out mosquitto/config/certs/ca.crt -keyout mosquitto/config/certs/ca.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=MQTT-CA"`

`openssl genrsa -out mosquitto/config/certs/server.key 2048`

`openssl req -new -out mosquitto/config/certs/server.csr -key mosquitto/config/certs/server.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=172.16.39.52"`

`openssl x509 -req -in mosquitto/config/certs/server.csr -CA mosquitto/config/certs/ca.crt -CAkey mosquitto/config/certs/ca.key -CAcreateserial -out mosquitto/config/certs/server.crt -days 365`

## 4. Configuração mosquitto.conf
echo 'listener 1883  
listener 8883  
cafile /mosquitto/config/certs/ca.crt  
certfile /mosquitto/config/certs/server.crt  
keyfile /mosquitto/config/certs/server.key  
allow_anonymous false  
password_file /mosquitto/config/auth/passwd  
acl_file /mosquitto/config/auth/acl' > mosquitto/config/mosquitto.conf

## 5. Subir e testar
Iniciar o sistema:  
`docker compose up -d --build`

`docker compose logs -f mqtt-subscriber`

Teste não criptografado:  
`docker run --rm eclipse-mosquitto:2.0.20 mosquitto_pub -h 172.16.39.60 -p 1883 -t sensor/test -m "teste não criptografado" -u mqttadminuser -P admin@mqtt`

Teste criptografado:
`docker run --rm -v $(pwd)/mosquitto/config/certs:/tmp/certs eclipse-mosquitto:2.0.20 mosquitto_pub -h 172.16.39.60 -p 8883 -t sensor/test -m "teste criptografado" -u mqttadminuser -P admin@mqtt --cafile /tmp/certs/ca.crt`
