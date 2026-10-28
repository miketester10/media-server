# Media Server - Stack completo con Jellyfin

Stack Docker completo per gestire automaticamente **film** e **serie TV** con audio in **italiano** e qualità minima **1080p**.

## Servizi inclusi

| Servizio         | Cosa fa                                                                 | Porta interna |
|------------------|-------------------------------------------------------------------------|---------------|
| **Jellyfin**     | Media server (riproduzione)                                             | 8096          |
| **qBittorrent**  | Client torrent (download)                                               | 8080          |
| **Radarr**       | Gestione automatica dei film                                            | 7878          |
| **Sonarr**       | Gestione automatica delle serie TV                                      | 8989          |
| **Prowlarr**     | Gestore centralizzato degli indexer                                     | 9696          |
| **Jellyseerr**   | Interfaccia per richiedere film e serie                                 | 5055          |
| **FlareSolverr** | Risolve i controlli Cloudflare (“Non sono un robot”) degli indexer     | 8191          |

Tutte le interfacce web sono accessibili **solo** attraverso la rete Docker chiamata `proxy-manager` (esempio: Nginx Proxy Manager).  
Sull'host viene esposta **solo** la porta torrent di qBittorrent (`6881`).  
**FlareSolverr non ha interfaccia web**: lavora solo in background e non va aggiunto a Nginx Proxy Manager.

---

## Come funziona il flusso (spiegato semplice)

1. Apri **Jellyseerr** dal browser.
2. Cerchi un film o una serie e clicchi **Request**.
3. Jellyseerr invia la richiesta a **Radarr** (se è un film) oppure a **Sonarr** (se è una serie).
4. Radarr o Sonarr chiedono a **Prowlarr** di cercare release **in italiano** e **minimo 1080p**.
5. Prowlarr interroga gli indexer che hai configurato.
6. Se un indexer è protetto da Cloudflare, Prowlarr chiede aiuto a **FlareSolverr**, che supera il controllo “Non sono un robot”.
7. Prowlarr restituisce i risultati a Radarr o Sonarr.
8. Radarr o Sonarr scelgono la release migliore e la mandano a **qBittorrent**.
9. qBittorrent scarica il file dentro la cartella `data/downloads/completed`.
10. Quando il download è finito:
    - **Radarr** crea la cartella `Film/Nome Film (Anno)/` e rinomina il file.
    - **Sonarr** crea la cartella `Serie TV/Nome Serie (Anno)/Season 01/` e rinomina gli episodi.
11. **Jellyfin** vede i nuovi file e li rende disponibili per la visione.

Dopo la configurazione iniziale tutto avviene in automatico.

---

## Struttura delle cartelle sul disco

```
media-server/
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
├── AGENT.md
│
├── config/                     # Configurazioni dei servizi (non versionate su Git)
│   ├── jellyfin/
│   ├── qbittorrent/
│   ├── radarr/
│   ├── sonarr/
│   ├── prowlarr/
│   └── jellyseerr/
│
└── data/
    ├── downloads/              # Dove scarica qBittorrent
    │   ├── incomplete/         # File ancora in download
    │   └── completed/          # File download completati
    │
    └── media/                  # Libreria definitiva per Jellyfin
        ├── Film/               # Film organizzati da Radarr
        │   └── Nome Film (2024)/  # Esempio cartella con anno
        └── Serie TV/           # Serie organizzate da Sonarr
            └── Nome Serie (2020)/
                └── Season 01/
```

**Importante:** le cartelle principali dei film e delle serie includono **sempre l'anno** nel nome.

Esempi reali:
- Film: `Film/Il Padrino (1972)/`
- Serie: `Serie TV/Breaking Bad (2008)/Season 01/`

---

## Installazione passo-passo

### Passo 1 - Prerequisiti

Devi avere installati:
- Docker
- Docker Compose

Crea la rete Docker esterna (esegui una sola volta):

```bash
docker network create proxy-manager
```

