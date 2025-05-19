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

Este projeto configura um ambiente completo para executar o n8n, uma poderosa ferramenta de automação de fluxos de trabalho. Ele utiliza o Docker Compose para facilitar a configuração e o gerenciamento dos serviços necessários. O MongoDB é utilizado para armazenar os dados do n8n de forma persistente, garantindo que seus fluxos de trabalho e configurações sejam preservados entre reinicializações. O Redis é empregado para otimizar o desempenho do n8n, gerenciando filas de tarefas e fornecendo um mecanismo de cache eficiente. Esta configuração permite que você aproveite ao máximo os recursos do n8n em um ambiente robusto e escalável.

## 🏁 Começando <a name="comecando"></a>

Estas instruções irão ajudá-lo a obter uma cópia do projeto em execução na sua máquina local para fins de desenvolvimento e teste. Veja [implantação](#implantacao) para notas sobre como implantar o projeto em um sistema ativo.

### Pré-requisitos

Você precisa ter o Docker e o Docker Compose instalados na sua máquina.

\`\`\`

\# Exemplo para verificar a instalação do Docker

docker --version

\# Exemplo para verificar a instalação do Docker Compose

docker-compose --version

\`\`\`

### Instalando

Siga estes passos para configurar o ambiente de desenvolvimento:

1.  Clone este repositório (ou crie um arquivo \`docker-compose.yml\`):

    \`\`\`

    git clone <https://github.com/seu-usuario/seu-repositorio.git> \# Se você tiver um repositório

    cd seu-repositorio

    \# Ou, se você criou o arquivo manualmente:

    \# Crie um arquivo chamado docker-compose.yml com o conteúdo fornecido anteriormente.

    \`\`\`

2.  Execute o Docker Compose:

    \`\`\`

    docker-compose up -d

    \`\`\`

    Este comando irá iniciar os containers do n8n, MongoDB e Redis.

3.  Acesse o n8n no seu navegador:

    \`\`\`

    Abra <http://localhost:5678> no seu navegador.

### Instalação do MongoDB Compass no Ubuntu <a name="instalacao-mongodb-compass-ubuntu"></a>

O MongoDB Compass é uma GUI para o MongoDB. Ele pode ser instalado no Ubuntu da seguinte forma:

1.  Importe a chave GPG pública do MongoDB:

    \`\`\`
    wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
    \`\`\`

2.  Crie uma lista de fontes para o MongoDB:

    \`\`\`
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
    \`\`\`

    Substitua `focal` pela sua versão do Ubuntu, se necessário (e.g., `bionic`, `groovy`, `hirsute`).  Verifique a versão correta em <https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/>

3.  ReCarregue o cache do pacote:

    \`\`\`
    sudo apt update
    \`\`\`

4.  Instale o MongoDB Compass:

    \`\`\`
    sudo apt install mongodb-compass
    \`\`\`    
    
5.  Execute o MongoDB Compass:
    
    \`\`\`
    mongocompass
    \`\`\`

### Instalação do Another Redis Desktop Manager no Ubuntu <a name="instalacao-ardm-ubuntu"></a>

Another Redis Desktop Manager (ARDM) é uma GUI para o Redis. Ele pode ser instalado no Ubuntu da seguinte forma:

1.  Baixe o pacote .deb do ARDM no site do GitHub:

    * Acesse a página de lançamentos do ARDM no GitHub: <https://github.com/qishibo/AnotherRedisDesktopManager/releases>
    * Encontre a versão mais recente e baixe o arquivo \`.deb\` para a sua arquitetura (geralmente `amd64`).
    
2.  Instale o pacote .deb usando o `dpkg`:

    \`\`\`
    sudo dpkg -i <caminho_do_arquivo_deb>
    \`\`\`

    Substitua `<caminho_do_arquivo_deb>` pelo caminho do arquivo que você baixou.

3.  Corrija quaisquer dependências ausentes:

    \`\`\`
    sudo apt -f install
    \`\`\`

4.  Execute o ARDM:

    \`\`\`
    ardm
    \`\`\`

## 🔧 Executando os testes <a name="testes"></a>

Não há testes automatizados incluídos neste projeto específico de provisionamento. Este projeto se concentra na configuração do ambiente. Os testes para o *n8n* em si devem ser tratados separadamente, seguindo a documentação do n8n.

## 🎈 Uso <a name="uso"></a>

Depois que o ambiente estiver configurado, você pode usar o n8n para criar e executar fluxos de trabalho. O MongoDB armazenará os dados do n8n, e o Redis melhorará o desempenho.

## 🚀 Implantação <a name="implantacao"></a>

Para implantar isso em um sistema ativo, você pode usar um orquestrador de contêineres como o Kubernetes ou o Docker Swarm. Você precisará provisionar uma infraestrutura com recursos suficientes para executar os três serviços. Além disso, considere configurar volumes persistentes para o MongoDB e Redis em um armazenamento externo para garantir a durabilidade dos dados. Um proxy reverso (como Nginx ou Traefik) também é recomendado para rotear o tráfego para o n8n.

## ⛏️ Construído Usando <a name="construido_usando"></a>

* [n8n](https://n8n.io/) - Plataforma de Automação de Fluxo de Trabalho
* [MongoDB](https://www.mongodb.com/) - Banco de Dados
* [Redis](https://redis.io/) - Armazenamento de Estrutura de Dados em Memória
* [Docker](https://www.docker.com/) - Plataforma de Contêineres
* [Docker Compose](https://docs.docker.com/compose/) - Ferramenta de Orquestração de Contêineres

## ✍️ Autores <a name="autores"></a>

* Guilherme Namura Felix