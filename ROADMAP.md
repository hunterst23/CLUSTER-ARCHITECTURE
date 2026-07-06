# HomeLab Dashboard – Roadmap

Inkrementální učební plán: každá fáze přidá novou vrstvu stacku a je samostatně funkční a nasaditelná.

## Cílový výsledek

Webová aplikace pro monitoring domácího Kubernetes clusteru:
- seznam K8s nodes + podů (stav, CPU, paměť)
- metriky z Prometheus (grafy CPU, memory, teplota RPi)
- logy z Loki
- metadata zařízení v PostgreSQL (vlastní poznámky)
- Redis cache (Prometheus dotazy jsou drahé)

## Architektura

```
apps/
  homelab-api/          ← Spring Boot (Java 21)
  homelab-ui/           ← Angular
k8s/
  homelab-api/          ← Deployment, Service, Ingress
  homelab-ui/           ← Deployment, Service, Ingress
docker-compose.yml      ← lokální dev (api + ui + postgres + redis)
```

---

## Fáze 1 – Spring Boot REST API ✅

**Naučíš se:** Java projekt, Maven, Spring Boot, REST kontroler, JSON

Cíl: endpoint `GET /api/nodes` vrací hardcoded list RPi nodes jako JSON.

```
apps/homelab-api/
  src/main/java/dev/projektil/homelab/
    HomelabApiApplication.java    ← vstupní bod
    node/
      NodeRole.java               ← enum CONTROL_PLANE / WORKER
      NodeDto.java                ← Java 21 record
      NodeController.java         ← @RestController, @GetMapping
  src/main/resources/
    application.properties
  pom.xml
  Dockerfile                      ← multi-stage build
```

**Koncepty:** anotace, IoC (dependency injection), HTTP metody, JSON serializace, multi-stage Docker build.

**Ověření:**
```bash
cd apps/homelab-api && mvn spring-boot:run
curl http://localhost:8080/api/nodes
```

---

## Fáze 2 – Angular frontend ✅

**Naučíš se:** Angular komponenty, HttpClient, TypeScript, Angular Material

Cíl: stránka se seznamem nodes načtená z Fáze 1 API.

```
apps/homelab-ui/
  src/app/
    nodes/
      nodes.component.ts
      nodes.component.html
    app.config.ts        ← HttpClient provider
```

**Koncepty:** komponenty, template binding `{{ }}`, `*ngFor`, services, observables (RxJS).

---

## Fáze 3 – Docker Compose ✅

**Naučíš se:** docker-compose, sítě mezi kontejnery, environment variables

Cíl: `docker compose up` spustí celý stack lokálně.

```
docker-compose.yml       ← services: api, ui, postgres, redis
apps/homelab-ui/Dockerfile
```

**Koncepty:** volumes, healthcheck, závislosti mezi službami.

---

## Fáze 4 – PostgreSQL ✅

**Naučíš se:** Spring Data JPA, entity, migrace (Flyway), repozitáře

Cíl: uložit vlastní poznámky ke každému node do DB.

```
apps/homelab-api/src/main/java/dev/projektil/homelab/
  device/
    Device.java               ← @Entity
    DeviceRepository.java     ← JpaRepository
    DeviceService.java
    DeviceController.java
  resources/
    db/migration/V1__init.sql ← Flyway
```

**Koncepty:** ORM, SQL, transakce, REST CRUD (GET/POST/PUT/DELETE).

---

## Fáze 5 – Prometheus integrace ✅

**Naučíš se:** Spring RestClient, DTO mapování, PromQL

Cíl: Spring Boot volá Prometheus HTTP API a vrací aktuální CPU/memory/teplotu RPi.

```
apps/homelab-api/src/main/java/dev/projektil/homelab/
  prometheus/
    PrometheusClient.java     ← RestClient → http://prometheus:9090/api/v1/query
    MetricsController.java    ← GET /api/metrics/{nodeName}
    MetricsDto.java
```

**Koncepty:** HTTP klient, parsing JSON, PromQL dotazy.

---

## Fáze 6 – Redis cache ✅

**Naučíš se:** Spring Cache, @Cacheable, TTL

Cíl: Prometheus dotazy se cachují na 30 sekund.

```
apps/homelab-api/src/main/java/dev/projektil/homelab/
  config/
    RedisConfig.java    ← CacheManager s TTL
```

**Koncepty:** cache invalidace, serializace, Redis datové struktury.

---

## Fáze 7 – Angular metriky ✅

**Naučil ses:** Angular `@Input()`, RxJS polling, `async pipe`, `mat-progress-bar`, dynamický binding

Cíl: každá node karta zobrazuje CPU %, RAM %, teplotu z Prometheus; automaticky se obnovuje každých 30 s.

```
apps/homelab-ui/src/app/nodes/
  metrics.model.ts                    ← interface NodeMetrics (ip, cpuPercent, memoryPercent, temperatureCelsius)
  metrics.service.ts                  ← getMetrics(ip): Observable<NodeMetrics> → GET /api/metrics/{ip}
  node-card/
    node-card.component.ts            ← @Input() node, interval(30s)+startWith+switchMap
    node-card.component.html          ← mat-progress-bar CPU/RAM/Temp + | number:'1.0-0'
    node-card.component.scss          ← grid layout metriky, barevné chipy
```

**Klíčové koncepty:**
- `@Input()` – předávání dat z rodičovské do child komponenty
- `interval(30_000).pipe(startWith(0), switchMap(...))` – polling pattern: první dotaz ihned, pak každých 30 s
- `switchMap` – ruší předchozí HTTP požadavek při novém ticku (žádné race conditions)
- `async pipe` – automatický subscribe + unsubscribe při zničení komponenty (žádný memory leak)
- `| number:'1.0-0'` – formátování čísla na celé číslo
- `[color]="tempColor(temp)"` – dynamický Material color binding (`primary` / `accent` / `warn`)

**Barevná škála teploty:**
- `primary` (modrá) – < 60 °C – normální provoz
- `accent` (žlutá) – 60–79 °C – zvýšená zátěž
- `warn` (červená) – ≥ 80 °C – přehřívání

**Ověření:**
```bash
# Musí běžet Prometheus port-forward + backend
curl http://localhost:8080/api/metrics/192.168.10.10
# → {"ip":"...","cpuPercent":...,"memoryPercent":...,"temperatureCelsius":...}
# V prohlížeči: progress bary se zobrazí ihned a obnoví se každých 30s
```

---

## Fáze 8 – Kubernetes nasazení ✅

**Naučil ses:** K8s manifesty, Namespace, Deployment, Service, Ingress, ConfigMap, Secret, PVC, Kustomize, resource limits, readinessProbe

