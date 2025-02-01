# Enhancing Security: TLS Encryption & Authentication for Kafka Ingress

* Currently, the Kafka brokers are exposed unencrypted on port 9092 through the Nginx Ingress Controller, making it vulnerable to security risks such as eavesdropping and unauthorized access. 
* Below, we implement TLS encryption and client authentication to secure communication between Kafka clients and the ingress.
* NOTE: You may have to customise this approach heavily to your specific use-case and setup. Ultimately it is the process we present here that should guide you to the details.

# 🚀 Step 1: Secure Kafka Ingress with TLS

We'll set up:

* 1. TLS termination at the ingress controller to encrypt traffic from external clients.
* 2. TLS passthrough to forward the encrypted Kafka traffic to the brokers.
* 3. Mutual TLS (mTLS) to ensure Kafka clients authenticate before accessing Kafka brokers.


## Step 1.1: Generate a TLS Certificate

* Since the Ingress Controller handles external traffic, it must have a TLS certificate.

Option 1: Use Let's Encrypt (Recommended for Public Access)

Install Cert-Manager:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

* Create a Certificate Issuer:

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

* Create the TLS certificate for Kafka Ingress:

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kafka-ingress-cert
  namespace: kafka-ingress
spec:
  secretName: kafka-ingress-tls
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  commonName: kafka.example.com
  dnsNames:
    - kafka.example.com

```

* Apply:

```
kubectl apply -f kafka-ingress-cert.yaml
```

## Step 1.2: Configure Kafka Ingress to Use TLS

* Modify the Ingress ConfigMap to enable TLS passthrough:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: kafka-ingress
data:
  "9092": "kafka/bayleaf-kafka-cp-kafka-headless:9092"
```

* Modify the Ingress Configuration:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kafka-ingress
  namespace: kafka-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "TLS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
      - kafka.example.com
    secretName: kafka-ingress-tls
  rules:
  - host: kafka.example.com
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: kafka-ingress-controller
            port:
              number: 9092
```

* Apply the changes:

```
kubectl apply -f kafka-ingress.yaml
```

# 🚀 Step 2: Enable Kafka TLS and Authentication

* Next, we configure Kafka itself to require TLS authentication.

## Step 2.1: Generate Kafka TLS Certificates

* Kafka needs its own certificates for authentication.
* Generate Kafka CA Certificate:

```
openssl req -new -x509 -keyout kafka-ca-key.pem -out kafka-ca-cert.pem -days 365 -subj "/CN=Kafka-CA"
```

* Generate Kafka Broker Certificate:

```
openssl genrsa -out kafka-broker-key.pem 2048
openssl req -new -key kafka-broker-key.pem -out kafka-broker.csr -subj "/CN=kafka.example.com"
openssl x509 -req -in kafka-broker.csr -CA kafka-ca-cert.pem -CAkey kafka-ca-key.pem -CAcreateserial -out kafka-broker-cert.pem -days 365
```

* Create a Kubernetes secret for Kafka TLS:

```
kubectl create secret generic kafka-tls-secret --from-file=kafka-broker-cert.pem --from-file=kafka-broker-key.pem --from-file=kafka-ca-cert.pem -n kafka
```

## Step 2.2: Configure Kafka to Use TLS and Authentication

* Modify the Kafka ConfigMap:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-config
  namespace: kafka
data:
  server.properties: |
    listeners=SSL://0.0.0.0:9092
    advertised.listeners=SSL://kafka.example.com:9092
    ssl.keystore.location=/etc/kafka/secrets/kafka-broker-cert.pem
    ssl.keystore.password=kafka-secret
    ssl.key.password=kafka-secret
    ssl.truststore.location=/etc/kafka/secrets/kafka-ca-cert.pem
    ssl.truststore.password=kafka-secret
    ssl.client.auth=required
    security.inter.broker.protocol=SSL
```

* Modify the Kafka StatefulSet:

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  template:
    spec:
      containers:
        - name: kafka
          volumeMounts:
            - name: kafka-tls-secret
              mountPath: "/etc/kafka/secrets"
              readOnly: true
      volumes:
        - name: kafka-tls-secret
          secret:
            secretName: kafka-tls-secret
```

* Apply the Kafka changes (as applicable, depending on how you deployed kafka)

```
kubectl apply -f kafka-config.yaml
kubectl apply -f kafka-statefulset.yaml
```

# Step 3: Configure Kafka Clients to Use TLS

* Kafka clients must also be configured to use TLS when connecting.
* Create a client certificate:


```
openssl genrsa -out kafka-client-key.pem 2048
openssl req -new -key kafka-client-key.pem -out kafka-client.csr -subj "/CN=kafka-client"
openssl x509 -req -in kafka-client.csr -CA kafka-ca-cert.pem -CAkey kafka-ca-key.pem -CAcreateserial -out kafka-client-cert.pem -days 365
```

* Copy these certificates to your Kafka client machine and configure the client properties:

```
security.protocol=SSL
ssl.truststore.location=/path/to/kafka-ca-cert.pem
ssl.keystore.location=/path/to/kafka-client-cert.pem
ssl.keystore.password=kafka-secret
ssl.key.password=kafka-secret
```

* Run the Kafka client:

```
kafka-console-producer --broker-list kafka.example.com:9092 --producer.config client.properties
```


##  Final Verification

* Check if Kafka is using SSL:

```
kubectl logs -n kafka kafka-0 | grep "SSL"
```

(You should see logs indicating SSL connections.)

* Check if clients can connect securely:

```
kafka-console-producer --broker-list kafka.example.com:9092 --producer.config client.properties
```

* If the client connects, TLS is working.