Se la rete esiste già, il comando darà un avviso: è normale, puoi continuare.

### Passo 2 - Clona il repository

```bash
git clone https://github.com/miketester10/media-server.git
cd media-server
```

### Passo 3 - Crea le cartelle (se non sono già presenti)

```bash
mkdir -p config/jellyfin
mkdir -p config/qbittorrent
mkdir -p config/radarr
mkdir -p config/sonarr
mkdir -p config/prowlarr
mkdir -p config/jellyseerr

mkdir -p data/downloads/incomplete
mkdir -p data/downloads/completed
mkdir -p data/media/Film
mkdir -p "data/media/Serie TV"

touch config/jellyfin/.gitkeep
touch config/qbittorrent/.gitkeep
touch config/radarr/.gitkeep
touch config/sonarr/.gitkeep
touch config/prowlarr/.gitkeep
touch config/jellyseerr/.gitkeep
touch data/downloads/.gitkeep
touch data/downloads/incomplete/.gitkeep
touch data/downloads/completed/.gitkeep
touch data/media/Film/.gitkeep
touch "data/media/Serie TV/.gitkeep"
```

### Passo 4 - Configura il file `.env`

Apri il file `.env` con un editor di testo:

```bash
nano .env
```

Contenuto da impostare:

```env
PUID=1001
PGID=1001
TZ=Europe/Rome
CONFIG_DIR=./config
DATA_DIR=./data
```

Per sapere il tuo PUID e PGID reali esegui:

```bash
id
```

Sostituisci `1001` con i numeri che vedi (esempio: `uid=1000(ubuntu) gid=1000(ubuntu)` → usa `1000`).

Salva e chiudi il file.

### Passo 5 - Avvia tutti i container

```bash
docker compose up -d
```

Attendi qualche secondo. Controlla che siano tutti avviati (deve comparire anche `flaresolverr`):

```bash
docker compose ps
```

### Passo 6 - Configura Nginx Proxy Manager (o il tuo reverse proxy)

Crea un **Proxy Host** per ogni servizio che ha interfaccia web.  
**Non** creare un Proxy Host per FlareSolverr (non ha interfaccia web).

Usa questi valori:

| Servizio     | Hostname interno (nel campo Forward Hostname / IP) | Porta | Esempio dominio da usare        |
|--------------|----------------------------------------------------|-------|---------------------------------|
| Jellyfin     | jellyfin                                           | 8096  | jellyfin.tuodominio.com         |
| qBittorrent  | qbittorrent                                        | 8080  | qbittorrent.tuodominio.com      |
| Radarr       | radarr                                             | 7878  | radarr.tuodominio.com           |
| Sonarr       | sonarr                                             | 8989  | sonarr.tuodominio.com           |
| Prowlarr     | prowlarr                                           | 9696  | prowlarr.tuodominio.com         |
| Jellyseerr   | jellyseerr                                         | 5055  | requests.tuodominio.com         |

Dopo aver creato i Proxy Host puoi accedere a ogni servizio dal browser usando i domini che hai scelto.

---

## Configurazione dettagliata di ogni servizio

Segui **esattamente** l'ordine qui sotto. Non saltare passaggi.

---

### 1. Configurazione di qBittorrent

1. Apri nel browser l'indirizzo di qBittorrent (esempio: `https://qbittorrent.tuodominio.com`).
2. Al primo accesso imposta username e password (ricordali, ti serviranno dopo).
3. Clicca sull'icona dell'**ingranaggio** in alto a destra (Impostazioni).
4. Vai nella sezione **Download**:
   - Nel campo **Cartella di salvataggio predefinita** scrivi esattamente: `/downloads/completed`
   - Spunta l'opzione per tenere i file incompleti in un'altra cartella.
   - Nel campo della cartella incompleti scrivi esattamente: `/downloads/incomplete`
5. Vai nella sezione **Connessione**:
   - Porta di ascolto: `6881`
