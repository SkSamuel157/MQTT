# Projeto MQTT - Sensor Python com TLS e Autenticação

Este projeto demonstra um ambiente **MQTT seguro**, com:

* Um **broker Mosquitto** configurado com autenticação e TLS;
* Um **sensor de temperatura em Python** publicando dados periodicamente;
* Um **subscriber** capaz de receber mensagens com segurança.

O objetivo é garantir **autenticação, segregação de portas e criptografia**, mantendo o sensor funcional na rede interna.

---

## Estrutura do Projeto

| Diretório/Arquivo                   | Descrição                                               |
| ----------------------------------- | ------------------------------------------------------- |
| `docker-compose.yml`                | Orquestra os serviços do broker, sensor e subscriber    |
| `Dockerfile.temperature-sensor-1`   | Cria a imagem Docker do sensor Python                   |
| `mosquitto/config/mosquitto.conf`   | Configuração do broker                                  |
| `mosquitto/config/mosquitto.passwd` | Arquivo de senhas para autenticação                     |
| `mosquitto/config/mosquitto.acl`    | Regras de controle de acesso por tópico                 |
| `mosquitto/certs/`                  | Certificados TLS (`ca.crt`, `server.crt`, `server.key`) |
| `src/temperature-sensor-1.py`       | Código Python do sensor                                 |

---

## Diagrama de Comunicação

```
                       ┌─────────────────────────────┐
                       │        Host Windows         │
                       │-----------------------------│
                       │ VSCode (SSH na VM Debian)   │
                       │ MQTT Explorer (portable)    │
                       └─────────────┬───────────────┘
                                     │
           SSH/VSCode Terminal       │
           para gerenciar Docker     │
                                     ▼
                       ┌─────────────────────────────┐
                       │         VM Debian           │
                       │-----------------------------│
                       │ Docker Engine + Compose     │
                       └─────────────┬───────────────┘
                                     │
                  ┌──────────────────┴─────────────────┐
                  │                                    │
                  ▼                                    ▼
       ┌─────────────────────────┐           ┌─────────────────────────┐
       │     mqtt-broker         │           │ temperature-sensor-1    │
       │ Eclipse Mosquitto 2.0.20│           │ Python 3.12 + Paho MQTT │
       │-------------------------│           │-------------------------│
       │ listener 0.0.0.0:1883   │           │ Publica em sensor/temperature │
       │ listener 0.0.0.0:8883 TLS│          │ Intervalo: 5s │
       │ allow_anonymous false    │           │ Usuário não-root │
       │ password_file + ACLs     │           │                 │
       │ certificados TLS         │           │                 │
       └─────────────┬───────────┘           └───────────────┬─────────┘
                     │                                     │
                     │                                     │
                     ▼                                     ▼
             ┌───────────────┐                     ┌───────────────┐
             │ mqtt-subscriber│                     │ MQTT Explorer │
             │ TLS + Auth      │                     │ (portable)    │
             │ Inscreve sensor/#│                    │ Observa dados │
             └─────────────────┘                     └───────────────┘
```

**Legenda:**

* **Porta 1883:** Comunicação interna (TCP, sem TLS) para sensores.
* **Porta 8883:** Conexões externas seguras (TLS + autenticação).
* **Broker:** Responsável por autenticação, ACLs e criptografia.
* **Subscriber externo:** Recebe mensagens de forma segura.
* **MQTT Explorer:** Ferramenta de monitoramento opcional.

---

## Alterações e Correções

### Sensor Python

* Atualizado para **MQTTv3.1.1**, garantindo compatibilidade com Mosquitto 2.0.20.
* Publica leituras contínuas de temperatura no tópico `sensor/temperature`.
* Conexão interna via **porta 1883**, sem TLS para comunicação local.

### Broker MQTT

* **Porta 1883:** comunicação interna para sensores sem TLS.
* **Porta 8883:** conexão externa segura com **TLS e autenticação**.
* **Segurança:** `allow_anonymous false`, `password_file` e ACLs configurados.
* **Criptografia:** certificados TLS para integridade e confidencialidade das mensagens.

**Exemplo de ACL:**

```text
user <USUÁRIO_CRIADO>
topic readwrite sensor/#
```

---

## Como Executar o Projeto

### 1. Criar Autenticação

1. Criar diretórios necessários:

```bash
mkdir -p mosquitto/{config,data,log,certs}
```

2. Criar usuário e senha:

```bash
mosquitto_passwd -c mosquitto/config/mosquitto.passwd <seu_usuario>
```

3. Configurar broker (`mosquitto.conf`):

```ini
allow_anonymous false
password_file /mosquitto/config/mosquitto.passwd
listener 1883
listener 8883
certfile /mosquitto/certs/server.crt
keyfile /mosquitto/certs/server.key
```

---

### 2. Iniciar Serviços

```bash
docker compose up -d --build
```

---

### 3. Testes

**Publicar manualmente (sensor interno):**

```bash
mosquitto_pub -h localhost -p 1883 -t 'sensor/temperature' -m '27.5' -d
```

**Subscriber externo via TLS:**

```bash
mosquitto_sub -h localhost -p 8883 --cafile ./mosquitto/certs/ca.crt -t 'sensor/#' -v --tls-version tlsv1.2 -u <USUÁRIO_CRIADO> -P <SENHA> -d
```

---

## Resumo Técnico

* Sensor Python funcional via **TCP interno (1883)**.
* Broker configurado com **TLS, autenticação e ACLs** para conexões externas (8883).
* Segregação de portas garante **segurança de tráfego interno e externo**.
* Certificados TLS garantem **integridade, confidencialidade e autenticidade** das mensagens.
