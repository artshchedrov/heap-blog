## Кейс # Запуск Ollama в docker compose

Пример запуска Ollama в контейнерах с оркестрацией. Можно использовать если нужно несколько различных активных моделей с доступом по API. Можно применять для создания агентов. 

Перед использование необходимо установить NVIDIA Container Toolkit.

docker compose с одним контейнером:

```yaml
version: '2.2'

services:
  ollama_one:
    volumes:
      - /srv/models/ollama:/root/.ollama
    container_name: ollama_one
    pull_policy: missing
    tty: true
    restart: unless-stopped
    image: ollama/ollama:latest
    ports:
      - 7869:11434
    environment:
      - OLLAMA_KEEP_ALIVE=1
    networks:
      - ollama-docker
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
networks:
  ollama-docker:
    external: false
```
Некоторые параметры:

`/srv/models/ollama:/root/.ollama` - путь куда будут сохраняться модели и метаданные.

`pull_policy: missing` - пуллить образ только если его нет на локальной машине.

`OLLAMA_KEEP_ALIVE=1` - переменная окружения указывающая сколько держать в памяти модель, после завершения запроса. Значение -1 укажет держать модель в памяти вечно. 