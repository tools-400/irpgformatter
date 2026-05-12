# GitHub Actions Workflows

## release.yml — Release-Veröffentlichung

Veröffentlicht bei einem Release-Build die folgenden Dateien als Assets eines GitHub Release:

- `docs/files/iRpgFormatter for RDi.pdf`
- `docs/files/rdi8.0/iRpgFormatter for RDi <RDi-Version>+ (v<VERSION> Update Site).zip`

### Trigger

Der Workflow läuft ausschließlich beim Push eines Tags im Release-Format:

```
v<major>.<minor>.<patch>.r       z. B. v1.0.0.r
```

Beta-Tags (`v1.0.0.b001`, `v1.0.0.b010` usw.) werden bewusst ignoriert.

### Ablauf

1. **Checkout** des Repos.
2. **Version aus dem Tag-Namen ableiten** (`v1.0.0.r` → `1.0.0.r`). Der Tag ist die einzige Quelle der Versionsnummer.
3. **Assets validieren**:
   - Die PDF muss unter `docs/files/iRpgFormatter for RDi.pdf` existieren.
   - Im Verzeichnis `docs/files/rdi8.0/` muss genau ein ZIP existieren, dessen Name auf `(v<VERSION> Update Site).zip` endet. Bei 0 oder mehreren Treffern bricht der Workflow mit klarer Fehlermeldung ab.
4. **GitHub Release erstellen** mit Titel `iRpgFormatter v<VERSION>` und beiden Dateien als Assets (`softprops/action-gh-release`).

### Veröffentlichen

```bash
git tag v1.0.0.r
git push origin v1.0.0.r
```

### Vorbedingung

Vor dem Tag-Push muss das Update-Site-ZIP unter dem Release-Namen im Repo liegen. Heißt das ZIP noch `... (v1.0.0.b010 Update Site).zip`, muss es vorher in `... (v1.0.0.r Update Site).zip` umbenannt und committet werden, sonst schlägt der Workflow fehl.

### Berechtigungen

Der Job läuft mit `contents: write`, was zum Anlegen des Release nötig ist. Es werden keine zusätzlichen Secrets benötigt — `GITHUB_TOKEN` reicht.

### Beta-Builds

Aktuell nicht im Scope. Falls Beta-Tags (`v*.b*`) später ebenfalls automatisch als Pre-Release veröffentlicht werden sollen, lässt sich das über einen zweiten Tag-Filter und `prerelease: true` ergänzen.
