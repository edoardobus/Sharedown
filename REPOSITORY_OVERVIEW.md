# Spiegazione del repository Sharedown

## Cos'è
Sharedown è un'app desktop **Electron** per scaricare video da SharePoint/OneDrive, con interfaccia grafica e opzioni di login automatico.

## Tecnologie chiave
- **Electron**: processo main (`app.js`) + processo renderer (UI in `sharedown/`).
- **Node.js**: filesystem, processi esterni, IPC.
- **Puppeteer**: apertura browser e automazione login/raccolta URL video.
- **FFmpeg** (via `fessonia`) e **yt-dlp**: motori di download.
- **Bootstrap** + **Font Awesome**: UI.
- **keytar**: salvataggio credenziali nel password manager del sistema.
- **axios** + **iso8601-duration**: chiamate HTTP e parsing durata video.
- **electron-builder**: packaging desktop multi-piattaforma.

## Struttura del codice

### Root del progetto
- `package.json`: dipendenze, script (`start`, `pack`, `dist`) e configurazione build.
- `app.js`: processo main Electron, finestre, menu applicativo, IPC verso renderer.
- `preload.js`: bridge sicuro (`contextBridge`) che espone API `window.sharedown` al frontend.
- `puppeteer.config.js`: cache/config Puppeteer.
- `buildHooks/`: hook di build per preparare Chromium e metadati versione.

### Cartella `sharedown/` (frontend e logica UI)
- `sharedown.html` + `sharedown.css`: layout e stile principali.
- `sharedown.js`: orchestrazione UI, coda download, stato globale, eventi utente.
- `utils.js`: helper applicativi (validazione URL, output path, integrazione API preload).
- `uiUtils.js`: gestione dinamica componenti UI (moduli login, toggle opzioni).
- `downloadQue.js`: modello coda download.
- `video.js`: modello entità video.
- `messageBoxType.js`, `timeoutMessage.js`: tipi messaggi e feedback temporaneo.
- `about.*`: finestra “About”.
- `loginModules/`: moduli login estensibili.

### Cartella `sharedown/loginModules/`
- `loginModule.js`: registry + selezione modulo login attivo.
- `Basic.js`: classe base/contratto per implementare moduli personalizzati.
- `SimpleUniversity*.js`, `Unina.js`, `UniParma.js`, `UniRoma3.js`: implementazioni concrete.

## Come è organizzato il flusso applicativo
1. `app.js` crea la finestra e carica `sharedown/sharedown.html`.
2. `preload.js` espone API native a `window.sharedown`.
3. `sharedown.js` usa quell’API per:
   - leggere/salvare stato e impostazioni,
   - ottenere metadati video o URL da cartelle SharePoint via Puppeteer,
   - avviare download con FFmpeg o yt-dlp,
   - aggiornare progresso e UI.
4. I moduli di login (`loginModules/`) astraggono i vari flussi di autenticazione.

## Come vengono gestite URL con lettere accentate
- In input, Sharedown normalizza gli URL tramite l'oggetto nativo `URL` (`sharedown/utils.js`, `setAsWebPlayerURL`) prima di metterli in coda.
- Questo passaggio serializza l'URL in forma valida (`urlObj.href`), quindi eventuali caratteri non ASCII nel path/query (incluse lettere accentate) vengono trattati come URL-encoded.
- Nel flusso “direct”, prima di chiamare `yt-dlp`, il link viene di nuovo ricostruito con `new URL(...).toString()` (`preload.js`), mantenendo la forma codificata.
- Per l'import da cartelle SharePoint, i percorsi file arrivano dall'API come `ServerRelativeUrl` e vengono composti in URL complete (`preload.js`, `_getVideoURLsInFold`), poi riutilizzati dal downloader.
- Eccezione specifica gestita manualmente: il carattere `#` viene convertito in `%23` in fase di inserimento URL (`sharedown/sharedown.js`) per evitare troncamenti/interpretazioni errate del fragment.

## Build e distribuzione
- `npm start`: avvio locale.
- `npm run pack`: build cartella applicazione.
- `npm run dist`: pacchetti distribuibili.
- Hook build (`buildHooks/beforePack.js`) scarica Chromium adatto alla piattaforma e genera `version.js`.

## Note pratiche sul repository
- Non c'è una suite test vera e propria: lo script `npm test` attualmente fallisce intenzionalmente.
- Il repository è organizzato in modo “feature-oriented” lato UI (coda, video, utility, login modules), mentre l’integrazione con OS/processi esterni è centralizzata in `preload.js`.
