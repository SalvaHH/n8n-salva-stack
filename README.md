# n8n self-hosted su Hetzner + Coolify (con runner Python)

Documentazione del setup realizzato il 15 ottobre 2026.

---

## 1. Panoramica

| Componente | Dove / cosa |
|---|---|
| Server | Hetzner Cloud, Ubuntu 24.04, x86, 2 vCPU / 4 GB RAM |
| Pannello di gestione | Coolify 4 → `https://coolify.wait4fan.app` |
| n8n | `https://n8n.wait4fan.app` |
| Dominio / DNS | `wait4fan.app`, registrato e gestito su Cloudflare |
| Repo con la configurazione | GitHub `SalvaHH/n8n-salva-stack` (pubblico, nessun segreto dentro) |
| Versione n8n / runner | `2.42.7` (devono essere **sempre identiche**) |

### Architettura

```
Internet ──► Cloudflare DNS (solo DNS, nuvoletta grigia)
                │
                ▼
        Server Hetzner (firewall: 22, 80, 443)
                │
          Traefik (proxy di Coolify, certificati Let's Encrypt)
                │
     ┌──────────┼───────────────┐
     ▼          ▼               ▼
   n8n ◄──► task-runners     postgres
  (5678)   (JS + Python,     (database n8n)
            broker :5679,
            solo rete interna)
```

- **n8n**: editor, webhook, esecuzioni.
- **task-runners**: immagine custom basata su `n8nio/runners`, esegue i nodi Code (JavaScript e Python) in un container separato. Da n8n 2.0 il Python funziona **solo** così.
- **postgres**: database di n8n.

---

## 2. Cosa abbiamo fatto (in ordine)

1. **Chiave SSH**: generata/usata `~/.ssh/id_ed25519.pub` e aggiunta su Hetzner.
2. **Server Hetzner** creato (Ubuntu 24.04) con firewall.
3. **Coolify installato** via SSH:
   ```bash
   curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
   ```
4. **Account admin Coolify** creato su `http://IP_SERVER:8000`, onboarding con server *This Machine*.
5. **Dominio** `wait4fan.app` comprato su Cloudflare; record DNS:

   | Tipo | Nome | Valore | Proxy |
   |---|---|---|---|
   | A | `n8n` | IPv4 del server | DNS only (grigia) |
   | A | `coolify` | IPv4 del server | DNS only (grigia) |

6. **Dominio di Coolify** impostato in *Settings → Instance's Domain* = `https://coolify.wait4fan.app`.
7. **Repo GitHub** con `docker-compose.yaml` e `runners/Dockerfile` (vedi sezione 3).
8. **Deploy in Coolify**: *Public Repository* → Build pack *Docker Compose* → dominio del servizio n8n = `https://n8n.wait4fan.app` porta `5678` (rimossa la variante `www.` aggiunta in automatico).
9. **Test runner Python** riuscito (stdlib + pandas).
10. **Firewall chiuso**: rimosse le porte di setup 8000, 6001, 6002.

### Firewall Hetzner (stato finale)

