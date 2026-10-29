# Rimuovere automaticamente i file da qBittorrent dopo l'import

Obiettivo: quando Radarr o Sonarr hanno importato il film/episodio in `media/Film` o `media/Serie TV`, **cancellare da qBittorrent** torrent + file in `downloads`.

Così non seedi (o seedi pochissimo) e non accumuli file doppi in download.

---

## Prima di iniziare

1. Radarr e Sonarr devono già essere collegati a qBittorrent (Test verde).
2. Le categorie consigliate:
   - Radarr → category `radarr`
   - Sonarr → category `sonarr`
3. Non cancellare a mano i torrent **prima** che l'import sia completato (History / Activity verdi).

---

## Parte A – Radarr (film)

### Passo 1 – Apri Radarr
Apri nel browser l'indirizzo di Radarr.

### Passo 2 – Download Clients
1. Clicca **Settings** (ingranaggio).
2. Clicca **Download Clients**.

### Passo 3 – Apri qBittorrent
Nella lista clicca sul client **qBittorrent** (quello che usi già).

### Passo 4 – Completed Download Handling
Nella finestra di modifica cerca la sezione **Completed Download Handling** (o opzioni simili sul client).

Imposta così:

| Opzione | Valore |
|---------|--------|
| **Enable** (Completed Download Handling) | **ON** (attivo) |
| **Remove** | **ON** (attivo) |

Significato:
- **Enable** = Radarr gestisce i download completati (import automatico).
- **Remove** = dopo un import riuscito, Radarr chiede a qBittorrent di **rimuovere il torrent e i file** dalla cartella download.

### Passo 5 – Controlla categoria e path
Verifica ancora:

- Host: `qbittorrent`
- Port: `8080`
- Category: `radarr`
- Username / Password: quelli di qBittorrent

### Passo 6 – Test e Save
1. Clicca **Test** → deve essere verde.
2. Clicca **Save**.

### Passo 7 (consigliato) – Media Management
1. Settings → **Media Management**
2. Se usi hardlink sullo stesso disco: **Use Hardlinks instead of Copy** = ON  
   (se poi Remove cancella il path in downloads, in `media/Film` il file resta comunque se era hardlink o se era già stato copiato/spostato correttamente).
3. **Import Extra Files** = ON, estensioni: `srt,sub,idx,nfo,txt`
4. Save

---

## Parte B – Sonarr (serie TV)

### Passo 1 – Apri Sonarr
Apri nel browser l'indirizzo di Sonarr.

### Passo 2 – Download Clients
1. Clicca **Settings**.
2. Clicca **Download Clients**.

### Passo 3 – Apri qBittorrent
Clicca sul client **qBittorrent**.

### Passo 4 – Completed Download Handling
Stesse impostazioni di Radarr:

| Opzione | Valore |
|---------|--------|
| **Enable** (Completed Download Handling) | **ON** |
| **Remove** | **ON** |

### Passo 5 – Controlla categoria
- Host: `qbittorrent`
- Port: `8080`
- Category: `sonarr`
- Username / Password corretti

### Passo 6 – Test e Save
1. **Test** → verde
2. **Save**

### Passo 7 (consigliato) – Media Management
1. Settings → **Media Management**
2. Use Hardlinks instead of Copy = ON (se stesso filesystem)
3. Import Extra Files = ON → `srt,sub,idx,nfo,txt`
4. Save

---

## Parte C – qBittorrent (opzionale ma utile)

Non è obbligatorio, ma evita confusione:

1. Apri qBittorrent
2. Impostazioni → **BitTorrent** (o Download)
3. Puoi lasciare il seed libero: tanto Radarr/Sonarr rimuoveranno il torrent dopo l'import
4. Assicurati che le cartelle siano ancora:
   - Completati: `/downloads/completed` (o `/data/downloads/completed` se usi mount unificato)
   - Incompleti: `/downloads/incomplete`

Non attivare “elimina torrent all’arrivo al ratio” in conflitto con Radarr se non sai cosa fai: lascia che **Remove** di Radarr/Sonarr faccia il lavoro dopo l’import.

---

## Parte D – Test completo

### Test film
1. Da **Seerr** (o da Radarr) richiedi/aggiungi un film di prova.
2. In **qBittorrent** deve comparire un torrent con categoria `radarr`.
3. Attendi fine download.
4. In **Radarr** → Activity / History: deve risultare **Imported**.
5. Controlla sul disco: esiste `data/media/Film/Nome Film (Anno)/...`
6. Dopo poco, in **qBittorrent** il torrent **scompare** e i file sotto `downloads/completed` relativi a quel film non ci sono più.

### Test serie
1. Richiedi una serie / episodio.
2. Categoria qBit: `sonarr`
3. Dopo import in `data/media/Serie TV/Nome (Anno)/Season XX/`
4. Il torrent viene rimosso da qBittorrent.

Se il file è in `media/` ma il torrent resta in qBit:
- ricontrolla che **Remove** sia ON
- ricontrolla Test del download client
- guarda Radarr/Sonarr → System → Logs per errori di rimozione

---

## Cosa non fa questa guida

- **Cleanuparr** non è necessario solo per “rimuovi dopo import”. Serve se vuoi anche pulire download bloccati, malware, code ferme.
- Seed prolungato: con **Remove = ON** non tieni i torrent a seedare a lungo.

---

## Riepilogo

| App | Dove | Cosa attivare |
|-----|------|----------------|
| Radarr | Settings → Download Clients → qBittorrent | Completed Download Handling **ON**, Remove **ON** |
| Sonarr | Settings → Download Clients → qBittorrent | Completed Download Handling **ON**, Remove **ON** |

Ordine: **download → import in media → cancellazione da qBittorrent**.
