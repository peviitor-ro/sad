# Software Architecture Design (SAD) — peviitor.ro

## Version 1.0 — June 2026

---

## 1. Introducere

### 1.1 Scop
Acest document descrie arhitectura software a platformei **peviitor.ro** — un motor de cautare a locurilor de munca din Romania. Platforma agregeaza joburi din peste 100 de surse (site-uri de cariera, ANAFM, companii direct) si le pune la dispozitie printr-un search engine full-text.

### 1.2 Domeniu
Sistemul acopera: colectarea automata a datelor (scraping), validarea, indexarea full-text in Apache SOLR, expunerea printr-un API BFF (Backend for Frontend) si interfata web React.

### 1.3 URL-uri principale
| Componenta | URL |
|---|---|
| Platforma live | https://peviitor.ro |
| API (BFF) | https://api.peviitor.ro |
| SOLR (search index) | https://solr.peviitor.ro |
| Admin/Validator | https://admin.peviitor.ro |
| Test | https://test.peviitor.ro |

---

## 2. Arhitectural Drivers

### 2.1 Constrangeri
- **Hardware**: Doua **Raspberry Pi 5** — SOLR pe unul (16GB RAM), API BFF pe celalalt (4GB RAM)
- **Cost**: Proiect open-source, zero buget operational
- **OS**: SOLR Pi — Debian 13 (Trixie), kernel 6.18.29; API Pi — Debian 12 (Bookworm), kernel 6.12.87
- **Stocare**: SOLR Pi — microSD 64GB (33GB folositi); API Pi — microSD 59.4GB (31GB liberi)
- **Retea**: SOLR Pi — Ethernet 1000 Mbps (eth0: 192.168.1.134); API Pi — Ethernet 1000 Mbps (eth0: 192.168.1.135); IP public prin DDNS (zimbor.go.ro), ISP RCS&RDS

### 2.2 Atribute de calitate
| Atribut | Abordare |
|---|---|
| **Performanta** | SOLR edismax + caches; frontend React static pe Vite; API PHP usor |
| **Scalabilitate** | Scraping distribuit prin GitHub Actions (mai multe repo-uri paralele) |
| **Disponibilitate** | Single-node SOLR pe RPi; downtime acceptabil |
| **Mentenabilitate** | Cod open-source, documentat in README-uri, GitFlow conventional |
| **Securitate** | Basic Auth SOLR; CORS restrictionat; API keys in env vars |
| **Corectitudine date** | Pipeline de validare: scraped -> tested -> verified -> published |

---

## 3. Stakeholders si Viewpoints

| Stakeholder | Interes principal |
|---|---|
| **Cautatori de locuri de munca** | Interfata rapida, cautare relevanta, date corecte |
| **Contributori open-source** | Scrapers, bug fixes, features |
| **QA Team** | Validare date, testare automata |
| **Administrator platforma** | Monitorizare, mentenanta, securitate |

---

## 4. Context View (C4 Level 1)

```
                    +-------------------+
                    |   Cautator de     |
                    |   munca (Browser) |
                    +--------+----------+
                             |
                        HTTPS (peviitor.ro)
                             |
                    +--------v----------+
                    |   search-engine   |
                    |  (React + Vite)   |
                    +--------+----------+
                             |
                     HTTPS (api.peviitor.ro)
                             |
                     +--------v----------+
                     |   API (BFF) v1    |
                     |  PHP 8.x, RPi 5   |
                     |    (4GB RAM)      |
                     +--------+----------+
                              |
                    HTTP (192.168.1.x:8983)
                    Basic Auth (LAN)
                              |
                     +--------v----------+
                     |  Apache SOLR 10.x |
                     |  RPi 5 (16GB RAM) |
                     +--------+----------+
                             ^
                             |
              +--------------+--------------+
              |              |              |
      +--------v------+ +----v------+ +-----v--------+ +------------------+
      | Scrapers (Py) | | JMeter    | | ANOFM         | | Scrapers (Node)  |
      | GitHub Action | | Scraper   | | Scraper       | | JS / TypeScript  |
      +---------------+ +-----------+ +---------------+ +------------------+
```

