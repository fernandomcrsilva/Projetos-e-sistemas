# Detector de Gás Natural (Metano) – Arduino Mega + ESP8266

> Projeto acadêmico IBM3119 • Protótipo open-source para detecção de gás natural (CH₄) usando **Arduino Mega 2560 + ESP8266 + sensor MQ-5**

**Status atual**

* Sensor MQ-5 **não disponível no Ibmec** – protótipo foi montado com material próprio (uso em casa) e documentação aberta para replicação no laboratório.
* Arquitetura final:

  * **Arduino Mega 2560** lê o sensor **MQ-5** (analógico + digital) e calcula **Rs/R₀** e “ppm equivalente”.
  * **ESP8266 (NodeMCU)** lê **temperatura/umidade (DHT11)**, recebe os dados do Mega via **UART** e publica tudo via **MQTT** (broker Mosquitto + dashboard web).
* Planejada **impressão 3D** de:

  * **compartimento para pilhas**,
  * gabinete ventilado para o conjunto MQ-5 + eletrônica.

---

## 1) Requisitos do Projeto

### Requisitos funcionais

* Detectar presença/aumento de **metano (CH₄)** e GLP em ambiente interno.
* Calcular **Rs/R₀** e estimar um **“ppm equivalente”** usando a curva log-log do MQ-5.
* Medir **temperatura** e **umidade relativa** (DHT11) para considerar influência termo-úmida no sensor.
* Disparar **alerta lógico** quando:

  * temperatura ≥ **60 °C** (sobreaquecimento),
  * **ppm** ultrapassar limiar de segurança definido em software.
* Enviar leituras periódicas via **Wi-Fi (MQTT)** para:

  * broker **Mosquitto**,
  * dashboard web em **HTML + JavaScript** (MQTT over WebSocket).
* Registrar no broker / dashboard:

  * data/hora das leituras,
  * data/hora da **calibração** (R₀ em ar limpo),
  * histórico de alarmes.

### Requisitos não funcionais

* Baixo custo e reprodutibilidade usando apenas componentes comuns (Mega, ESP8266, MQ-5, DHT11).
* Comunicação robusta entre Mega e ESP8266 usando **UART com divisor de tensão** (5 V → 3,3 V).
* Operação contínua com alimentação **5 V** (USB ou pack de pilhas com step-up).
* Caixa/gabinete com:

  * ventilação adequada para o sensor,
  * isolamento mecânico das partes energizadas.
* **Aviso de segurança:** protótipo acadêmico – **não substitui detector comercial certificado**.

---

## 2) Especificações (mapeamento dos requisitos)

### 2.1 Hardware (BOM)

* **Microcontroladores**

  * **Arduino Mega 2560**

    * Lê o **sensor MQ-5**.
    * Calcula `Rs`, `Rs/R₀` e `ppm`.
    * Envia os dados para o ESP8266 via **Serial3 (TX3)**.
  * **ESP8266 NodeMCU**

    * Lê o **DHT11** (temperatura/umidade).
    * Recebe os dados do Mega pela **Serial hardware RXD0 (GPIO3)**.
    * Publica em MQTT para o broker Mosquitto.

* **Sensores**

  * **MQ-5 – sensor de gás combustível**

    * AO → A0 do Mega (leitura analógica).
    * DO → pino 52 do Mega (detecção digital ajustável via trimpot).
  * **DHT11 – temperatura e umidade**

    * DATA → D2 do ESP8266.

* **Interface e comunicação**

  * Conexão **TX3 (pino 14 do Mega)** → **RXD0 (GPIO3 do ESP)**
    
  * GND comum entre Mega, ESP e MQ-5.

* **Alimentação**

  * Protótipo atual alimentado via **USB** (Mega + ESP).
  * Fase seguinte: compartimento 3D com **pack de pilhas** + módulo **step-up 5 V**.

* Protoboard, jumpers e, futuramente, PCB simples.

### 2.2 Software – Arduino Mega (firmware MQ-5)

* Ambiente: **Arduino IDE**.
* Principais pontos:

  * Leitura analógica do MQ-5 com **média de N amostras** para reduzir ruído.
  * Cálculo da tensão no sensor `Vrs` e da resistência `Rs` usando o divisor real do módulo:

    * `Rs = (Vrs * RL) / (Vc - Vrs)`.
  * Rotina de **calibração automática de R₀** em ar limpo:

    * sensor mede `Rs` por ~10 s,
    * assume‐se `Rs(ar)/R₀ ≈ 5` (datasheet) ⇒ `R₀ = Rs(ar)/5`.
  * Conversão de `Rs/R₀` para **ppm** via curva log–log do MQ-5 (ajuste linear em escala log).
  * **Filtro exponencial** em `ratio` e `ppm` para suavizar leituras.
  * Envio periódico via **Serial3** no formato de linha:

    ```text
    ADC:<valor>;VRS:<V>;RS:<ohms>;RATIO:<x>;PPM:<y>;D0:<0/1>
    ```

### 2.3 Software – ESP8266 (bridge + MQTT + DHT11)

* Bibliotecas:

  * `ESP8266WiFi` – conexão Wi-Fi.
  * `PubSubClient` – cliente MQTT.
  * `DHT` – leitura do DHT11.

