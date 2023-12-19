# Laboratório Prometheus

> **Status:** Em desenvolvimento

## Descrição do Laboratório:

Este laboratório é baseado no exemplo disponível no site [prometheus.io](https://prometheus.io/docs/introduction/overview/).

## Tecnologias Utilizadas:

* Linux (baseado no Ubuntu)
* Docker
* Golang
* Prometheus
* Grafana
* Alertmanager

## Antes de começar

* Você pode fazer esse laboratório em um ambiente Windows ou Linux, local ou em uma cloud pública de sua preferência.
* Recomendo fazer todo o laboratório em um ambiente Linux.
* Caso queira, você pode utilizar o [Play With Docker](https://labs.play-with-docker.com/)
* Certifique-se de ter instalado o Docker e o Docker Compose.

## Objetivo 01 - Coletar métricas do host local utilizando o Node exporter e o Prometheus:

### Passo 01 - Arquivo de configuração do Prometheus

- **Criação do arquivo de configuração `prometheus.yml`**

```yaml
# Configurações globais aplicadas a todas as configurações de raspagem (scrape).
global:
  # Intervalo de raspagem global. Define com que frequência o Prometheus coleta métricas.
  scrape_interval: 5s
scrape_configs:
  # Configurações para o job chamado 'prometheus'.
  - job_name: 'prometheus'
    # Configuração estática para definir os alvos (targets) para raspagem.
    static_configs:
      # Lista de alvos para o job 'prometheus'. Neste caso, o Prometheus está sendo raspado em 'prometheus:9090'.
      - targets: ['prometheus:9090']

  # Configurações para o job chamado 'node-exporter'.
  - job_name: 'node-exporter'
    # Configuração estática para definir os alvos (targets) para raspagem.
    static_configs:
      # Lista de alvos para o job 'node-exporter'. Aqui, o node-exporter está sendo raspado em 'node-exporter:9100'.
      - targets: ['node-exporter:9100']
```

### Passo 02 - Arquivo Docker compose para ambiente

- **Criação do arquivo `docker-compose.yml`**

```yaml
# Versão do Docker Compose
version: '3'

# Definição dos serviços
services:
  # Serviço para o Prometheus
  prometheus:
    # Imagem a ser utilizada para o serviço Prometheus
    image: prom/prometheus
    # Mapeamento de portas - o Prometheus estará acessível na porta 9090 do host
    ports:
      - 9090:9090
    # Mapeamento de volumes - vincula o arquivo prometheus.yml local ao caminho no contêiner
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    # Comando a ser executado pelo contêiner, especificando o arquivo de configuração
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    # Dependência do serviço node-exporter - o Prometheus inicia após o node-exporter
    depends_on:
      - node-exporter

  # Serviço para o Node Exporter
  node-exporter:
    # Imagem a ser utilizada para o serviço Node Exporter
    image: prom/node-exporter
    # Mapeamento de portas - o Node Exporter estará acessível na porta 9100 do host
    ports:
      - 9100:9100
```
### Passo 03 - Testando o ambiente

#### **Comandos Utilizados**

```shell
# Criação de um diretório chamado "lab" para organizar os arquivos relacionados ao laboratório.
mkdir lab

# Navegação para o diretório recém-criado "lab".
cd lab

# Abre o editor Vim para criar/editar o arquivo de configuração do Prometheus chamado "prometheus.yml".
vim prometheus.yml

# Exibe o conteúdo do arquivo "prometheus.yml" usando o comando 'cat'.
cat prometheus.yml

# Abre o editor Vim para criar/editar o arquivo de configuração do Docker Compose chamado "docker-compose.yml".
vim docker-compose.yml

# Exibe o conteúdo do arquivo "docker-compose.yml" usando o comando 'cat'.
cat docker-compose.yml

# Inicia os serviços definidos no arquivo "docker-compose.yml" em segundo plano (detached).
docker-compose up -d

# Exibe informações sobre os contêineres em execução definidos no arquivo "docker-compose.yml".
docker-compose ps
```

#### **Acessando o Ambiente**

* Se você estiver utilizando o Play With Docker, abra o navegador e você verá as portas expostas, clique na porta 9090 que é referênte ao Prometheus

* Se você estiver em um ambiente local, acesse o host e adicione a porta do prometheus: http://localhost:9090

#### **Explore o Ambiente**

* Após acessar o Prometheus, explore os recursos disponíveis.

### Passo 04 - Entendendo os tipos de métricas disponíveis no Prometheus

#### Acesse a documentação oficial e faça a leitura
- Tipos de Métricas [documentação oficial](https://prometheus.io/docs/concepts/metric_types/)

## Objetivo 02 - Instrumentar uma aplicação Go e exportar suas métricas:

### Passo 01 - Criar e testar aplicação Go sem instrumentação

1. **Crie o arquivo `server.go`**

```go
// Pacote principal da aplicação em Go
package main

// Importação das bibliotecas necessárias
import (
   "fmt"
   "net/http"
)

// Função que responde a requisições HTTP com "pong"
func ping(w http.ResponseWriter, req *http.Request){
   fmt.Fprintf(w, "pong")
}

// Função principal que configura um manipulador para a rota "/ping"
// e inicia o servidor HTTP na porta 8090
func main() {
   // Configuração do manipulador para a rota "/ping"
   http.HandleFunc("/ping", ping)

   // Inicia o servidor HTTP na porta 8090
   http.ListenAndServe(":8090", nil)
}
```


2. **Build do servidor, execução e teste**

    ```bash
    # No ubuntu
    sudo add-apt-repository ppa:longsleep/golang-backports
    sudo apt update
    sudo apt install golang-1.18

    # No Play with docker
    apk update
    apk add go

    # Buildar o arquivo go
    go build server.go
    # Executar o servidor
    ./server
    # Acessar http://localhost:8090/ping
    ```

### Passo 02 - Instrumentar aplicação Go 

3. **Instrumentação da aplicação no arquivo `server.go`**

```go
// Pacote principal da aplicação em Go
package main

// Importação das bibliotecas necessárias
import (
   "fmt"
   "net/http"

   "github.com/prometheus/client_golang/prometheus"
   "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Definição de um contador Prometheus para contabilizar as requisições ao manipulador "/ping"
var pingCounter = prometheus.NewCounter(
   prometheus.CounterOpts{
       Name: "ping_request_count",
       Help: "No of requests handled by Ping handler",
   },
)

// Função que responde a requisições HTTP com "pong" e incrementa o contador
func ping(w http.ResponseWriter, req *http.Request) {
   pingCounter.Inc()
   fmt.Fprintf(w, "pong")
}

// Função principal que registra o contador Prometheus, configura manipuladores para as rotas "/ping" e "/metrics",
// e inicia o servidor HTTP na porta 8090
func main() {
   // Registro do contador Prometheus
   prometheus.MustRegister(pingCounter)

   // Configuração do manipulador para a rota "/ping"
   http.HandleFunc("/ping", ping)

   // Configuração do manipulador para a rota "/metrics" para expor métricas Prometheus
   http.Handle("/metrics", promhttp.Handler())

   // Inicia o servidor HTTP na porta 8090
   http.ListenAndServe(":8090", nil)
}
```


4. **Execução do exemplo**

```shell
# Inicializa um novo módulo Go chamado "prom_example". Isso cria um arquivo go.mod no diretório atual.
go mod init prom_example

# Atualiza as dependências do módulo para garantir que o arquivo go.mod reflete as dependências necessárias.
go mod tidy

# Compila o código fonte contido no arquivo server.go e gera um executável chamado "server".
go build server.go

# Executa o arquivo binário gerado, iniciando o servidor criado a partir do código fonte.
./server


# Acessar http://localhost:8090/metrics

```

### Passo 03 - Atualização das configurações do Promethues

5. **Atualização do arquivo de configuração do Prometheus (`prometheus.yml`)**

```yaml
# Configurações globais aplicadas a todas as configurações de raspagem (scrape).
global:
  # Intervalo de raspagem global. Define com que frequência o Prometheus coleta métricas.
  scrape_interval: 5s
scrape_configs:
  # Configurações para o job chamado 'prometheus'.
  - job_name: 'prometheus'
    # Configuração estática para definir os alvos (targets) para raspagem.
    static_configs:
      # Ajusta o target para eviter problemas de rede, buscando pelo endereço IP
      - targets: ['<ip_do_seu_host>:9090']

  # Configurações para o job chamado 'node-exporter'.
  - job_name: 'node-exporter'
    # Configuração estática para definir os alvos (targets) para raspagem.
    static_configs:
      # Lista de alvos para o job 'node-exporter'. Aqui, o node-exporter está sendo raspado em 'node-exporter:9100'.
      - targets: ['<ip_do_seu_host>:9100']`
  
  # Configurações para o job chamado 'simple_server'.
  - job_name: 'simple_server'
    # Configuração estática para definir os alvos (targets) para raspagem.
    static_configs:
      # Lista de alvos para o job 'node-exporter'. Aqui, o node-exporter está sendo raspado em 'node-exporter:9100'.
      - targets: ['<ip_do_seu_host>:8090']`
```

### Passo 04 - Subir o ambiente com as novas configurações.

6. **Subir o container do Prometheus novamente para atualizar as configurações e visualizar a métrica `ping_request_count`**

```shell
# Para todos os serviços definidos no arquivo docker-compose.yml, interrompe a execução dos contêineres.
docker-compose stop

# Remove todos os contêineres associados aos serviços definidos no arquivo docker-compose.yml.
docker-compose rm

# Inicia todos os serviços definidos no arquivo docker-compose.yml em segundo plano (detached mode).
docker-compose up -d

# Exibe informações sobre os contêineres em execução definidos no arquivo "docker-compose.yml".
docker-compose ps

# Lembre-se de manter o server em execução.
./server
```

## Objetivo 03 - Visualizando Métricas Utilizando o Grafana:

### Passo 01 - Ajustar o arquivo do Docker Compose.

1. **Ajuste do arquivo `docker-compose.yml` para incluir o Grafana**

```yaml
# Versão do Docker Compose
version: '3'

# Definição dos serviços
services:
  # Serviço para o Prometheus
  prometheus:
    # Imagem a ser utilizada para o serviço Prometheus
    image: prom/prometheus
    # Mapeamento de portas - o Prometheus estará acessível na porta 9090 do host
    ports:
      - 9090:9090
    # Mapeamento de volumes - vincula o arquivo prometheus.yml local ao caminho no contêiner
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    # Comando a ser executado pelo contêiner, especificando o arquivo de configuração
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    # Dependência do serviço node-exporter - o Prometheus inicia após o node-exporter
    depends_on:
      - node-exporter

  # Serviço para o Node Exporter
  node-exporter:
    # Imagem a ser utilizada para o serviço Node Exporter
    image: prom/node-exporter
    # Mapeamento de portas - o Node Exporter estará acessível na porta 9100 do host
    ports:
      - 9100:9100

  # Serviço para o Grafana
  grafana:
    # Imagem a ser utilizada para o serviço Grafana
    image: grafana/grafana
    # Mapeamento de portas - o Grafana estará acessível na porta 3000 do host
    ports:
      - 3000:3000
```

### Passo 02 - Subir o novo container no ambiente

```shell
# Inicia todos os serviços definidos no arquivo docker-compose.yml em segundo plano (detached mode).
docker-compose up -d

# Exibe informações sobre os contêineres em execução definidos no arquivo "docker-compose.yml".
docker-compose ps
```

### Passo 03 - Adicionando Prometheus como fonte de dados no Grafana


### Passo 04 - Criando o primeiro painel no Grafana

* [Documentação](https://grafana.com/docs/grafana/getting-started/getting-started-prometheus/)

## Objetivo 03 - Criando alertas baseados em métricas:

### Passo 01 - Ajustar o arquivo do Docker Compose.

- **Implementação de alertas utilizando o Alertmanager**
- **Configuração de alertas no arquivo `prometheus.yml`**

Com esses passos, você estará pronto para explorar e entender melhor o Prometheus, Grafana e Alertmanager no contexto deste laboratório. Boas explorações!
