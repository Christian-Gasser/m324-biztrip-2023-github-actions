# EX-03 – Deployment einer Docker-Anwendung auf AWS ECS (Fargate) mit GitHub Actions

## Lernziele

Nach dieser Übung könnt ihr:

- den Unterschied zwischen einem direkten EC2-Deployment (EX-01) und einem containerbasierten ECS-Deployment einordnen
- ein Amazon ECR (Elastic Container Registry) Repository anlegen und ein Image dorthin pushen
- eine ECS-Task-Definition, einen Cluster und einen Service (Fargate-Launch-Type) anlegen
- einen Application Load Balancer vor einen ECS-Service schalten
- eine GitHub-Actions-Pipeline schreiben, die bei jedem Push auf `main` ein neues Image baut, nach ECR pusht und den ECS-Service aktualisiert (Rolling Deployment)
- AWS-Zugangsdaten per OIDC (statt langlebiger Access Keys) an GitHub Actions vergeben

## Voraussetzungen

- Abgeschlossene [EX-02](./EX-02-create-Docker-Image-DockerHub.md) — das Repository enthält bereits ein funktionierendes `Dockerfile`, `.dockerignore` und `nginx.conf`
- Ein AWS-Account mit Rechten, ECR-, ECS-, IAM- und ELB-Ressourcen anzulegen
- AWS CLI lokal installiert und konfiguriert: `aws --version`, `aws sts get-caller-identity`
- `gh` CLI weiterhin eingeloggt (siehe EX-01)

---

## EC2- vs. ECS-Deployment im Vergleich

| | EC2-Deployment (EX-01) | ECS-Deployment (diese Übung) |
| --- | --- | --- |
| **Deploy-Einheit** | Statische Dateien (`dist/`), per `rsync` auf einen konkreten Server kopiert | Ein Docker-Image, das aus einer **Task Definition** heraus als Container gestartet wird |
| **Server-Management** | Ihr verwaltet die EC2-Instanz selbst: OS-Updates, nginx-Installation/-Konfiguration, Prozess am Leben halten | Bei **Fargate** keine Server sichtbar/verwaltbar — AWS betreibt die Rechenkapazität. Bei **EC2-Launch-Type** laufen weiterhin eigene EC2-Instanzen, aber ECS übernimmt das Scheduling der Container darauf |
| **Skalierung** | Manuell, oder selbst eine Auto Scaling Group + Load Balancer aufbauen | Eingebaut über die ECS-**Service**-Definition (`desiredCount`) + Application Auto Scaling |
| **Self-Healing** | Nicht vorhanden — ein abgestürzter Prozess muss manuell/durch ein eigenes Skript neu gestartet werden | Der ECS-Service ersetzt automatisch Tasks, die crashen oder den Health-Check nicht bestehen |
| **Deploy-Mechanismus** | SSH-Verbindung, Dateien kopieren, nginx neu laden | Neues Image nach ECR pushen, Task Definition aktualisieren, `aws ecs update-service` löst ein Rolling Deployment aus — kein SSH nötig |
| **Rollback** | Manuell: alten Build erneut deployen | Alte Task-Definition-Revision erneut als Service-Deployment auswählen |
| **Typische Zugriffskontrolle** | SSH-Key (`EC2_SSH_KEY`) mit vollem Server-Zugriff | IAM-Rolle mit fein granulierten Rechten nur auf ECR/ECS-APIs (idealerweise per OIDC, kein Long-Lived-Key) |
| **Netzwerk** | Direkt gegen die Public-IP/DNS der Instanz, ggf. nginx als Reverse Proxy | Application Load Balancer + Target Group vor dem Service, Health Checks über die ALB |

Kurz gesagt: EX-01 verschiebt Dateien auf einen Server, den ihr komplett selbst betreibt. Diese Übung verschiebt ein **Image** in eine **Registry** und überlässt Scheduling, Skalierung und Self-Healing der Container der ECS-Kontrollebene.