Cíl: `kubectl apply -k k8s/` nasadí celý dashboard stack na RPi cluster.

```
k8s/
  kustomization.yaml                  ← seznam všech resources; kubectl apply -k k8s/
  namespace.yaml                      ← namespace: homelab
  postgres/
    pvc.yaml                          ← PVC 2Gi (MicroK8s storage addon)
    secret.yaml                       ← POSTGRES_DB/USER/PASSWORD (base64)
    deployment.yaml                   ← postgres:16-alpine + readinessProbe pg_isready
    service.yaml                      ← ClusterIP:5432
  redis/
    deployment.yaml                   ← redis:7-alpine + readinessProbe redis-cli ping
    service.yaml                      ← ClusterIP:6379
  homelab-api/
    configmap.yaml                    ← DATASOURCE_URL, REDIS_HOST, PROMETHEUS_URL
    secret.yaml                       ← DATASOURCE_USERNAME/PASSWORD (base64)
    deployment.yaml                   ← readinessProbe /api/nodes, initialDelay 60s, resource limits
    service.yaml                      ← ClusterIP:8080
  homelab-ui/
    deployment.yaml                   ← nginx serving Angular SPA, resource limits
    service.yaml                      ← ClusterIP:80
    ingress.yaml                      ← homelab.local /api→homelab-api:8080, /→homelab-ui:80
```

**Klíčové koncepty:**
- `Namespace` – izolace všech resources od ostatního cluster workloadu
- `ConfigMap` – nekritická konfigurace (URL, porty); env přes `envFrom: configMapRef`
- `Secret` – citlivé hodnoty jako base64; `echo -n "heslo" | base64`
- `PersistentVolumeClaim` – trvalé úložiště pro postgres; vyžaduje MicroK8s `storage` addon
- `readinessProbe` – K8s nepošle traffic na pod dokud není probe úspěšná
- `resources.requests` – kolik scheduler rezervuje při plánování na node
- `resources.limits` – hard strop; překročení paměti = OOMKill
- `Ingress` s `kubernetes.io/ingress.class: public` – MicroK8s specifické
- Krátké DNS (`postgres:5432`) funguje v rámci stejného namespace; cross-namespace vyžaduje FQDN
- Kustomize – jednopříkazový deploy celého stacku bez helm

**Nginx routing po prostředích:**
- `npm start` – proxy.conf.json → localhost:8080
- `docker compose` – nginx.dev.conf (volume mount) s proxy_pass → homelab-api:8080
- Kubernetes – Ingress Controller přebírá /api/ routing; nginx.conf v image jen SPA

**Proč nginx.conf neobsahuje proxy_pass pro K8s:**
nginx řeší upstream hostname staticky při startu. Pokud K8s DNS ještě nepropagoval Service,
nginx selže s `host not found`. Řešení: routing na Ingress úrovni, nginx servuje jen soubory.

**Build a deploy workflow:**
```bash
# ARM64 image (cluster je RPi ARM64)
docker buildx build --platform linux/arm64 -t hunterst23/homelab-api:latest --push apps/homelab-api/
docker buildx build --platform linux/arm64 -t hunterst23/homelab-ui:latest  --push apps/homelab-ui/

kubectl apply -k k8s/
kubectl get pods -n homelab -w

# DNS (jednorázově na Mac)
echo "192.168.10.242 homelab.local" | sudo tee -a /etc/hosts
# → http://homelab.local
```

**Prometheus URL v K8s** (přímý přístup, bez port-forward):
`http://kube-prom-stack-kube-prome-prometheus.observability.svc.cluster.local:9090`

**Reálné startup časy na RPi ARM64:**
- `homelab-api` – 48 s (Spring Boot + Flyway + JVM warmup)
- `postgres`, `redis`, `homelab-ui` – < 50 s

**Ověření:**
```bash
curl http://homelab.local/api/nodes
# → 9 RPi nodes jako JSON ✅

kubectl run curl-test --image=curlimages/curl:latest \
  --restart=Never -n homelab --rm -i -- curl -s http://homelab-api:8080/api/nodes
# test z uvnitř clusteru bez DNS

kubectl get pods -n homelab
# NAME                        READY  STATUS   RESTARTS
# homelab-api-xxx             1/1    Running  1        ← 1 restart = postgres nebyl ready, normální
# homelab-ui-xxx              1/1    Running  0
# postgres-xxx                1/1    Running  0
# redis-xxx                   1/1    Running  0
```

**Pozorování z reálného nasazení:**
- `Ingress ADDRESS: 127.0.0.1` – normální pro MicroK8s, přístup funguje přes 192.168.10.242
- `annotation "kubernetes.io/ingress.class" is deprecated` – varování, neblokuje; v Fázi 10 opravit na `spec.ingressClassName: public`
- homelab-api se restartoval 1× – postgres ještě nebyl ready; readinessProbe to zachytila a traffic nepřišel dokud API nebylo připraveno

---

## Fáze 9 – Loki log viewer ✅

**Naučil ses:** LogQL, Loki HTTP API, Spring RestClient URI template encoding, Angular `FormsModule` + `ngModel`, `mat-tab-group`, `DatePipe`

Cíl: tab „Logy" v Angular dashboardu zobrazuje logy K8s podů z Loki s výběrem podu a tlačítkem Refresh.

```
apps/homelab-api/src/main/java/dev/projektil/homelab/loki/
  LokiResponse.java     ← nested records: Response → Data → Stream(stream labels, values[][])
  LogEntry.java         ← record: timestamp (ISO string), message
  LokiClient.java       ← RestClient → /loki/api/v1/query_range, LogQL, ns timestamp parsing
  LogsController.java   ← GET /api/logs?app=homelab-api&limit=200

apps/homelab-ui/src/app/logs/
  log.model.ts          ← interface LogEntry
  logs.service.ts       ← getLogs(app, limit) → Observable<LogEntry[]>
  logs.component.ts     ← apps[], selectedApp, load(), inject(LogsService)
  logs.component.html   ← mat-select dropdown + refresh button + monospace log výstup
  logs.component.scss   ← dark terminal theme (#1e1e1e bg, monospace font)

apps/homelab-ui/src/app/
  app.ts                ← přidán MatTabsModule + LogsComponent
  app.html              ← mat-tab-group: tab "Nodes" + tab "Logy"
```

