## Projeto IoT de Monitoramento para Vinheria Agnello (FIAP)

Solução de Internet das Coisas para monitoramento em tempo real de temperatura, umidade e luminosidade da adega da Vinheria Agnello, utilizando a plataforma FIWARE, Node-RED e o padrão NGSIv2.

## Funcionalidades Principais

* **Coleta de Dados:** O hardware (ESP32) coleta dados de sensores:
    * Temperatura (DHT11)
    * Umidade (DHT11)
    * Luminosidade (LDR)
* **Comunicação em Tempo Real:** Os dados são enviados via Wi-Fi para um broker MQTT (Aedes, rodando em Node-RED) a cada 10 segundos.
* **Orquestração e Tradução:** O Node-RED atua como middleware, recebendo os dados brutos em JSON, processando-os e traduzindo-os para o padrão **NGSIv2**.
* **Plataforma de Contexto:** Os dados formatados são enviados para o **FIWARE Orion Context Broker**, criando uma entidade "WineCellar:001" que representa o estado atual da adega (em conformidade com Smart Data Models).
* **Armazenamento Histórico:** Através de uma subscrição no Orion, todos os dados recebidos são encaminhados para o **STH-Comet**, que os armazena no MongoDB para análise histórica.
* **Visualização:** Um dashboard em Node-RED exibe os dados dos sensores (temperatura, umidade, luz) em tempo real através de medidores (gauges).

## Tecnologias e Componentes Utilizados

### Hardware
* **MCU:** ESP32
* **Sensor:** DHT11 (Temperatura e Umidade)
* **Sensor:** LDR (Luminosidade)
* **Resistor:** 10k $\Omega$

### Software (Middleware e Visualização)
* **Node-RED:** Orquestrador principal do fluxo de dados.
    * **Broker MQTT:** `node-red-contrib-aedes`
    * **Conector FIWARE:** `node-red-contrib-letsfiware-ngsi`
    * **Dashboard:** `node-red-dashboard`
* **Arduino IDE:** Para programação do firmware do ESP32.

### Backend (Plataforma FIWARE)
* **Docker & Docker Compose:** Utilizado para orquestrar e executar todos os serviços de backend.
* **FIWARE Orion Context Broker:** Gerenciador de contexto que armazena o "estado atual" da adega (NGSIv2).
* **FIWARE STH-Comet:** Componente para armazenamento de dados históricos.
* **MongoDB:** Banco de dados NoSQL utilizado tanto pelo Orion quanto pelo STH-Comet.

## Como Executar o Projeto

Para rodar este projeto, você precisará de três componentes principais funcionando: o Backend FIWARE (no Docker), o Middleware (Node-RED) e o Hardware (ESP32).

### Pré-requisitos
* [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado e rodando.
* [Node.js](https://nodejs.org/en/) (que inclui o NPM) instalado.
* [Node-RED](https://nodered.org/docs/getting-started/local) instalado globalmente (`npm install -g --unsafe-perm node-red`).
* [Arduino IDE](https://www.arduino.cc/en/software) instalado com o suporte ao ESP32.

### 1. Backend (FIWARE + Docker)

1.  Clone este repositório:
    ```bash
    git clone [URL_DO_SEU_REPOSITORIO]
    cd [NOME_DO_REPOSITORIO]
    ```
2.  Inicie os contêineres do FIWARE (Orion, STH-Comet e MongoDB) usando o arquivo `docker-compose.yml` fornecido:
    ```bash
    docker-compose up -d
    ```
3.  **Verificação:**
    * **Orion:** Abra `http://localhost:1026/version` no seu navegador. Você deve ver uma resposta JSON.
    * **STH-Comet:** Abra `http://localhost:8666/version`.

### 2. Middleware (Node-RED)

1.  Inicie o Node-RED em um terminal:
    ```bash
    node-red
    ```
2.  Abra o editor do Node-RED no seu navegador: `http://127.0.0.1:1880`.
3.  Vá ao menu (canto superior direito) > **Manage palette** > **Install**. Instale os seguintes pacotes:
    * `node-red-contrib-aedes`
    * `node-red-contrib-letsfiware-ngsi`
    * `node-red-dashboard`
4.  Vá ao menu > **Import** e importe o arquivo `flow.json` deste repositório.
5.  Clique no botão vermelho **Deploy** (no canto superior direito) para iniciar o broker MQTT e o fluxo de dados.
6.  Acesse o dashboard para ver os medidores (ainda vazios): `http://127.0.0.1:1880/ui`.

### 3. Hardware (ESP32)

1.  Monte o circuito conectando os sensores DHT11 e LDR ao ESP32 conforme o código (Pino D4 para o DHT, Pino A0/VP para o LDR).
2.  Abra o arquivo `.ino` (disponível na pasta `esp32_code/`) na sua Arduino IDE.
3.  No código, **altere as seguintes variáveis** para refletir sua rede:
    ```cpp
    const char* SSID = "NOME_DA_SUA_REDE_WIFI";
    const char* PASSWORD = "SENHA_DA_SUA_REDE_WIFI";
    const char* MQTT_BROKER = "IP_DO_SEU_PC_COM_NODE-RED"; // Ex: "192.168.1.10"
    ```
4.  Instale as bibliotecas necessárias na Arduino IDE (`Tools > Manage Libraries...`):
    * `PubSubClient`
    * `DHT sensor library`
    * `Adafruit Unified Sensor`
5.  Conecte seu ESP32, selecione a placa e a porta corretas, e clique em **Upload**.

### 4. Habilitar o Armazenamento Histórico

Para que o Orion avise o STH-Comet sobre novas leituras, precisamos criar uma "Subscrição".

1.  Com tudo rodando, envie a seguinte requisição (usando Postman, Insomnia, ou cURL) para o Orion:

    **POST** `http://localhost:1026/v2/subscriptions`
    **Headers:** `Content-Type: application/json`

    **Body (Corpo):**
    ```json
    {
      "description": "Avisar o STH-Comet quando a Adega mudar",
      "subject": {
        "entities": [
          {
            "id": "urn:ngsi-ld:WineCellar:001",
            "type": "WineCellar"
          }
        ],
        "condition": {
          "attrs": [ "temperature", "humidity", "luminosity" ]
        }
      },
      "notification": {
        "http": {
          "url": "http://sth-comet:8666/notify"
        },
        "attrs": [ "temperature", "humidity", "luminosity" ],
        "metadata": ["timestamp"]
      },
      "throttling": 1
    }
    ```
    *(Ou use o arquivo `subscription.json` deste repositório)*

### 5. Verificando o Funcionamento

Se tudo estiver correto, o ESP32 enviará dados, o Node-RED irá processar e:
1.  O dashboard em `http://127.0.0.1:1880/ui` mostrará os medidores funcionando.
2.  A entidade no Orion será atualizada: `http://localhost:1026/v2/entities/urn:ngsi-ld:WineCellar:001?options=keyValues`.

##  Integrantes

* FELIPE RAMALHO          RM 561148
* GUILHERME DE MEDEIROS   RM 561699
* MIKAEL DE ALBUQUERQUE   RM 566507
* MURILO VIEIRA DA CRUZ   RM 563743
* OTAVIO MAGNO DA SILVA   RM 566149
* VICTOR PUCCI            RM 561736
