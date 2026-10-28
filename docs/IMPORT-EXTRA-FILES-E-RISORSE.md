# Import Extra Files + risorse utili

## Import Extra Files (sottotitoli e file extra)

Nella guida principale **non era ancora impostato**. Va attivato **sia in Radarr sia in Sonarr**.

Serve a far importare, insieme al video, anche i file extra che spesso arrivano nel torrent:

| Estensione | Cosa è |
|------------|--------|
| `.srt` | Sottotitoli testo (i più comuni) |
| `.sub` | Sottotitoli (a volte con `.idx`) |
| `.idx` | Indice per alcuni `.sub` |
| `.nfo` | Info sulla release |
| `.txt` | Note varie |

Nota: l'estensione corretta dei sottotitoli di testo è di solito **`.srt`**, non `.str`.

### Radarr – Import Extra Files (passo-passo)

1. Apri Radarr nel browser.
2. Vai su **Settings** → **Media Management**.
3. Scorri fino alla sezione **Importing** (o **Import Extra Files**).
4. Attiva l'interruttore **Import Extra Files** (deve essere ON).
5. Nel campo **Import Extra Files** (lista estensioni) scrivi esattamente:
   ```
   srt,sub,idx,nfo,txt
   ```
   (senza punti, separate da virgola, senza spazi oppure con spazi: entrambe le forme di solito funzionano; forma consigliata senza spazi)
6. Clicca **Save** in alto a destra.

Così, quando Radarr sposta il film in `Film/Nome Film (Anno)/`, porta con sé anche eventuali sottotitoli e file `.nfo` presenti nella cartella del download.

### Sonarr – Import Extra Files (passo-passo)

1. Apri Sonarr nel browser.
2. Vai su **Settings** → **Media Management**.
3. Scorri fino alla sezione **Importing** (o **Import Extra Files**).
4. Attiva l'interruttore **Import Extra Files** (deve essere ON).
5. Nel campo delle estensioni scrivi esattamente:
   ```
   srt,sub,idx,nfo,txt
   ```
6. Clicca **Save** in alto a destra.

Stesso comportamento di Radarr, ma per gli episodi nelle cartelle `Serie TV/Nome Serie (Anno)/Season XX/`.

### Differenza con Bazarr

- **Import Extra Files** = tiene i sottotitoli **già presenti** nel torrent al momento dell'import.
- **Bazarr** = **scarica** i sottotitoli da internet se mancano (dopo l'import).

Le due cose si completano: attiva Import Extra Files ora; Bazarr è opzionale in un secondo momento.

---

## Video guida consigliato

Video con molte spiegazioni e consigli utili sullo stack media (ARR + server):

**https://youtu.be/-PQtE6Nb0Cw**

Consigliato come risorsa di supporto oltre a questo README: mostra flussi, impostazioni e trucchi pratici.  
Lo stack di questo repository resta basato su **Jellyfin** (non Plex): dove il video parla di Plex, per noi vale Jellyfin + Jellyseerr.
