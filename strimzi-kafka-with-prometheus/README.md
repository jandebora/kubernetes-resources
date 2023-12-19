# Kafka con Prometheus (Para DEV)

En esta guía se pretende hacer una sencilla instalación de Kafkaen Kubernetes (usando la versión de Zookeeper en primera instancia) con Prometheus y Grafana. Para ello, debemos tener en cuenta los siguientes requisitos:

* Cliente Kubectl
* Cliente Helm
* Comodidad administrando vía terminal

# Primeros pasos

## Instalación de Prometheus con Grafana

Existe un Helm Chart para ello muy sencillo de usar:

```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 52.1.0 -n <namespace_de_instalacion>
```

En este caso instalará 6 Pods que contienen lo siguiente:

* Prometheus Alert Manager (AlertManager y Config Reloader)
* Grafana (Grafana, Grafana Dashboards y Grafana Datasources)
* Prometheus (Prometheus y Config Reloader)
* Prometheus State Metrics
* Prometheus Stack Operator
* Prometheus Node Exporter

## Instalación de operador Strimzi

Previo a instalar el operador, hay que indicarle mediante un archivo de valores que necesitamos que nos cree los archivos de configuración de los dashboards que usaremos con Grafana. Esto se hace de la siguiente manera:

**Archivo strimzi-values.yaml**
```yaml
dashboards:
  enabled: true
  namespace: default
```

Una vez hecho esto, el siguiente paso sería instalar el operador:

```
helm install strimzi-kafka-operator strimzi/strimzi-kafka-operator -n <namespace_de_instalacion> -f strimzi-values.yaml
```

Esto creará todos los servicios necesarios para poder ejecutar las operaciones de Strimzi.

## Instalación de Kafka

### Primer paso: Archivo de configuración de métricas

Nuestro Kafka tiene que tener cierta configuración de para enviar métricas a Prometheus. Podemos obtener en su documentación y GitHub oficiales ciertos ejemplos, pero nosotros hemos usado uno más completo obtenido en el [Blog de tecnología de Piotr](https://piotrminkowski.com). Sería el siguiente:

**kafka-metrics-config.yaml**
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: kafka-metrics
  labels:
    app: kafka
data:
  kafka-metrics-config.yml: |
    lowercaseOutputName: true
    rules:
    - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), topic=(.+), partition=(.*)><>Value
      name: kafka_server_$1_$2
      type: GAUGE
      labels:
        clientId: "$3"
        topic: "$4"
        partition: "$5"
    - pattern: kafka.server<type=(.+), name=(.+), clientId=(.+), brokerHost=(.+), brokerPort=(.+)><>Value
      name: kafka_server_$1_$2
      type: GAUGE
      labels:
        clientId: "$3"
        broker: "$4:$5"
    - pattern: kafka.server<type=(.+), cipher=(.+), protocol=(.+), listener=(.+), networkProcessor=(.+)><>connections
      name: kafka_server_$1_connections_tls_info
      type: GAUGE
      labels:
        cipher: "$2"
        protocol: "$3"
        listener: "$4"
        networkProcessor: "$5"
    - pattern: kafka.server<type=(.+), clientSoftwareName=(.+), clientSoftwareVersion=(.+), listener=(.+), networkProcessor=(.+)><>connections
      name: kafka_server_$1_connections_software
      type: GAUGE
      labels:
        clientSoftwareName: "$2"
        clientSoftwareVersion: "$3"
        listener: "$4"
        networkProcessor: "$5"
    - pattern: "kafka.server<type=(.+), listener=(.+), networkProcessor=(.+)><>(.+):"
      name: kafka_server_$1_$4
      type: GAUGE
      labels:
        listener: "$2"
        networkProcessor: "$3"
    - pattern: kafka.server<type=(.+), listener=(.+), networkProcessor=(.+)><>(.+)
      name: kafka_server_$1_$4
      type: GAUGE
      labels:
        listener: "$2"
        networkProcessor: "$3"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>MeanRate
      name: kafka_$1_$2_$3_percent
      type: GAUGE
    - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*><>Value
      name: kafka_$1_$2_$3_percent
      type: GAUGE
    - pattern: kafka.(\w+)<type=(.+), name=(.+)Percent\w*, (.+)=(.+)><>Value
      name: kafka_$1_$2_$3_percent
      type: GAUGE
      labels:
        "$4": "$5"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+), (.+)=(.+)><>Count
      name: kafka_$1_$2_$3_total
      type: COUNTER
      labels:
        "$4": "$5"
        "$6": "$7"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*, (.+)=(.+)><>Count
      name: kafka_$1_$2_$3_total
      type: COUNTER
      labels:
        "$4": "$5"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*><>Count
      name: kafka_$1_$2_$3_total
      type: COUNTER
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Value
      name: kafka_$1_$2_$3
      type: GAUGE
      labels:
        "$4": "$5"
        "$6": "$7"
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Value
      name: kafka_$1_$2_$3
      type: GAUGE
      labels:
        "$4": "$5"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)><>Value
      name: kafka_$1_$2_$3
      type: GAUGE
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+), (.+)=(.+)><>Count
      name: kafka_$1_$2_$3_count
      type: COUNTER
      labels:
        "$4": "$5"
        "$6": "$7"
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*), (.+)=(.+)><>(\d+)thPercentile
      name: kafka_$1_$2_$3
      type: GAUGE
      labels:
        "$4": "$5"
        "$6": "$7"
        quantile: "0.$8"
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.+)><>Count
      name: kafka_$1_$2_$3_count
      type: COUNTER
      labels:
        "$4": "$5"
    - pattern: kafka.(\w+)<type=(.+), name=(.+), (.+)=(.*)><>(\d+)thPercentile
      name: kafka_$1_$2_$3
      type: GAUGE
      labels:
        "$4": "$5"
        quantile: "0.$6"
    - pattern: kafka.(\w+)<type=(.+), name=(.+)><>Count
      name: kafka_$1_$2_$3_count
      type: COUNTER
    - pattern: kafka.(\w+)<type=(.+), name=(.+)><>(\d+)thPercentile
      name: kafka_$1_$2_$3
      type: GAUGE
      labels:
        quantile: "0.$4"
    - pattern: "kafka.server<type=raft-metrics><>(.+-total|.+-max):"
      name: kafka_server_raftmetrics_$1
      type: COUNTER
    - pattern: "kafka.server<type=raft-metrics><>(.+):"
      name: kafka_server_raftmetrics_$1
      type: GAUGE
    - pattern: "kafka.server<type=raft-channel-metrics><>(.+-total|.+-max):"
      name: kafka_server_raftchannelmetrics_$1
      type: COUNTER
    - pattern: "kafka.server<type=raft-channel-metrics><>(.+):"
      name: kafka_server_raftchannelmetrics_$1
      type: GAUGE
    - pattern: "kafka.server<type=broker-metadata-metrics><>(.+):"
      name: kafka_server_brokermetadatametrics_$1
      type: GAUGE
