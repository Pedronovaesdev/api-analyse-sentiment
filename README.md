# ⚡ API Análise de Sentimento (Squad-06)

Uma API intermediária construída com FastAPI para receber textos, encaminhar para um serviço de inferência BERT externo (outra API), processar e enriquecer as respostas (normalização, regras de negócio, logging e persistência opcional) e expor resultados de análise de sentimento prontos para consumo por clientes front-end ou pipelines.

Resumo rápido:
- Recebe dados textuais via endpoint HTTP.
- Empacota e publica mensagens para uma fila RabbitMQ (mensageria resiliente).
- Um worker/consumer consome da fila, chama a API BERT externa (inference), processa resultados e persiste/retorna os resultados via callback ou armazenamento.
- Agrega e transforma a resposta do modelo (score, rótulo, metadata).
- Opcionalmente persiste resultados, apresenta métricas e expõe endpoints de saúde/status.

---

## 🔎 Visão geral do fluxo de dados

1. Cliente envia payload (texto) para esta API.
2. API valida e prepara a requisição.
3. API publica uma mensagem na fila RabbitMQ (publisher) contendo o input + metadata.
4. Um ou mais workers consomem mensagens da fila, chamam a API BERT externa (BERT_INFERENCE), recebem a resposta e aplicam regras de normalização/thresholds.
5. Worker persiste resultado (DB) e/ou publica resultado em outra fila / endpoint de callback (webhook) para que o cliente ou outro serviço recupere o resultado.
6. API retorna confirmação imediata ao cliente (aceite/queued) e o resultado final é obtido após processamento assíncrono.

Observação: o uso de RabbitMQ melhora resiliência, desacoplamento e escalabilidade para alto volume de requisições.

---

## 📦 Funcionalidades principais

- Endpoint de análise de sentimento (single / batch) — aceita e enfileira.
- Arquitetura assíncrona com RabbitMQ (publisher + consumer).
- Lógica de retry, DLQ (dead-letter queue) e histórico de falhas.
- Encaminhamento seguro para API BERT (com timeouts, retries e headers) feito pelos workers consumers.
- Pydantic models para validação de payloads.
- Respostas padronizadas com confidências e rótulos.
- Health-check, metrics (basic) e monitoramento do estado da fila.
- Configuração via .env (facilita integração com CI/CD).
- Testes unitários com pytest (skeleton).

---

## 📘 Arquitetura de Mensageria — RabbitMQ (detalhes importantes)

- Papel: desacoplar recepção de requisições (API) do processamento de inferência (worker). A API atua como publisher; worker(s) consomem, chamam BERT externo e persistem/retornam resultados.
- Exchanges & Queues recomendadas:
  - exchange: sentiment.inference (type: direct/topic)
  - queue: sentiment.requests (durable)
  - queue de resposta/callback: sentiment.results (durable) ou usar webhook/DB
  - dead-letter-exchange: sentiment.dlx → dead-letter queue: sentiment.failed
- Mensagens:
  - Formato JSON, id único (uuid), timestamp, payload.text, metadata, reply_to/callback_url opcional.
  - Exemplo:
    {
      "id": "uuid-v4",
      "text": "Amei o serviço!",
      "metadata": {"source":"app","user_id":"u123"},
      "enqueue_at": "2025-11-30T12:00:00Z",
      "reply_to": "http://client/callback" // opcional
    }
- Garantias / melhores práticas:
  - Filas duráveis + mensagens persistent.
  - Acknowledgements explícitos (ack / nack).
  - Prefetch (QoS) configurado no consumer para controlar concorrência (ex.: prefetch_count=10).
  - Dead-lettering para mensagens mal-formatadas ou falhas repetidas.
  - Retry exponencial no worker (retries controlados) antes de enviar para DLQ.
  - Idempotência no processamento (track processed ids) para evitar reprocessamento.
  - Monitoração (RabbitMQ Management UI) e métricas exportadas (Prometheus).
- Escalabilidade:
  - Escalar consumers horizontalmente para aumentar throughput.
  - Separar filas por prioridade ou payload size.
  - Use conexão / channel pooling para performance.

---

## 🛠 Tech stack

- Python 3.10+
- FastAPI
- Uvicorn
- httpx (cliente HTTP assíncrono)
- Pydantic
- aio-pika / pika / kombu (client RabbitMQ, assíncrono recomendado)
- pytest (testes)
- Optional: Redis/Cache, PostgreSQL/SQLite

---

## ⚙️ Variáveis de ambiente (exemplo .env)

Crie `.env` a partir de `.env.example` e configure:

