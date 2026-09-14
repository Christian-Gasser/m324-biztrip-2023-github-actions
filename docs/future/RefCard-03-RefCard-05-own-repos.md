# Planung: RefCard 03–05 — eigene Repos pro Service

> Status: RefCard 03 ist als eigene Repos aufgesetzt und in Arbeit; 04/05
> weiterhin in Planung. Dieses Dokument hält die Entscheidungen fest, damit
> die Struktur über die ganze Sequenz hinweg konsistent bleibt.

## Überblick über die Sequenz

| RefCard | Status | Neues Thema | Frontend | Backend | Datenbank | Storage |
| --- | --- | --- | --- | --- | --- | --- |
| 02 (dieses Repo) | 🟢 fertig | GitHub Actions, EC2, Docker, ECS | React + Vite | – | – | – |
| 03 | 🟡 in Arbeit | Spring Boot Backend | React + Vite | Spring Boot | MariaDB | – |
| 04 | ⚪ geplant | Storage (Bilder) | React + Vite | Spring Boot | MariaDB | lokal/Volume → Vorstufe zu S3 |
| 05 | ⚪ geplant | Fullstack auf AWS | React + Vite | Spring Boot | MariaDB | S3 |

RefCard 03 liegt bereits in zwei separaten Repos vor:

