# Projeto n8n com Docker Compose, MongoDB, Redis e ELK Stack

Este projeto configura um ambiente completo para executar o n8n, uma poderosa ferramenta de automação de fluxos de trabalho, juntamente com serviços de banco de dados, cache e um stack de logs centralizado.

## 📝 Tabela de Conteúdo

- [Sobre](#sobre)
- [Começando](#começando)
  - [Pré-requisitos](#pré-requisitos)
  - [Instalando](#instalando)
- [Acessando a Interface Gráfica do MongoDB (Compass)](#acessando-a-interface-gráfica-do-mongodb-compass)
- [Acessando a Interface Gráfica do Redis (RedisInsight)](#acessando-a-interface-gráfica-do-redis-redisinsight)
- [Visualizando os Logs com Kibana](#visualizando-os-logs-com-kibana)
- [Uso](#uso)
- [Implantação](#implantação)
- [Construído Usando](#construído-usando)
- [Contribuindo](#contribuindo)
- [Autores](#autores)

## 🧐 Sobre

Este projeto configura um ambiente completo para executar o n8n, uma poderosa ferramenta de automação de fluxos de trabalho. Ele utiliza o Docker Compose para facilitar a configuração e o gerenciamento dos serviços necessários.

- **MongoDB**: Utilizado para armazenar os dados do n8n de forma persistente, garantindo que seus fluxos de trabalho e configurações sejam preservados entre reinicializações.
- **Redis**: Empregado para otimizar o desempenho do n8n, gerenciando filas de tarefas e fornecendo um mecanismo de cache eficiente.
- **Interfaces Gráficas**: Inclui interfaces gráficas para o MongoDB (via conexão externa com Compass) e para o Redis (RedisInsight, incluído no Docker Compose).
- **ELK Stack (Elasticsearch, Logstash, Kibana)**: Configurado para coletar, armazenar e visualizar centralizadamente os logs de todos os seus serviços. Isso é crucial para depuração, monitoramento e compreensão do comportamento do seu assistente.

Esta configuração permite que você aproveite ao máximo os recursos do n8n em um ambiente robusto, escalável e observável.

## 🏁 Começando

Estas instruções irão ajudá-lo a obter uma cópia do projeto em execução na sua máquina local para fins de desenvolvimento e teste.

### Pré-requisitos

Você precisa ter o Docker e o Docker Compose instalados na sua máquina.

```bash
# Exemplo para verificar a instalação do Docker
docker --version

# Exemplo para verificar a instalação do Docker Compose
docker-compose --version
```

### Instalando

Siga estes passos para configurar o ambiente de desenvolvimento:

1.  **Crie a estrutura de pastas e arquivos:**

    Na pasta raiz do seu projeto, crie a seguinte estrutura:

    ```
    .
    ├── docker-compose.yml
    └── logstash/
        └── config/
            └── logstash.conf
    ```

2.  **Preencha o arquivo `docker-compose.yml`:**

    Cole o conteúdo abaixo no seu arquivo `docker-compose.yml`.

    **⚠️ Aviso de Segurança:** Este arquivo contém senhas diretamente no código (`minhaSenhaForte123`, `outraSenhaForte456`). Para produção, é altamente recomendável movê-las para um arquivo `.env`.

    ```yaml
    version: '3.8'

    services:
      # --- N8N Core Services ---
      n8n:
        image: n8nio/n8n:latest
        container_name: n8n_core_app
        ports:
          - "5678:5678"
        volumes:
          - n8n_data:/home/node/.n8n
        environment:
          - N8N_HOST=${N8N_HOST:-localhost}
          - N8N_PORT=${N8N_PORT:-5678}
          - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
          - NODE_ENV=production
          - N8N_DB=mongodb
          - N8N_DB_MONGODB_URL=mongodb://admin:minhaSenhaForte123@n8n_mongo_db:27017/n8n?authSource=admin
          - REDIS_HOST=n8n_redis_cache
          - REDIS_PORT=6379
          - REDIS_PASSWORD=outraSenhaForte456
          - REDIS_ENABLE=true
          - N8N_LOG_LEVEL=info
          - N8N_LOG_OUTPUT=stdout
        restart: unless-stopped
        networks:
          - assistant_network
        logging:
          driver: gelf
          options:
            gelf-address: "udp://logstash_gelf:12201"
            tag: "n8n"

      n8n_mongo_db:
        image: mongo:latest
        container_name: n8n_mongo_db_data
        restart: unless-stopped
        ports:
          - "27017:27017" # Porta exposta para acesso da GUI
        environment:
          - MONGO_INITDB_ROOT_USERNAME=admin
          - MONGO_INITDB_ROOT_PASSWORD=minhaSenhaForte123
        volumes:
          - n8n_mongo_data:/data/db
        networks:
          - assistant_network
        logging:
          driver: gelf
          options:
            gelf-address: "udp://logstash_gelf:12201"
            tag: "n8n-mongo"

      n8n_redis_cache:
        image: redis:latest
        container_name: n8n_redis_cache_data
        restart: unless-stopped
        command: redis-server --requirepass outraSenhaForte456 # Inicia o Redis com senha
        volumes:
          - n8n_redis_data:/data
        networks:
          - assistant_network
        logging:
          driver: gelf
          options:
            gelf-address: "udp://logstash_gelf:12201"
            tag: "n8n-redis"

      redis_client: # Interface Gráfica para o Redis
        image: redislabs/redisinsight:1.14.0
        container_name: redis_gui_client
        ports:
          - "8001:8001"
        volumes:
          - redisinsight_data:/db
        networks:
          - assistant_network
        restart: unless-stopped

      # --- ELK Stack Services for Logging ---
      elasticsearch_log:
        image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4
        container_name: elasticsearch_log_data
        environment:
          - discovery.type=single-node
          - xpack.security.enabled=false
          - ES_JAVA_OPTS=-Xms512m -Xmx512m
        ulimits:
          memlock:
            soft: -1
            hard: -1
        volumes:
          - elasticsearch_data:/usr/share/elasticsearch/data
        ports:
          - "9200:9200"
        restart: unless-stopped
        networks:
          - assistant_network

      logstash_gelf:
        image: docker.elastic.co/logstash/logstash:8.13.4
        container_name: logstash_gelf_processor
        command: logstash -f /usr/share/logstash/config/logstash.conf
        volumes:
          - ./logstash/config:/usr/share/logstash/config
        ports:
          - "12201:12201/udp"
        environment:
          - LS_JAVA_OPTS=-Xms256m -Xmx256m
        depends_on:
          - elasticsearch_log
        restart: unless-stopped
        networks:
          - assistant_network
        logging:
          driver: "json-file"
          options:
            max-size: "10m"
            max-file: "5"

      kibana_viewer:
        image: docker.elastic.co/kibana/kibana:8.13.4
        container_name: kibana_log_viewer
        ports:
          - "5601:5601"
        environment:
          - ELASTICSEARCH_HOSTS=http://elasticsearch_log:9200
          - XPACK_SECURITY_ENABLED=false
        depends_on:
          - elasticsearch_log
        restart: unless-stopped
        networks:
          - assistant_network

    # --- Volumes para persistência de dados ---
    volumes:
      n8n_data:
      n8n_mongo_data:
      n8n_redis_data:
      elasticsearch_data:
      redisinsight_data: # Volume para a GUI do Redis

    # --- Rede customizada para comunicação entre os serviços ---
    networks:
      assistant_network:
        name: personal_ai_assistant_network
    ```

3.  **Crie e Preencha o arquivo `logstash/config/logstash.conf`:**

    Cole o conteúdo abaixo no seu arquivo `logstash/config/logstash.conf`:

    ```conf
    # logstash/config/logstash.conf
    input {
      gelf {
        port => 12201
        codec => json
      }
    }

    filter {
      # Adicione filtros aqui se necessário
    }

    output {
      elasticsearch {
        hosts => ["elasticsearch_log:9200"]
        index => "personal-ai-assistant-logs-%{+YYYY.MM.dd}"
      }
    }
    ```

4.  **Inicie os Contêineres:**

    Antes de iniciar pela primeira vez (ou após fazer alterações nas senhas), é uma boa prática limpar volumes antigos para evitar conflitos.

    ```bash
    # Opcional, mas recomendado na primeira execução:
    docker-compose down --volumes
    ```

    No terminal, na pasta onde está seu `docker-compose.yml`, execute:

    ```bash
    docker-compose up -d
    ```

    Este comando irá iniciar todos os contêineres.

5.  **Acesse o n8n no seu navegador:**

    Abra [http://localhost:5678](http://localhost:5678) no seu navegador.

## 💻 Acessando a Interface Gráfica do MongoDB (Compass)

Você pode usar uma ferramenta como o MongoDB Compass para se conectar ao banco de dados.

### Instalação (Exemplo para Ubuntu)

```bash
# 1. Baixe e instale o MongoDB Compass do site oficial:
# https://www.mongodb.com/try/download/compass

# 2. Conecte-se usando os dados abaixo.
```

### Como se Conectar

-   **Connection String:** `mongodb://admin:minhaSenhaForte123@localhost:27017/?authSource=admin`
    Você pode colar esta string diretamente no MongoDB Compass.

-   **Ou preencha manualmente:**
    -   Hostname: `localhost`
    -   Port: `27017`
    -   Authentication: `Username / Password`
    -   Username: `admin`
    -   Password: `minhaSenhaForte123`
    -   Authentication Database: `admin`

## 💻 Acessando a Interface Gráfica do Redis (RedisInsight)

Este projeto já inclui o RedisInsight, uma ferramenta gráfica web para gerenciar o Redis. Nenhuma instalação adicional é necessária.

1.  **Acesse o RedisInsight no seu navegador:**

    Abra [http://localhost:8001](http://localhost:8001) no seu navegador.

2.  **Adicione a Conexão do Banco de Dados:**

    Clique em "Add Redis Database" e preencha os campos manualmente:

    -   Host: `n8n_redis_cache` (**Importante:** Use o nome do serviço Docker, não `localhost`)
    -   Port: `6379`
    -   Name: `n8n Cache` (ou qualquer apelido que preferir)
    -   Password: `outraSenhaForte456`

## 🔎 Visualizando os Logs com Kibana

O Kibana é a interface de visualização que permite explorar e analisar os logs coletados no Elasticsearch.

1.  **Acesse o Kibana no navegador:**

    Abra [http://localhost:5601](http://localhost:5601).

2.  **Crie um "Data View" (Padrão de Índice):**

    -   No menu, vá para `Stack Management` > `Kibana` > `Data Views`.
    -   Clique em `Create data view`.
    -   Name: `personal-ai-assistant-logs-*`
    -   Timestamp field: `@timestamp`
    -   Clique em `Create data view`.

3.  **Explore seus Logs no "Discover":**

    -   No menu, vá para `Analytics` > `Discover`.
    -   Você verá os logs de todos os contêineres. Use a barra de pesquisa para filtrar, por exemplo: `tag: "n8n"` para ver apenas os logs do n8n.

## 🎈 Uso

Depois que o ambiente estiver em execução, você pode:

-   **Acessar o n8n:** Vá para [http://localhost:5678](http://localhost:5678) para criar seus fluxos de trabalho.
-   **Monitorar Logs:** Utilize o Kibana em [http://localhost:5601](http://localhost:5601) para uma visão centralizada dos eventos.
-   **Gerenciar Dados:** Conecte-se ao MongoDB com o Compass e acesse o RedisInsight em [http://localhost:8001](http://localhost:8001) para inspecionar os dados.

## 🚀 Implantação

Para implantar isso em produção, considere:

-   **Gerenciamento de Segredos:** Mova as senhas do `docker-compose.yml` para um arquivo `.env` ou para um sistema de gerenciamento de segredos.
-   **Segurança do Elasticsearch:** Habilite o `xpack.security.enabled=true` e configure usuários, senhas e TLS para o Elasticsearch e Kibana.
-   **Reverse Proxy:** Use um proxy reverso (Nginx, Traefik) para gerenciar o acesso externo e configurar HTTPS.
-   **Volumes Persistentes:** Garanta que os volumes Docker estejam em um storage robusto e com backup.

## ⛏️ Construído Usando

-   [n8n](https://n8n.io/) - Plataforma de Automação de Fluxo de Trabalho
-   [MongoDB](https://www.mongodb.com/) - Banco de Dados NoSQL
-   [Redis](https://redis.io/) - Cache e Fila em Memória
-   [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) - GUI para Redis
-   [Elasticsearch, Logstash, Kibana (ELK Stack)](https://www.elastic.co/elastic-stack) - Coleta e Visualização de Logs
-   [Docker & Docker Compose](https://www.docker.com/) - Plataforma de Contêineres

## 🤝 Contribuindo

Contribuições são bem-vindas!

1.  Faça um Fork do Projeto
2.  Crie sua Feature Branch (`git checkout -b feature/AmazingFeature`)
3.  Faça Commit das suas Mudanças (`git commit -m 'Add some AmazingFeature'`)
4.  Faça Push para a Branch (`git push origin feature/AmazingFeature`)
5.  Abra um Pull Request

## ✍️ Autores

-   Guilherme Namura Felix - Desenvolvimento Inicial