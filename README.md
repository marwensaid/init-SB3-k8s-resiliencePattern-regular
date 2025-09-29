# init-SB3-k8s-resiliencePattern-regular


# 📝 TD – Résilience des microservices Spring Boot sur Kubernetes

## 🎯 Objectifs

* Comprendre les enjeux de la résilience dans les architectures microservices.
* Découvrir les patterns Retry, Timeout et Circuit Breaker.
* Mettre en œuvre ces patterns dans une application Spring Boot (avec **Resilience4j**).
* Déployer et tester le comportement sur Kubernetes.

---

## 1. 📖 Notions de cours

### 🔹 Qu’est-ce que la résilience ?

La **résilience logicielle** est la capacité d’un système à **continuer à fonctionner malgré des pannes partielles**, des délais de réponse élevés ou des défaillances réseau.

Sans résilience, un petit incident (ex : une API lente) peut provoquer un **effet domino** et faire tomber tout le système.

---

### 🔹 Les patterns de résilience

1. **Retry (réessai automatique)**

   * Tentative de relancer une requête après un échec.
   * Utile pour les erreurs temporaires (réseau, surcharge).
   * ⚠️ Attention à ne pas surcharger davantage un service déjà en panne.

2. **Timeout (délai maximum)**

   * Définit un temps limite pour attendre la réponse d’un service.
   * Évite de bloquer un appel indéfiniment.

3. **Circuit Breaker (disjoncteur logiciel)**

   * Coupe les appels vers un service en échec répété.
   * Permet d’éviter l’épuisement des ressources en détectant rapidement qu’un service est indisponible.
   * Inspiré du **fusible électrique** 🔌.

4. **Fallback (plan B)**

   * Réponse de secours (par ex. renvoyer des données en cache ou un message par défaut).

---

### 🔹 Spring Boot et Resilience4j

* **Resilience4j** est une librairie Java légère pour implémenter Retry, Timeout, Circuit Breaker et Fallback.
* Intégrée facilement avec **Spring Boot** (via annotations).

---

## 2. ⚙️ Mise en place d’un projet résilient

### Architecture du TD

* **Service A (gateway)** : expose une API `/api/data` qui appelle le **Service B**.
* **Service B (backend)** : simule parfois des lenteurs ou des erreurs (pour tester Retry et Circuit Breaker).

---

### Exemple Service B (simuler des erreurs)

```java
@RestController
@RequestMapping("/backend")
public class BackendController {

    private final Random random = new Random();

    @GetMapping("/data")
    public String getData() throws InterruptedException {
        int n = random.nextInt(10);

        if (n < 3) {
            throw new RuntimeException("Erreur simulée !");
        } else if (n < 6) {
            Thread.sleep(3000); // Simule un service lent
        }

        return "Données OK (n=" + n + ")";
    }
}
```

---

### Exemple Service A (client résilient avec Resilience4j)

```java
@RestController
@RequestMapping("/api")
public class ApiController {

    private final RestTemplate restTemplate;

    public ApiController(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }

    @GetMapping("/data")
    @Retry(name = "backendRetry", fallbackMethod = "fallbackResponse")
    @CircuitBreaker(name = "backendCircuitBreaker", fallbackMethod = "fallbackResponse")
    @TimeLimiter(name = "backendTimeout")
    public CompletableFuture<String> getData() {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.getForObject("http://service-b:8080/backend/data", String.class)
        );
    }

    public CompletableFuture<String> fallbackResponse(Throwable t) {
        return CompletableFuture.completedFuture("⚠️ Service B indisponible, réponse fallback !");
    }
}
```

---

### Configuration `application.yml`

```yaml
resilience4j:
  retry:
    instances:
      backendRetry:
        maxAttempts: 3
        waitDuration: 500ms

  circuitbreaker:
    instances:
      backendCircuitBreaker:
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowSize: 10

  timelimiter:
    instances:
      backendTimeout:
        timeoutDuration: 2s
```

---

## 3. 🚀 Déploiement sur Kubernetes

### Dockerfile (Service A et B)

```dockerfile
FROM eclipse-temurin:17-jdk
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

### Manifeste Kubernetes Service B (`service-b.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      containers:
        - name: service-b
          image: service-b:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: service-b
spec:
  selector:
    app: service-b
  ports:
    - port: 8080
      targetPort: 8080
```

### Manifeste Kubernetes Service A (`service-a.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      containers:
        - name: service-a
          image: service-a:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: service-a
spec:
  selector:
    app: service-a
  ports:
    - port: 8080
      targetPort: 8080
  type: LoadBalancer
```

---

## 4. 🧪 Tests pratiques (en TD)

1. **Déployer Service B seul**

   ```bash
   kubectl apply -f service-b.yaml
   kubectl get pods
   kubectl port-forward svc/service-b 8080:8080
   curl http://localhost:8080/backend/data
   ```

2. **Déployer Service A**

   ```bash
   kubectl apply -f service-a.yaml
   kubectl get svc
   ```

3. **Tester les appels**

   * Accéder à `http://<EXTERNAL_IP>:8080/api/data`
   * Observer que :

     * Parfois l’appel réussit.
     * Parfois il tombe en **timeout**.
     * Parfois le **circuit breaker** coupe les appels.
     * Le **fallback** s’active.

---

## 5. ✅ Bilan 

* **Retry** permet de gérer des erreurs temporaires.
* **Timeout** protège contre des services trop lents.
* **Circuit Breaker** évite l’effet domino.
* **Fallback** garantit une réponse minimale.

👉 Ces patterns sont indispensables pour des microservices Cloud-native robustes.

---