**Klíčové koncepty:**
- `LogQL` – dotazovací jazyk Loki; `{namespace="homelab",app="homelab-api"}` vybere stream
- Loki `values` = `List<List<Object>>` kde `[0]` je timestamp v nanosekundách (String), `[1]` je log řádek
- Nanoseconds → Instant: `Instant.ofEpochSecond(nanos / 1_000_000_000L, nanos % 1_000_000_000L)`
- `direction=backward` vrátí nejnovější logy první → Java stream je reversnout pro chronologické pořadí
- `FormsModule` + `[(ngModel)]` – two-way binding pro `<mat-select>`
- `mat-tab-group` – navigace mezi Nodes a Logy bez routeru
- `DatePipe` s formátem `'HH:mm:ss.SSS'` – formátování ISO timestamp

**Kritická gotcha – LogQL a Spring RestClient:**
```java
// ❌ { } v LogQL query → Spring UriComponentsBuilder interpretuje jako URI template proměnnou
.uri(b -> b.path("/loki/api/v1/query_range").queryParam("query", query).build())
// → Loki dostane špatný dotaz → vrátí []

// ✅ Předat query jako pojmenovanou template proměnnou – Spring URL-enkóduje hodnotu
.uri("/loki/api/v1/query_range?query={query}&limit={limit}&direction=backward", query, limit)
```

**Loki service v clusteru:**
- `loki.observability.svc.cluster.local:3100`
- Promtail nastavuje labels: `namespace`, `app`, `pod`, `container`, `node_name`
- Hodnoty `app` labelu v namespace homelab: `homelab-api`, `homelab-ui`, `postgres`, `redis`

**Lokální vývoj – nutný port-forward:**
```bash
kubectl port-forward -n observability svc/loki 3100:3100
curl -s http://localhost:3100/ready    # → "ready"
curl -s "http://localhost:8080/api/logs?app=homelab-api&limit=5"
# → [{"timestamp":"2026-05-02T08:40:37.453Z","message":"Started HomelabApiApplication..."}]
```

**K8s deploy (po dokončení Fáze 8):**
Přidat `LOKI_URL` do `k8s/homelab-api/configmap.yaml` (už hotovo) a rebuildovat api image.

---

## Fáze 10 – CI/CD ✅

**Naučil ses:** GitHub Actions workflow, Docker Buildx ARM64 v CI, self-hosted runner, secrets, `paths` trigger, rollout restart

Cíl: push na `main` → build ARM64 image → push Docker Hub → kubectl rollout restart na clusteru.

```
.github/workflows/
  deploy-api.yml    ← trigger: apps/homelab-api/** nebo k8s/homelab-api/**
  deploy-ui.yml     ← trigger: apps/homelab-ui/** nebo k8s/homelab-ui/**
```

**Pipeline (oba workflows mají stejnou strukturu):**
```
push na main
  └── build-push job (ubuntu-latest – GitHub runner)
      ├── docker/login-action  (DOCKERHUB_USERNAME + DOCKERHUB_TOKEN secrets)
      ├── docker/setup-buildx-action
      └── docker/build-push-action --platform linux/arm64 → hunterst23/homelab-api:latest
  └── deploy job (self-hosted, rpi – runner na rpi-master-1)
      └── kubectl rollout restart + rollout status --timeout
```

**Klíčové koncepty:**
- `on.push.paths` – workflow se spustí jen při změně relevantních souborů
- `needs: build-push` – deploy čeká na dokončení build jobu
- `runs-on: ubuntu-latest` – GitHub cloud runner pro build (má Docker Buildx)
- `runs-on: [self-hosted, rpi]` – vlastní runner na RPi pro deploy (má přístup ke clusteru)
- `docker/build-push-action` s `platforms: linux/arm64` – cross-compile na x86 runneru
- `kubectl rollout status --timeout=180s` – čeká na úspěšný deploy nebo selže

**Nastavení self-hosted runneru (jednorázově na rpi-master-1):**
```bash
# GitHub repo → Settings → Actions → Runners → New self-hosted runner
# Vyber: Linux, ARM64, přidat label "rpi"
# Zkopíruj a spusť příkazy z GitHub UI, pak:
sudo ./svc.sh install && sudo ./svc.sh start
```

**GitHub Secrets (repo → Settings → Secrets and variables → Actions):**

| Secret | Hodnota |
|---|---|
| `DOCKERHUB_USERNAME` | `hunterst23` |
| `DOCKERHUB_TOKEN` | Docker Hub → Account Settings → Security → Access Tokens |

**Oprava deprecated ingress anotace (provedena v Fázi 10):**
```yaml
# ❌ Deprecated (použito v Fázi 8)
annotations:
  kubernetes.io/ingress.class: public

# ✅ Moderní způsob (k8s/homelab-ui/ingress.yaml)
spec:
  ingressClassName: public
```

---

## Fáze 10.5 – Lokální storage (SSD disky) ✅

**Naučil ses:** ext4 formátování, fdisk GPT partition, `/etc/fstab` auto-mount, K8s `local` PV s `nodeAffinity`

Cíl: připravit fyzické SSD disky pro K8s persistent storage; oddělit primary a backup storage na různé nody.

**Hardware:**
- 2× ADATA SE880 465 GB SSD (USB připojení na RPi)
- SSD 1 → rpi-worker-1 (192.168.10.20) – primary storage
- SSD 2 → rpi-worker-4 (192.168.10.30) – backup storage

**Co bylo uděláno:**
1. Formátování ext4 (`mkfs.ext4 -L storage-1/2 /dev/sda1`)
2. Připojení fyzicky k worker-1 a worker-4
3. Mount point vytvoření (`/mnt/storage-1`, `/mnt/storage-2`)
4. Přidání do `/etc/fstab` (UUID-based, `nofail`)
5. Adresářová struktura (`gitea/`, `registry/`, `backups/gitea|postgres|registry`)
6. K8s manifesty: `StorageClass local-storage` + 3 `PersistentVolume` s `nodeAffinity`

```
k8s/storage/
  storageclass.yaml   ← local-storage (no-provisioner, WaitForFirstConsumer, Retain)
  pv-gitea.yaml       ← 200Gi, rpi-worker-1, /mnt/storage-1/gitea
  pv-registry.yaml    ← 200Gi, rpi-worker-1, /mnt/storage-1/registry
  pv-backups.yaml     ← 400Gi, rpi-worker-4, /mnt/storage-2/backups
  kustomization.yaml
```

**Klíčové koncepty:**
- `local` StorageClass – PV přímo mapuje adresář na konkrétním nodu; není sdílené přes nody
- `nodeAffinity` – scheduler vždy naplánuje pod na správný node kde je disk fyzicky přítomen
- `WaitForFirstConsumer` – PV se neváže na PVC dokud není pod naplánován; zabrání uváznutí
- `Retain` reclaim policy – data přežijí smazání PVC; musí být manuálně vyčištěno
- UUID v fstab místo device path – `/dev/sda1` se může změnit po restartu, UUID je stabilní
- `nofail` v fstab – systém nastartuje i bez SSD (ochrana proti bootovacím problémům)

