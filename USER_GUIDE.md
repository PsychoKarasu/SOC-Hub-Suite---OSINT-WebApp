# SOC Hub Suite — Guida Operativa

**Versione:** 1.0
**Pubblico:** Analisti SOC del team Brightstar Lottery
**URL produzione:** `https://soc-hub-suite.pages.dev`

---

## 1. Cos'è SOC Hub Suite

SOC Hub Suite è una piattaforma OSINT multi-hub progettata per accelerare le attività di threat intelligence, breach analysis e phishing investigation del team. Aggrega dati da fonti commerciali e gratuite in un'interfaccia unica, con analisi AI integrata.

La suite è composta da **due hub specializzati**:

| Hub | Caso d'uso | Quando usarlo |
|---|---|---|
| **Intel-Hub** | Identity & Breach Intelligence | Verifica se un'email/dominio/utente è coinvolto in data breach noti |
| **Phish-Hub** | Email & Phishing OSINT | Analisi di domini sospetti, validazione email, reputation check, URL scanning |

---

## 2. Accesso

### 2.1 Prerequisiti

Per accedere alla piattaforma serve:

1. **Account email aziendale** `@brightstarlottery.com` (il dominio è whitelistato a livello Cloudflare Access — accessi da altri domini vengono respinti)
2. **Master Key** del SOC (chiave di sblocco unica per tutta la suite, condivisa dal team SOC)
3. **Browser moderno** (Chrome/Edge/Firefox aggiornati — IE non supportato)

### 2.2 Dove ottenere le credenziali

| Credenziale | Dove |
|---|---|
| Email accesso | Account aziendale già attivo (`nome.cognome@brightstarlottery.com`) |
| Master Key SOC | Bitwarden vault del team → entry `SOC-Hub / Master Key` |
| Eventuali API key personali | Bitwarden personale (solo per analisti con piani provider dedicati) |

> ⚠️ **Mai condividere la Master Key fuori dal vault Bitwarden.** Non incollarla in chat, email, ticket. Se sospetti compromissione, contatta il SOC Lead immediatamente.

### 2.3 Procedura di primo accesso

1. Apri `https://soc-hub-suite.pages.dev`
2. **Cloudflare Access** ti chiede l'email aziendale → ricevi un PIN via email → inseriscilo
3. Ti reindirizza al **portale SOC Hub**
4. Inserisci la **Master Key** dal vault → click **Sblocca**
5. Vedi la dashboard con i due hub disponibili → click sull'hub che ti serve

> 💡 La sessione resta valida per **8 ore** anche tra hub diversi. Non devi rifare login passando da Intel-Hub a Phish-Hub.

### 2.4 Logout

Click sull'avatar in alto a destra → **Logout**, oppure chiudi tutti i tab. La sessione viene invalidata automaticamente dopo 8h di inattività.

---

## 3. Architettura (overview)

```
┌─────────────────────────────────────────────────────┐
│  Browser analista (HTTPS, sessione cifrata)         │
└──────────────────────┬──────────────────────────────┘
                       │
         ┌─────────────▼──────────────┐
         │   Cloudflare Access        │  ← email PIN, dominio whitelist
         │   (zero-trust gateway)     │
         └─────────────┬──────────────┘
                       │
         ┌─────────────▼──────────────┐
         │   Cloudflare Pages         │  ← static SPA (HTML+JS+CSS)
         │   soc-hub-suite.pages.dev  │
         └──────┬───────────────┬─────┘
                │               │
        ┌───────▼──────┐ ┌──────▼─────────────┐
        │  Intel-Hub   │ │  Phish-Hub         │
        └───────┬──────┘ └──────┬─────────────┘
                │               │
        ┌───────▼───────────────▼────────────┐
        │  CORS Proxy (Render)               │
        │  intel-hub-proxy.onrender.com      │
        └───────┬────────────────────────────┘
                │
    ┌───────────▼────────────────────────────┐
    │  Provider OSINT (IntelX, HIBP,         │
    │  BreachDirectory, Hunter, VT, ...)     │
    └────────────────────────────────────────┘
```

**Punti chiave:**
- **Frontend statico** su Cloudflare Pages — no backend custom, zero superficie d'attacco server-side
- **Cifratura client-side** — la config con le API keys è cifrata in `localStorage` con AES-GCM derivato dalla Master Key (PBKDF2 600k iterazioni). Senza Master Key i dati sono illeggibili anche con accesso fisico al browser
- **Autenticazione a doppio fattore implicito** — Cloudflare Access (email PIN) + Master Key (knowledge factor)
- **AI analysis on-demand** — il modello Claude (`claude-haiku-4-5`) viene chiamato solo se l'analista richiede l'analisi sui risultati, mai automaticamente
- **Sanitizzazione output** — DOMPurify pulisce ogni markdown prodotto da Claude prima del rendering

---

## 4. Intel-Hub — Guida operativa

### 4.1 Cosa fa

Intel-Hub interroga in parallelo cinque provider per dirti se un identificativo (email, username, dominio, hash password) è apparso in data breach noti, leak, dump o database compromessi.

### 4.2 Connettori disponibili

| Provider | Tipo dati | Note |
|---|---|---|
| **IntelX** | Email, dominio, password, IP — accesso a archivi leak | Free tier disponibile, paid per archivio completo |
| **HIBP** (Have I Been Pwned) | Email/dominio in breach pubblici | Limite illimitato (UNLIMITED tag) |
| **BreachDirectory** | Email + password plaintext quando disponibile | RapidAPI, quota mensile |
| **LeakIX** | Servizi esposti, credentials in dump | Free tier con API key |
| **DeHashed** | Email, username, password, IP, telefono — multi-attributo | **Solo paid** — molto completo |

### 4.3 Workflow tipico

