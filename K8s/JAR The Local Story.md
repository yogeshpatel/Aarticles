# JAR → Docker: The Local Story

## Part 1: What a JAR Cannot Carry | Part 2: Packaging the Runtime

**Series:** JAR → Docker → Kubernetes → Production  
**Lab time:** ~90 min (Part 1: ~40 min · Part 2: ~50 min)  
**Sample app:** [Retail-CRUD-API](https://github.com/yogeshpatel/Retail-CRUD-API) — a real Spring Boot REST API you clone and run. No bootstrapping needed.

---

## The mental model before you write a line of code

A JAR file contains your compiled code and declared library dependencies. It does not contain:

- The Java version it was tested on
- Environment variables it reads at startup
- The OS library versions it links against
- Certificates it needs to make outbound HTTPS calls
- The startup script that sets `JAVA_OPTS`
- The log directory it expects to exist

Every item on that list is an assumption your service makes about the machine it lands on. On your laptop those assumptions are true because you set them up. In QA, in production, on a colleague's machine — they may not be.

This is not a Docker problem. It is a **deployment contract** problem. Docker is one way to make the contract explicit. Part 1 makes the gap visible. Part 2 closes it.

---

# Part 1 — JAR: The Local Story

---

## Lab 1.1 — Clone and explore the app

```bash
# Clone the sample app
git clone https://github.com/yogeshpatel/Retail-CRUD-API
cd Retail-CRUD-API

# Understand what you have
ls -la
cat README.md

# Check the project structure
find src/main -name "*.java" | head -20
cat src/main/resources/application.yml
```

Before running anything, look at `application.yml` and notice this pattern:

```yaml
server:
  port: ${SERVER_PORT:8080}
```

> **Note:** `${SERVER_PORT:8080}` — the port is read from the environment with a fallback to 8080. This is the first step toward making an app environment-aware. Every config value that could differ between environments should follow this pattern.

Now check what environment variables the whole app reads:

```bash
grep -r 'System.getenv\|@Value\|\${' src/main/
```

Write down every variable you find. This is your deployment contract — the list of things the machine must provide that the JAR cannot carry itself.

---

## Lab 1.2 — Build and run as a plain JAR

```bash
# Build
mvn clean package -DskipTests

# See what was created — note the exact JAR name
ls -lh target/*.jar

# Set the JAR name as a variable so every command below works
export APP_JAR=$(ls target/*.jar | grep -v original | head -1)
echo "JAR: $APP_JAR"

# Run
java -jar $APP_JAR
```

In a second terminal, test the app is running. Check your repo README for the exact endpoints — every CRUD API exposes slightly different paths. At minimum, the Spring Boot Actuator health endpoint is always available:

```bash
# Health check — works on any Spring Boot app
curl -s http://localhost:8080/actuator/health | python3 -m json.tool
# Expected: {"status": "UP"}

# Check the repo README for API endpoints, then test one
# Example for a product API — adjust path to match your repo:
curl -s http://localhost:8080/api/products | python3 -m json.tool

# Or use the stats/info endpoint if available
curl -s http://localhost:8080/actuator/info | python3 -m json.tool
```

If the app started and health returns `UP` — you are ready for the audit.

---

## Lab 1.3 — The dependency audit (the uncomfortable part)

This is where you make the invisible visible. Answer each question honestly.

### Question 1: Which Java version is this JAR compiled for?

```bash
# What does your machine have?
java --version

# What Java version did Maven use to compile?
mvn help:effective-pom | grep "<source>"

# What bytecode version is inside the JAR?
javap -verbose -cp $APP_JAR \
  $(jar tf $APP_JAR | grep 'Application\|App\|Main' | head -1 | sed 's/.class//;s|/|.|g') \
  2>/dev/null | grep "major version"

# Simpler alternative — check all class files
jar tf $APP_JAR | grep ".class" | head -1 | xargs -I{} javap -verbose -cp $APP_JAR {} 2>/dev/null | grep "major version"
```

> **Bytecode major version reference:**  
> 65 = Java 21 | 61 = Java 17 | 55 = Java 11

**Now the question:** If you copy this JAR to a machine running Java 17, what happens?

```bash
# Simulate by overriding JAVA_HOME temporarily (if you have Java 17 installed)
# macOS:
JAVA_HOME=$(/usr/libexec/java_home -v 17 2>/dev/null) java -jar $APP_JAR

# Linux (adjust path to your Java 17 installation):
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 java -jar $APP_JAR
```

If compiled on Java 21, you will see `UnsupportedClassVersionError`. The JAR carries no information about the Java version it requires. **The contract is implicit.**

---

### Question 2: What environment variables does this app read?

```bash
# Already ran this above — look at the output again
grep -r 'System.getenv\|@Value\|\${' src/main/
```

Now ask honestly:

- Does every deployment environment have these set?
- Is there a `.env.example` file in this repo?
- If someone clones this at 2am and needs to run it in production — where is the variable list?

---

### Question 3: Where do logs go?

```bash
# Start the app in the background
java -jar $APP_JAR &
APP_PID=$!

# Send some traffic
curl -s http://localhost:8080/actuator/health

# Now look — where are the log files?
ls -la *.log 2>/dev/null || echo "No log files in current directory"
ls -la logs/ 2>/dev/null || echo "No logs/ directory"

# Kill the app
kill $APP_PID
```

By default, Spring Boot logs to stdout only. On a VM, stdout disappears unless someone configured a log shipper. On Docker and Kubernetes, stdout is collected automatically by the platform.

This is a design decision: **write to stdout, let the platform handle collection.**

---

### Question 4: What is the startup command?

```bash
# Right now you are running:
java -jar $APP_JAR

# In production you likely need:
java \
  -Xms512m \
  -Xmx2g \
  -XX:+UseG1GC \
  -XX:MaxRAMPercentage=75.0 \
  -Dspring.profiles.active=prod \
  -jar $APP_JAR

# Where is this command documented?
# Where is it version-controlled?
# If the answer is "in a wiki page" or "someone's head" — that is a fragility.
```

---

### Question 5: Does the app signal readiness before accepting traffic?

```bash
# Start the app
java -jar $APP_JAR &
APP_PID=$!

# How long until health returns UP?
for i in $(seq 1 20); do
  STATUS=$(curl -s http://localhost:8080/actuator/health 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin).get('status','NOT_READY'))" 2>/dev/null)
  echo "$(date +%H:%M:%S) → $STATUS"
  [ "$STATUS" = "UP" ] && break
  sleep 2
done

# Kill the app
kill $APP_PID
```

Note how long startup takes. Now ask: if a load balancer starts sending traffic on port open — before the JVM finishes initialising — what happens to those first requests?

This is exactly what Docker `HEALTHCHECK` and Kubernetes `startupProbe` solve. You will wire both in Part 2.

---

## Lab 1.4 — The failure mode inventory

Run these one at a time and observe what happens:

```bash
# Failure 1: port already in use
# Bind port 8080 in another terminal
nc -l 8080 &
java -jar $APP_JAR
# Result: Application fails to start
# Question: Does the error message tell ops what to do?
kill %1

# Failure 2: missing required environment variable
# Edit application.yml — change server.port: ${SERVER_PORT:8080}
# to server.port: ${SERVER_PORT}   (remove the default)
# Then run:
java -jar $APP_JAR
# Result: Application fails — but is the error obvious?

# Restore application.yml before continuing
```

---

## What you learned in Part 1

|JAR carries|JAR does NOT carry|
|---|---|
|Compiled bytecode|Java version requirement|
|Library dependencies (shaded)|OS-level library versions|
|Default config values|Environment-specific config|
|Startup class|Startup flags (`-Xmx`, `-D`)|
|Log format code|Log destination / collection|
|Health endpoint code|Health check wiring to infra|
|Application code|Signal: "I am ready for traffic"|

**The gap between what a JAR carries and what a running service needs is exactly what Docker makes explicit.**

---

## Self-check questions — Part 1

1. If you handed this JAR to a colleague with no instructions, what is the minimum they need to know to run it correctly in production?
2. Your ops team needs to roll back to yesterday's version at 2am. Is the startup command stored anywhere reliable?
3. Your JAR was compiled on Java 21. The production VM was patched last week to Java 17. How do you find out before the incident?
4. The app took 15 seconds to reach `UP` in Lab 1.3 Question 5. A load balancer with no health check starts routing traffic at second 5. What is the user experience during those 10 seconds?

---

# Part 2 — Docker: Packaging the Runtime

## Repeatability is the product

The implicit contract from Part 1 was: "this JAR works on a machine with the right Java version, the right env vars, and the right startup command."

Docker makes that contract a file — `Dockerfile` — that lives in your repository, goes through code review, and produces an identical artifact on every machine that builds it.

```
Docker image = JAR + Java runtime + startup command + health check + non-root user
```

When the image works in dev, it works in prod. Not because environments are the same — they are not. Because the image carries everything it needs.

---

## Lab 2.1 — The naive Dockerfile (deliberately wrong)

Start here so every improvement in 2.2 has a clear reason.

```dockerfile
# Dockerfile.naive — DO NOT USE IN PRODUCTION
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y openjdk-21-jre

COPY target/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

Build and measure:

```bash
# Build the JAR first
mvn clean package -DskipTests

docker build -f Dockerfile.naive -t retail-api:naive .

# How big?
docker images retail-api:naive
# Expect: 500–700 MB — write this number down
```

Now find the problems:

```bash
# Problem 1: Who runs the process?
docker run --rm retail-api:naive whoami
# Output: root
# If this container is compromised, the attacker has root inside it.

# Problem 2: No health check
docker inspect retail-api:naive | grep -A5 Healthcheck
# Output: "Test": []
# Docker has no idea if your app is actually healthy or just running.

# Problem 3: No startup readiness signal
# The container starts — but is the JVM ready?
# Traffic arrives on port open, not on app ready.

# Problem 4: JVM flags are not version-controlled
# java -jar app.jar uses JVM defaults.
# Your -Xmx, -XX:+UseG1GC from Part 1 are gone. They live nowhere.
```

---

## Lab 2.2 — The production Dockerfile (multi-stage)

Multi-stage builds solve the build/runtime split: Stage 1 compiles using a full JDK. Stage 2 runs using a minimal JRE. Build tools never reach the final image.

```dockerfile
# Dockerfile
# ─── Stage 1: Build ──────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy dependency descriptor first — Docker layer cache skips
# this expensive step if pom.xml has not changed
COPY pom.xml .
RUN mvn dependency:go-offline -q 2>/dev/null || true

# Copy source and build
COPY src ./src
RUN mvn clean package -DskipTests -q

# Extract Spring Boot layers for optimised rebuild caching
RUN java -Djarmode=layertools \
    -jar target/*.jar extract \
    --destination target/extracted

# ─── Stage 2: Runtime ────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Non-root user — never run production services as root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy Spring Boot layers: least-changed first, most-changed last
# Dependencies layer almost never changes → cached on every redeploy
COPY --from=builder --chown=appuser:appgroup \
    /build/target/extracted/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup \
    /build/target/extracted/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup \
    /build/target/extracted/snapshot-dependencies/ ./
# Application layer changes on every code change → always rebuilt
COPY --from=builder --chown=appuser:appgroup \
    /build/target/extracted/application/ ./

USER appuser

EXPOSE 8080

# Health check wired into the image — answers Part 1 Question 5
# start-period gives the JVM time to initialise before retries begin
HEALTHCHECK --interval=15s --timeout=3s --start-period=40s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# JVM flags are now version-controlled — not in a wiki, not in someone's head
ENTRYPOINT ["java", \
    "-XX:MaxRAMPercentage=75.0", \
    "-XX:+UseG1GC", \
    "-XX:+DisableExplicitGC", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

Build and compare:

```bash
docker build -t retail-api:v1 .

# Compare sizes
docker images | grep retail-api
# naive:  ~600MB
# v1:     ~180MB  — JRE-only Alpine strips ~420MB

# Verify non-root
docker run --rm retail-api:v1 whoami
# Output: appuser  ✅

# Verify health check is wired into the image
docker inspect retail-api:v1 | grep -A 8 '"Healthcheck"'
# Should show your HEALTHCHECK command  ✅
```

---

## Lab 2.3 — Run it and validate every Part 1 gap is closed

```bash
# Run the container
docker run -d \
  --name retail-api \
  -p 8080:8080 \
  -e SERVER_PORT=8080 \
  -e SPRING_PROFILES_ACTIVE=local \
  retail-api:v1

# Watch health status update — starts as "starting", moves to "healthy"
watch docker inspect retail-api --format='{{.State.Health.Status}}'
# starting → healthy  (takes up to 40s per start-period)

# Once healthy — test the app
curl -s http://localhost:8080/actuator/health | python3 -m json.tool
# {"status":"UP"}  ✅

# Inspect who is running what
docker exec retail-api whoami          # → appuser  ✅
docker exec retail-api java --version  # → openjdk 21  ✅
docker exec retail-api ps aux          # → java owned by appuser  ✅

# Cleanup
docker rm -f retail-api
```

**Every Part 1 gap is now closed:**

|Part 1 implicit assumption|Part 2 explicit declaration|
|---|---|
|Java 21 on the host|`FROM eclipse-temurin:21-jre-alpine`|
|`java -jar app.jar` startup|`ENTRYPOINT [...]` in Dockerfile|
|JVM flags in someone's head|`-XX:MaxRAMPercentage=75.0` in ENTRYPOINT|
|Run as root by default|`USER appuser`|
|No readiness signal|`HEALTHCHECK` with `start-period=40s`|
|Startup command undocumented|Dockerfile is version-controlled|

---

## Lab 2.4 — Docker Compose: wire in an external dependency

Real services do not run alone. A retail API typically needs a database. In Part 1 you probably ran with an in-memory H2 database. Compose lets you replace that with a real database and a mock for any other external service — without installing anything extra on your machine.

### Step 1: Create a WireMock stub for an external pricing service

This simulates any downstream HTTP dependency your app might call — a pricing engine, a discount service, a product catalogue. No code changes to the app. Just Docker Compose wiring.

```bash
mkdir -p wiremock/mappings
```

```json
// wiremock/mappings/pricing-stub.json
{
  "request": {
    "method": "GET",
    "urlPattern": "/pricing/.*"
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "body": "{\"discount\": 10, \"currency\": \"USD\"}"
  }
}
```

### Step 2: Write docker-compose.yml

```yaml
# docker-compose.yml
services:

  retail-api:
    build: .
    ports:
      - "8080:8080"
    environment:
      SERVER_PORT: "8080"
      SPRING_PROFILES_ACTIVE: "local"
      # Any downstream service URLs go here — app reads from env
      PRICING_SERVICE_URL: "http://pricing-service:8080"
    depends_on:
      pricing-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 3s
      start_period: 40s
      retries: 3
    restart: unless-stopped

  pricing-service:
    image: wiremock/wiremock:3.3.1
    ports:
      - "9090:8080"
    volumes:
      - ./wiremock:/home/wiremock
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/__admin/health"]
      interval: 10s
      timeout: 3s
      start_period: 10s
      retries: 3
```

### Step 3: Start the full stack

```bash
mvn clean package -DskipTests

docker compose up --build
```

In a second terminal:

```bash
# Confirm the app is healthy
curl -s http://localhost:8080/actuator/health | python3 -m json.tool
# Expected: {"status": "UP"}

# Confirm the mock pricing service is responding
curl -s http://localhost:9090/pricing/product-123 | python3 -m json.tool
# Expected: {"discount": 10, "currency": "USD"}

# Add a new mock stub dynamically without restarting
curl -s -X POST http://localhost:9090/__admin/mappings \
  -H "Content-Type: application/json" \
  -d '{
    "request": {"method":"GET","url":"/pricing/vip"},
    "response": {
      "status": 200,
      "body": "{\"discount\": 30, \"currency\": \"USD\"}",
      "headers": {"Content-Type": "application/json"}
    }
  }'

curl -s http://localhost:9090/pricing/vip | python3 -m json.tool
# Expected: {"discount": 30, "currency": "USD"}

# Shut down cleanly
docker compose down
```

**What this demonstrates:** Every developer on your team can now run the full stack — app plus all external dependencies — with one command. No database to install. No external service to call. No "it works on my machine."

---

## Lab 2.5 — Security and image hygiene

```bash
# Scan for CVEs
# Option A: Docker Scout (built into Docker Desktop)
docker scout cves retail-api:v1

# Option B: Trivy (open source, free)
brew install trivy      # macOS
trivy image retail-api:v1

# Verify no secrets are inside the image
docker history retail-api:v1
# Look at ENV layers — no passwords, tokens, or API keys should appear

# Confirm image filesystem
docker run --rm retail-api:v1 ls -la /app
docker run --rm retail-api:v1 cat /etc/passwd | grep appuser

# Attempt to write as root — should be blocked in a hardened setup
docker run --rm --user root retail-api:v1 touch /app/root-test 2>&1
```

---

## Lab 2.6 — Layer cache: why build order matters for CI speed

```bash
# Cold build — all layers from scratch
time docker build -t retail-api:cache-test .

# Make a code-only change (no dependency change)
echo "// cache test" >> src/main/java/$(find src/main/java -name "*.java" | head -1 | sed 's|src/main/java/||')

# Rebuild — watch what is CACHED vs what rebuilds
time docker build -t retail-api:cache-test .
```

With the layered Spring Boot approach from Lab 2.2:

- `dependencies/` layer → **CACHED** (no pom.xml change)
- `spring-boot-loader/` layer → **CACHED**
- `snapshot-dependencies/` layer → **CACHED**
- `application/` layer → **rebuilt** (only this — your code change)

In a CI pipeline processing 50 builds per day, the difference between rebuilding all layers and rebuilding only the application layer is **minutes per build**.

---

## What you now have

```
Retail-CRUD-API/
├── Dockerfile                  ✅ multi-stage, non-root, health check, JVM flags
├── docker-compose.yml          ✅ app + mock pricing service
└── wiremock/mappings/
    └── pricing-stub.json       ✅ mock stubs version-controlled alongside code
```

---

## Self-check questions — Part 2

1. The `COPY pom.xml` + `RUN mvn dependency:go-offline` step happens before `COPY src`. Why does this order matter for Docker layer caching?
2. What is the difference between `CMD` and `ENTRYPOINT` in a Dockerfile? Which is correct for a production Java service and why?
3. The `HEALTHCHECK` has `start-period=40s`. What happens if you set this to 5s on an app that takes 20s to start? What is the production symptom?
4. A colleague says "just raise `MaxRAMPercentage` to 90 to give the JVM more memory." What is the risk and what would you check first?

---

## What comes next

**Article 2: Kubernetes — The Production Move**

Your image is production-shaped. Now you will deploy it to Kubernetes — adding Deployments, Services, ConfigMaps, Secrets, readiness and liveness probes, and a Horizontal Pod Autoscaler that scales automatically under load.

The key difference: Docker runs containers. Kubernetes keeps them running.