---

## Schritt 1: ECR-Repository anlegen

```bash
aws ecr create-repository --repository-name biztrips
```

Notiert euch die zurückgegebene `repositoryUri`, z. B. `123456789012.dkr.ecr.eu-central-1.amazonaws.com/biztrips`.

---

## Schritt 2: IAM-Rolle für GitHub Actions per OIDC einrichten

Statt langlebiger `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` als Secrets zu hinterlegen (Risiko bei Leak), richtet GitHub als OIDC-Identity-Provider in AWS ein und erstellt eine IAM-Rolle, die GitHub Actions per kurzlebigem Token annehmen kann.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Danach eine Rolle `github-actions-biztrips-ecs` mit einer Trust Policy anlegen, die nur auf euer Repository/euren Branch eingeschränkt ist (`repo:<user>/<repo>:ref:refs/heads/main`), und ihr die Policies `AmazonEC2ContainerRegistryPowerUser` sowie eine eingeschränkte ECS-Update-Policy anhängen.

> Details zur Trust-Policy-Syntax: [`aws-actions/configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials#configuring-the-role-and-trust-policy) — folgt der dortigen Anleitung, statt die Policy von Hand zu tippen.

---

## Schritt 3: ECS-Cluster anlegen

```bash
aws ecs create-cluster --cluster-name biztrips-cluster
```

Ein Fargate-Cluster braucht keine eigenen EC2-Instanzen — der Cluster ist zunächst nur ein logischer Namespace für Services/Tasks.

---

## Schritt 4: Application Load Balancer + Target Group

- Eine **Target Group** vom Typ `ip` (Fargate-Tasks bekommen eine ENI mit eigener IP, keine Instance-ID) anlegen, Port 80, Health-Check-Pfad `/`
- Einen **Application Load Balancer** in mindestens zwei Subnets anlegen, Listener auf Port 80 → leitet an die Target Group weiter
- Security Group des ALB: eingehend Port 80 aus dem Internet
- Security Group der Fargate-Tasks: eingehend Port 80 **nur von der Security Group des ALB**

---

## Schritt 5: Task Definition schreiben

`task-definition.json` im Projektroot:

```json
{
  "family": "biztrips",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "biztrips",
      "image": "123456789012.dkr.ecr.eu-central-1.amazonaws.com/biztrips:latest",
      "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/biztrips",
          "awslogs-region": "eu-central-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

`ecsTaskExecutionRole` ist eine von AWS vorgegebene Standardrolle (`AmazonECSTaskExecutionRolePolicy`), die dem Container erlaubt, das Image von ECR zu ziehen und Logs nach CloudWatch zu schreiben — analog zur Rolle, die in EX-01 der SSH-User implizit über `sudo`-Rechte auf der Instanz hatte.

---

## Schritt 6: ECS-Service anlegen

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json

aws ecs create-service \
  --cluster biztrips-cluster \
  --service-name biztrips-service \
  --task-definition biztrips \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-tasks],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=biztrips,containerPort=80"
```

`desired-count 2` sorgt dafür, dass immer zwei Tasks laufen — fällt eine aus, ersetzt der Service sie automatisch. Das ist der Punkt, an dem ECS sich am deutlichsten von EX-01 unterscheidet: Dort gab es genau **eine** Instanz ohne eingebaute Redundanz.

---

## Schritt 7: GitHub-Actions-Job `deploy-ecs`

Neuer Job in `.github/workflows/deploy.yml`, der auf dem bestehenden `docker`-Job aus EX-02 aufbaut:

```yaml
  deploy-ecs:
    name: Image nach ECR pushen und ECS-Service aktualisieren
    runs-on: ubuntu-latest
    needs: docker
    if: github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    permissions:
      id-token: write   # notwendig für OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: AWS-Credentials via OIDC beziehen
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-biztrips-ecs
          aws-region: eu-central-1

      - name: Bei ECR anmelden
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2

      - name: Image bauen und nach ECR pushen
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: biztrips
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --build-arg VITE_API_BASE_URL="${{ vars.VITE_API_BASE_URL }}" \
            -t "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
            -t "$ECR_REGISTRY/$ECR_REPOSITORY:latest" .
          docker push "$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"
          docker push "$ECR_REGISTRY/$ECR_REPOSITORY:latest"

      - name: Task-Definition mit neuem Image aktualisieren
        id: render-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: biztrips
          image: ${{ steps.ecr-login.outputs.registry }}/biztrips:${{ github.sha }}

      - name: ECS-Service deployen
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.render-task-def.outputs.task-definition }}
          cluster: biztrips-cluster
          service: biztrips-service
          wait-for-service-stability: true