1. Click **Intel-Hub** dal portale
2. Inserisci il selector (es. email del target) nella search bar
3. Scegli i provider attivi (di default: tutti quelli con quota disponibile)
4. Click **Search** — l'avatar Sakura entra in modalità ricerca
5. I risultati appaiono come **card per provider** con counters dei match
6. Espandi una card per vedere i dettagli (breach name, date, dati esposti)
7. Click **Analyze with AI** se vuoi un sommario sintetico generato da Claude

### 4.4 Lettura quote

In alto trovi la barra **Limits** che mostra le quote real-time:
- Tag `API` → quota dal provider stesso (sempre veritiera)
- Tag `SESSION` → conteggio chiamate fatte in questa sessione (provider senza quota API)
- Tag `UNLIMITED` → nessun limite di chiamate

> 📊 Se vedi quota in rosso (≥80% consumata), avvisa il SOC Lead per upgrade del piano.

### 4.5 Best practice

- **Email come selector**: il più comune, dà i risultati più ricchi
- **Username**: utile su DeHashed e LeakIX, scarso su HIBP
- **Dominio aziendale**: usa solo su HIBP `/breacheddomain` per panoramica
- **Mai cercare dati di clienti senza autorizzazione documentata** — lo strumento è per threat intel difensiva, non per profilazione

---

## 5. Phish-Hub — Guida operativa

### 5.1 Cosa fa

Phish-Hub è il complemento offensivo: analizza email sospette, domini di phishing, URL malevoli e fornisce un giudizio aggregato sulla loro pericolosità.

### 5.2 Connettori disponibili

| Provider | Tipo dati | Note |
|---|---|---|
| **Hunter.io** | Email pattern di un dominio, MX, sources | Free tier 50 ricerche/mese |
| **EmailRep** | Reputation singola email | No auth, no quota |
| **DNS Recon** | A, MX, TXT, SPF, DMARC, DKIM | Google DoH, illimitato |
| **URLScan.io** | Screenshot e analisi URL sospetti | Free 5000/mese |
| **VirusTotal** | Reputation file/URL/domini, vendor consensus | Free 500/giorno |

### 5.3 Workflow tipico — analisi email sospetta

1. Click **Phish-Hub** dal portale
2. Inserisci l'email/dominio sospetto
3. Click **Search** — Sakura cambia colore (rossa per Phish-Hub)
4. Le card popolano:
   - **EmailRep** → score di reputation (0-100, basso = sospetto)
   - **Hunter** → conferma se l'email pattern è plausibile per quel dominio
   - **DNS Recon** → SPF/DMARC/DKIM mancanti = forte indicatore di spoofing
   - **VT** → vendor che lo flaggano malicious
   - **URLScan** → se ci sono URL nel payload, screenshot e indicators
5. Click **Analyze with AI** → Claude sintetizza un verdetto (legittima / sospetta / phishing confermato) con motivazione

### 5.4 Workflow tipico — analisi URL sospetto

1. Inserisci l'URL completo (`https://...`)
2. URLScan effettua submit + screenshot in ~10s
3. VirusTotal restituisce vendor consensus
4. DNS Recon verifica il dominio (registrazione recente = red flag)

### 5.5 Lettura risultati

- **EmailRep score < 50** → email sospetta, indagare
- **SPF/DMARC failing** → spoofing probabile
- **VT detection > 3 vendors** → malicious confermato
- **URLScan screenshot vuoto/redirect** → domain probabilmente già takedown o cloaking

---

## 6. Lingua interfaccia

In alto a destra c'è il toggle **EN/IT** — switcha tutta l'UI istantaneamente. Persistente per browser/sessione.

---

## 7. Troubleshooting

| Sintomo | Causa probabile | Soluzione |
|---|---|---|
| **"Cloudflare Access denied"** | Email non `@brightstarlottery.com` | Usa account aziendale |
| **"Master Key invalid"** | Chiave digitata male o cambiata | Verifica su Bitwarden, ricopiala |
| **Search lenta (>30s) prima volta del giorno** | Render proxy in cold start | Normale, rifai search dopo 30s |
| **IntelX restituisce 401** | Tier sbagliato (free vs paid) | Contatta SOC Lead per toggle config |
| **Quota in rosso** | Mese al limite | Avvisa SOC Lead per upgrade |
| **Sakura "rotta" a tutto schermo** | Bug rendering CSS | Hard refresh (Ctrl+Shift+R) |
| **AI Analysis non risponde** | Quota Claude esaurita o rate limit | Aspetta 60s e riprova |

Per problemi non risolti, apri ticket interno o ping diretto al SOC Lead con:
- Screenshot dell'errore
- Browser + versione
- Selector usato (oscurato se sensibile)
- Timestamp

---

## 8. Sicurezza & Policy

- **Mai esportare i risultati fuori dalla VPN aziendale**
- **Mai cercare dati personali di colleghi/clienti senza autorizzazione scritta**
- **Mai loggare la Master Key in note locali, terminal, file di lavoro**
- **Sospetto compromissione del browser/sessione?** Logout immediato + rotazione Master Key (responsabilità SOC Lead)
- **Ogni ricerca è loggata** lato Cloudflare Access (audit trail conformità)

---

## 9. Contatti

| Necessità | Contatto |
|---|---|
| Accesso non funzionante | SOC Lead |
| Richiesta upgrade quota provider | SOC Lead |
| Bug funzionali | Issue su `github.com/PsychoKarasu/SOC-Hub-Suite---OSINT-WebApp` |
| Suggerimenti feature | SOC Lead o canale Slack `#soc-hub` |
| Sospetto incidente di sicurezza | SOC Lead **immediatamente** + email a security@ |

---

**Buon hunting. 🛡️**