---

## 5. Container View (C4 Level 2)

### 5.1 Frontend — search-engine

| Caracteristica | Valoare |
|---|---|
| Limbaj | JavaScript (JSX) |
| Framework | React 18 |
| State Management | Redux Toolkit + redux-thunk |
| Routing | react-router-dom v6 |
| Styling | Tailwind CSS 3 + shadcn/ui (Radix) |
| Build | Vite 5 |
| Monitoring | Sentry (frontend) |
| Analytics | Microsoft Clarity |
| Testare | Jest + React Testing Library |
| Moduri build | local, qa, production, final |

**Structura src:**
```
src/
  components/   -> UI components (shadcn/ui + custom)
  context/      -> React context providers
  pages/        -> Route pages (Search, JobDetail, etc.)
  reducers/     -> Redux slices
  lib/          -> Utility library
  utils/        -> Helpers
  assets/       -> Statics (SVG, images)
  store.js      -> Redux store config
  App.jsx       -> Root component + routes
  index.jsx     -> Entry point
```

### 5.2 API BFF — api.peviitor.ro

| Caracteristica | Valoare |
|---|---|
| Limbaj | PHP 8.3 |
| Runtime | Container Docker (`php:8.3-apache`) pe port 8080 |
| Hardware | **Raspberry Pi 5** (4GB RAM, microSD 59.4GB, 1 Gbps Ethernet) |
| OS | Debian 12 (Bookworm), kernel 6.12.87+rpt-rpi-2712 |
| Runtime-uri | Node.js v20.20.2, Python 3.11.2, GCC 12 |
| Hostname | `api` |
| Reverse proxy | Nginx Proxy Manager (Docker, `jc21/nginx-proxy-manager`) pe porturile 80/81/443 |
| Alte containere | `orase-api` (php:8.3-apache, port 8081) — serviciul orase.peviitor.ro |
| Versiune API productie | **v1** |
| Alte versiuni | v0, v3, v4, v5, v6 (experimentale/work-in-progress) |
| Autentificare SOLR | Basic Auth (SOLR_USER + SOLR_PASS din api.env) |
| Conexiune SOLR | HTTP pe LAN la RPi-ul SOLR (port 8983) |
| CORS | Restrictionat la domenii cunoscute (admin.zira.ro, api.peviitor.ro) |

**Endpoint-uri v1:**
```
GET  /v1/search       -> Cautare full-text (q, company, city, workmode, page, rows, sort)
GET  /v1/jobs         -> Listare joburi
GET  /v1/companies    -> Listare companii
GET  /v1/company      -> Detalii companie
GET  /v1/total        -> Numar total joburi
GET  /v1/suggest      -> Autocomplete sugestii
GET  /v1/random       -> Joburi aleatoare
GET  /v1/health       -> Status sanatate
GET  /v1/firme        -> Firme
POST /v1/add          -> Adaugare documente
PUT  /v1/update       -> Actualizare documente
DELETE /v1/cleanjobs  -> Curatare joburi expirate
DELETE /v1/empty      -> Golire core
```

**Flux cautare (v1/search):**
1. Normalizare query (diacritice -> ASCII)
2. Construire SOLR query (edismax, fq, bq, sort)
3. Fetch de la http://192.168.1.134:8983/solr/job/select
4. Mapare campuri SOLR -> API response
5. Return JSON

### 5.3 SOLR Index — solr.peviitor.ro

| Caracteristica | Valoare |
|---|---|
| Versiune | Apache SOLR 10.x |
| Hardware | Raspberry Pi 5 Rev 1.1 (16GB RAM + 16GB swap + 2GB zram, microSD 64GB) |
| SO | Debian 13 (Trixie) v13.5, kernel 6.18.29+rpt-rpi-2712 |
| Hostname | `solr-pi` |
| Retea | Ethernet 1000 Mbps (eth0: 192.168.1.134/24) |
| Runtime | Docker container `solr:10-slim` (port 8983 expus) |
| Autentificare | Basic Auth (configurat prin security.json) |
| URL public | https://solr.peviitor.ro (prin API BFF / Nginx Proxy Manager) |

