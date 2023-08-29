# Laboratório Prometheus

> Status: Developing 


## Descrição do Laboratório:

Laboratório baseado no exemplo disponível no site [prometheus.io](https://prometheus.io/docs/introduction/overview/)

## Technologies Used:

* Linux (Ubuntu based)
* Golang
* Prometheus
* Grafana
* Alertmanager




1. Acessar a documentação do Promethues:

- Introdução ao Prometheus
    - ****MOSTRE-ME COMO É FEITO****
    1. Instalar o Docker e o Docker-compose
    2. Criar o arquivo de configuração do promethues e nodeexporter
    
    ```
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
    
    1. Criar o arquivo docker-compose
        
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
        		# network_mode: "host"
        
          node-exporter:
            image: prom/node-exporter
            ports:
              - 9100:9100
        		# network_mode: "host"
        ```
        
        Métrica: node_cpu-seconds_total
        
- Entendendo os tipos de Métricas
    
    Demonstrar na documentação oficial
    
- Instrumentando um servidor HTTP escrito em GO
    1. Criar o arquivo server.go
        
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
        
    2. Build do server, executar e testar
        
        ```go
        sudo add-apt-repository ppa:longsleep/golang-backports
        sudo apt update
        sudo apt install golang-1.18
        
        export PATH=$PATH:/usr/local/go/bin
        
        #buildar o arquivo go
        go build server.go
        # Executar o server
        ./server
        #acessar o http://localhost:8090/ping
        ```
        
    3. Fazer a instrumentação da aplicação
        
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
               Help: "No of request handled by Ping handler",
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
        
    4. Executar o exemplo
        
        ```go
        go mod init prom_example
        go mod tidy
        go build server.go
        ./server
        
        #Acessar http://localhost:8090/metrics
        
        ```
        
    5. Atualizar o arquivo de configuração do prometheus
        
        ```go
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
        
    6. Subir o container do promethues novamente para atualizar as configurações e visualizar a métrica 
        
        ping_request_count
        
- Visualizando métricas utilizando o Grafana
    1. Vamos ajustar nosso arquivo docker-compose para incluir o grafana.
        
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
        
    2. Executar o docker compose e subir os containers.
    3. ****Adicionando Prometheus como fonte de dados no Grafana.****
    4. ****Criando nosso primeiro painel. (Doc)****
- Alertas baseados em métricas