| Porta | Protocollo | Uso |
|---|---|---|
| 22 | TCP | SSH |
| 80 | TCP | HTTP (serve a Let's Encrypt) |
| 443 | TCP | HTTPS |

Nessuna regola in uscita (tutto permesso). La porta 5679 del broker **non** va mai aperta.

---

## 3. I file del repo

### `runners/Dockerfile`

```dockerfile
FROM n8nio/runners:2.42.7

USER root

# pacchetti Python che vuoi usare nei Code node
RUN cd /opt/runners/task-runner-python && uv pip install pandas numpy requests

# patch del config del launcher: allowlist dentro il JSON (le env var da sole non bastano)
RUN /opt/runners/task-runner-python/.venv/bin/python -c "import json;p='/etc/n8n-task-runners.json';c=json.load(open(p));[r.setdefault('env-overrides',{}).update({'N8N_RUNNERS_STDLIB_ALLOW':'*','N8N_RUNNERS_EXTERNAL_ALLOW':'*'}) for r in c['task-runners'] if r['runner-type']=='python'];json.dump(c,open(p,'w'),indent=2)"

USER runner
```

Perché la patch del JSON: il launcher dei runner legge gli allowlist dal file `/etc/n8n-task-runners.json`, non dalle variabili d'ambiente del container. Con `EXTERNAL_ALLOW='*'` sono permessi tutti i pacchetti installati nell'immagine (servono anche le dipendenze interne, es. pandas importa numpy, dateutil, pytz…).

### `docker-compose.yaml`

```yaml
services:
  n8n:
    image: n8nio/n8n:2.42.7
    restart: unless-stopped
    environment:
      - N8N_HOST=n8n.wait4fan.app
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.wait4fan.app/
      - N8N_PROXY_HOPS=1
      - GENERIC_TIMEZONE=Europe/Madrid
      - TZ=Europe/Madrid
      - N8N_ENCRYPTION_KEY=${SERVICE_PASSWORD_64_N8NKEY}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=${SERVICE_USER_POSTGRES}
      - DB_POSTGRESDB_PASSWORD=${SERVICE_PASSWORD_POSTGRES}
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNERS_MODE=external
      - N8N_RUNNERS_BROKER_LISTEN_ADDRESS=0.0.0.0
      - N8N_RUNNERS_AUTH_TOKEN=${SERVICE_PASSWORD_64_RUNNERS}
    volumes:
      - n8n-data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

  task-runners:
    build: ./runners
    restart: unless-stopped
    environment:
      - N8N_RUNNERS_TASK_BROKER_URI=http://n8n:5679
      - N8N_RUNNERS_AUTH_TOKEN=${SERVICE_PASSWORD_64_RUNNERS}
      - N8N_RUNNERS_AUTO_SHUTDOWN_TIMEOUT=15
    depends_on:
      - n8n

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${SERVICE_USER_POSTGRES}
      - POSTGRES_PASSWORD=${SERVICE_PASSWORD_POSTGRES}
      - POSTGRES_DB=n8n
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d n8n"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  n8n-data:
  pg-data:
```

Le variabili `SERVICE_PASSWORD_*` / `SERVICE_USER_*` sono generate da Coolify al primo deploy e restano fisse. Si vedono in Coolify → app → **Environment Variables**.

---

## 4. ➕ Aggiungere un nuovo pacchetto Python (step by step)

Esempio: aggiungere `openpyxl`.

1. **Apri il Dockerfile su GitHub**
   Vai su `github.com/SalvaHH/n8n-salva-stack` → cartella `runners` → file `Dockerfile` → icona matita ✏️.

2. **Aggiungi il nome del pacchetto alla riga `uv pip install`**
   Prima:
   ```dockerfile
   RUN cd /opt/runners/task-runner-python && uv pip install pandas numpy requests
   ```
   Dopo:
   ```dockerfile
   RUN cd /opt/runners/task-runner-python && uv pip install pandas numpy requests openpyxl
   ```
   - Solo questa riga. **Non toccare** la riga lunga con il JSON: l'allowlist è già `*`.
   - Per fissare una versione: `openpyxl==3.1.5`.

3. **Salva**: pulsante **Commit changes** (in alto a destra) → di nuovo **Commit changes**.

4. **Redeploy in Coolify**
   `https://coolify.wait4fan.app` → progetto → applicazione → pulsante **Redeploy** in alto a destra.
   Aspetta nel log la riga `New container started` (qualche minuto: ricostruisce l'immagine del runner).

5. **Verifica sul server** (opzionale, via SSH):
   ```bash
   docker exec $(docker ps -qf name=task-runners) /opt/runners/task-runner-python/.venv/bin/python -c "import openpyxl; print('ok')"
   ```
   Deve stampare `ok`.

6. **Usalo in n8n**: in un nodo Code (Python):
   ```python
   import openpyxl
   return [{"json": {"ok": True}}]
   ```

### Regole del runner Python da ricordare

- Niente attributi/nomi con doppio underscore (`__version__`, `__import__`, …): vengono bloccati come *Security violation*.
- I dati in ingresso sono in `_items` (lista; ogni elemento ha la chiave `"json"`).
- Si restituisce una lista di dizionari: `return [{"json": {...}}]`.
- Pacchetti non installati nel Dockerfile → `ModuleNotFoundError`. Non si può fare `pip install` dal nodo.

---

## 5. Aggiornare n8n

1. Controlla l'ultima versione **stable** e che esista lo **stesso tag** sia per `n8nio/n8n` sia per `n8nio/runners` su Docker Hub.
2. Su GitHub cambia il numero di versione in **entrambi** i file:
   - `docker-compose.yaml` → `image: n8nio/n8n:X.Y.Z`
   - `runners/Dockerfile` → `FROM n8nio/runners:X.Y.Z`
3. **Redeploy** in Coolify.

Versioni diverse tra n8n e runner = nodi Code che vanno in timeout.

---

## 6. Accessi

| Cosa | Come |
|---|---|
| Coolify | `https://coolify.wait4fan.app` (email + password admin) |
| n8n | `https://n8n.wait4fan.app` (account owner) |
| Server | `ssh root@IP_SERVER` (chiave SSH) |
| Coolify se il dominio non va | tunnel: `ssh -L 8000:localhost:8000 root@IP_SERVER` poi `http://localhost:8000` |
| Password admin Coolify dimenticata | via SSH: `docker exec -ti coolify sh -c "php artisan root:reset-password"` |

### Da tenere nel password manager

- IP del server
- Password admin Coolify e owner n8n
- Contenuto di `/data/coolify/source/.env` (chiavi interne di Coolify)
- Valore di `SERVICE_PASSWORD_64_N8NKEY` (chiave di cifratura n8n: se si perde, le credenziali salvate in n8n diventano illeggibili)

---

## 7. Troubleshooting rapido

| Problema | Causa probabile | Cosa fare |
|---|---|---|
| "Connessione non privata" | certificato non ancora emesso | aspettare 2–3 min; nuvoletta Cloudflare grigia; porta 80 aperta; nessun dominio extra (es. `www.`) senza DNS |
| "404 page not found" sull'IP | Traefik sulla porta 80 senza sito | normale; usare i domini |
| `Security violation detected` | import bloccato o uso di `__qualcosa__` | aprire il dettaglio errore; verificare allowlist: `docker exec $(docker ps -qf name=task-runners) cat /etc/n8n-task-runners.json \| grep -i allow` |
| `unterminated string literal` | copia-incolla sporco nel nodo | Ctrl+A, Canc, incollare di nuovo solo il codice |
| `Task request timed out after 60 seconds` | runner non connesso | versioni n8n/runner diverse, oppure token/broker URI errati |
| `ModuleNotFoundError` | pacchetto non installato | aggiungerlo al Dockerfile (sezione 4) |
| Comando `docker` non riconosciuto | lanciato sul PC Windows | prima `ssh root@IP_SERVER`, poi il comando |

### Comandi utili (sul server)

```bash
# stato dei container
docker ps --format "table {{.Names}}\t{{.Status}}"

# log del runner
docker logs $(docker ps -qf name=task-runners) --tail 30

# log del proxy (certificati)
docker logs coolify-proxy --tail 30
```
