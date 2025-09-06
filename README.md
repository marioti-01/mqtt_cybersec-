Passo 1: Aplicar as Correções de Segurança
Vamos modificar os arquivos de configuração para remover o acesso anônimo, fortalecer as senhas e ajustar as permissões.

1.1. Crie Usuários e Senhas Fortes
Primeiro, vamos criar um usuário para o sensor e definir senhas fortes para ambos os usuários (sensor e liss).

Abra seu terminal na pasta do projeto e execute os seguintes comandos. Substitua SENHA_FORTE_AQUI por senhas seguras.

Bash

# Cria/substitui o arquivo de senhas com o novo usuário "sensor"
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" eclipse-mosquitto:2.0.20 mosquitto_passwd -c /mosquitto/config/mosquitto.passwd sensor

# Adiciona o usuário "liss" ao mesmo arquivo (sem a flag -c)
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" eclipse-mosquitto:2.0.20 mosquitto_passwd -b /mosquitto/config/mosquitto.passwd liss SENHA_FORTE_AQUI_2
O primeiro comando pedirá para você digitar a senha para o usuário sensor.

1.2. Atualize o Arquivo de Configuração (mosquitto.conf)
Vamos remover o acesso anônimo do listener interno (porta 1883).

Arquivo: mosquitto/config/mosquitto.conf

Ini, TOML

# mosquitto.conf
per_listener_settings true

# Listener interno (sensor) - porta 1883
# Acesso anônimo foi REMOVIDO para aumentar a segurança.
listener 1883
allow_anonymous false
password_file /mosquitto/config/mosquitto.passwd
acl_file /mosquitto/config/mosquitto.acl

# Listener externo (subscriber) com TLS
listener 8883
cafile /mosquitto/certs/ca.crt
certfile /mosquitto/certs/server.crt
keyfile /mosquitto/certs/server.key
allow_anonymous false
password_file /mosquitto/config/mosquitto.passwd
acl_file /mosquitto/config/mosquitto.acl

# Persistência e logs
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
1.3. Atualize as Regras de Acesso (mosquitto.acl)
Agora, vamos adicionar a permissão correta para o novo usuário sensor.

Arquivo: mosquitto/config/mosquitto.acl

# O usuário 'sensor' só pode escrever no tópico de temperatura.
user sensor
topic write sensor/temperature

# O usuário 'liss' só pode ler de todos os tópicos dentro de 'sensor/'.
user liss
topic read sensor/#
1.4. Use um Arquivo .env para Gerenciar Segredos
Para evitar senhas fixas no docker-compose.yml, vamos usar um arquivo de ambiente.

Crie um arquivo chamado .env na raiz do seu projeto.

Adicione as senhas a este arquivo:

Arquivo: .env

SENSOR_PASSWORD=SENHA_FORTE_AQUI_1
SUBSCRIBER_PASSWORD=SENHA_FORTE_AQUI_2
Crie um arquivo .gitignore para garantir que o arquivo .env não seja enviado ao GitHub.

Arquivo: .gitignore

.env
1.5. Atualize o docker-compose.yml
Finalmente, modifique o docker-compose.yml para usar as variáveis do arquivo .env.

Arquivo: docker-compose.yml

YAML

services:
  mqtt-broker:
    image: eclipse-mosquitto:2.0.20
    container_name: mqtt-broker
    ports:
      - "1883:1883"
      - "8883:8883"
    volumes:
      - ./mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./mosquitto/config/mosquitto.passwd:/mosquitto/config/mosquitto.passwd
      - ./mosquitto/config/mosquitto.acl:/mosquitto/config/mosquitto.acl
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
      - ./mosquitto/certs:/mosquitto/certs
    restart: unless-stopped
    networks:
      - mqtt_network

  temperature-sensor-1:
    build:
      context: .
      dockerfile: Dockerfile.temperature-sensor-1
    container_name: temperature-sensor-1
    depends_on:
      - mqtt-broker
    environment:
      BROKER_HOST: "mqtt-broker"
      BROKER_PORT: "1883"
      BROKER_TLS: "false"
      BROKER_USERNAME: "sensor" # Usuário definido
      BROKER_PASSWORD: "${SENSOR_PASSWORD}" # Senha via .env
      TOPIC: "sensor/temperature"
      INTERVAL_SECONDS: "5"
    restart: unless-stopped
    networks:
      - mqtt_network

  mqtt-subscriber:
    image: eclipse-mosquitto:2.0.20
    container_name: mqtt-subscriber
    depends_on:
      - mqtt-broker
    environment:
      BROKER_PORT: "8883"
      BROKER_TLS: "true"
      BROKER_USERNAME: "liss"
      BROKER_PASSWORD: "${SUBSCRIBER_PASSWORD}" # Senha via .env
    command: >
      sh -c "mosquitto_sub 
      -h mqtt-broker 
      -p $$BROKER_PORT 
      -t 'sensor/#' 
      -u $$BROKER_USERNAME 
      -P $$BROKER_PASSWORD 
      --cafile /mosquitto/certs/ca.crt 
      -v"
    volumes:
      - ./mosquitto/certs:/mosquitto/certs
    restart: on-failure
    networks:
      - mqtt_network

networks:
  mqtt_network:
    driver: bridge
Passo 2: Atualizar a Documentação (README.md)
Substitua o conteúdo do seu README.md pelo texto abaixo. Ele explica as melhorias de segurança e como executar o projeto corrigido.