**Ověření:**
```bash
kubectl get storageclass         # → local-storage + microk8s-hostpath (default)
kubectl get pv                   # → pv-gitea, pv-registry, pv-backups: Available
```

---

## Fáze 11 – Gitea (lokální Git server) ✅

**Naučil ses:** Více deploymentů v jednom namespace, cross-service DNS v namespace, `fsGroup` vs `runAsUser`, s6-overlay init systém v Docker obrazech

Cíl: Gitea běží v K8s namespace `gitea`, data na SSD 1 (`pv-gitea`), přístupné na `gitea.local`.

```
k8s/gitea/
  namespace.yaml
  postgres-secret.yaml      ← credentials pro Gitea DB (gitea/gitea123)
  postgres-pvc.yaml         ← 10Gi microk8s-hostpath (jen metadata)
  postgres-deployment.yaml  ← postgres:16-alpine, readinessProbe pg_isready
  postgres-service.yaml     ← ClusterIP:5432 (name: postgres)
  pvc-gitea.yaml            ← 200Gi local-storage → pv-gitea (SSD 1, rpi-worker-1)
  secret.yaml               ← SECRET_KEY + DB_PASSWORD
  configmap.yaml            ← GITEA__ env vars (DB, server, security)
  deployment.yaml           ← gitea/gitea:latest, nodeSelector worker-1, fsGroup:1000
  service.yaml              ← NodePort: HTTP:3000, SSH:30022
  ingress.yaml              ← gitea.local → :3000
  kustomization.yaml
```

**Klíčové koncepty:**
- `GITEA__sekce__klic` env var formát – dvojité podtržítko = hierarchie v `app.ini`
- `GITEA__security__INSTALL_LOCK: "true"` – přeskočí web setup wizard, nutné pro automatizovaný deploy
- `fsGroup: 1000` (NE `runAsUser`) – `gitea/gitea` image startuje jako root (s6-overlay init), interně přepíná na `git` (UID 1000); `fsGroup` zajistí přístup k mounted volume
- `pvc-gitea.yaml` s `volumeName: pv-gitea` – explicitní binding na konkrétní PV (obchází risk bindnutí na pv-registry)
- `nodeSelector: kubernetes.io/hostname: rpi-worker-1` – pod musí běžet tam kde je SSD

**Kritická gotcha – chown před prvním deployem:**
```bash
# Na rpi-worker-1 před kubectl apply:
ssh ubuntu@192.168.10.20
sudo chown -R 1000:1000 /mnt/storage-1/gitea
```
Bez toho s6-svscan selže s `Permission denied` na `.s6-svscan/lock`.

**Admin user (jen první deploy):**
```bash
# runAsUser by způsobil Permission denied – musíme přes su git
kubectl exec -n gitea deploy/gitea -- su git -c \
  "gitea admin user create --username admin --password <heslo> --email admin@homelab.local --admin"
```

**Git remote:**
```bash
git remote add gitea http://admin@gitea.local/admin/PROJEKTIL.git
git push gitea main
# SSH: git clone ssh://git@192.168.10.20:30022/admin/PROJEKTIL.git
```

**Ověření:**
```bash
kubectl get pods -n gitea        # gitea + gitea-postgres: 1/1 Running
# http://gitea.local             → Gitea web UI
```

---

## Fáze 12 – Lokální Container Registry ✅

**Naučil ses:** Docker Registry HTTP API, htpasswd bcrypt autentizace, K8s `imagePullSecret`, insecure registry konfigurace (containerd hosts.toml), MetalLB LoadBalancer s fixní IP, passwordless sudo přes sudoers.d

Cíl: `registry:2` v K8s namespace `registry`, data na SSD 1 (`pv-registry`), přístupný na `192.168.10.243:5000`.

```
k8s/registry/
  namespace.yaml
  pvc-registry.yaml         ← 200Gi local-storage → pv-registry (SSD 1, worker-1)
  secret-htpasswd.yaml      ← bcrypt hash: admin:<heslo>
  secret-pullsecret.yaml    ← dockerconfigjson pro imagePullSecret v namespace registry
  deployment.yaml           ← registry:2, nodeSelector worker-1, htpasswd mount
  service.yaml              ← LoadBalancer, loadBalancerIP: 192.168.10.243, port 5000
  kustomization.yaml
```

**Klíčové koncepty:**
- `registry:2` – oficiální Docker Distribution image, ARM64 kompatibilní
- `htpasswd -Bbn` – bcrypt hash; silnější než MD5 (`-m`); generovat přes `docker run --rm httpd:2`
- `REGISTRY_AUTH=htpasswd` + `REGISTRY_AUTH_HTPASSWD_PATH` – env var konfigurace registry
- `loadBalancerIP: 192.168.10.243` – MetalLB přidělí fixní IP z poolu; registry je tak dostupný přímo na IP bez Ingress (Ingress není vhodný pro Docker registry protokol)
- `kubernetes.io/dockerconfigjson` – typ secretu pro docker credentials; musí existovat v každém namespace kde pody pullují privátní images
- `imagePullSecrets` v Deployment – K8s předá credentials containerd při pull

**Insecure registry – proč a jak:**
Docker a containerd vyžadují TLS pro registry – jedinou výjimkou je `localhost`. Pro HTTP registry na LAN je nutné explicitní povolení:

Mac Docker Desktop → Settings → Docker Engine:
```json
{"insecure-registries": ["192.168.10.243:5000"]}
```

MicroK8s containerd (na každém nodu):
```
/var/snap/microk8s/current/args/certs.d/192.168.10.243:5000/hosts.toml
```
Aplikovat: `sudo microk8s stop && sudo microk8s start`

**Passwordless sudo (nakonfigurováno při Fázi 12):**
```bash
# Jednorázově na každém nodu (ssh -t pro interaktivní heslo):
echo 'sator ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/sator
# → umožní budoucí automatizaci bez hesla
```

**Generování htpasswd:**
```bash
docker run --rm httpd:2 htpasswd -Bbn admin <heslo>
# výstup | base64 → do secret-htpasswd.yaml pole htpasswd
```

