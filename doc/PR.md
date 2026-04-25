# Pull Request: Miglioramento Processo di Ingestion (HTML to Markdown)

## Descrizione
Questa PR si propone di migliorare i contenuti di WordPress inviati alla memoria dichiarativa dello Cheshire Cat. Invece di rimuovere semplicemente tutti i tag HTML (perdendo la struttura semantica), il sistema ora converte il contenuto in **Markdown**.

## Modifiche Principali

### 1. Integrazione Libreria `league/html-to-markdown`
- Aggiunta la dipendenza nel file `composer.json`.
- Questa libreria permette una conversione dell'HTML in Markdown, preservando titoli (`#`), grassetti (`**`), liste e link, elementi fondamentali per il "ragionamento" dei modelli LLM.

### 2. Ottimizzazione Logica di Ingestion (`inc/declarative-memory.php`)
- **Espansione Shortcode**: Utilizzo di `do_shortcode()` per processare e includere il testo generato dai plugin WP.
- **Supporto Paragrafi (`wpautop`)**: Applicazione del filtro automatico di WordPress per garantire che i testi scritti con l'editor classico mantengano la corretta divisione in paragrafi durante la conversione.
- **Configurazione Ottimizzata**:
    - Utilizzo di titoli in stile ATX (`#`).
    - Rimozione automatica di tag tecnici rumorosi (`script`, `style`, `iframe`, `form`).
    - Pulizia degli spazi bianchi e dei ritorni a capo eccessivi per ottimizzare il consumo di token.

### 3. Supporto WooCommerce
- La stessa logica di conversione in Markdown è stata estesa alla **Short Description** dei prodotti, garantendo coerenza tra i vari tipi di post.

## File Modificati
- `composer.json`: Aggiunta dipendenza `league/html-to-markdown`.
- `inc/declarative-memory.php`: Refactoring della funzione `cheshirecat_upload_to_declarative_memory`.

## Note per il Reviewer
Dopo aver applicato la PR eseguire il comando seguente nella root del plugin per installare la nuova libreria:
```bash
composer update
```
È stato implementato un controllo (`class_exists`) che garantisce il funzionamento del plugin tramite fallback al vecchio metodo anche qualora la libreria non fosse ancora installata via Composer.
