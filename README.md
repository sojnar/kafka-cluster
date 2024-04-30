# Índice

- [Configuração do Kafka com Strimzi no Kubernetes](#configuração-do-kafka-com-strimzi-no-kubernetes)
  - [Pré-requisitos](#pré-requisitos)
  - [Baixar os Binários do Kafka e Configurar o PATH](#baixar-os-binários-do-kafka-e-configurar-o-path)
- [Ingress Controller](#ingress-controller)
  - [Criando um Ingress Controller (Nginx)](#criando-um-ingress-controller-nginx)
  - [Listando o namespace criado](#listando-o-namespace-criado)
  - [Listando todos os recursos criados no namespaces ingress-nginx](#listando-todos-os-recursos-criados-no-namespaces-ingress-nginx)
  - [Habilidado o TLS passthrough](#habilidado-o-tls-passthrough)
- [Cluster-Operator kafka](#cluster-operator-kafka)
  - [Criando um Namespace chamado Kafka](#criando-um-namespace-chamado-kafka)
  - [Deployando o Cluster Operator](#deployando-o-cluster-operator)
  - [Acompanhando os logs do operator](#acompanhando-os-logs-do-operator)
  - [Aplicando as Configurações de Métricas do Monitoramento](#aplicando-as-configurações-de-métricas-do-monitoramento)
  - [Consultando o ConfigMap criado](#consultando-o-configmap-criado)
  - [Provisionando o Cluster Kafka](#provisionando-o-cluster-kafka)
  - [Acompanhando a criação dos pods](#acompanhando-a-criação-dos-pods)
  - [Acompanhando a crição dos volumes persistentes](#acompanhando-a-crição-dos-volumes-persistentes)
- [Criação de Tópicos](#criação-de-tópicos)
  - [Crie os tópicos necessários para sua aplicação](#crie-os-tópicos-necessários-para-sua-aplicação)
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

## Ingress Controller
- ### Criando um Ingress Controller (Nginx)
```bash
$ kubectl apply -f kafka-cluster/ingress-nginx-controller.yaml
```

- ### Listando o namespace criado
```bash 
$ kubectl get namespacess | grep ingress
```

- ### Listando todos os recursos criados no namespaces ingress-nginx
```bash
$ kubectl get all -n ingress-nginx
```

- ### O suporte Ingress no Strimzi foi adicionado no Strimzi 0.12.0. Ele usa TLS passthrough e foi testado com o controlador NGINX Ingress. Antes de usá-lo, certifique-se de que TLS passthrough esteja habilitada no controlador.

- ### Habilidado o TLS passthrough

```bash
$ kubectl -n ingress-nginx edit deployment/ingress-nginx-controller
```

```yaml
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --watch-ingress-without-class=true
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
        - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --enable-ssl-passthrough
        env:
```

## Cluster-Operator kafka

- ### Criando um Namespace chamado Kafka
Execute o comando:
```bash
$ kubectl create namespace kafka
```

- ### Deployando o Cluster Operator
  * #### O Cluster-Operator é responsável por gerenciar os recursos do Kafka dentro do Kubernetes.
```bash
$ kubectl apply -f strimzi-cluster-operator.yaml
```

- ### Acompanhando os logs do operator:
```bash
$ kubectl logs <nome-do-pod> -n kafka
```

- ### Aplicando as Configurações de Métricas do Monitoramento:
```bash
$ kubectl apply -f kafka-metrics-configMap.yaml
```

- ### Consultando o ConfigMap criado:
```bash
$ kubectl get configMap -n kafka
```

- ### Provisionando o Cluster Kafka:
```bash
$ kubectl apply -f kafka-cluster-persistent.yaml
```

- ### Acompanhando a criação dos pods:
```bash
$ kubectl get pod -n kafka -w
```

- ### Acompanhando a crição dos volumes persistentes:
```bash
$ kubectl get pvc -n kafka -w
```

- ### Para deletar um volume persistente:
```bash
$ kubectl delete pvc/<nome-do-pvc> -n kafka
```

## Criação de Tópicos
- ### Crie os tópicos necessários para sua aplicação:
```bash
$ kubectl apply -f kafka-topic.yaml
```

- ### Verifique a configuração do tópico criado:
```bash
$ kubectl get KafkaTopic -n kafka
```

- ### Verificação dos Serviços Criados:
```bash
$ kubectl get svc -n kafka
```

## Uso dos Binários do Kafka para Testes

- ### Para Testar o envio e recebimento de mensagens usando o client do Kafka é preciso baixar o crt criado no provisionamento do cluster, para isso precisamos verificar se as secretes foram criadas.

  - ### Verificando as secrets
  ```bash 
  $ kubectl get secrets -n kafka
  NAME                                TYPE     DATA   AGE
  my-cluster-cluster-ca-cert          Opaque   3      18d
  ```

  - ### Verificando se a chave foi criada corretamante, o resultado deve ser semelhante ao exemplo abaixo, o CN deve ser igual o nome do seu cluster.
  ```bash
  $ openssl s_client -connect  -servername kafka-bootstrap.pmrun.com.br -showcerts | grep CN

  depth=1 O = io.strimzi, CN = cluster-ca v0
  verify error:num=19:self-signed certificate in certificate chain
  
  depth=1 O = io.strimzi, CN = cluster-ca v0
  
  depth=0 O = io.strimzi, CN = my-cluster-kafka
  
  0 s:O = io.strimzi, CN = my-cluster-kafka
    i:O = io.strimzi, CN = cluster-ca v0
  1 s:O = io.strimzi, CN = cluster-ca v0
    i:O = io.strimzi, CN = cluster-ca v0
  subject=O = io.strimzi, CN = my-cluster-kafka
  issuer=O = io.strimzi, CN = cluster-ca v0
  ```

  - ### Baixando o .crt
  ```bash
  $ kubectl get secret my-cluster-cluster-ca-cert -o jsonpath='{.data.ca\.crt}' -n kafka | base64 -d > ca.crt
  ```

  - ### Importando o .crt
  ```bash
  $ keytool -import -trustcacerts -alias root \
    -file ca.crt -keystore truststore.jks -storepass password -noprompt
  ```

- ### Liste os tópicos disponíveis:
```bash
$ echo -e "security.protocol=SSL\nssl.truststore.location=./truststore.jks\nssl.truststore.password=password" > config.properties

$ kafka-topics.sh --list --bootstrap-server kafka-bootstrap.pmrun.com.br:443 --command-config config.properties
```

- ### Teste o envio de mensagens:
```bash
$ kafka-console-producer.sh --broker-list kafka-bootstrap.pmrun.com.br:443 \
  --producer-property security.protocol=SSL \
  --producer-property ssl.truststore.password=password \
  --producer-property ssl.truststore.location=./truststore.jks --topic itss
```

- ### Teste o recebimento de mensagens:
```bash
$ kafka-console-consumer.sh --bootstrap-server kafka-bootstrap.pmrun.com.br:443 \
  --topic itss --from-beginning --consumer-property security.protocol=SSL \
  --consumer-property ssl.truststore.password=password \
  --consumer-property ssl.truststore.location=./truststore.jks
```

## Configuração do prometheus + grafana
- ### Defina o namespace padrao no contexto
```bash
$ kubectl config set-context --current --namespace=kafka
```

- ### Instale o prometheus operator
```bash
$ kubectl apply -f prometheus-operator-deployment.yaml 
```

- ### Verifique se o prometheus operator foi inicializado
```bash
$ kubectl get pods|grep prometheus
```

- ### Crie o additional scrape
```bash
$ kubectl create secret generic additional-scrape-configs --from-file=./prometheus-additional.yaml -n kafka

$ kubectl apply -f prometheus-additional.yaml 
```

- ### Crie o pod-monitor
```bash
$ kubectl apply -f strimzi-pod-monitor.yaml
```

- ### Crie as regras do prometheus
```bash
$ kubectl apply -f prometheus-rules.yaml
```

- ### Inicialize o prometheus
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

- ### Inicialize o Grafana
```bash
$ kubectl apply -f grafana.yaml
```

```bash
$ kubectl get pvc -n kafka
```

- ### Edite o arquivo ingress-path.yml com a url do seu grafana e crie o recurso
```bash
$ kubectl apply -f ingress-path.yml
```

- ### No browser acesse a url do grafana e configure o datasource
    -  >datasource > config > http://prometheus-operated.kafka.svc.cluster.local:9090
- ### Importe os templates
    - strimzi-zookeeper.json
    - strimzi-kafka.json
    - strimzi-kafka-exporter.json


## Configuração do loki
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