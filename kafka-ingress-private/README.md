# Troubleshooting Tips


```
kubectl auth can-i list ingressclasses --as=system:serviceaccount:kafka-ingress:default
```


## KAFKA_ADVERTISED_LISTENERS:

```
KAFKA_LISTENERS=EXTERNAL://0.0.0.0:9092
KAFKA_ADVERTISED_LISTENERS=EXTERNAL://kafka.acn-aisles.com:9092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=EXTERNAL:PLAINTEXT
```
