1. Deploy

```bash 
cd docker/
```

```bash 
docker compose -p mini-pubsub-service up -d
```

2. Spring Boot Integration

```java
<dependency>
    <groupId>io.github.hoanggm</groupId>
    <artifactId>minipubsub-client</artifactId>
    <version>1.0.2</version>
</dependency>
```

```java
mini.pubsub.server.cluster=127.0.0.1:9801,127.0.0.1:9802,127.0.0.1:9803
mini.pubsub.server.connect-timeout-ms=5000
mini.pubsub.server.auto-reconnect=true
```

- Config

```java
@Configuration
public class MiniPubSubConfig {
    @Value("${mini.pubsub.server.cluster}")
    private String cluster;

    @Value("${mini.pubsub.server.connect-timeout-ms:5000}")
    private int connectTimeoutMs;

    @Value("${mini.pubsub.server.auto-reconnect:true}")
    private boolean autoReconnect;

    @Bean(destroyMethod = "close")
    public PubSubClient pubSubClient() throws InterruptedException {
        PubSubClient client = new PubSubClient(
                new ClientConfig(cluster, connectTimeoutMs, autoReconnect)
        );

        client.connect();
        return client;
    }
}
```

```java
@Component
public class MiniPubSubListenerProcessor implements BeanPostProcessor {

    private final PubSubClient client;

    public MiniPubSubListenerProcessor(PubSubClient client) {
        this.client = client;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        for (Method method : bean.getClass().getDeclaredMethods()) {
            if (method.isAnnotationPresent(PubSubListener.class)) {
                PubSubListener annotation = method.getAnnotation(PubSubListener.class);
                String topic = annotation.topic();

                client.subscribe(topic, (t, payload) -> {
                    try {
                        method.setAccessible(true);
                        method.invoke(bean, (Object) payload);
                    } catch (Exception ignored) {
                    }
                });
            }
        }
        return bean;
    }
}
```

- Publisher

```java
 public int publish(String message) {
    pubSubClient.publish("notifications", message);
    return 1;
}
```

- Listener

```java
@PubSubListener(topic = "notifications")
public void onMessage(byte[] payload) {
    String data = new String(payload, StandardCharsets.UTF_8);
    System.out.println(data);
}
```