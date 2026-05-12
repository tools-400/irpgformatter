# Alternativer Release-Workflow ohne eingecheckte Binärdateien

Ziel: PDF und Update-Site-ZIP nicht mehr im Repo halten, sondern ausschließlich als GitHub-Release-Assets bereitstellen. Das hält die Git-Historie dauerhaft schlank und macht regelmäßige History-Rewrites überflüssig.

## Grundidee

- Quellen (Plugin-Code, PDF-Sources) bleiben im Repo.
- Build der Update-Site und der PDF passiert weiterhin lokal in RDi/Eclipse — ein CI-Build entfällt, weil die RDi-Toolchain dort nicht reproduzierbar abbildbar ist.
- Ein **Ant-Script** im Projekt übernimmt nach dem lokalen Build sowohl das Tagging als auch den Upload der Artefakte direkt an die GitHub-Release-API.
- Ein GitHub-Actions-Workflow ist optional und nur dann nötig, wenn man zusätzliche Server-seitige Schritte will (z. B. Pages-Deploy, Mail-Benachrichtigung).

## Variante A (empfohlen): Ant-Script lädt direkt hoch

Voraussetzungen:
- `gh` CLI installiert und mit `gh auth login` einmalig autorisiert (oder Personal Access Token in einer Datei außerhalb des Repos, gelesen via `<property file="..."/>`). `gh` ist deutlich einfacher als die nackte REST-API.
- Lokal gebaute Artefakte liegen an einem festen Ort, z. B. `build/dist/`.

Beispiel `release.xml` (Skizze):

```xml
<project name="release" default="release">

    <property name="version" value="1.0.0.r"/>
    <property name="tag"     value="v${version}"/>
    <property name="pdf"     value="build/dist/iRpgFormatter for RDi.pdf"/>
    <property name="zip"     value="build/dist/iRpgFormatter for RDi 9.6.0.13+ (v${version} Update Site).zip"/>

    <target name="check">
        <available file="${pdf}" property="pdf.ok"/>
        <available file="${zip}" property="zip.ok"/>
        <fail unless="pdf.ok" message="PDF fehlt: ${pdf}"/>
        <fail unless="zip.ok" message="ZIP fehlt: ${zip}"/>
    </target>

    <target name="tag" depends="check">
        <exec executable="git" failonerror="true">
            <arg value="tag"/>
            <arg value="${tag}"/>
        </exec>
        <exec executable="git" failonerror="true">
            <arg value="push"/>
            <arg value="origin"/>
            <arg value="${tag}"/>
        </exec>
    </target>

    <target name="release" depends="tag">
        <exec executable="gh" failonerror="true">
            <arg value="release"/>
            <arg value="create"/>
            <arg value="${tag}"/>
            <arg value="--title"/>
            <arg value="iRpgFormatter v${version}"/>
            <arg value="--notes"/>
            <arg value="Release ${version}"/>
            <arg file="${pdf}"/>
            <arg file="${zip}"/>
        </exec>
    </target>

</project>
```

Aufruf:

```bash
ant -f release.xml -Dversion=1.0.0.r release
```

Vorteile:
- Lokaler Build und Upload in einem Schritt, keine zwei Systeme zu koordinieren.
- Kein Workflow-Run, keine GitHub-Actions-Minuten verbraucht.
- Funktioniert auch, wenn man kein Tag pushen will (`gh release create` kann Tag implizit anlegen).

Nachteile:
- Releases hängen am lokalen Rechner des Maintainers.
- Token-Handling (bei PAT) muss sauber gelöst sein — niemals einchecken.

## Variante B (Fallback): Manuell über die GitHub-UI

Wenn weder Workflow noch Ant gewünscht: Tag pushen, im Releases-Tab „Draft a new release" wählen, Tag eintragen, Dateien per Drag&Drop hochladen, „Publish" klicken. Voll manuell, kein Setup nötig, aber fehleranfällig (Tippfehler im Tag, vergessene Datei).

## Migration vom aktuellen Stand

Einmalig:
1. **Historie bereinigen** mit `git filter-repo`:
   ```bash
   git filter-repo \
     --path docs/files/rdi8.0/ \
     --path "docs/files/iRpgFormatter for RDi.pdf" \
     --path docs/beta-version/ \
     --invert-paths
   ```
2. **Tags neu setzen** auf die umgeschriebenen Commits (alte Tags werden durch `filter-repo` entfernt).
3. **Force-Push** von `main` und allen Tags:
   ```bash
   git push --force --all
   git push --force --tags
   ```
4. `.gitignore` ergänzen, damit Binärdateien nicht versehentlich wieder eingecheckt werden:
   ```
   docs/files/*.pdf
   docs/files/rdi8.0/*.zip
   docs/beta-version/
   build/
   ```
5. Bestehende Klones (eigene und fremde) müssen neu geklont oder hart resettet werden.
6. `.github/workflows/release.yml` entfernen oder leeren — der Tag-getriggerte Workflow wird durch das Ant-Script ersetzt.

## Was bleibt unverändert

- Bestehende Release-Downloads unter `releases/download/v.../...` funktionieren auch nach dem History-Rewrite weiter — Release-Assets liegen außerhalb der Git-Historie.
- GitHub Pages funktioniert weiter, solange `docs/index.html` und die referenzierten Assets im finalen Tree erhalten bleiben. PDF-Links in der Pages-Site müssen auf die Release-URLs umgestellt werden:
  ```
  https://github.com/tools-400/irpgformatter/releases/latest/download/iRpgFormatter%20for%20RDi.pdf
  ```

## Empfehlung

**Variante A (Ant + `gh`) + einmaliger History-Purge.** Lokaler Build bleibt im RDi-Workflow, Upload ist ein einziger `ant`-Aufruf, das Repo bleibt dauerhaft schlank, und es gibt keinen CI-Workflow mehr zu pflegen.
