# Configuração do Kafka com Strimzi no Kubernetes

### Pré-requisitos

- Ter um cluster Kubernetes pronto para uso.
- Ter o `kubectl` configurado para se comunicar com seu cluster Kubernetes.
- Acesso ao terminal ou linha de comando.

### Baixar os Binários do Kafka e Configurar o PATH

1. Baixe os binários do Kafka do [site oficial](https://kafka.apache.org/downloads).
2. Extraia o conteúdo do arquivo baixado em uma pasta de sua escolha.
3. Adicione o caminho da pasta `bin` dentro da pasta extraída ao seu `PATH` para poder executar os comandos do Kafka diretamente do terminal.

### Criar um Namespace Chamado Kafka

Execute o comando:
```bash
kubectl create namespace kafka
```

## Provisionar o Cluster-Operator
### O Cluster-Operator é responsável por gerenciar os recursos do Kafka dentro do Kubernetes.
```bash
kubectl apply -f strimzi-cluster-operator.yaml
```

### Acompanhe os logs do operator:
```bash
kubectl logs <nome-do-pod> -n kafka
```

### Aplicar as Configurações de Métricas do Monitoramento:
```bash
kubectl apply -f kafka-metrics-configMap.yaml
```

### Consulte o ConfigMap criado:
```bash
kubectl get configMap -n kafka
```

### Provisionar o Cluster Kafka:
```bash
kubectl apply -f kafka-cluster-persistent.yaml
```

### Acompanhe a criação dos pods:
```bash
kubectl get pod -n kafka -w
```

### Acompanhe a crição dos volumes persistentes:
```bash
kubectl get pvc -n kafka -w
```

### Para deletar um volume persistente:
```bash
kubectl delete pvc/<nome-do-pvc> -n kafka
```

## Criação de Tópicos
### Crie os tópicos necessários para sua aplicação.:
```bash
kubectl apply -f kafka-topic.yaml
```

### Verifique a configuração do tópico criado:
```bash
kubectl get KafkaTopic -n kafka
```

### Verificação dos Serviços Criados:
```bash
kubectl get svc -n kafka
```

## Uso dos Binários do Kafka para Testes
### Liste os tópicos disponíveis:
```bash
kafka-topics.sh --bootstrap-server <seu-ip-externo-do-broker>:9094 --list
```

### Teste o envio de mensagens:
```bash
kafka-console-producer.sh --broker-list <seu-ip-externo-do-broker>:9094 --topic <nome-do-topico>
```

### Teste o recebimento de mensagens:
```bash
kafka-console-consumer.sh --bootstrap-server <seu-ip-externo-do-broker>:9094 --topic <nome-do-topico> --from-beginning
```