**Ověření:**
```bash
docker login 192.168.10.243:5000          # admin / <heslo>
curl -u admin:<heslo> http://192.168.10.243:5000/v2/_catalog
# → {"repositories":[]}

# Push test:
docker buildx build --platform linux/arm64 \
  -t 192.168.10.243:5000/homelab-api:latest --push apps/homelab-api/
curl -u admin:<heslo> http://192.168.10.243:5000/v2/_catalog
# → {"repositories":["homelab-api"]}
```

---

## Fáze 12.5 – cert-manager + TLS ✅

**Naučil ses:** cert-manager, ClusterIssuer, Certificate, PKI (CA chain), distribuce CA certifikátu, containerd trust store

Cíl: vlastní Certificate Authority pro homelab; registry běží přes HTTPS bez `insecure-registries`.

```
k8s/cert-manager/
  clusterissuer-selfsigned.yaml   ← bootstrap issuer (self-signed, bez CA)
  ca-certificate.yaml             ← CA cert (isCA: true) v namespace cert-manager
  clusterissuer-ca.yaml           ← homelab-ca ClusterIssuer (vydává všechny další certy)
  kustomization.yaml

k8s/registry/
  certificate.yaml                ← TLS cert pro registry (dnsNames + ipAddresses)
  deployment.yaml                 ← přidány REGISTRY_HTTP_TLS_CERTIFICATE/KEY + volume mount
```

**Klíčové koncepty:**
- **PKI chain:** `selfsigned ClusterIssuer` → vydá `homelab-ca Certificate` (isCA:true) → z toho vznikne `homelab-ca ClusterIssuer` → vydává servery certifikáty (registry, gitea, ...)
- `Certificate.spec.secretName` – cert-manager vytvoří Secret s `tls.crt` a `tls.key`; pod ho mountuje jako volume
- `duration` + `renewBefore` – cert-manager automaticky obnoví certifikát před expirací bez manuálního zásahu
- `ipAddresses` v SAN – nutné aby certifikát platil i při přístupu přes IP (ne jen hostname)
- `tcpSocket` readinessProbe – registry s TLS vrací 401 na `/v2/` bez credentials → `httpGet` probe by selhala; `tcpSocket` jen ověří otevřený port

**Distribuce CA (jednorázová operace):**
```bash
kubectl get secret homelab-ca-secret -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/homelab-ca.crt

# K8s nody: system CA store → update-ca-certificates → microk8s restart
# Mac: sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain
```

**Omezení – Docker Desktop na Macu:**
Docker Desktop běží v Linux VM, která nemá přímý přístup na LAN (192.168.10.x). `curl` z Mac hostu funguje (CA trusted), ale `docker login` volá daemon v VM. Workaround: SSH tunel. Trvalé řešení: CI/CD runner na clusteru (Fáze 13).

**Ověření:**
```bash
curl -u admin:<heslo> https://192.168.10.243:5000/v2/_catalog   # bez -k → TLS validní
```

---

## Fáze 13 – Gitea Actions + migrace CI/CD ✅

**Naučil ses:** Gitea Actions (GitHub Actions kompatibilní syntax), act_runner instalace a registrace, systemd service pro runner, K8s ClusterIP jako workaround pro systemd-resolved DNS

Cíl: push na Gitea → act_runner buildí ARM64 image → push na lokální registry → kubectl rollout restart.

**Architektura:**
```
git push → Gitea (http://10.152.183.47:3000 interně)
  → act_runner (rpi-master-1, labels: rpi,linux,arm64)
  → docker buildx --platform linux/arm64
  → 192.168.10.243:5000/homelab-api:latest
  → kubectl rollout restart deployment/homelab-api -n homelab
```

```
.gitea/workflows/
  deploy-api.yml    ← trigger: apps/homelab-api/** nebo k8s/homelab-api/**
  deploy-ui.yml     ← trigger: apps/homelab-ui/** nebo k8s/homelab-ui/**
```

**Klíčové koncepty:**
- Gitea Actions je záměrně kompatibilní s GitHub Actions – stejná YAML syntax, stejné `uses:` akce
- `runs-on: [self-hosted, rpi]` – runner se vybírá podle labelů
- Registry secrets v Gitea: repo → Settings → Actions → Secrets
- IP adresy místo hostnames v workflows – runner nemůže resolvovat `.local` domény

**Kritická gotcha – systemd-resolved na Ubuntu 24.04:**
`systemd-resolved` (`127.0.0.53`) blokuje `.local` (mDNS rezervace) i `.cluster.local` (K8s DNS).
`/etc/hosts` pro `.local` se ignoruje. Řešení: ClusterIP přímo.
```bash
# Registrace runneru – použij ClusterIP místo gitea.local
act_runner register --no-interactive \
  --instance http://10.152.183.47:3000 \
  --token <TOKEN> \
  --name rpi-master-1 \
  --labels rpi,linux,arm64

# systemd service
sudo systemctl enable --now act_runner
```

**Gitea Secrets (repo → Settings → Actions → Secrets):**

| Secret | Hodnota |
|---|---|
| `REGISTRY_USERNAME` | `admin` |
| `REGISTRY_PASSWORD` | heslo k lokálnímu registry |

**Ověření runneru:**
http://gitea.local/-/admin/runners → `rpi-master-1`: Online

**Gotchas při zprovoznění (zaznamenáno pro příště):**
- `actions/checkout@v4` vyžaduje Node.js na hostu → `sudo apt install nodejs`
- `docker/build-push-action` (BuildKit) nezdědí `/etc/docker/certs.d/` → nahradit přímým `docker build` + `docker push`
- `kubectl` není v PATH runneru → použít `microk8s kubectl`
- `imagePullSecret` musí existovat v namespace před deployem
- Containerd hosts.toml musí být HTTPS + CA cert (ne HTTP + skip_verify) aby K8s mohl pullovat z TLS registry
- `docker login` credentials v `~/.docker/config.json` jsou base64 (ne šifrované) – smazat po registraci, workflow používá secrets

**Výsledná pipeline (ověřeno funkční):**
```
git push gitea main
  → deploy-api.yml (trigger: apps/homelab-api/**)
  → actions/checkout@v4  (1s)
  → docker login 192.168.10.243:5000  (0s)
  → docker build + push  (31s)
  → microk8s kubectl set image + rollout status  (1s)
  → Úspěch ✅
```


---

## Fáze 14 – Automatické zálohy ✅

**Naučíš se:** K8s CronJob, shell scripty pro zálohy, pg_dump, Gitea backup API

Cíl: automatické noční zálohy všech klíčových dat na SSD 2 (rpi-worker-4, `/mnt/storage-2/backups`).

**Co zálohovat:**