#### Core: `job` (locuri de munca)

| Field | Tip | Stocat | Indexat | MultiValued | Required |
|---|---|---|---|---|---|
| `url` | string | da | da | nu | **DA** (uniqueKey) |
| `title` | text_general | da | da | nu | nu |
| `company` | string | da | da | nu | nu |
| `cif` | string | da | da | nu | nu |
| `location` | text_general | da | da | **da** | nu |
| `workmode` | string | da | da | nu | nu |
| `status` | string | da | da | nu | nu |
| `salary` | text_general | da | da | nu | nu |
| `date` | pdate | da | da | nu | nu |
| `vdate` | pdate | da | da | nu | nu |
| `expirationdate` | pdate | da | da | nu | nu |
| `tags` | text_general | da | da | **da** | nu |

**Copy fields** in `_text_`: url, title, company, location, tags, workmode, salary

**Suggester**: FuzzyLookupFactory pe campul `title`

#### Core: `company` (companii)

| Field | Tip | Stocat | Indexat | MultiValued |
|---|---|---|---|---|
| `id` (cif) | string | da | da | nu |
| `company` | string | da | da | nu |
| `status` | string | da | da | nu |
| `location` | text_general | da | da | da |
| `website` | string | da | da | da |
| `career` | string | da | da | da |
| `brand` | string | da | da | nu |
| `group` | string | da | da | nu |
| `lastScraped` | string | da | nu | nu |
| `scraperFile` | string | da | nu | nu |

### 5.4 Data Model

#### Status Flow (Job)
```
scraped -> tested -> published
scraped -> verified -> published
```

- `scraped` — abia colectat, nevalidat
- `tested` — URL functional, dar date incomplete (CAPTCHA, lipsa salariu etc.)
- `verified` — complet, toate campurile extrase
- `published` — importat in indexul principal

#### Expirare automata
- `expirationdate` = `vdate` + maxim 30 de zile
- Job-uri cu `expirationdate < NOW()` sterse automat (cron zilnic 02:00)

#### Validare URL
- Selectare joburi `verified` cu `date` > 1 zi
- HEAD request paralel (max 1000 concurrent)
- 404 -> DELETE
- Continut invalid ("expirat", "ocupat", "no longer available") -> `tested`
- Valid -> `verified` refresh

---

## 6. Deployment View

### 6.1 Infrastructura

```
+-------------------------------------------------------------+
|                        Acasa / RCS&RDS                       |
|                                                              |
|  +------------------+    +-----------------------+          |
|  | Router TP-Link   |    | RPi 5 (4GB) — API     |          |
|  | 192.168.1.1      |    | 192.168.1.135         |          |
  |  | Ethernet 1 Gbps  +----+ Nginx Proxy :80/443   |          |
  |  | DDNS:            |    | Nginx UI :81          |          |
  |  | zimbor.go.ro     |    | PHP BFF :8080         |          |
  |  |                  |    +-----------------------+          |
  |  |                  |    +------------------------+          |
  |  |                  +----+ RPi 5 (16GB) — SOLR    |          |
  |  |                       | 192.168.1.134          |          |
  |  |                       | Docker solr:10-slim    |          |
  |  |                       | SOLR :8983             |          |
  |  +--------+---------+    +------------------------+          |
  |           |                                                   |
  +-----------|---------------------------------------------------+
            | Internet (NAT)
            |
    +-------v--------+
    |  IP Public      |
    |  86.122.35.88   |
    +-----------------+
            |
    +-------v--------+
    |  Frontend       |
    |  (GitHub Pages) |
    +-----------------+
```

### 6.2 Componente externe
- **API BFF**: Ruleaza pe RPi 5 (4GB) in reteaua locala
- **Frontend**: Gazduit pe **GitHub Pages** (cdn extern)
- **Scrapers**: Ruleaza prin **GitHub Actions** in repo-uri separate
 - **DNS**: zimbor.go.ro (DDNS -> IP RCS&RDS)
- **Monitoring**: Sentry (frontend + backend), Netdata (API Pi)
- **Analytics**: Microsoft Clarity

