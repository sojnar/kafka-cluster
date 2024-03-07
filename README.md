# Índice

- [Configuração do Kafka com Strimzi no Kubernetes](#configuração-do-kafka-com-strimzi-no-kubernetes)
  - [Pré-requisitos](#pré-requisitos)
  - [Baixar os Binários do Kafka e Configurar o PATH](#baixar-os-binários-do-kafka-e-configurar-o-path)
  - [Criar um Namespace Chamado Kafka](#criar-um-namespace-chamado-kafka)
  - [Provisionar o Cluster-Operator](#provisionar-o-cluster-operator)
  - [Aplicar as Configurações de Métricas do Monitoramento](#aplicar-as-configurações-de-métricas-do-monitoramento)
  - [Provisionar o Cluster Kafka](#provisionar-o-cluster-kafka)
  - [Criação de Tópicos](#criação-de-tópicos)
  - [Verificação dos Serviços Criados](#verificação-dos-serviços-criados)
  - [Uso dos Binários do Kafka para Testes](#uso-dos-binários-do-kafka-para-testes)
- [Configuração do Prometheus + Grafana](#configuração-do-prometheus--grafana)
- [Configuração do Loki](#configuração-do-loki)


# Configuração do Kafka com Strimzi no Kubernetes

### Pré-requisitos

- Ter um cluster Kubernetes pronto para uso.
- Ter o `$ kubectl` configurado para se comunicar com seu cluster Kubernetes.
- Acesso ao terminal ou linha de comando.

### Baixar os Binários do Kafka e Configurar o PATH

1. Baixe os binários do Kafka do [site oficial](https://kafka.apache.org/downloads).
2. Extraia o conteúdo do arquivo baixado em uma pasta de sua escolha.
3. Adicione o caminho da pasta `bin` dentro da pasta extraída ao seu `PATH` para poder executar os comandos do Kafka diretamente do terminal.

### Criar um Namespace Chamado Kafka

Execute o comando:
```bash
$ kubectl create namespace kafka
```

## Provisionar o Cluster-Operator
### O Cluster-Operator é responsável por gerenciar os recursos do Kafka dentro do Kubernetes.
```bash
$ kubectl apply -f strimzi-cluster-operator.yaml
```

### Acompanhe os logs do operator:
```bash
$ kubectl logs <nome-do-pod> -n kafka
```

### Aplicar as Configurações de Métricas do Monitoramento:
```bash
$ kubectl apply -f kafka-metrics-configMap.yaml
```

### Consulte o ConfigMap criado:
```bash
$ kubectl get configMap -n kafka
```

### Provisionar o Cluster Kafka:
```bash
$ kubectl apply -f kafka-cluster-persistent.yaml
```

### Acompanhe a criação dos pods:
```bash
$ kubectl get pod -n kafka -w
```

### Acompanhe a crição dos volumes persistentes:
```bash
$ kubectl get pvc -n kafka -w
```

### Para deletar um volume persistente:
```bash
$ kubectl delete pvc/<nome-do-pvc> -n kafka
```

## Criação de Tópicos
### Crie os tópicos necessários para sua aplicação.:
```bash
$ kubectl apply -f kafka-topic.yaml
```

### Verifique a configuração do tópico criado:
```bash
$ kubectl get KafkaTopic -n kafka
```

### Verificação dos Serviços Criados:
```bash
$ kubectl get svc -n kafka
```

## Uso dos Binários do Kafka para Testes
### Liste os tópicos disponíveis:
```bash
$ kafka-topics.sh --bootstrap-server <seu-ip-externo-do-broker>:9094 --list
```

### Teste o envio de mensagens:
```bash
$ kafka-console-producer.sh --broker-list <seu-ip-externo-do-broker>:9094 --topic <nome-do-topico>
```

### Teste o recebimento de mensagens:
```bash
$ kafka-console-consumer.sh --bootstrap-server <seu-ip-externo-do-broker>:9094 --topic <nome-do-topico> --from-beginning
```

#
# Configuração do prometheus + grafana
### Defina o namespace padrao no contexto
```bash
$ kubectl config set-context --current --namespace=kafka
```

### Instale o prometheus operator
```bash
$ kubectl apply -f prometheus-operator-deployment.yaml 
```

- #### Verifique se o prometheus operator foi inicializado
```bash
$ kubectl get pods|grep prometheus
```

### Crie o additional scrape
```bash
$ kubectl create secret generic additional-scrape-configs --from-file=./prometheus-additional.yaml -n kafka

$ kubectl apply -f prometheus-additional.yaml 
```

### Crie o pod-monitor
```bash
$ kubectl apply -f strimzi-pod-monitor.yaml
```

### Crie as regras do prometheus
```bash
$ kubectl apply -f prometheus-rules.yaml
```

### Inicialize o prometheus
```bash
$ kubectl apply -f prometheus.yaml
```
- ### Verifique se o prometheus foi inicializado
```bash
$ kubectl get pod -w
```

- ### Em uma outra aba da sua console monitore os services
```bash
$ kubectl get svc -w
```

### Inicialize o Grafana
```bash
$ kubectl apply -f grafana.yaml
```

```bash
$ kubectl get pvc -n kafka
```

- ### No browser acesse a url do grafana e configure o datasource
    -  >datasource > config > http://prometheus-operated.kafka.svc.cluster.local:9090
- ### Importe os templates
    - strimzi-zookeeper.json
    - strimzi-kafka.json
    - strimzi-kafka-exporter.json


# Configuração do loki
- ### Adicionando repositorio helm grafana
```bash
$ helm repo add grafana https://grafana.github.io/helm-charts
```

- ### Listando os repositórios instalados
```bash
$ helm repo list
```

- ### Criando um namespace para o loki
```bash
$ kubectl create ns loki
```

- ### Definindo namespace padrão
```bash
$ kubectl config set-context --current --namespace=loki
```

- ### Instalando loki-stack
```bash
$ helm upgrade --install loki grafana/loki-stack -f values.yaml -n loki
```

- ### Verifique se todos os objetos subiram com sucesso
```bash
$ kubectl get all
```

- ### Ajuste o client.url no daemonset
```bash
$ kubectl edit daemonset loki-promtail

    spec:
      containers:
      - args:
        - -config.file=/etc/promtail/promtail.yaml
        - -client.url=http://loki:3100/loki/api/v1/push
```

- ### Consulte os logs do pod/loki-promtail
```bash
$ kubectl logs -f pod/loki-promtail
```

- ### Verifique se os logs que o Promtail esta lendo
```bash
$ kubectl port-forward pod/loki-promtail-m64r9  8081:3101
```

- > Acesse no seu navegador a url= http://localhost:8081
    
- ### Configure o loki no grafana
    >datasouce>Add data source>loki > http://loki.loki.svc.cluster.local:3100