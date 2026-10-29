# Rimuovere i file dopo il download (qBittorrent)

## Importante (leggi prima)

**Chi cancella torrent e file da `downloads` è qBittorrent**, non Radarr/Sonarr.

Nelle versioni recenti di Radarr:

- **Completed Download Handling → Enable** = importa il film in `media/Film`
- **Remove Completed** sul client = toglie il download dalla *history* del client
- **Non** è affidabile per cancellare i file dal disco dopo un torrent

Per eliminare davvero **torrent + file** da `data/downloads` usi i **Seeding Limits** di qBittorrent.

Ordine corretto:

1. qBittorrent **scarica**
2. Radarr/Sonarr **importano** in `media/Film` o `Serie TV`
3. Dopo pochi minuti qBittorrent **cancella** torrent e file in downloads
4. Il film/episodio **resta** nella cartella media

---

## Parte A – Impostazione in qBittorrent (obbligatoria)

### Passo 1 – Apri qBittorrent
Apri nel browser l’interfaccia web di qBittorrent.

### Passo 2 – Impostazioni
Clicca l’**ingranaggio** (Options / Impostazioni).

### Passo 3 – Sezione BitTorrent
Nel menu a sinistra (o in alto) apri **BitTorrent**.

### Passo 4 – Seeding Limits
Scorri fino a **Seeding Limits**.

Vedrai qualcosa di simile a:

- When ratio reaches
- When total seeding time reaches
- When inactive seeding time reaches
- then → menu a tendina (`Stop torrent`, ecc.)

### Passo 5 – Valori da impostare

1. **Non** spuntare “When ratio reaches” (lascialo spento), oppure lascialo com’è se non ti interessa.
2. **Spunta** la casella **When total seeding time reaches**.
3. Nel campo numerico accanto scrivi **`5`** (significa 5 minuti).
4. **Non** spuntare “When inactive seeding time reaches” (opzionale: lascialo spento).
5. Nel menu **then** scegli una di queste voci (dipende dalla lingua):
   - **Remove torrent and files**
   - **Rimuovi torrent e file**
   - **Delete torrent and files**

   **Non** lasciare `Stop torrent` / `Ferma torrent`: quello ferma solo il seed e **non** cancella i file.

### Passo 6 – Salva
Clicca **Save** / **Salva** / **OK** in fondo alla finestra.

### Riepilogo valori

| Campo | Valore |
|-------|--------|
| When total seeding time reaches | **spuntato**, valore **5** minuti |
| then | **Remove torrent and files** |

Perché 5 minuti e non 0?  
Radarr/Sonarr devono avere tempo di **importare** il file in `media/` prima che qBittorrent lo cancelli. Con 5 minuti di solito l’import è già finito. Se vedi che a volte cancella troppo presto, alza a **10** minuti.

---

## Parte B – Cosa lasciare in Radarr (necessario ma non cancella i file)

Radarr deve solo **importare**. Non affidarti a Radarr per svuotare `downloads`.

### Passo 1
Apri **Radarr** → **Settings** → **Download Clients**.

### Passo 2
Nella sezione **Completed Download Handling**:

- **Enable** = **spuntato**  
  (“Automatically import completed downloads from download client”)

### Passo 3
Clicca il riquadro **qBittorrent**.

### Passo 4
Se vedi **Remove Completed** e **Remove Failed**, lasciali **spuntati** (puliscono la history; non sostituiscono i Seeding Limits).

### Passo 5
Category: `radarr` · Test verde · **Save**.

---

## Parte C – Cosa lasciare in Sonarr

### Passo 1
Apri **Sonarr** → **Settings** → **Download Clients**.

### Passo 2
**Completed Download Handling → Enable** = **spuntato**.

### Passo 3
Apri **qBittorrent** → Category `sonarr` → Remove Completed/Failed spuntati se presenti → **Test** → **Save**.

---

## Parte D – Test passo-passo

### Film
1. Richiedi un film da Seerr (o Interactive Search in Radarr).
2. In qBittorrent compare il torrent (categoria `radarr`).
3. Aspetta la fine del download.
4. In Radarr → History: deve comparire **Imported**.
5. Sul disco deve esistere:  
   `data/media/Film/Nome Film (Anno)/file.mkv`
6. Aspetta **5–10 minuti**.
7. In qBittorrent il torrent **scompare**.
8. In `data/downloads/completed` i file di quel film **non ci sono più**.
9. In `data/media/Film/...` il file **c’è ancora**.

### Serie
Stesso schema con categoria `sonarr` e cartella  
`data/media/Serie TV/Nome (Anno)/Season XX/`.

---

## Se qualcosa va storto

| Problema | Cosa controllare |
|----------|------------------|
| Il torrent resta in qBit per sempre | Seeding Limits non salvati; “then” è ancora `Stop torrent` |
| Sparisce anche il file in `media/Film` | L’import non era finito prima della cancellazione → alza il tempo a **10** minuti; verifica History = Imported |
| Non importa in media | Completed Download Handling Enable OFF; path sbagliati; categoria diversa da `radarr`/`sonarr` |
| Nella tendina non c’è “Remove torrent and files” | Aggiorna qBittorrent; cerca voci tipo “Delete files” / “Rimuovi con file” |

---

## Cosa non serve per questo obiettivo

- **Cleanuparr**: utile per code bloccate/malware, non obbligatorio solo per “svuota downloads dopo import”.
- Cercare altre opzioni nascoste in Radarr: nella UI recente **non ci sono** seed ratio/time sul client che cancellano i file come qBittorrent.

---

## Riepilogo in una frase

**Radarr/Sonarr importano. qBittorrent (Seeding Limits → dopo 5 minuti → Remove torrent and files) cancella da downloads.**