| Zdroj | Metoda | Cíl |
|---|---|---|
| PostgreSQL (homelab) | `pg_dump` z K8s CronJob | `/mnt/storage-2/backups/postgres/` |
| PostgreSQL (Gitea) | `pg_dump` z K8s CronJob | `/mnt/storage-2/backups/gitea/` |
| Gitea repozitáře | Gitea backup API nebo `gitea dump` | `/mnt/storage-2/backups/gitea/` |
| Container Registry | `rsync` image layers | `/mnt/storage-2/backups/registry/` |

**Architektura:**
```
k8s/backup/
  namespace.yaml              ← namespace: backup
  cronjob-postgres.yaml       ← pg_dump každou noc ve 2:00
  cronjob-gitea.yaml          ← gitea dump každou noc ve 3:00
  cronjob-registry.yaml       ← rsync registry data každou noc ve 4:00
  configmap.yaml              ← skripty jako ConfigMap (mountovány jako soubory)
  secret.yaml                 ← DB credentials
  pv-backups.yaml             ← PV pro přístup k /mnt/storage-2/backups (nodeAffinity: worker-4)
  pvc-backups.yaml            ← PVC → pv-backups
```

**Klíčové koncepty:**
- `CronJob` – K8s resource pro plánované úlohy (cron syntax); vytváří Job → Pod při každém spuštění
- `successfulJobsHistoryLimit` + `failedJobsHistoryLimit` – kolik historických Jobů K8s uchovává
- `pg_dump` – PostgreSQL nástroj pro export databáze do SQL souboru
- `nodeAffinity` na backup PV – zálohovací pody musí běžet na worker-4 kde je SSD 2
- Rotace záloh – uchovávat posledních N záloh, starší mazat (find + mtime)

**Ověření:**
```bash
kubectl get cronjobs -n backup
kubectl create job --from=cronjob/postgres-backup manual-test -n backup
kubectl logs -n backup -l job-name=manual-test
ls /mnt/storage-2/backups/postgres/
```

---

## Fáze 15 – TLS pro homelab.local ✅

**Naučíš se:** cert-manager Certificate resource, Ingress TLS sekce, HTTP→HTTPS redirect

Cíl: dashboard `homelab.local` přístupný přes HTTPS (certifikát od interní CA, stejné jako registry).

**Co vzniklo:**
- `k8s/homelab-ui/certificate.yaml` – Certificate resource pro `homelab.local`, issuer `homelab-ca`
- `k8s/homelab-ui/ingress.yaml` – doplněna `tls:` sekce + anotace `ssl-redirect: "true"`
- HTTP automaticky přesměrováno na HTTPS

---

## Fáze 16 – ArgoCD (GitOps CD) ✅

**Naučíš se:** GitOps princip, ArgoCD Application CRD, deklarativní CD, sync policy, drift detection, rollback

Cíl: nahradit imperativní `kubectl set image` v CI/CD pipeline za GitOps model – ArgoCD sleduje Git repozitář a automaticky synchronizuje stav clusteru s manifesty.

**GitOps model:**
```
git push
  → Gitea Actions: docker build → docker push → aktualizuj image tag v manifestu → git commit
                                                          ↓
                                               ArgoCD detekuje změnu → sync do K8s → health check
```

**Architektura:**
```
k8s/argocd/
  kustomization.yaml
  namespace.yaml              <- namespace: argocd
  install.yaml                <- ArgoCD official manifests
  ingress.yaml                <- argocd.local (UI + API)
  certificate.yaml            <- TLS cert pro argocd.local
  apps/
    app-homelab.yaml          <- Application CR pro homelab namespace
    app-gitea.yaml            <- Application CR pro gitea namespace
    app-registry.yaml         <- Application CR pro registry namespace
    app-backup.yaml           <- Application CR pro backup namespace
```

**Změny v CI/CD workflows:**
```yaml
# PŘED (imperative):
- microk8s kubectl set image deployment/homelab-api homelab-api=... -n homelab
- microk8s kubectl rollout status ...

# PO (GitOps):
- IMAGE_TAG=git-${{ github.sha }}
- docker build -t registry/homelab-api:$IMAGE_TAG
- docker push registry/homelab-api:$IMAGE_TAG
- sed -i "s|image:.*|image: registry/homelab-api:$IMAGE_TAG|" k8s/homelab-api/deployment.yaml
- git commit -am "ci: update homelab-api image to $IMAGE_TAG"
- git push
# ArgoCD auto-sync zbytek udělá sám
```

**Klíčové koncepty:**
- `Application` CRD – ArgoCD resource definující zdroj (Git repo + path) a cíl (cluster + namespace)
- Auto-sync policy – ArgoCD automaticky aplikuje změny při detekci diffu
- Drift detection – ArgoCD hlásí pokud se stav clusteru liší od Gitu (manuální změny)
- Rollback – `argocd app rollback <app> <revision>` obnoví předchozí stav z Git history
- Health status – ArgoCD rozumí K8s resources a reportuje Healthy/Degraded/Progressing
- Kustomize nativní podpora – ArgoCD přímo spouští `kustomize build` bez nutnosti pre-generace

**Prerequisity:**
- Gitea běží a je dostupná z clusteru (Fáze 11 ✅)
- TLS pro argocd.local přes cert-manager (homelab-ca ✅)

**Co vzniklo:**

```
k8s/argocd/
  namespace.yaml              ← namespace: argocd
  certificate.yaml            ← TLS cert pro argocd.local (homelab-ca)
  ingress.yaml                ← argocd.local, backend-protocol: HTTPS
  argocd-cm-patch.yaml        ← ConfigMap: server.insecure: false
  apps/
    app-homelab.yaml          ← Application CR (path: k8s, ns: homelab)
    app-gitea.yaml            ← Application CR (path: k8s/gitea)
    app-registry.yaml         ← Application CR (path: k8s/registry)
    app-backup.yaml           ← Application CR (path: k8s/backup)

.gitea/workflows/
  deploy-api.yml              ← aktualizováno: git commit + push místo kubectl set image
  deploy-ui.yml               ← aktualizováno: image tag git-<sha>, GITEA_TOKEN secret
```

**Klíčová technická rozhodnutí:**

| Problém | Řešení |
|---|---|
| install.yaml > 262 KB (client-side apply limit) | `kubectl apply --server-side --force-conflicts` |
| `gitea.local` neresolví v ArgoCD podech | RepoURL přepsán na `http://gitea.gitea.svc.cluster.local:3000/...` |
| Repository autentizace | K8s Secret s labelem `argocd.argoproj.io/secret-type=repository` |
| CI nesmí spustit nový run po manifest commitu | `[skip ci]` v commit message |

