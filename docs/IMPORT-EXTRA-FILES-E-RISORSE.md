# Import Extra Files + risorse utili

## Import Extra Files (sottotitoli e file extra)

Attivare **sia in Radarr sia in Sonarr**.

| Estensione | Uso |
|------------|-----|
| `.srt` | Sottotitoli testo (i più comuni) |
| `.sub` | Sottotitoli |
| `.idx` | Indice spesso insieme a `.sub` |
| `.nfo` | Info release |
| `.txt` | Note |

### Radarr

1. Settings → Media Management
2. Import Extra Files: **ON**
3. Estensioni: `srt,sub,idx,nfo,txt`
4. Save

### Sonarr

1. Settings → Media Management
2. Import Extra Files: **ON**
3. Estensioni: `srt,sub,idx,nfo,txt`
4. Save

### Import Extra Files vs Bazarr

- **Import Extra Files** = tiene i sottotitoli già presenti nel download
- **Bazarr** = scarica i sottotitoli se mancano (servizio opzionale da aggiungere dopo)

---

## Custom Format regex (italiano, case-insensitive)

Usare in Radarr e Sonarr:

**Italiano** (score +100):

```regex
(?i)\b(ita|italian|italiano|subita|multi)\b
```

**Non Italiano** (score -200):

```regex
(?i)\b(eng|english)\b
```

Il prefisso `(?i)` rende la ricerca **case-insensitive** (ita = ITA = Italiano = MULTI, ecc.).

`MULTI` spesso indica dual audio. Per solo italiano “puro” togliere `multi` dalla regex.

---

## Video guida

https://youtu.be/-PQtE6Nb0Cw

Risorsa di supporto con spiegazioni sullo stack ARR / media server.  
Questo repository usa **Jellyfin** + **Seerr** per richieste e riproduzione.