- BERT_API_URL=https://bert-inference.example.com/predict
- BERT_API_KEY=seu_token (se necessário)
- BERT_TIMEOUT=10
- RABBITMQ_URL=amqp://user:password@localhost:5672/
- RABBITMQ_EXCHANGE=sentiment.inference
- RABBITMQ_REQUEST_QUEUE=sentiment.requests
- RABBITMQ_RESULT_QUEUE=sentiment.results
- RABBITMQ_DLX=sentiment.dlx
- RABBITMQ_PREFETCH=10
- RETRY_MAX_ATTEMPTS=5
- APP_HOST=0.0.0.0
- APP_PORT=8000
- LOG_LEVEL=info
- DATABASE_URL=sqlite:///./data/app.db (opcional)
- CACHE_URL=redis://localhost:6379/0 (opcional)

---

## 🚀 Como rodar local (Windows) — com RabbitMQ

1. Clone
   ```
   git clone <repo-url>
   cd api-analyse-sentiment
   ```

2. Ambiente virtual e dependências
   ```
   python -m venv venv
   .\venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. Iniciar RabbitMQ (via Docker Compose recomendado)
   Exemplo docker-compose mínimo:
   ```yaml
   version: "3.8"
   services:
     rabbitmq:
       image: rabbitmq:3-management
       ports:
         - "5672:5672"
         - "15672:15672"
       environment:
         RABBITMQ_DEFAULT_USER: user
         RABBITMQ_DEFAULT_PASS: password
   ```
   Acesse UI: http://localhost:15672 (user/password)

4. Preparar .env (copiar .env.example → .env) e ajustar RABBITMQ_URL / BERT_API_URL.

5. Executar API (publisher)
   ```
   uvicorn main:app --reload --host 0.0.0.0 --port 8000
   ```

6. Executar worker (consumer)
   - Script worker separado: ex.: worker.py que consome da fila e chama BERT_API.
   - Rodar worker(s) em background / container separado.

---

## 🧭 Endpoints principais

1) /health
- GET — Retorna status da API e integração básica com RabbitMQ (connection ok).

2) /api/v1/analyze
- POST — Enfileira texto para processamento assíncrono.
Exemplo request:
```json
{
  "id": "abc123",
  "text": "Eu adorei o produto! Recomendaria para amigos.",
  "metadata": { "source": "web" },
  "callback_url": "http://cliente/callback" // opcional
}
```

Resposta imediata exemplo:
```json
{
  "id": "abc123",
  "status": "queued",
  "queued_at": "2025-11-30T12:34:56Z"
}
```

3) /api/v1/analyze/batch
- POST — Recebe array de textos e enfileira em lote.

4) /api/v1/results/{id}
- GET — (opcional) Recupera resultado final (se persistido no DB).

5) /metrics
- GET — métricas simples / contadores / health da fila.

---

## 💡 Boas práticas implementadas

- Publisher confirma publicação; consumers usam ack/nack.
- Retry exponencial e DLQ para falhas persistentes.
- Prefetch/limitação para controlar memória/throughput.
- Logging estruturado e trace_id para correlacionar mensagens.
- Testes com mocking de RabbitMQ (aio-pika / fixtures).
- Separar worker em um processo/container independente.

---

## 🧪 Testes

Rodar testes:
```
pytest -q
```

Sugestão: adicionar testes para mocks de chamadas à API BERT (httpx.MockTransport) e para comportamento de fila (ex.: RabbitMQ stub, aio-pika Testing).

---

## 📦 Deploy / Produção

- Containerize com Docker (adicionar Dockerfile) e um serviço worker separado.
- Orquestração: Kubernetes / Docker Compose com configuração das filas, readiness/liveness probes e limitação de recursos.
- Use filas duráveis e réplica de brokers (HA) se necessário.
- Segredos para credenciais (Key Vault / Secrets Manager).
- Monitoramento: Prometheus node-exporter + RabbitMQ exporter + logs centralizados.

---

## 🔧 Observações importantes sobre BERT externo e RabbitMQ

- Esta API não executa BERT localmente — consumidores chamam a API BERT.
- Contrato: BERT externo deve retornar label/probabilities/embedding conforme schema esperado.
- Gateway de mensageria garante entrega confiável; configure DLQ e idempotência.
- Ajuste thresholds e mapeamento de labels (POS/NEG/NEU) na camada de processamento do worker.

---

## 🤝 Como contribuir

1. Faça fork do repositório.
2. Crie branch: feature/minha-melhoria
3. Adicione testes.
4. PR com descrição clara.

---

## 📜 Licença

Escolha a licença do projeto (ex.: MIT). Atualize o arquivo LICENSE no repositório.

---

Se quiser, eu gero:
- template de Dockerfile + docker-compose com worker e broker;
- worker.py exemplo (consumer) com retry, DLQ e chamada ao BERT;
- testes unitários exemplares para endpoints, mocks do BERT e integração com RabbitMQ.