**Aplikace a jejich stav:**

| Application | Git path | Namespace | SYNC | HEALTH |
|---|---|---|---|---|
| homelab | `k8s` | homelab | Synced | Healthy |
| gitea | `k8s/gitea` | gitea | Synced | Healthy |
| registry | `k8s/registry` | registry | Synced | Healthy |
| backup | `k8s/backup` | backup | Synced | Healthy |

**Ověření:**
```bash
kubectl get pods -n argocd
# → 7× Running (application-controller, applicationset-controller, dex-server,
#               notifications-controller, redis, repo-server, server)

kubectl get applications -n argocd
# → NAME       SYNC STATUS   HEALTH STATUS
# → backup     Synced        Healthy
# → gitea      Synced        Healthy
# → homelab    Synced        Healthy
# → registry   Synced        Healthy

# ArgoCD UI
https://argocd.local   # username: admin, password: viz argocd-initial-admin-secret
```

---

## Fáze 17 – Longhorn Storage ✅

**Naučil ses:** Distributed block storage, Helm install, dynamic provisioner, PVC namespace isolation, backup CronJob architektura

Cíl: nahradit statické `local-storage` PV a `microk8s-hostpath` za Longhorn – dynamic distributed block storage s 2 replikami.

**Co se změnilo:**

| Před | Po |
|---|---|
| `local-storage` StorageClass (no-provisioner) | `longhorn` (dynamic, výchozí) |
| 3 statické PV s nodeAffinity | žádné PV – Longhorn provisioner vytváří automaticky |
| `microk8s-hostpath` pro gitea-postgres | `longhorn` |
| backup CronJoby v namespace `backup` s hostPath | CronJoby v app namespace, mountují PVC přímo |
| `backup-storage` na hostPath worker-4 | Longhorn PVC `backup-storage` 80 Gi |
| `gitea-data` 200 Gi (local-storage) | `gitea-data` 10 Gi (longhorn) |
| `registry-data` 200 Gi (local-storage) | `registry-data` 50 Gi (longhorn) |

**Architektura backup CronJobů po migraci:**

```
namespace: backup
  cronjob-postgres  → Longhorn PVC backup-storage → pg_dump homelab + gitea DB

namespace: gitea
  cronjob-gitea-backup → PVC gitea-data (mount) → rsync SSH → worker-4:/mnt/storage-2/backups/gitea/

namespace: registry
  cronjob-registry-backup → PVC registry-data (mount) → rsync SSH → worker-4:/mnt/storage-2/backups/registry/
```

**Helm install:**
```bash
helm repo add longhorn https://charts.longhorn.io && helm repo update

# Prerekvizita open-iscsi na každém nodu
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.7.0/deploy/prerequisite/longhorn-iscsi-installation.yaml

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --values k8s/longhorn/values.yaml --version 1.7.0
```

> **MicroK8s poznámka:** `k8s/longhorn/values.yaml` musí obsahovat:
> ```yaml
> csi:
>   kubeletRootDir: /var/snap/microk8s/common/var/lib/kubelet
> ```
> Bez toho `longhorn-driver-deployer` spadne s `failed to get kubelet root dir`.

**Kopírování SSH secret do app namespace (jednorázově):**
```bash
kubectl get secret backup-ssh-key -n backup -o yaml \
  | sed 's/namespace: backup/namespace: gitea/' | kubectl apply -f -
kubectl get secret backup-ssh-key -n backup -o yaml \
  | sed 's/namespace: backup/namespace: registry/' | kubectl apply -f -
```

**Nasazení:**
```bash
kubectl apply -k k8s/gitea/
kubectl apply -k k8s/registry/
kubectl apply -k k8s/backup/
```

**Klíčové koncepty:**
- Longhorn dynamic provisioner – PVC vytvoří volume automaticky bez manuálního PV
- Repliky (2) – data jsou na 2 různých nodech → výpadek worker-1 neztratí data
- PVC jsou namespace-scoped – backup CronJob v `backup` namespace nemůže mountovat PVC z `gitea` namespace → proto jsou gitea/registry backup CronJoby přesunuty do svých namespace
- RWO concurrent mount – Longhorn dovoluje mount RWO volume z více podů na STEJNÉM nodu (app pod + backup pod)
- `Retain` reclaim policy – data přežijí smazání PVC
- MicroK8s kubeletRootDir – MicroK8s ukládá kubelet data do `/var/snap/microk8s/common/var/lib/kubelet` (nikoliv `/var/lib/kubelet`). Longhorn nedokáže cestu autodetekovat → nutno nastavit `csi.kubeletRootDir` v `values.yaml`
- PostgreSQL + Longhorn: lost+found problem – Longhorn formátuje volume ext4, čímž vznikne adresář `lost+found` v kořenu disku. PostgreSQL `initdb` odmítá inicializovat neprázdný adresář → nastavit env `PGDATA` na subdirektoru (např. `/var/lib/postgresql/data/pgdata`)
- RWO Recreate strategy – Deployment s RWO PVC smí mít `strategy: type: Recreate`. Výchozí RollingUpdate zkouší startovat nový pod na jiném nodu před ukončením starého → nový pod čeká na attach PVC → deadlock. Recreate nejprve ukončí starý pod, pak spustí nový.
- Datová migrace – source data (hostPath `/mnt/storage-1/gitea`, `/mnt/storage-1/registry`) zkopírována přes dočasný migration pod (alpine + rsync) s `nodeName: rpi-worker-1`. PostgreSQL data migrována přes `pg_dumpall` → `pg_restore`.

**Ověření:**
```bash
kubectl get storageclass
# → longhorn (default) ✅

kubectl get pvc -n gitea
# → gitea-data: Bound, longhorn ✅

kubectl get pvc -n backup
# → backup-storage: Bound, longhorn ✅

# Longhorn UI
echo "192.168.10.242 longhorn.local" | sudo tee -a /etc/hosts
# → http://longhorn.local
```

**Potíže při nasazení:**