Markdown

# Projeto MQTT Seguro: Sensor Python com Broker Mosquitto

Este projeto demonstra um ambiente MQTT seguro, com um **broker Mosquitto**, um **sensor de temperatura em Python** e um **subscriber de teste**. O objetivo é garantir uma arquitetura de segurança robusta, aplicando autenticação obrigatória, criptografia TLS para acesso externo e controle de acesso granular por usuário (ACLs).

---

## Arquitetura de Segurança Implementada

A segurança deste projeto foi aprimorada para seguir o princípio de "confiança zero" (*zero-trust*). As seguintes medidas foram implementadas:

1.  **Autenticação Obrigatória:** O acesso anônimo foi **desabilitado** em todas as portas. Todos os clientes (sensores internos e subscribers externos) devem se autenticar com um usuário e senha válidos.
2.  **Criptografia TLS:** A porta externa (`8883`) opera exclusivamente com TLS, garantindo que a comunicação fora da rede interna seja sempre criptografada.
3.  **Controle de Acesso Granular (ACLs):** As permissões são estritamente controladas:
    * O usuário `sensor` tem permissão **apenas para escrever** (`write`) no tópico `sensor/temperature`.
    * O usuário `liss` (subscriber) tem permissão **apenas para ler** (`read`) dos tópicos em `sensor/#`.
4.  **Gerenciamento de Segredos:** As senhas foram removidas do arquivo `docker-compose.yml` e são gerenciadas através de um arquivo de ambiente (`.env`), que não é versionado no Git.

---

## Como Executar o Projeto

### Pré-requisitos

* Docker
* Docker Compose

### 1. Configuração Inicial

Antes de iniciar, você precisa criar os usuários e o arquivo de ambiente.

#### a. Crie os Usuários e Senhas

Execute os comandos abaixo para criar o arquivo `mosquitto.passwd` com os usuários `sensor` e `liss`. Você precisará definir as senhas durante o processo.

```bash
# Cria o arquivo de senhas com o usuário "sensor" (será solicitada a senha)
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" eclipse-mosquitto:2.0.20 mosquitto_passwd -c /mosquitto/config/mosquitto.passwd sensor

# Adiciona o usuário "liss" ao mesmo arquivo (defina uma senha forte no comando)
docker run -it --rm -v "$(pwd)/mosquitto/config:/mosquitto/config" eclipse-mosquitto:2.0.20 mosquitto_passwd -b /mosquitto/config/mosquitto.passwd liss <SUA_SENHA_FORTE_PARA_LISS>
```

#### b. Crie o Arquivo de Ambiente

1.  Crie um arquivo chamado `.env` na raiz do projeto.
2.  Adicione as senhas que você acabou de definir:

    ```
    SENSOR_PASSWORD=<SENHA_QUE_VOCE_DEFINIU_PARA_O_SENSOR>
    SUBSCRIBER_PASSWORD=<SENHA_QUE_VOCE_DEFINIU_PARA_LISS>
    ```

### 2. Inicie os Serviços

Com a configuração pronta, inicie todos os contêineres:

```bash
docker compose up -d --build
```

### 3. Verifique os Logs

Você pode verificar os logs do sensor e do subscriber para confirmar que estão se conectando e funcionando corretamente:

```bash
# Ver logs do sensor
docker compose logs -f temperature-sensor-1

# Ver logs do subscriber (que está recebendo as mensagens)
docker compose logs -f mqtt-subscriber
```

---

## Testes Manuais

Você pode usar o `mosquitto_pub` e `mosquitto_sub` para interagir com o broker. Lembre-se que agora a autenticação é **obrigatória em todas as portas**.

### Publicar como o Sensor (Porta 1883, sem TLS)

```bash
mosquitto_pub -h localhost -p 1883 -t 'sensor/temperature' -m '35.5' -u 'sensor' -P '<SENHA_DO_SENSOR>' -d
```

### Subscrever como Cliente Externo (Porta 8883, com TLS)

```bash
mosquitto_sub -h localhost -p 8883 --cafile ./mosquitto/certs/ca.crt -t 'sensor/#' -u 'liss' -P '<SENHA_DA_LISS>' -v -d
```
Passo 3: Subir o Projeto Corrigido no GitHub
Agora que os arquivos estão corrigidos e a documentação atualizada, siga estes passos para enviar ao GitHub.

Verifique o Status dos Arquivos
Abra o terminal na raiz do projeto.

Bash

git status
Você verá os arquivos que foram modificados: mosquitto.conf, mosquitto.acl, docker-compose.yml, README.md, e os novos arquivos .env (que deve ser ignorado) e .gitignore.

Adicione os Arquivos ao "Stage" do Git

Bash

git add mosquitto/config/mosquitto.conf mosquitto/config/mosquitto.acl docker-compose.yml README.md .gitignore
Note que não adicionamos o arquivo .env, pois ele está no .gitignore.

Crie um Commit
Um commit é um "snapshot" das suas alterações. Use uma mensagem clara que descreva o que foi feito.

Bash

git commit -m "feat(security): Implementa autenticação e ACLs em todos os listeners"
Mensagens de commit como esta são uma excelente prática profissional.

Envie as Alterações para o GitHub
Supondo que seu repositório remoto se chama origin e a branch é main.

Bash

git push origin main