- Backend: [`RefCard-03-be-SpringBootBusinessTrips`](https://github.com/bbwlc/RefCard-03-be-SpringBootBusinessTrips)
- Frontend: [`RefCard-03-fe-React-vite`](https://github.com/bbwlc/refcard-03-fe-react-vite)

Jede RefCard führt genau **ein** neues Konzept ein. Das Frontend bleibt über
die ganze Sequenz React + Vite (keine Umstellung auf Next.js o. Ä.), damit die
neue Komplexität jeweils vom Backend/Storage/AWS-Teil kommt und nicht von
einem Frontend-Framework-Wechsel.

## Entscheidung: eigenes Repo pro Service statt Monorepo

**Entscheidung:** Frontend und Backend bekommen je ein eigenes Repository mit
eigener GitHub-Actions-Pipeline und eigenem Image in ECR — kein Monorepo.

**Begründung:**

- Setzt das bestehende Muster fort: RefCard 02 (dieses Repo) ist bereits
  Frontend-only, RefCard 03 wird analog Backend-only sein.
- Jede Pipeline bleibt ein in sich geschlossenes Lehrbeispiel: "build → test →
  Image nach ECR pushen → deployen" — das ist die eigentliche Lektion, nicht
  Monorepo-Tooling.
- Vermeidet zusätzliche Komplexität, die für die CI/CD-Lektion nicht nötig
  wäre (Path-Filter in Workflows, Workspaces, gemeinsame Versionierung).
- Entspricht dem realen Pattern unabhängig deploybarer Services, das RefCard
  05 (ECR/ECS) ohnehin vermitteln soll.

**Kompromiss:** Für RefCard 05 sehen Studierende Frontend, Backend, DB und
Storage nicht aus einem einzigen `git clone` heraus per `docker-compose up`.
Sie brauchen entweder zwei Repos parallel ausgecheckt, oder ein separates,
kleines "Compose"-Repo/Verzeichnis nur für die lokale Entwicklung, das beide
Images referenziert (siehe Abschnitt "Lokale Entwicklung in RefCard 05").

## RefCard 03 — Spring Boot Backend + MariaDB (eigene Repos)

Umgesetzt als zwei Repos, analog zur Entscheidung oben:

- **Backend** — [`RefCard-03-be-SpringBootBusinessTrips`](https://github.com/bbwlc/RefCard-03-be-SpringBootBusinessTrips):
  Spring Boot REST-API (Java 21, Spring Boot 4.1, Web + Data JPA, Maven
  Wrapper) für Mitarbeitende, Business Trips, Flüge, Meetings. MariaDB lokal
  via Docker Compose, H2 als Default für Tests/schnellen lokalen Start. Hat
  eigenes `Dockerfile`, eigene `deploy.yml`, eigene `task-definition.json`
  und eigenes `docs/`-Verzeichnis (inkl. eigener Kopie dieser Planung unter
  `docs/future/`).
- **Frontend** — [`RefCard-03-fe-React-vite`](https://github.com/bbwlc/refcard-03-fe-react-vite):
  Fortsetzung von RefCard 02 (React + Vite), an die neue Backend-API
  angebunden statt an Mock-Daten. `VITE_API_BASE_URL` zeigt auf die neue
  Backend-URL — das Env-Var-Pattern aus RefCard 02 wurde direkt
  wiederverwendet.

Namenskonvention damit etabliert: `RefCard-<nn>-be-<Beschreibung>` /
`RefCard-<nn>-fe-<Beschreibung>` pro Service.

## RefCard 04 — Storage (Bilder)

- Baut auf RefCard 03 auf (gleiches Backend-Repo oder Fortsetzung davon).
- Bewusst **zuerst lokaler/Volume-Storage**, nicht direkt S3 — damit
  Studierende den Unterschied zu RefCard 05 später konkret erleben ("warum
  reicht ein lokales Volume in der Cloud nicht mehr").
- Spring Boot Endpoint zum Hoch-/Runterladen von Bildern, Ablage auf einem
  gemounteten Volume.

## RefCard 05 — Fullstack auf AWS (ECR, ECS, S3)

- Bringt alle Teile zusammen: React+Vite-Frontend, Spring-Boot-Backend,
  MariaDB, Storage — jetzt auf **S3** umgestellt statt lokalem Volume.
- Jeder Service (Frontend, Backend) hat weiterhin sein eigenes Repo, eigenes
  ECR-Repository und eigenen ECS-Service — analog zum Muster aus
  [EX-03](../exercises/EX-03-deploy-AWS-ECS.md), nur mit zwei Services statt
  einem.
- Backend bekommt eine IAM-Rolle mit Zugriff nur auf den konkreten S3-Bucket
  (Least Privilege, gleiches Prinzip wie die OIDC-Rolle in EX-03).
- Frontend bleibt ein reiner statischer Build; ob er über nginx-in-ECS oder
  S3+CloudFront ausgeliefert wird, ist ein offener Punkt (siehe unten).

### Lokale Entwicklung in RefCard 05

Da Frontend und Backend in getrennten Repos liegen, braucht es für die lokale
Entwicklung eine Möglichkeit, beide zusammen laufen zu lassen:

- Option A: Kurze Anleitung in beiden READMEs, beide Repos nebeneinander
  auszuchecken und zwei Terminals/`docker-compose`-Aufrufe zu starten.
- Option B: Ein drittes, kleines Repo/Verzeichnis nur mit einer
  `docker-compose.yml`, die die (lokal gebauten oder von ECR gezogenen)
  Images beider Services referenziert — kein eigener Anwendungscode.

Entscheidung zwischen A und B folgt beim Bauen von RefCard 05.

## Entscheidung: MariaDB auf RDS statt als Container

**Entscheidung:** MariaDB läuft in RefCard 03/05 auf **RDS**, nicht als eigener
Container in ECS/Fargate.

**Begründung:** Fargate hat keinen direkten persistenten Storage (kein
EBS-Mount im `awsvpc`-Netzwerkmodus, nur EFS mit zusätzlichem Komplexitäts-
Aufwand) — eine DB als Fargate-Task zu betreiben würde Studierende mit einem
Storage-Problem konfrontieren, das nichts mit der eigentlichen Lektion zu tun
hat. RDS ist ausserdem in AWS Academy Learner Lab verfügbar (siehe unten) und
passt zum bereits vermittelten Prinzip "AWS verwaltet zustandsbehaftete
Ressourcen, die Container-Flotte bleibt stateless".

## Wichtige Rahmenbedingung: AWS Academy Learner Lab

Die Studierenden arbeiten mit **AWS Academy Learner Lab**-Accounts. Das hat
zwei konkrete Auswirkungen auf die Planung:

- **RDS funktioniert** — RDS ist einer der in Learner Lab freigegebenen
  Services (Regionen meist auf `us-east-1`/`us-west-2` beschränkt, dazu
  Limits bei Instance-Typen, Storage-Grösse und vCPUs). Die
  RDS-Entscheidung oben ist damit für Learner-Lab-Accounts umsetzbar.
- **OIDC/eigene IAM-Rollen funktionieren nicht.** Learner-Lab-Accounts
  erlauben **kein** `iam:CreateRole` und **keine** eigene OIDC-Identity-
  Provider-Einrichtung — Studierende haben nur die vorgegebene `LabRole` und
  dürfen dieser höchstens zusätzliche Policies anhängen. Das betrifft direkt
  das in [EX-03](../exercises/EX-03-deploy-AWS-ECS.md) Schritt 2 beschriebene
  Vorgehen (`aws iam create-open-id-connect-provider`, eigene Trust Policy) —
  dieser Schritt schlägt in Learner-Lab-Accounts fehl.

  **Fallback für RefCard 05 (und ggf. Korrektur/Ergänzung von EX-03):**
  Statt OIDC die von AWS Academy pro Sitzung bereitgestellten temporären
  Zugangsdaten (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` /
  `AWS_SESSION_TOKEN` aus dem "AWS Details"-Panel des Labs) als GitHub
  Secrets hinterlegen und über `aws-actions/configure-aws-credentials`
  direkt einsetzen (`aws-access-key-id` / `aws-secret-access-key` /
  `aws-session-token` statt `role-to-assume`). Nachteil: Diese Tokens laufen
  mit der Lab-Session ab und müssen von Studierenden manuell aktualisiert
  werden, sobald die Session neu gestartet wird — das sollte in der
  Übungsanleitung explizit als Lab-Sandbox-Einschränkung erklärt werden, mit
  Verweis darauf, dass OIDC (wie in EX-03 gezeigt) der reale
  Best-Practice-Ansatz in einem produktiven AWS-Account wäre.

## Offene Punkte

- Frontend-Auslieferung in RefCard 05: nginx-Container in ECS (konsistent zu
  EX-02/EX-03) vs. S3+CloudFront (würde S3 doppelt nutzen — einmal für
  Storage, einmal für Hosting — ggf. verwirrend für Studierende).
- Ob RefCard 04 in den bestehenden RefCard-03-Repos weitergebaut wird oder
  eigene `RefCard-04-be-...`/`RefCard-04-fe-...`-Repos bekommt (Namenskonvention
  `RefCard-<nn>-be/fe-<Beschreibung>` ist mit RefCard 03 etabliert).
- Soll EX-03 (dieses Repo) um einen Hinweis/Alternativ-Abschnitt zu
  Learner-Lab-Zugangsdaten ergänzt werden, oder bleibt EX-03 bewusst bei
  "echtem" OIDC als Referenz und der Learner-Lab-Fallback wird nur in
  RefCard 05 dokumentiert?