| Problém | Příčina | Řešení |
|---|---|---|
| `longhorn-driver-deployer` CrashLoop: `failed to get kubelet root dir` | MicroK8s ukládá kubelet data do nestandardní cesty | Přidat `csi.kubeletRootDir: /var/snap/microk8s/common/var/lib/kubelet` do `values.yaml` |
| `open-iscsi` pods: `cannot create NETLINK_ISCSI socket` na worker-4, worker-5 | Kernel modul `iscsi_tcp` chybí na Ubuntu 22.04 RPi 4 s kernelem 5.15.0-1098 | `apt install linux-modules-extra-raspi` + drain + reboot + `modprobe iscsi_tcp` + persistence v `/etc/modules-load.d/iscsi.conf` |
| PVC 200 Gi nelze provisionovat: `insufficient storage` | Největší volný disk v clusteru ~109 Gi, 200 Gi se nevleze ani na jeden node | Reálné PVC velikosti: gitea 10 Gi, registry 50 Gi, backup 80 Gi |
| PostgreSQL `initdb`: `directory exists but is not empty` (lost+found) | Longhorn ext4 volume má `lost+found` v kořenu disku | Nastavit `PGDATA=/var/lib/postgresql/data/pgdata` v env |
| RollingUpdate deadlock u postgres deploymentu s RWO PVC | Nový pod schedulován na jiný node – nemůže attachovat PVC dokud staré PVC připojeno | `strategy: type: Recreate` v deployment spec |

---

## Fáze 18 – PostgreSQL HA (CloudNativePG)

**Naučíš se:** K8s Operator pattern, CRD (Custom Resource Definition), PostgreSQL streaming replication, WAL archiving

Cíl: nahradit single-instance PostgreSQL (homelab + gitea) za HA cluster spravovaný CloudNativePG operátorem. RPO klesne z 24 h (nightly dump) na minuty (kontinuální WAL archiving).

**Proč CloudNativePG:**
- Nativní K8s operator – spravuje PostgreSQL jako CRD `Cluster`
- Streaming replication out-of-the-box (primary + N standbys)
- Automatický failover při výpadku primary
- WAL archiving přímo do K8s PVC
- Lehčí alternativa k Patroni, ideální pro homelab

**Architektura:**
```
k8s/cloudnativepg/
  kustomization.yaml
  operator.yaml                  <- CNPG operator (CRD + controller)
  cluster-homelab.yaml           <- Cluster CR: 3 instance, homelab DB
  cluster-gitea.yaml             <- Cluster CR: 3 instance, gitea DB
  scheduledbackup-homelab.yaml   <- WAL archiving na backup PVC
  scheduledbackup-gitea.yaml
```

**Klíčové koncepty:**
- `Cluster` CRD – deklarativní definice PostgreSQL clusteru (instances, storage, backup)
- Streaming replication – primary posílá WAL streamy standby nodům v reálném čase
- WAL archiving – každý WAL segment archivován → obnova do libovolného bodu v čase (PITR)
- Failover – při výpadku primary automatická volba nového primary ze standby
- `ScheduledBackup` CRD – plánované base-backupy doplňující WAL archiving

**Migrace dat:**
1. Nainstalovat CNPG operator
2. Exportovat data z aktuálního postgres: `pg_dump`
3. Vytvořit `Cluster` resource (nahradí Deployment)
4. Importovat data do nového clusteru
5. Aktualizovat connection stringy v homelab-api a gitea ConfigMap

**Ověření:**
```bash
kubectl get cluster -n homelab
kubectl get pods -n homelab -l cnpg.io/cluster=homelab-postgres
# Simulace failover:
kubectl delete pod homelab-postgres-1 -n homelab
kubectl get cluster -n homelab -w
```

---

## Fáze 19 – Gitea HA

**Naučíš se:** Horizontal scaling stateful aplikace, RWX storage, session affinity, PVC migrace

Cíl: Gitea běží ve více replikách (replicas: 3) na sdíleném Longhorn RWX volume. Výpadek jednoho podu neovlivní dostupnost SCM.

**Prerequisity:** Fáze 18 (CloudNativePG pro Gitea DB) + Fáze 17 (Longhorn – pro RWX přepnout storageClass na `longhorn-rwx`)

**Změny:**
```
k8s/gitea/
  deployment.yaml   <- replicas: 1 -> 3, PVC: longhorn -> longhorn-rwx
  pvc-gitea.yaml    <- storageClassName: longhorn-rwx, accessMode: RWX
  service.yaml      <- SessionAffinity: ClientIP
```

**Klíčové koncepty:**
- Horizontal scaling stateful app – Gitea sdílí repo files přes RWX volume
- Session affinity – Gitea nemá distribuovanou session cache, sticky sessions nutné
- PVC migrace – přesun dat z local-storage PVC na Longhorn RWX PVC
- Ingress affinity – `nginx.ingress.kubernetes.io/affinity: cookie`

**Migrace PVC:**
1. Scale down Gitea na 0 replik
2. Zkopírovat data z starého PVC na Longhorn RWX PVC (rsync přes tmp pod)
3. Aktualizovat Deployment na nový PVC
4. Scale up na 3 repliky

**Ověření:**
```bash
kubectl get pods -n gitea  # 3x Running
kubectl delete pod -n gitea -l app=gitea  # kill all pods najednou
curl -I https://gitea.local  # musí stále odpovídat
```

---

## Fáze 20 – Harbor (Enterprise Container Registry)

**Naučíš se:** Helm charts, enterprise registry features, image scanning (Trivy), RBAC, replication policies

Cíl: nahradit `registry:2` za Harbor – enterprise-grade registry s HA, image scanningem a RBAC.

**Prerequisity:** Fáze 17 (Longhorn ✅ – Harbor potřebuje více PVC pro jednotlivé komponenty)

**Proč Harbor:**
- HA architektura (core, jobservice, registry, database, redis – vše separátně)
- Image scanning (Trivy) – detekce CVE v každém pushnutém image
- RBAC – různá práva pro různé projekty/týmy
- Webhooky – notifikace při push (integrace s Gitea)
- Web UI pro správu images a scan reportů

**Architektura:**
```
k8s/harbor/
  kustomization.yaml
  namespace.yaml
  helm-values.yaml    <- Harbor Helm values (persistence, ingress, TLS)
  certificate.yaml    <- TLS cert pro harbor.local
  ingress.yaml        <- harbor.local (UI) + harbor.local/v2 (registry API)
```

**Migrace images:**
1. Nainstalovat Harbor přes Helm
2. Vytvořit projekt `homelab` v Harbor UI
3. Re-tag a push existujících images na nový endpoint
4. Aktualizovat CI/CD workflows na Harbor endpoint
5. Aktualizovat imagePullSecrets ve všech Deploymentech

**Klíčové koncepty:**
- Helm chart – package manager pro K8s, Harbor má oficiální chart
- Trivy scanner – open-source CVE scanner integrovaný do Harbor
- Harbor projekt – namespace pro images s vlastním RBAC
- Replication policy – synchronizace mezi registry instancemi

**Ověření:**
```bash
kubectl get pods -n harbor
# Otevřít https://harbor.local
# Pushnout testovací image a ověřit scan report v Harbor UI
```