### 6.3 Scrapers & Data Pipeline
```
Sursa (site companie)
    | (Python/Scrapy/NodeJS/JMeter)
    v
GitHub Actions (cron zilnic)
    |
    v
Script de upload (upload_to_solr.sh)
    |
    v
SOLR index (job core)
    |
    v (cron daily 06:00)
Validare URL-uri (HEAD requests)
    |
    v (cron daily 02:00)
Stergere joburi expirate (expirationdate < NOW())
```

---

## 7. Decizii arhitecturale (ADR)

| ID | Decizie | Rational |
|---|---|---|
| ADR-001 | **SOLR pe RPi 5 (16GB)** | Cost zero, consum energetic mic, suficient pentru ~50k joburi |
| ADR-002 | **PHP BFF pe RPi 5 separat (4GB)** | Separarea API de SOLR imbunatateste securitatea si izolarea; API-ul e expus public, SOLR ramane in LAN |
| ADR-003 | **PHP BFF (nu API direct SOLR)** | Izolare: frontendul nu stie de SOLR; normalizare query/response; securitate (Basic Auth ascuns) |
| ADR-004 | **React + Redux + Vite** | Performanta, ecosistem matur, tooling modern (HMR, build rapid) |
| ADR-005 | **DDNS zimbor.go.ro** | IP dinamic RCS&RDS; DDNS asigura acces extern la API |
| ADR-006 | **Scrapers in repo-uri separate** | Paralelism prin GitHub Actions; mentenabilitate; contributii distribuite |
| ADR-007 | **edismax + copyField _text_** | Cautare full-text pe mai multe campuri simultan, cu boost configurabil |
| ADR-008 | **status flow scraped->verified->published** | Pipeline de calitate: datele trec prin validare inainte de publicare |

---

## 8. Riscuri si compromisuri

| Risc | Impact | Mitigare |
|---|---|---|
| **Single point of failure** (2 RPi-uri) | API si/sau SOLR indisponibile | Backup config; scripturi de restore rapide |
| **Stocare microSD** | Fiabilitate redusa, uzura | Monitoring health; backup zilnic |
| **IP dinamic RCS&RDS** | Pierdere conexiune externa | DDNS cu TTL scazut; fallback manual |
| **Fara replica SOLR** | Pierdere date la defectiune | Backup configurable; rebuild din scrapers |
| **Dependenta GitHub Actions** | Scrapers nu ruleaza fara GitHub | Self-hosted runner ca alternativa |

---

## 9. Tehnologii utilizate

| Tehnologie | Rol | Versiune |
|---|---|---|
| Apache SOLR | Search & indexare | 10-slim (Docker) |
| PHP | API BFF | 8.x |
| React | Frontend UI | 18.x |
| Redux Toolkit | State management | 2.x |
| Vite | Build tool | 5.x |
| Tailwind CSS | Styling | 3.x |
| shadcn/ui (Radix) | Componente UI | - |
| Python / Scrapy | Web scraping | 3.x |
| NodeJS / JavaScript | Web scraping | - |
| Apache JMeter | Web scraping | 5.x |
| Sentry | Error monitoring | - |
| Netdata | Monitoring infrastructura | 2.10.0 |
| Microsoft Clarity | User analytics | - |
| Docker | Containerizare (dev) | - |
| Debian Linux | OS (RPi) | 13 (Trixie) |
| Nginx / Apache | Reverse proxy | - |
| Playwright | Testare E2E automata | - |
| PHPUnit / Jest / Vitest | Testare unitara | - |
| k6 | Testare de performanta | - |
| OWASP ZAP | Testare de securitate | - |
| TestLink | Management teste | - |
| GitHub Actions | CI/CD, scrapers | - |

---

## 10. Testare

### 10.1 Document de referinta