6. Vai nella sezione **Web UI**:
   - Controlla che username e password siano quelli che hai scelto.
7. Clicca **Salva** in basso a destra.

---

### 2. Configurazione di Prowlarr

#### 2.1 API Key e indexer di base

1. Apri nel browser l'indirizzo di Prowlarr (esempio: `https://prowlarr.tuodominio.com`).
2. Vai su **Settings** (icona ingranaggio) → **General**.
3. Nella sezione **Security** trova il campo **API Key**.
4. Copia l'API Key e salvala in un blocco note: ti servirà tra poco.
5. Vai su **Indexers** (menu a sinistra).
6. Clicca **+ Add Indexer**.
7. Cerca e aggiungi gli indexer che vuoi usare.
8. Per ogni indexer che aggiungi:
   - Il campo **Base URL** è già compilato. Se non funziona, scegli un dominio alternativo dal menu a tendina.
   - Imposta una **Priority** più alta (numero più basso = priorità maggiore) per quelli che di solito hanno buone release in italiano.
   - Salva l'indexer.
9. Per ora non collegare ancora Radarr e Sonarr: lo faremo dopo averli configurati.

#### 2.2 Configurazione di FlareSolverr (obbligatoria per massima compatibilità)

Molti indexer mostrano il controllo Cloudflare (“Verifica che sei un umano” / “Non sono un robot”).  
Prowlarr da solo non può superare quel controllo. **FlareSolverr** lo fa al posto suo.

1. In Prowlarr vai su **Settings** (icona ingranaggio) → **Indexers**.
2. Scorri verso il basso fino alla sezione **FlareSolverr Servers** (o **FlareSolverr**).
3. Clicca il pulsante **+** per aggiungere un server FlareSolverr.
4. Compila i campi così:
   - **Name**: `FlareSolverr`
   - **Tags**: scrivi la parola `flaresolverr` e premi Invio sulla tastiera (così crei il tag)
   - **Host**: `http://flaresolverr:8191`
5. Clicca **Test**. Deve diventare verde / successo.
6. Clicca **Save**.

#### 2.3 Assegnare FlareSolverr agli indexer protetti da Cloudflare

