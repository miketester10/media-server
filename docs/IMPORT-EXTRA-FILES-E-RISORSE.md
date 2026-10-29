# Import Extra Files + risorse

## Import Extra Files

In **Radarr** e **Sonarr** → Settings → Media Management:

- Import Extra Files: **ON**
- Estensioni: `srt,sub,idx,nfo,txt`

## Custom Format regex (solo italiano)

```regex
(?i)\b(ita|italian|italiano|subita)\b
```

Score +100. **Senza** `multi`.

Non Italiano:

```regex
(?i)\b(eng|english)\b
```

Score -200.

## Video guida

https://youtu.be/-PQtE6Nb0Cw

Vedi anche `docs/VIOLARR-E-REGEX.md` per Violarr.
