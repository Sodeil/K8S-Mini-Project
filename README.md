# Kubernetes Mini-Projekt  
## StatefulSet mit externem Load Balancer (HAProxy)

## Überblick

Dieses Projekt deployt eine einfache **statische Website** in Kubernetes und stellt sie **bewusst ohne Kubernetes-LoadBalancer oder Ingress** bereit.  
Der Zugriff erfolgt über einen **eigenständigen HAProxy-Pod**, der TLS terminiert und per DNS direkt mit den Pods spricht.

**Ziel:** zeigen, dass Kubernetes-Primitive gezielt eingesetzt werden können.

Das Projekt demonstriert:

- bewusste Auswahl von Kubernetes-Ressourcen  
- saubere Anbindung externer Komponenten  
- deterministisches, reproduzierbares Verhalten  
- klare Security- und Architekturentscheidungen  

---

## Architektur

    Client
    ↓ HTTPS :8443
    HAProxy (Pod, eigenständig)
    ↓ HTTP :8080
    StatefulSet (Apache x N)


---

## Komponenten & Begründung

### 1. StatefulSet (`project.yaml`)

**Warum StatefulSet und kein Deployment?**

- Jeder Apache-Pod besitzt eine **stabile Identität** (`project-0`, `project-1`, …)
- Diese Identitäten sind **per DNS auflösbar**
- HAProxy kann gezielt einzelne Pods ansprechen  
  → notwendig, da **kein Kubernetes Service als LoadBalancer** verwendet wird

**Technisch:**

- `replicas: 2`
- Jeder Pod:
  - nutzt ein gemeinsames Webroot (`emptyDir`, befüllt per Init-Container)
  - erhält eine feste Apache-Konfiguration aus einer ConfigMap
  - besitzt Liveness- und Readiness-Probes auf `/index.html`

**Security:**

- `runAsNonRoot` mit fixer UID/GID  
- alle Linux-Capabilities gedroppt  
- `seccompProfile: RuntimeDefault`

---

### 2. Headless Service (`dns-service.yaml`)

**Warum `clusterIP: None`?**

- kein virtueller Load Balancer
- Kubernetes DNS liefert **direkt die Pod-IPs**

HAProxy übernimmt selbst:

- Round-Robin
- Healthchecks
- dynamisches Hinzufügen/Entfernen von Pods

---

### 3. ConfigMap (`project-config.yaml`)

Enthält sämtliche Konfiguration:

- URL der HTML-Datei
- HAProxy-Config-Template
- Apache `httpd.conf`

**Warum so?**

- klare Trennung von Code & Konfiguration
- Änderungen an Website, Apache oder HAProxy  
  → **kein Image-Build**, nur `kubectl apply`

---

### 4. Init-Container (loadbalancer.yaml)

#### Apache Init-Container

- lädt HTML per `curl`
- legt sie im Shared Volume (`webroot`) ab

**Begründung:**

- Apache-Image bleibt unverändert
- klare Trennung von Build- und Laufzeit
- reproduzierbar, deterministisch

#### HAProxy Init-Container

Aufgaben:

- Installation minimaler Tools (`curl`, `openssl`, `gettext`)
- Laden des HAProxy-Config-Templates
- automatische Ermittlung des Cluster-DNS aus `/etc/resolv.conf`
- Rendering der Config via `envsubst`
- Erzeugung eines Self-Signed TLS-Zertifikats

**Warum kein Custom-Image?**

- volle Transparenz im YAML
- kein versteckter Build-Step
- reproduzierbar in jedem Cluster

---

### 5. HAProxy Pod (`loadbalancer.yaml`)

**Warum ein eigener Pod und kein Service?**

Explizite Vorgabe der Aufgabe:  
> *kein Kubernetes Service*

**HAProxy:**

- hört auf `:8443` (HTTPS)
- terminiert TLS selbst
- Round-Robin Load-Balancing
- DNS-basierte Service Discovery
- automatische Entfernung fehlerhafter Pods

**DNS-Integration:**

server-template pod 10 dns-service.default.svc.cluster.local:8080 check


- automatische Skalierung bis 10 Pods
- keine statische Pod-Liste
- dynamische DNS-Auswertung

---

## Nutzung

### Ressourcen ausrollen

```bash
kubectl apply -f https://raw.githubusercontent.com/Sodeil/K8S-Mini-Project/a802497a6cc9d1d8e4d0f0c11cd1f01447b747a7/project-config.yaml
kubectl apply -f https://raw.githubusercontent.com/Sodeil/K8S-Mini-Project/a802497a6cc9d1d8e4d0f0c11cd1f01447b747a7/dns-service.yaml
kubectl apply -f https://raw.githubusercontent.com/Sodeil/K8S-Mini-Project/a802497a6cc9d1d8e4d0f0c11cd1f01447b747a7/project.yaml
kubectl apply -f https://raw.githubusercontent.com/Sodeil/K8S-Mini-Project/a802497a6cc9d1d8e4d0f0c11cd1f01447b747a7/loadbalancer.yaml

Status prüfen:

kubectl get pods
kubectl get endpoints dns-service

Zugriff:

kubectl port-forward pod/loadbalancer 8443:8443

Test:
curl -k https://localhost:8443/

Welcher Pod:
curl -Ik https://localhost:8443/ 2>/dev/null | grep -i '^X-served-by:'

Browser:

https://localhost:8443

SSL-Warnung ist erwartbar (Self-Signed Zertifikat)