1. Vai su **Indexers** (menu a sinistra).
2. Apri un indexer (clicca sul nome o sull'icona di modifica).
3. Nel campo **Tags** aggiungi il tag `flaresolverr` (quello creato al punto precedente).
4. Clicca **Test** sull'indexer.
5. Se il test diventa verde, clicca **Save**.
6. Se il test è ancora rosso:
   - Prova a cambiare il **Base URL** scegliendo un altro dominio dal menu a tendina.
   - Ripeti il Test.
7. Ripeti i punti da 2 a 6 per **ogni** indexer che fallisce a causa di Cloudflare.

**Regola pratica:**
- Test verde **senza** tag `flaresolverr` → lascia senza tag.
- Test rosso / errore Cloudflare → aggiungi il tag `flaresolverr` e riprova.

Non è obbligatorio mettere il tag su tutti gli indexer: solo su quelli che ne hanno bisogno.

---

### 3. Configurazione di Radarr (Film)

#### 3.1 Media Management (cartelle e nomi con anno)

1. Apri nel browser l'indirizzo di Radarr (esempio: `https://radarr.tuodominio.com`).
2. Vai su **Settings** → **Media Management**.
3. Nella sezione **Root Folders** clicca **Add Root Folder**.
4. Nel percorso scrivi esattamente: `/movies`
5. Clicca **Ok**.
6. Scorri fino alla sezione **Movie Naming**.
7. Attiva l'interruttore **Rename Movies** (deve essere verde / ON).
8. Attiva anche **Replace Illegal Characters**.
9. Nel campo **Standard Movie Format** scrivi esattamente:
   ```
   {Movie Title} ({Release Year})
   ```
10. Nel campo **Movie Folder Format** scrivi esattamente:
    ```
    {Movie Title} ({Release Year})
    ```
11. Clicca **Save** in alto a destra.

Risultato: ogni film avrà una cartella del tipo `Il Padrino (1972)`.

#### 3.2 Quality Profile (solo 1080p)

1. Vai su **Settings** → **Profiles**.
2. Clicca **+ Add** per creare un nuovo profilo.
3. Nome del profilo: `Italiano 1080p`
4. Nella lista delle qualità **disattiva tutto** tranne queste quattro:
   - Bluray-1080p
   - WEBDL-1080p
   - WEBRip-1080p
   - HDTV-1080p
5. Lascia disattivate tutte le qualità 720p, 480p e inferiori.
6. Clicca **Save**.
7. Torna alla lista dei profili e imposta `Italiano 1080p` come profilo **Default** (se c'è l'opzione).

#### 3.3 Custom Formats (preferenza Italiano)

1. Vai su **Settings** → **Custom Formats**.
2. Clicca **+ Add**.
3. Nome: `Italiano`
4. Clicca **+** sotto Conditions.
5. Scegli il tipo **Release Title**.
6. Nel campo Regular Expression scrivi esattamente:
   ```
   \b(ita|italian|iTALiAN|iTA|subita)\b
   ```
7. Lascia **Negate** disattivato e **Required** disattivato.
8. Clicca **Save**.
9. Clicca di nuovo **+ Add** per creare il secondo formato.
10. Nome: `Non Italiano`
11. Clicca **+** sotto Conditions → tipo **Release Title**.
12. Regular Expression:
    ```
    \b(eng|english|sub|subs|subbed)\b
    ```
13. Clicca **Save**.
14. Torna su **Settings** → **Profiles**.
15. Apri il profilo `Italiano 1080p`.
16. Nella sezione **Custom Formats** assegna questi punteggi:
    - `Italiano` → **100**
    - `Non Italiano` → **-200**
17. Nel campo **Minimum Custom Format Score** metti: `50`
18. Nel campo **Upgrade Until Custom Format Score** metti: `100`
19. Clicca **Save**.

#### 3.4 Collegamento a qBittorrent

1. Vai su **Settings** → **Download Clients**.
2. Clicca **+ Add** → scegli **qBittorrent**.
3. Compila i campi così:
   - Name: `qBittorrent`
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: (quello che hai impostato in qBittorrent)
   - Password: (quella che hai impostato in qBittorrent)
   - Category: `radarr`
4. Clicca **Test**. Deve risultare verde / successo.
5. Clicca **Save**.

#### 3.5 Collegamento di Prowlarr a Radarr

1. Torna nell'interfaccia di **Prowlarr**.
2. Vai su **Settings** → **Apps**.
3. Clicca **+ Add Application** → scegli **Radarr**.
4. Compila i campi così:
   - Name: `Radarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: (incolla l'API Key di Radarr)
5. Per trovare l'API Key di Radarr: apri Radarr → **Settings** → **General** → copia l'API Key.
6. Clicca **Test**. Deve essere ok.
7. Clicca **Save**.
8. Nella lista delle App clicca sull'icona di sincronizzazione (Sync App Indexers) se presente.

---

### 4. Configurazione di Sonarr (Serie TV)

#### 4.1 Media Management (cartelle con anno + stagioni)

1. Apri nel browser l'indirizzo di Sonarr (esempio: `https://sonarr.tuodominio.com`).
2. Vai su **Settings** → **Media Management**.
3. Nella sezione **Root Folders** clicca **Add Root Folder**.
4. Nel percorso scrivi esattamente: `/tv`
5. Clicca **Ok**.
6. Scorri fino alla sezione **Episode Naming**.
7. Attiva l'interruttore **Rename Episodes** (deve essere ON).
8. Attiva anche **Replace Illegal Characters**.
9. Nel campo **Standard Episode Format** scrivi esattamente:
   ```
   {Series Title} - S{season:00}E{episode:00} - {Episode Title}
   ```
10. Nel campo **Daily Episode Format** puoi lasciare il valore di default oppure usare lo stesso schema.
11. Nel campo **Season Folder Format** scrivi esattamente:
    ```
    Season {season:00}
    ```
12. Nel campo **Series Folder Format** scrivi esattamente:
    ```
    {Series Title} ({Series Year})
    ```
13. Clicca **Save** in alto a destra.

Risultato: ogni serie avrà una cartella del tipo `Breaking Bad (2008)/Season 01/`.

#### 4.2 Quality Profile (solo 1080p)

1. Vai su **Settings** → **Profiles**.
2. Clicca **+ Add**.
3. Nome del profilo: `Italiano 1080p`
4. Nella lista delle qualità **disattiva tutto** tranne:
   - Bluray-1080p
   - WEBDL-1080p
   - WEBRip-1080p
   - HDTV-1080p
5. Clicca **Save**.
6. Imposta `Italiano 1080p` come profilo di default se l'opzione è disponibile.

#### 4.3 Custom Formats (preferenza Italiano)

1. Vai su **Settings** → **Custom Formats**.
2. Clicca **+ Add**.
3. Nome: `Italiano`
4. Clicca **+** sotto Conditions → tipo **Release Title**.
5. Regular Expression:
   ```
   \b(ita|italian|iTALiAN|iTA|subita)\b
   ```
6. Clicca **Save**.
7. Clicca di nuovo **+ Add**.
8. Nome: `Non Italiano`
9. Condition → Release Title → Regular Expression:
   ```
   \b(eng|english|sub|subs|subbed)\b
   ```
10. Clicca **Save**.
11. Vai su **Settings** → **Profiles**.
12. Apri il profilo `Italiano 1080p`.
13. Nella sezione **Custom Formats** assegna:
    - `Italiano` → **100**
    - `Non Italiano` → **-200**
14. **Minimum Custom Format Score**: `50`
15. **Upgrade Until Custom Format Score**: `100`
16. Clicca **Save**.

#### 4.4 Collegamento a qBittorrent

1. Vai su **Settings** → **Download Clients**.
2. Clicca **+ Add** → scegli **qBittorrent**.
3. Compila i campi così:
   - Name: `qBittorrent`
   - Host: `qbittorrent`
   - Port: `8080`
   - Username: (quello di qBittorrent)
   - Password: (quella di qBittorrent)
   - Category: `sonarr`
4. Clicca **Test**. Deve essere ok.
5. Clicca **Save**.

#### 4.5 Collegamento di Prowlarr a Sonarr

1. Torna nell'interfaccia di **Prowlarr**.
2. Vai su **Settings** → **Apps**.
3. Clicca **+ Add Application** → scegli **Sonarr**.
4. Compila i campi così:
   - Name: `Sonarr`
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key: (incolla l'API Key di Sonarr)
5. Per trovare l'API Key di Sonarr: apri Sonarr → **Settings** → **General** → copia l'API Key.
6. Clicca **Test**. Deve essere ok.
7. Clicca **Save**.
8. Se presente, clicca Sync App Indexers.

---

### 5. Configurazione di Jellyseerr

1. Apri nel browser l'indirizzo di Jellyseerr (esempio: `https://requests.tuodominio.com`).
2. Segui la procedura guidata iniziale.
3. Quando chiede di collegare **Jellyfin**:
   - URL: `http://jellyfin:8096`
   - Inserisci username e password di un utente **amministratore** di Jellyfin.
   - Clicca Test / Continua.
4. Quando chiede di collegare **Radarr**:
   - URL: `http://radarr:7878`
   - API Key: (quella di Radarr, la trovi in Radarr → Settings → General)
   - Root Folder: seleziona `/movies`
   - Quality Profile: seleziona `Italiano 1080p`
   - Clicca Test e poi Salva.
5. Quando chiede di collegare **Sonarr**:
   - URL: `http://sonarr:8989`
   - API Key: (quella di Sonarr, la trovi in Sonarr → Settings → General)
   - Root Folder: seleziona `/tv`
   - Quality Profile: seleziona `Italiano 1080p`
   - Clicca Test e poi Salva.
6. Completa la procedura guidata fino alla fine.

---

### 6. Configurazione di Jellyfin

1. Apri nel browser l'indirizzo di Jellyfin (esempio: `https://jellyfin.tuodominio.com`).
2. Se è la prima volta, completa la procedura di setup iniziale (crea utente admin, ecc.).
3. Vai su **Dashboard** → **Libraries** (o **Librerie**).
4. Clicca **Add Media Library** (Aggiungi libreria).
5. Content type: **Movies** (Film).
6. Display name: `Film`
7. Clicca **+** per aggiungere una cartella e seleziona / scrivi: `/media/Film`
8. Conferma e salva.
9. Clicca di nuovo **Add Media Library**.
10. Content type: **Shows** (Serie TV).
11. Display name: `Serie TV`
12. Aggiungi la cartella: `/media/Serie TV`
13. Conferma e salva.
14. (Opzionale) Vai nelle impostazioni della libreria e abilita la scansione automatica periodica.

---

## Test finale (verifica che tutto funzioni)

### Test film

1. Apri **Jellyseerr**.
2. Cerca un film qualsiasi.
3. Clicca **Request**.
4. Apri **Radarr** → vai su **Activity** o **Queue**: deve comparire il film in coda.
5. Apri **qBittorrent**: deve partire un download con categoria `radarr`.
6. Quando il download finisce, controlla sul disco che esista una cartella del tipo:
   ```
   data/media/Film/Nome Film (Anno)/
   ```
7. Apri **Jellyfin**: il film deve apparire nella libreria Film (eventualmente forza una scansione della libreria).

### Test serie TV

1. Apri **Jellyseerr**.
2. Cerca una serie qualsiasi.
3. Clicca **Request** (puoi richiedere solo la prima stagione per il test).
4. Apri **Sonarr** → **Activity** / **Queue**: deve comparire la serie.
5. Apri **qBittorrent**: deve partire un download con categoria `sonarr`.
6. Quando finisce, controlla sul disco:
   ```
   data/media/Serie TV/Nome Serie (Anno)/Season 01/
   ```
7. Apri **Jellyfin**: la serie deve apparire nella libreria Serie TV.

### Test FlareSolverr (opzionale ma consigliato)

1. In Prowlarr apri un indexer a cui hai assegnato il tag `flaresolverr`.
2. Clicca **Test**.
3. Se diventa verde, FlareSolverr sta funzionando correttamente.

---

## Comandi utili

```bash
# Avviare tutto
docker compose up -d

# Fermare tutto
docker compose down

# Aggiornare le immagini e riavviare
docker compose pull
docker compose up -d

# Vedere i log di un servizio (esempio Radarr)
docker compose logs -f radarr

# Vedere i log di Sonarr
docker compose logs -f sonarr

# Vedere i log di FlareSolverr
docker compose logs -f flaresolverr

# Vedere lo stato dei container
docker compose ps
```

---

## Note importanti

- Le cartelle `config/` e `data/` contengono dati generati e **non** vengono caricate su GitHub (grazie al file `.gitignore`).
- I file `.gitkeep` servono solo a far comparire le cartelle vuote su GitHub.
- Cambia sempre `PUID` e `PGID` nel file `.env` con i valori del tuo utente di sistema.
- La rete Docker `proxy-manager` deve esistere **prima** di avviare `docker compose up -d`.
- I nomi delle cartelle dei film e delle serie includono **sempre l'anno** grazie alle impostazioni di naming di Radarr e Sonarr.
- **FlareSolverr non ha interfaccia web**: non va aggiunto a Nginx Proxy Manager. Si collega solo dentro Prowlarr.
- Non serve installare Cloudflare sul server. Cloudflare è solo la protezione usata da alcuni siti; FlareSolverr la supera automaticamente.