Strategia de testare este definita integral in documentul separat:
**[Test Strategy — peviitor.ro](https://github.com/peviitor-ro/test_strategy_peviitor-ro/blob/master/Test_Strategy_Peviitor.ro.md)**


Autor: Diana Dragoi | Aprobat: Ana-Maria Talmacel (QA Lead), Boga Sebastian-Nicolae (PO)

### 10.2 Arii de testare

| Arie | Abordare |
|------|----------|
| **Frontend (search-engine)** | Testare manuala exploratorie + Automatizare E2E (Playwright) + Testare vizuala (Figma) |
| **API (api.peviitor.ro)** | Validare endpoint-uri v0/v1 in Swagger UI + Postman/Bruno + PHPUnit |
| **SOLR Index** | Conformitate schema, cautare diacritice, CRUD, facet search, validare URL pipeline |
| **Validare date** | Status flow (scraped→tested→verified→published), campuri obligatorii, matching companii |
| **Cross-component** | Integrare Frontend↔API↔SOLR↔Scrapers |
| **Non-functional** | Performanta (P95 < 2s), Securitate (OWASP Top 10), Accesibilitate (WCAG 2.2 AA), SEO |

### 10.3 Medii de testare

| Mediu | URL | Scop |
|-------|-----|------|
| **Local** | `http://localhost:3000` | Dezvoltare, unit testing (Docker) |
| **Test** | [test.peviitor.ro](https://test.peviitor.ro) | Integrare, UAT, regresie. SOLR: [testsolr.peviitor.ro](https://testsolr.peviitor.ro). Swagger: [test.peviitor.ro/swagger-ui](https://test.peviitor.ro/swagger-ui) |
| **Productie** | [peviitor.ro](https://peviitor.ro) | Live. SOLR: [solr.peviitor.ro](https://solr.peviitor.ro) |

### 10.4 Echipe si tool-uri

- **5 Manual QA** — testare functionala, exploratorie, regresie, validare date
- **2 Automation QA** — Playwright E2E, Postman, integrare GitHub Actions
- **1 Performance Tester** — k6, Lighthouse, monitorizare SOLR
- **1 Security Tester** — OWASP ZAP, GitHub Secret Scanning / CodeQL / Dependabot

Tool-uri principale: **Playwright**, **Postman/Bruno**, **PHPUnit/Jest/Vitest**, **Apache JMeter**, **k6**, **OWASP ZAP**, **axe DevTools/WAVE**, **TestLink**, **GitHub Issues**.

### 10.5 Livrabile si metrici

| Artifact | Frecventa |
|----------|-----------|
| Test cases & checklists | Per release |
| Bug reports (GitHub Issues) | Zilnic |
| Test Summary Report | Per release |
| **KPI** | Target |
| Test Pass Rate | ≥ 95% |
| Automation Coverage | ≥ 80% critical path |
| Defect Escape Rate | ≤ 3 high-severity/trimestru |
| Search Response Time | P95 < 2s |

### 10.6 Bug Lifecycle

```
[New] → [Assigned] → [In Progress] → [Fixed] → [Verified] → [Closed]
```
Bug severity: S1 (Blocker) → S4 (Trivial). Triaj zilnic; S1/S2 rezolvate in < 4h.

---

## 11. Glosar

| Termen | Definitie |
|---|---|
| **BFF** | Backend for Frontend — API care serveste specific unui client (frontend) |
| **SOLR** | Platforma de cautare full-text open-source (Apache Lucene) |
| **Core** | Index SOLR (echivalentul unui tabel in DB) |
| **edismax** | Query parser SOLR care permite boost si campuri multiple |
| **BFF API** | API-ul PHP (v1) care face proxy intre frontend si SOLR |
| **Scraper** | Script care extrage automat date de pe site-uri externe |
| **DDNS** | Dynamic DNS — serviciu care actualizeaza DNS-ul pentru IP-uri dinamice |
| **CIF/CUI** | Codul de Identificare Fiscala al companiilor din Romania |
| **vdate** | Verified date — data la care un job a fost verificat |
| **ANAF** | Agentia Nationala de Administrare Fiscala |

---

*Document generat pe baza explorarii organizatiei GitHub [peviitor-ro](https://github.com/peviitor-ro), a codului din repo-urile publice, a configuratiei hardware live si a documentului [Test Strategy — peviitor.ro](https://github.com/peviitor-ro/test_strategy_peviitor-ro/blob/master/Test_Strategy_Peviitor.ro.md).*