* Lógica:

  * Conecta na rede Wi-Fi (`ssid` / `password`).
  * Conecta ao **broker Mosquitto** (`mqtt_server`, `mqtt_port`).
  * Usa `Serial.begin(9600)` para receber do Mega pela RXD0.
  * Faz o **parse** das linhas recebidas do Mega usando `sscanf`.
  * Lê **temperatura** e **umidade** do DHT11.
  * Monta payloads JSON e publica periodicamente (ex. a cada 5 s):

    * Tópico `gas/dht`:

      ```json
      {"temp":27.9,"hum":64.0}
      ```
    * Tópico `gas/mq`:

      ```json
      {
        "adc":49,
        "v_rs":0.239,
        "rs":1006,
        "ratio":3.988,
        "ppm":2.8,
        "d0":1,
        "temp":27.9,
        "hum":64.0
      }
      ```
  * Regras de **alarme lógico** usadas no dashboard:

    * temperatura ≥ 60 °C;
    * ppm acima do limiar configurado (ex.: > 50 ppm equivalente).

### 2.4 Dashboard Web (HTML + JavaScript)

* Página estática (HTML/CSS/JS) acessível via browser no PC ou celular na mesma rede.
* Usa **MQTT over WebSocket** para se conectar ao Mosquitto (porta 8080).
* Mostra em tempo real:

  * cards com **temperatura**, **umidade** e **concentração de gás**;
  * área de **Alarms** (sem alarmes / alerta de gás / alta temperatura);
  * log de mensagens formatado:

    ```text
    [22:32:01] temperature: 27.1 °C | humidity: 44.0 % | ppm: 0.8
    ```
* Alarmes visuais mudam de cor quando algum limiar é excedido.

---

## 3) Disponibilidade no Ibmec

* Sensores MQ-4/MQ-5/MQ-8: **não disponíveis** no estoque do Ibmec no momento.

  * [ ] Solicitar compra / empréstimo para replicar o protótipo no laboratório.
* Arduino Mega, ESP8266, DHT11, jumpers e protoboards:
  verificar disponibilidade local para montagem de um segundo kit.
* Enquanto o Ibmec não tiver sensor MQ, é possível:

  * usar o firmware atual com **MQ-5 pessoal**,
  * ou adaptar um **modo simulado** para demonstração em sala.

---

## 4) Alimentação e Impressão 3D

### 4.1 Compartimento de pilhas (para impressão 3D)

* **Objetivo:** permitir uso autônomo (sem USB) com pack de pilhas e módulo step-up 5 V.
* Requisitos do modelo:

  * alojar 4×AA ou 2×18650 + módulo **step-up**;
  * tampa com trava e furo para cabo de alimentação;
  * espaço para chave liga/desliga;
  * furos de fixação (parede ou suporte).
* Arquivos STL serão enviados para **[ibmeclabs@gmail.com](mailto:ibmeclabs@gmail.com)** para impressão
  (provavelmente 2–3 iterações até caber placa, módulo e fiação).

### 4.2 Gabinete principal (fase seguinte)

* Aberturas de ventilação próximas ao módulo MQ-5 (gás tende a subir).
* Furação frontal para:

  * LEDs de status (opcional, fase futura),
  * saída acústica do buzzer (futuro),
  * passagem de cabo de alimentação.
* Suporte interno para fixar Mega, ESP e MQ-5 com segurança.

---

## 5) Montagem (versão atual)

| Função                | Placa / Pino                             | Observações                              |
| --------------------- | ---------------------------------------- | ---------------------------------------- |
| MQ-5 AOUT (analógico) | **Mega – A0**                            | Leitura para cálculo de Rs/R₀/ppm        |
| MQ-5 DOUT (digital)   | **Mega – 52**                            | Nível alto/baixo conforme trimpot        |
| UART Mega → ESP       | **Mega TX3 (14)** → **ESP RXD0 (GPIO3)** |                 |
| DHT11 DATA            | **ESP – D2**                             | Alimentado com 3,3 V                     |
| GND comum             | Mega ↔ ESP ↔ MQ-5                        | Referência compartilhada                 |
| Alimentação           | USB 5 V                                  | Em estudos para pack de pilhas + step-up |

> Não usamos TX do ESP para o Mega (comunicação é unidirecional Mega → ESP).

---

## 6) Firmware – passos rápidos

1. **Mega 2560**

   * Carregar o sketch do MQ-5 (cálculo de Rs/R₀/ppm + envio Serial3).
   * Abrir Serial Monitor (9600 bps) para conferir calibração e leituras.
2. **ESP8266**

   * Desconectar fio TX3→RXD0, gravar o sketch do ESP.
   * Reconectar o fio, abrir Serial Monitor (9600 bps) e conferir:

     * conexão Wi-Fi,
     * conexão MQTT,
     * mensagens “Linha do Mega:” e “Publicado MQ:…”.
3. **Broker Mosquitto**

   * Rodar `mosquitto` (porta 1883) + listener WebSocket (porta 8080).
   * Testar com:
     `mosquitto_sub -h <IP> -p 1883 -t "gas/#" -v`.
4. **Dashboard Web**

   * Abrir o HTML no navegador,
   * apontar para o IP do broker,
   * observar gráficos/cards e alarmes em tempo real.

---

## 7) Revisão de Literatura (Overleaf – padrão IEEE/SBrT)

* Princípio de funcionamento dos sensores **MQ-x** (camada de SnO₂ aquecida).
* Curvas **log(Rs/R₀) × log(ppm)** do MQ-5 para diferentes gases (CH₄, LPG, H₂) e limites.
* Influência de **temperatura** e **umidade** (referência: artigo “A Simplified Model for Calculating the Dependence of MOS Gas Sensors on Temperature and Humidity”).
* Métodos de calibração:

  * determinação de R₀ em ar limpo,
  * ajustes empíricos em ambiente real.
* Limitações de precisão e tempo de aquecimento (burn-in).
* Boas práticas de segurança: diferença entre protótipo acadêmico e sensores certificados.


---


