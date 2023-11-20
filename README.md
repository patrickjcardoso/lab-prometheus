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

## Documentação do Prometheus:

### Introdução ao Prometheus
- **Instalação do Docker e Docker-compose**
- **Criação do arquivo de configuração `prometheus.yml`**

    ```yaml
    global:
      scrape_interval: 5s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['prometheus:9090']

      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter:9100']
    ```

- **Criação do arquivo `docker-compose.yml`**

    ```docker
    version: '3'
    services:
      prometheus:
        image: prom/prometheus
        ports:
          - 9090:9090
        volumes:
          - ./prometheus.yml:/etc/prometheus/prometheus.yml
        command:
          - '--config.file=/etc/prometheus/prometheus.yml'
        depends_on:
          - node-exporter

      node-exporter:
        image: prom/node-exporter
        ports:
          - 9100:9100
    ```

    Métrica: `node_cpu-seconds_total`

### Entendendo os Tipos de Métricas
- Demonstrar na [documentação oficial](https://prometheus.io/docs/concepts/metric_types/)

### Instrumentando um Servidor HTTP Escrito em GO
1. **Criação do arquivo `server.go`**

    ```go
    package main

    import (
       "fmt"
       "net/http"
    )

    func ping(w http.ResponseWriter, req *http.Request){
       fmt.Fprintf(w,"pong")
    }

    func main() {
       http.HandleFunc("/ping",ping)

       http.ListenAndServe(":8090", nil)
    }
    ```

2. **Build do servidor, execução e teste**

    ```bash
    sudo add-apt-repository ppa:longsleep/golang-backports
    sudo apt update
    sudo apt install golang-1.18

    export PATH=$PATH:/usr/local/go/bin

    # Buildar o arquivo go
    go build server.go
    # Executar o servidor
    ./server
    # Acessar http://localhost:8090/ping
    ```

3. **Instrumentação da aplicação no arquivo `server.go`**

    ```go
    package main

    import (
       "fmt"
       "net/http"

       "github.com/prometheus/client_golang/prometheus"
       "github.com/prometheus/client_golang/prometheus/promhttp"
    )

    var pingCounter = prometheus.NewCounter(
       prometheus.CounterOpts{
           Name: "ping_request_count",
           Help: "No of requests handled by Ping handler",
       },
    )

    func ping(w http.ResponseWriter, req *http.Request) {
       pingCounter.Inc()
       fmt.Fprintf(w, "pong")
    }

    func main() {
       prometheus.MustRegister(pingCounter)

       http.HandleFunc("/ping", ping)
       http.Handle("/metrics", promhttp.Handler())
       http.ListenAndServe(":8090", nil)
    }
    ```

4. **Execução do exemplo**

    ```bash
    go mod init prom_example
    go mod tidy
    go build server.go
    ./server

    # Acessar http://localhost:8090/metrics
    ```

5. **Atualização do arquivo de configuração do Prometheus (`prometheus.yml`)**

    ```yaml
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets: ["localhost:9090"]
      - job_name: simple_server
        static_configs:
          - targets: ["localhost:8090"]
    ```

6. **Subir o container do Prometheus novamente para atualizar as configurações e visualizar a métrica `ping_request_count`**

### Visualizando Métricas Utilizando o Grafana
1. **Ajuste do arquivo `docker-compose.yml` para incluir o Grafana**

    ```docker
    version: '3'
    services:
      prometheus:
        image: prom/prometheus
        ports:
          - 9090:9090
        volumes:
          - ./prometheus.yml:/etc/prometheus/prometheus.yml
        command:
          - '--config.file=/etc/prometheus/prometheus.yml'
        depends_on:
          - node-exporter
        network_mode: "host"

      node-exporter:
        image: prom/node-exporter
        ports:
          - 9100:9100
        network_mode: "host"

      grafana:
        image: grafana/grafana
        ports:
          - 3000:3000
        network_mode: "host"
    ```

2. **Execução do Docker Compose e subida dos containers**
3. **Adicionando Prometheus como fonte de dados no Grafana**
4. **Criando o primeiro painel ([documentação](https://grafana.com/docs/grafana/getting-started/getting-started-prometheus/))**

## Alertas Baseados em Métricas
- **Implementação de alertas utilizando o Alertmanager**
- **Configuração de alertas no arquivo `prometheus.yml`**

Com esses passos, você estará pronto para explorar e entender melhor o Prometheus, Grafana e Alertmanager no contexto deste laboratório. Boas explorações!