```

Wichtige Design-Entscheidungen:

- **`permissions: id-token: write`** — ohne diese Berechtigung kann der Job kein OIDC-Token anfordern und `configure-aws-credentials` schlägt fehl
- **`wait-for-service-stability: true`** — der Job wartet, bis ECS bestätigt, dass alle neuen Tasks laufen und die alten Tasks abgelöst wurden (Rolling Deployment), statt sofort grün zu melden, während im Hintergrund noch deployed wird
- Kein SSH-Key, kein `rsync` — die einzige "Zugangsdaten" ist die kurzlebige, auf dieses Repository/diesen Branch eingeschränkte IAM-Rolle

### Benötigte zusätzliche Repository-Variables

| Variable | Beispiel | Beschreibung |
| --- | --- | --- |
| `AWS_ROLE_ARN` | `arn:aws:iam::123456789012:role/github-actions-biztrips-ecs` | Rolle aus Schritt 2 (kann statt Klartext in der YAML auch als Variable referenziert werden) |

---

## Bekannte Stolpersteine

**`AccessDenied` beim `configure-aws-credentials`-Schritt**
Die Trust Policy der IAM-Rolle ist meist zu eng oder zu weit falsch konfiguriert — prüft, ob `sub` in der Trust Policy exakt `repo:<user>/<repo>:ref:refs/heads/main` entspricht (inkl. korrektem Repo-Namen und Branch).

**Task startet, aber Health-Check der Target Group schlägt dauerhaft fehl**
Meist Security-Group-Problem: Die Security Group der Tasks muss eingehenden Traffic von der Security Group des ALB auf Port 80 erlauben (Schritt 4) — nicht umgekehrt.

**`CannotPullContainerError` beim Task-Start**
Die `executionRoleArn` fehlt oder hat nicht die Policy `AmazonECSTaskExecutionRolePolicy` — ohne diese Rolle darf der Task-Agent das Image nicht von ECR ziehen.

**Service bleibt bei "PENDING", `desired-count` wird nie erreicht**
Häufig fehlende `assignPublicIp=ENABLED` bei Tasks in einem öffentlichen Subnet ohne NAT-Gateway — ohne Public IP kann der Task-Agent weder das Image ziehen noch Logs senden.

---

## Reflexionsfragen

1. Welche Aufgaben, die in EX-01 der SSH-Deploy-Schritt (`rsync` + `sudo systemctl reload nginx`) manuell erledigt hat, übernimmt in dieser Übung die ECS-Kontrollebene automatisch?
2. Warum ist eine über OIDC bezogene, kurzlebige IAM-Rolle sicherer als ein dauerhaft in GitHub Secrets hinterlegter AWS-Access-Key, wie ihn EX-01 in Form von `EC2_SSH_KEY` verwendet?
3. Was passiert mit den zwei laufenden Tasks eines Services, wenn ein `update-service` mit neuer Task-Definition ausgelöst wird — und warum ist das ein Vorteil gegenüber dem Single-Server-Deployment aus EX-01?
4. Warum braucht die Target Group in Schritt 4 den Typ `ip` statt `instance`, obwohl EX-01/EX-02 nie mit Target Groups gearbeitet haben?
