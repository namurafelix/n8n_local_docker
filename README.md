## 📝 Tabela de Conteúdos

* [Sobre](#sobre)
* [Começando](#comecando)
* [Instalação do MongoDB Compass no Ubuntu](#instalacao-mongodb-compass-ubuntu)
* [Instalação do Another Redis Desktop Manager no Ubuntu](#instalacao-ardm-ubuntu)
* [Implantação](#implantacao)
* [Uso](#uso)
* [Construído Usando](#construido_usando)
* [Contribuindo](#contribuindo)
* [Autores](#autores)

## 🧐 Sobre <a name="sobre"></a>


Este projeto configura um ambiente completo para executar o n8n, uma poderosa ferramenta de automação de fluxos de trabalho. Ele utiliza o Docker Compose para facilitar a configuração e o gerenciamento dos serviços necessários. O MongoDB é utilizado para armazenar os dados do n8n de forma persistente, garantindo que seus fluxos de trabalho e configurações sejam preservados entre reinicializações. O Redis é empregado para otimizar o desempenho do n8n, gerenciando filas de tarefas e fornecendo um mecanismo de cache eficiente.

Além disso, esta configuração inclui um stack de logs (Elasticsearch, Logstash e Kibana) para coletar, armazenar e visualizar centralizadamente os logs de todos os seus serviços. Isso é crucial para depuração, monitoramento e compreensão do comportamento do seu assistente. Esta configuração permite que você aproveite ao máximo os recursos do n8n em um ambiente robusto, escalável e observável.

🏁 Começando <a name="comecando"></a>

Estas instruções irão ajudá-lo a obter uma cópia do projeto em execução na sua máquina local para fins de desenvolvimento e teste. Veja implantação para notas sobre como implantar o projeto em um sistema ativo.

Pré-requisitos
Você precisa ter o Docker e o Docker Compose instalados na sua máquina.

Bash

# Exemplo para verificar a instalação do Docker
docker --version

# Exemplo para verificar a instalação do Docker Compose
docker-compose --version
Instalando
Siga estes passos para configurar o ambiente de desenvolvimento:

Crie a estrutura de pastas e arquivos:
Na pasta raiz do seu projeto, crie a seguinte estrutura:

.
├── docker-compose.yml
└── logstash/
    └── config/
        └── logstash.conf
Preencha o arquivo docker-compose.yml:
Cole o conteúdo abaixo no seu arquivo docker-compose.yml:

YAML

version: '3.8'

services:
  # --- N8N Core Services ---
  n8n:
    image: n8nio/n8n:latest # Usando a versão mais recente
    container_name: n8n_core_app
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n # Volume persistente para dados do n8n
    environment:
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PORT=${N8N_PORT:-5678}
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - NODE_ENV=production
      - N8N_DB=mongodb
      - N8N_DB_MONGODB_URL=mongodb://n8n_mongo_db:27017/n8n # Conecta ao serviço MongoDB
      - REDIS_HOST=n8n_redis_cache # Conecta ao serviço Redis
      - N8N_LOG_LEVEL=info # Nível de log do n8n (info, debug, warn, error)
      - N8N_LOG_OUTPUT=stdout # N8N envia logs para a saída padrão para o Docker capturar
    restart: unless-stopped
    networks:
      - assistant_network # Adiciona à rede customizada
    logging: # Configura o driver de log do Docker para GELF
      driver: gelf
      options:
        gelf-address: "udp://logstash_gelf:12201" # Aponta para o serviço Logstash
        tag: "n8n" # Tag para identificar os logs no Logstash/Elasticsearch

  n8n_mongo_db:
    image: mongo:latest
    container_name: n8n_mongo_db_data
    restart: unless-stopped
    volumes:
      - n8n_mongo_data:/data/db # Volume persistente para dados do MongoDB
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
    volumes:
      - n8n_redis_data:/data # Volume persistente para dados do Redis
    networks:
      - assistant_network
    logging:
      driver: gelf
      options:
        gelf-address: "udp://logstash_gelf:12201"
        tag: "n8n-redis"

  # --- ELK Stack Services for Logging ---
  elasticsearch_log:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.4 # Versão recomendada
    container_name: elasticsearch_log_data
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false # ATENÇÃO: NÃO USE EM PRODUÇÃO SEM CONFIGURAR A SEGURANÇA!
      - ES_JAVA_OPTS=-Xms512m -Xmx512m # Aloca 512MB de RAM. Ajuste conforme sua máquina.
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data # Volume persistente para dados do ES
    ports:
      - "9200:9200" # Porta da API REST do Elasticsearch
    restart: unless-stopped
    networks:
      - assistant_network

  logstash_gelf:
    image: docker.elastic.co/logstash/logstash:8.13.4 # Mesma versão principal do ES
    container_name: logstash_gelf_processor
    command: logstash -f /usr/share/logstash/config/logstash.conf # Aponta para o arquivo de config
    volumes:
      - ./logstash/config:/usr/share/logstash/config # Monta o diretório de config
    ports:
      - "12201:12201/udp" # Porta para receber logs GELF via UDP
    environment:
      - LS_JAVA_OPTS=-Xms256m -Xmx256m # Aloca 256MB de RAM. Ajuste conforme necessário.
    depends_on:
      - elasticsearch_log # Inicia depois do Elasticsearch
    restart: unless-stopped
    networks:
      - assistant_network
    logging: # Desabilita o driver padrão de logging do docker para evitar loop de logs do logstash
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  kibana_viewer:
    image: docker.elastic.co/kibana/kibana:8.13.4 # Mesma versão principal do ES e Logstash
    container_name: kibana_log_viewer
    ports:
      - "5601:5601" # Porta do Kibana UI
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch_log:9200 # Conecta ao Elasticsearch usando o novo nome do serviço
      - XPACK_SECURITY_ENABLED=false # ATENÇÃO: NÃO USE EM PRODUÇÃO SEM CONFIGURAR A SEGURANÇA!
    depends_on:
      - elasticsearch_log # Inicia depois do Elasticsearch
    restart: unless-stopped
    networks:
      - assistant_network

# --- Volumes para persistência de dados ---
volumes:
  n8n_data:
  n8n_mongo_data:
  n8n_redis_data:
  elasticsearch_data:

# --- Rede customizada para comunicação entre os serviços ---
networks:
  assistant_network:
    name: personal_ai_assistant_network
Crie e Preencha o arquivo logstash/config/logstash.conf:
Cole o conteúdo abaixo no seu arquivo logstash/config/logstash.conf:

# logstash/config/logstash.conf
input {
  gelf {
    port => 12201 # Porta para receber logs GELF
    codec => json
  }
}

filter {
  # O GELF já estrutura bem os logs, mas você pode adicionar filtros para parsear
  # campos específicos ou enriquecer os dados se necessário.
  # Exemplo: Renomear o campo 'host' para algo mais descritivo
  # mutate {
  #   rename => { "host" => "docker_container_host" }
  # }
}

output {
  elasticsearch {
    hosts => ["elasticsearch_log:9200"] # Aponta para o serviço Elasticsearch
    index => "personal-ai-assistant-logs-%{+YYYY.MM.dd}" # Índice diário no Elasticsearch
  }
  # Para depuração, descomente a linha abaixo para ver os logs no terminal do Logstash
  # stdout { codec => rubydebug }
}
Inicie os Contêineres:
No terminal, na pasta onde está seu docker-compose.yml, execute:

Bash

docker compose up -d
Este comando irá iniciar todos os contêineres: n8n, MongoDB, Redis, Elasticsearch, Logstash e Kibana.

Acesse o n8n no seu navegador:
Abra http://localhost:5678 no seu navegador.

Instalação do MongoDB Compass no Ubuntu <a name="instalacao-mongodb-compass-ubuntu"></a>
O MongoDB Compass é uma GUI para o MongoDB. Ele pode ser instalado no Ubuntu da seguinte forma:

Importe a chave GPG pública do MongoDB:

Bash

wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
Crie uma lista de fontes para o MongoDB:

Bash

echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
Substitua focal pela sua versão do Ubuntu, se necessário (e.g., bionic, groovy, hirsute). Verifique a versão correta em https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/

ReCarregue o cache do pacote:

Bash

sudo apt update
Instale o MongoDB Compass:

Bash

sudo apt install mongodb-compass
Execute o MongoDB Compass:

Bash

mongocompass
Instalação do Another Redis Desktop Manager no Ubuntu <a name="instalacao-ardm-ubuntu"></a>
Another Redis Desktop Manager (ARDM) é uma GUI para o Redis. Ele pode ser instalado no Ubuntu da seguinte forma:

Baixe o pacote .deb do ARDM no site do GitHub:

Acesse a página de lançamentos do ARDM no GitHub: https://github.com/qishibo/AnotherRedisDesktopManager/releases
Encontre a versão mais recente e baixe o arquivo .deb para a sua arquitetura (geralmente amd64).
Instale o pacote .deb usando o dpkg:

Bash

sudo dpkg -i <caminho_do_arquivo_deb>
Substitua <caminho_do_arquivo_deb> pelo caminho do arquivo que você baixou.

Corrija quaisquer dependências ausentes:

Bash

sudo apt -f install
Execute o ARDM:

Bash

ardm
🔎 Visualizando os Logs com Kibana <a name="visualizando-os-logs-com-kibana"></a>
O Kibana é a interface de visualização que permite explorar, analisar e criar dashboards a partir dos logs coletados no Elasticsearch.

Acesse o Kibana no navegador:
Abra http://localhost:5601 no seu navegador.

Crie um "Index Pattern" (Padrão de Índice):
O Kibana precisa de um padrão para saber onde buscar os logs no Elasticsearch.

Na barra de navegação do Kibana (geralmente à esquerda), navegue até "Stack Management" (ou apenas "Management" em versões mais antigas).
Dentro de "Stack Management", encontre a seção "Kibana" e clique em "Index Patterns" (ou "Data Views" em versões mais novas).
Clique em "Create data view" (ou "Create index pattern").
No campo "Index pattern name", digite personal-ai-assistant-logs-*. O asterisco (*) é um curinga que corresponde a todos os índices que começam com "personal-ai-assistant-logs-", que é o que o Logstash está criando diariamente (ex: personal-ai-assistant-logs-2025.06.08).
No campo "Timestamp field", selecione @timestamp. Este campo é crucial para que o Kibana possa organizar os logs cronologicamente.
Clique em "Create data view".
Explore seus Logs no "Discover":
Agora que o padrão de índice está configurado, você pode ver seus logs.

No menu de navegação do Kibana, vá para a seção "Analytics" e clique em "Discover".
Certifique-se de que o "Index Pattern" que você acabou de criar (personal-ai-assistant-logs-*) esteja selecionado no canto superior esquerdo da tela.
Você verá uma lista de todos os logs coletados de seus contêineres Docker (n8n, MongoDB, Redis).
Filtrar Logs: Você pode usar a barra de pesquisa na parte superior para filtrar logs (ex: digite tag: "n8n" para ver apenas os logs do n8n, ou level: "error" para ver apenas os logs de erro).
Ajustar Intervalo de Tempo: No canto superior direito, você pode ajustar o intervalo de tempo para ver logs de "últimos 15 minutos", "última hora", "último dia", etc.
Inspecionar Logs: Clique em uma linha de log para expandir e ver todos os campos e detalhes associados àquela entrada de log.
🔧 Executando os testes <a name="testes"></a>
Não há testes automatizados incluídos neste projeto específico de provisionamento. Este projeto se concentra na configuração do ambiente. Os testes para o n8n em si e para o seu assistente de IA devem ser tratados separadamente, seguindo a documentação do n8n e as melhores práticas de teste para aplicações de IA.

🎈 Uso <a name="uso"></a>
Depois que o ambiente estiver configurado e todos os serviços estiverem em execução, você pode:

Acessar o n8n: Vá para http://localhost:5678 para começar a criar e gerenciar seus fluxos de trabalho de automação.
Monitorar Logs: Utilize o Kibana em http://localhost:5601 para ter uma visão centralizada e em tempo real de tudo o que está acontecendo nos seus serviços. Isso será fundamental para depurar seus fluxos de trabalho e monitorar o comportamento do assistente.
Gerenciar Dados: Conecte-se ao MongoDB (usando o Compass) e Redis (usando o ARDM) para inspecionar os dados armazenados pelo n8n e, futuramente, os dados do seu assistente (metas, objetivos, diário de bordo).
🚀 Implantação <a name="implantacao"></a>
Para implantar isso em um sistema ativo (produção), você pode usar um orquestrador de contêineres como o Kubernetes ou o Docker Swarm. Você precisará provisionar uma infraestrutura com recursos suficientes para executar todos os serviços. Além disso, considere:

Volumes Persistentes: Configurar volumes persistentes para o MongoDB, Redis e Elasticsearch em um armazenamento externo (como NFS, EFS, ou provedores de armazenamento em nuvem) para garantir a durabilidade dos dados e a resiliência contra falhas de hardware.
Reverse Proxy: Um proxy reverso (como Nginx, Caddy ou Traefik) é altamente recomendado para rotear o tráfego para o n8n e o Kibana, adicionando uma camada de segurança (HTTPS com Let's Encrypt), balanceamento de carga e gerenciamento de domínio.
Segurança: Em produção, NUNCA use xpack.security.enabled=false. Configure a segurança do Elasticsearch/Kibana com usuários e senhas robustos e considere o uso de SSL/TLS para as comunicações internas entre os serviços da ELK Stack.
Monitoramento e Alertas: Implemente um monitoramento mais avançado para os recursos dos contêineres (CPU, memória, disco) e configure alertas para falhas ou comportamentos anormais.
Backup e Recuperação: Crie uma estratégia robusta de backup para todos os volumes de dados persistentes.
⛏️ Construído Usando <a name="construido_usando"></a>
n8n - Plataforma de Automação de Fluxo de Trabalho
MongoDB - Banco de Dados NoSQL
Redis - Armazenamento de Estrutura de Dados em Memória (Cache e Message Broker)
Elasticsearch - Motor de Busca e Análise Distribuído
Logstash - Pipeline de Processamento de Dados (Coleta, Transformação, Envio)
Kibana - Plataforma de Visualização e Exploração de Dados
Docker - Plataforma de Contêineres
Docker Compose - Ferramenta de Orquestração para Múltiplos Contêineres
🤝 Contribuindo <a name="contribuindo"></a>
Contribuições são o que tornam a comunidade open source um lugar incrível para aprender, inspirar e criar. Qualquer contribuição que você fizer será muito apreciada.

Faça um Fork do Projeto
Crie sua Feature Branch (git checkout -b feature/AmazingFeature)
Faça Commit das suas Mudanças (git commit -m 'Add some AmazingFeature')
Faça Push para a Branch (git push origin feature/AmazingFeature)
Abra um Pull Request

✍️ Autores <a name="autores"></a>

Guilherme Namura Felix - Desenvolvimento Inicial