```

El siguiente paso sería instalar este archivo:

```
kubectl create -f kafka-metrics-config.yaml -n <namespace_de_instalacion>
```

### Segundo paso: Instalación de Kafka de un solo broker (versión de desarrollo)

En primer lugar comentar que en esta sección, dada la fecha actual, sería conveniente hacer una instalación con Kafka KRaft. No obstante, al obtenerse algunos problemas pues el operador de Strimzi aún está puliendo detalles con esta configuración, se ha obtado por hacer una instalación "más antigua" con Zookeeper de un solo nodo. Veamos pues como instalarlo.

**kafka-persistence-single.yaml**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 3.6.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
      inter.broker.protocol.version: "3.6"
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics
          key: kafka-metrics-config.yml
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 5Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 5Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

En el anterior archivo, que usa las operaciones de Strimzi, hemos instalado lo siguiente:

* Kafka de 1 sóla réplica con:
    * Versión 3.6.0 del broker
    * 3 Listeners:
        * Interno sin seguridad en el puerto 9092
        * Interno con seguridad en el puerto 9093
    * Configuración básica de desarrollo para solo una réplica
    * Configuración de envío de métricas al archivo de configuración previamente creado
    * Persistencia muy básica de 5 Gi
* Zookeeper de una sóla réplica con:
    * Persistencia muy básica de 5 Gi
* Instalación de operadores de creación de usuarios y tópicos

Para instalarlo:

```
kubectl create -f kafka-persistence-single.yaml -n <namespace_de_instalacion>
```

### Tercer paso: PodMonitor para el envío de métricas

Con la instalación por defecto sin ningún tipo de indicación, Prometheus no va a entender que debe recopilar las métricas de Kafka. Para ello se debe de crear un recurso PodMonitor e indicarselo apropiadamente de la siguiente forma:

**kafka-resource-metrics.yaml**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: kafka-resources-metrics
  labels:
    app: kafka
    release: kube-prometheus-stack
spec:
  selector:
    matchExpressions:
      - key: "strimzi.io/kind"
        operator: In
        values: ["Kafka", "KafkaConnect", "KafkaMirrorMaker", "KafkaMirrorMaker2"]
  namespaceSelector:
    matchNames:
      - kafka
  podMetricsEndpoints:
  - path: /metrics
    port: tcp-prometheus
    relabelings:
    - separator: ;
      regex: __meta_kubernetes_pod_label_(strimzi_io_.+)
      replacement: $1
      action: labelmap
    - sourceLabels: [__meta_kubernetes_namespace]
      separator: ;
      regex: (.*)
      targetLabel: namespace
      replacement: $1
      action: replace
    - sourceLabels: [__meta_kubernetes_pod_name]
      separator: ;
      regex: (.*)
      targetLabel: kubernetes_pod_name
      replacement: $1
      action: replace
    - sourceLabels: [__meta_kubernetes_pod_node_name]
      separator: ;
      regex: (.*)
      targetLabel: node_name
      replacement: $1
      action: replace
    - sourceLabels: [__meta_kubernetes_pod_host_ip]
      separator: ;
      regex: (.*)
      targetLabel: node_ip
      replacement: $1
      action: replace
```

Y creándolo en kubernetes:

```
kubectl create -f kafka-resource-metrics.yaml -n <namespace_de_instalacion>
```

### Cuarto paso: Acceso a Grafana

Ya tenemos todo listo para hacer una primera prueba. En este caso, hagamos una sencilla. 

1. Creamos un port-forward del pod de grafana en su puerto 3000.
1. Accedemos a [http://localhost:3000](http://localhost:3000)
1. Añadimos el usuario y contraseña: `admin / prom-operator`
1. Seleccionamos el Dashboard `Strimzi Kafka`
1. Et voilà
