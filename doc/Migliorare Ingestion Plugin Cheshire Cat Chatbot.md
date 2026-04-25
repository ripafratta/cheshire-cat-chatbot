
Questa relazione documenta in dettaglio il processo di **ingestion** (acquisizione dati) implementato nel plugin WordPress per lo Cheshire Cat Chatbot. Il sistema è progettato per sincronizzare i contenuti del sito WordPress con la **Memoria Dichiarativa** dell'intelligenza artificiale.

## 1. Obiettivo del Processo

L'obiettivo principale è permettere allo Cheshire Cat di "leggere" e memorizzare i contenuti del sito (articoli, pagine, prodotti) per poter rispondere a domande basate sul contesto specifico del sito WordPress.

---

## 2. Architettura e Componenti Chiave

Il processo si basa su tre pilastri principali:

- **`inc/declarative-memory.php`**: Contiene la logica core per l'invio, l'aggiornamento e la cancellazione dei punti di memoria.
- **`inc/admin/declarative-memory-sync.php`**: Gestisce l'interfaccia di sincronizzazione massiva (Bulk Sync) nell'area di amministrazione.
- **`inc/classes/CustomCheshireCat.php`**: Fornisce l'interfaccia client per comunicare con le API REST dello Cheshire Cat.

---

## 3. Modalità di Ingestion

### A. Ingestion Automatica (Real-time)

Il sistema monitora i cambiamenti nei contenuti tramite gli hook standard di WordPress:

- **`save_post`**: Quando un post viene creato o aggiornato, il contenuto viene inviato alla memoria. Se è un aggiornamento, il vecchio punto di memoria viene rimosso prima di inserire il nuovo.
- **`before_delete_post`**: Quando un post viene eliminato definitivamente, i relativi dati vengono rimossi dalla memoria del Gatto.
- **`transition_post_status`**: Gestisce il passaggio allo stato "cestino" (trash), rimuovendo il contenuto dalla memoria se il post non è più pubblico.

### B. Ingestion Massiva (Bulk Sync)

Attraverso una pagina dedicata sotto "Cheshire Cat -> Sync Memory", l'amministratore può:

1. Filtrare per **Post Type** (Post, Pagine, Prodotti, ecc.).
2. Filtrare per **Stato** (es. solo i pubblicati).
3. Filtrare per **Data** (ultime 24 ore, ultima settimana, ecc.).
4. Avviare un processo AJAX che elabora i post a blocchi (batch di 5 post alla volta) per evitare il timeout del server.

---

## 4. Elaborazione e Pulizia dei Dati

Prima dell'invio, i dati subiscono un processo di raffinamento:

1. **Pulizia HTML**: Viene rimosso tutto il codice HTML tramite `wp_strip_all_tags`.
2. **Rimozione Shortcode**: Gli shortcode di WordPress vengono eliminati per evitare di inviare testo tecnico non interpretabile dall'AI.
3. **Integrazione WooCommerce**: Se il contenuto è un prodotto, il sistema include automaticamente:
    - Descrizione breve.
    - Attributi del prodotto (es. taglia, colore, marca) formattati come testo leggibile.

---

## 5. Struttura dei Dati Inviati (Metadata)

Ogni "punto di memoria" creato nello Cheshire Cat include metadati essenziali per il recupero delle informazioni:

- `origin`: Impostato su "WordPress".
- `wp_id`: L'ID univoco del post (utilizzato per rintracciare ed eliminare/aggiornare il dato).
- `url`: Il permalink pubblico del contenuto.
- `title`: Il titolo del post/pagina.
- `short_description` / `attribute_*`: (Solo per prodotti WooCommerce).

---

## 6. Sicurezza e Ottimizzazione

- **Nonce Verification**: Ogni operazione (AJAX o salvataggio) è protetta da nonce per prevenire attacchi CSRF.
- **Transient Cache**: Viene utilizzato un _transient_ di 10 secondi (`cheshire_memory_processed_{ID}`) per evitare che salvataggi multipli simultanei inviino lo stesso dato più volte.
- **Esclusioni**: Tramite una Meta Box nei singoli post, l'utente può decidere di escludere specifici contenuti dalla sincronizzazione.
- **Filtri Globali**: È possibile configurare quali tipi di post (Post Type) devono essere processati di default nelle impostazioni del plugin.

---

## 7. Flusso Tecnico dell'API

Il plugin utilizza l'endpoint `/memory/points/` (tramite il client PHP) con i seguenti metodi:

- **DELETE**: Richiamato tramite `deleteMemoryPointsByMetadata` filtrando per `wp_id`.
- **POST**: Richiamato tramite `createMemoryPoint` inviando il payload nella collezione `declarative`.

---

**Conclusione:** Il sistema di ingestion è robusto e ben integrato nell'ecosistema WordPress, garantendo che lo Cheshire Cat sia sempre aggiornato con i contenuti del sito, minimizzando al contempo il carico sul server e garantendo la pulizia dei dati inviati.

Ho completato l'analisi del codice relativo al processo di ingestion. Sopra trovi la relazione dettagliata che documenta il funzionamento del sistema, le modalità di sincronizzazione (automatica e massiva), il trattamento dei dati e l'integrazione con le API dello Cheshire Cat.

Se hai bisogno di approfondire una parte specifica o di modifiche tecniche, fammi sapere!


## Gli allegati di tipo documento vengono anch'essi caricati in Cheshire cat AI?

No, attualmente **gli allegati di tipo documento (come PDF, DOCX, ecc.) non vengono caricati** automaticamente nello Cheshire Cat AI.

Dall'analisi del codice (in particolare nel file `inc/declarative-memory.php`), emergono i seguenti punti tecnici:

1. **Esclusione esplicita**: Il codice contiene un controllo che ignora esplicitamente i post di tipo `attachment` (gli allegati):
    
    php
    
    // Ignora gli attachment
    
    if ( $ptype === 'attachment' ) {
    
        return;
    
    }
    
2. **Solo contenuto testuale**: Il processo di ingestion è progettato per estrarre e inviare il testo memorizzato nel database di WordPress (`post_content`). Non è presente una logica per l'estrazione di testo dai file fisici o per l'invio dei file stessi tramite l'endpoint `/rabbithole/` (che è quello solitamente usato per i documenti).
3. **Pulizia dei dati**: Prima dell'invio, il sistema esegue una pulizia degli shortcode e dei tag HTML, confermando che l'intento è quello di inviare solo il **testo pulito** degli articoli o delle pagine.

### Cosa viene caricato invece:

- Il **titolo** e il **testo** di articoli, pagine e prodotti.
- Per i prodotti WooCommerce, vengono inviati anche la **descrizione breve** e gli **attributi** (come caratteristiche tecniche o varianti) convertiti in formato testuale.

Se desideri caricare dei documenti (PDF/DOCX), al momento dovresti farlo manualmente tramite l'interfaccia (Admin) dello Cheshire Cat AI oppure estendere il plugin con una funzione specifica che utilizzi l'API `/rabbithole/` per l'upload dei file.


## L'attuale versione del codice potrebbe beneficiare dell'integrazione con la libreria microsoft MarkItDown per trasformare i contenuti in markdown?

Assolutamente sì, l'integrazione di una libreria come **MarkItDown** (o un convertitore Markdown in generale) porterebbe **benefici enormi** alla qualità dei dati forniti allo Cheshire Cat AI.

Attualmente, il tuo codice esegue questa operazione:

php

$content = wp_strip_all_tags($content);

Questa funzione "appiattisce" completamente il testo. Un titolo `<h1>`, un paragrafo `<p>`, una lista `<ul>` o un link `<a>` perdono tutta la loro struttura semantica, diventando un unico blocco di testo continuo.

Ecco perché passare al Markdown (MD) cambierebbe le regole del gioco:

### 1. Vantaggi Principali per lo Cheshire Cat (LLM)

- **Mantenimento del Contesto Strutturale:** I modelli linguistici (LLM) "amano" il Markdown. Sanno distinguere perfettamente un titolo (`# Titolo`) da un elenco puntato (`- elemento`) o da un link (`[testo](url)`). Mantenere questa struttura aiuta il Gatto a comprendere l'importanza gerarchica delle informazioni (ad esempio, capirà che un testo sotto un H2 appartiene a quell'argomento).
- **Migliore indicizzazione Vettoriale:** Quando il testo viene diviso in "chunks" (frammenti) per essere salvato nel database vettoriale (Qdrant), avere una sintassi Markdown pulita aiuta gli algoritmi di chunking a dividere il testo in modo logico (es. tagliando prima di un nuovo `# Titolo` anziché a metà di una frase).
- **Risoluzione del problema degli Allegati:** **MarkItDown** di Microsoft è formidabile nell'estrarre testo da PDF, Word (DOCX), Excel, PowerPoint e immagini, convertendoli direttamente in Markdown. Potresti usarlo proprio per risolvere il limite di cui parlavamo prima, permettendo al plugin di processare la libreria media di WordPress e inviare i documenti al Gatto.

### 2. Sfide Tecniche e Alternative (PHP vs Python)

C'è però un ostacolo architettonico importante da considerare:

- **MarkItDown è scritto in Python:** Il tuo plugin è in **PHP**. I server di hosting WordPress standard raramente permettono di eseguire script Python o hanno le librerie installate.
- **Come integrarlo?** Per usare MarkItDown dovresti configurare un piccolo microservizio esterno (es. in FastAPI/Flask) su un tuo server, a cui il plugin WordPress invia il contenuto/file, riceve indietro il Markdown e lo invia al Cheshire Cat.

### La Soluzione più Semplice (Native PHP)

Se il tuo obiettivo primario è **migliorare l'ingestion degli articoli/pagine** (da HTML a MD) senza impazzire con Python, la soluzione migliore è utilizzare una libreria PHP nativa e leggerissima come **`league/html-to-markdown`**.

Potresti installarla via Composer (`composer require league/html-to-markdown`) e sostituire la riga di pulizia nel tuo file `inc/declarative-memory.php` in questo modo:

php

// Invece di: $content = wp_strip_all_tags($content);

// Usa il convertitore:

$converter = new League\HTMLToMarkdown\HtmlConverter();

// Rimuovi gli script e gli stili prima di convertire

$converter->getConfig()->setOption('strip_tags', true); 

$content = $converter->convert($content);

### In Sintesi

- **Passare in formato Markdown?** È un grandissimo **SÌ**, migliorerà drasticamente la qualità delle risposte dell'AI.
- **Usare Microsoft MarkItDown?** È l'ideale se vuoi sbloccare l'ingestion di documenti complessi (PDF/Office) e sei disposto a creare un microservizio Python di appoggio.
- **Solo per i testi WP?** Usa una libreria PHP (`league/html-to-markdown`) per una modifica veloce, nativa e dal risultato assicurato.

## Fornisci suggerimenti per ottimizzare la configurazione della libreria league/html-to-markdown per una conversione ottimale dei testi wordpress?

Per ottimizzare la conversione dei contenuti WordPress per lo Cheshire Cat AI utilizzando `league/html-to-markdown`, ti suggerisco una configurazione mirata a eliminare il "rumore" del codice (specialmente quello generato da Gutenberg o dai Page Builder) e a massimizzare la leggibilità per l'LLM.

Ecco i suggerimenti chiave e un esempio di implementazione:

### 1. Parametri di Configurazione Consigliati

- **`header_style => 'atx'`**: Fondamentale. Converte i titoli nel formato `# Titolo`, `## Sottotitolo`. Questo è il formato che gli LLM riconoscono meglio rispetto allo stile _Setext_ (quello con le linee di `=` o `-` sotto il testo).
- **`strip_tags => true`**: WordPress (specialmente con i blocchi) genera molti `<div>` e `<span>` annidati che non hanno un equivalente in Markdown. Attivando questa opzione, la libreria estrarrà solo il testo pulito, eliminando i tag inutili ma mantenendo la struttura (grassetti, corsivi, ecc.).
- **`remove_nodes => 'script style iframe object'`**: Rimuove completamente i blocchi di codice che non contengono informazioni utili per la memoria del Gatto (script, stili CSS, mappe o video incorporati).
- **`hard_break => true`**: Converte i tag `<br>` in interruzioni di riga nette. Questo aiuta l'LLM a percepire meglio la separazione visiva dei testi.

### 2. Esempio di Implementazione Ottimizzata

Ecco come potresti modificare la funzione di caricamento in `inc/declarative-memory.php`:

php

use League\HTMLToMarkdown\HtmlConverter;

// ... all'interno della funzione di upload ...

$content = $post->post_content;

// 1. ESPANDI GLI SHORTCODE (Opzionale ma consigliato)

// Invece di rimuoverli e basta, renderizzali se contengono testo utile

$content = do_shortcode($content);

// 2. CONFIGURAZIONE OTTIMIZZATA

$config = [

    'header_style'    => 'atx',           // Usa # e ##

    'strip_tags'      => true,          // Pulisce i tag HTML senza equivalente MD

    'remove_nodes'    => 'script style iframe form', // Elimina spazzatura tecnica

    'italic_style'    => '*',             // Stile standard per corsivo

    'bold_style'      => '**',            // Stile standard per grassetto

    'hard_break'      => true,           // Gestione pulita dei <br>

    'preserve_comments' => false,        // Rimuove i commenti <!-- wp:paragraph --> di Gutenberg

];

$converter = new HtmlConverter($config);

// 3. CONVERSIONE

$markdown_content = $converter->convert($content);

// 4. PULIZIA EXTRA (Opzionale)

// Rimuovi spazi bianchi eccessivi per risparmiare token

$markdown_content = preg_replace("/\n\s+\n/", "\n\n", $markdown_content);

$data = [

    'content' => $markdown_content,

    'metadata' => $metadata

];

### 3. Suggerimento Bonus: Il "Filtro WordPress"

Prima di passare il contenuto al convertitore, WordPress memorizza i dati nel database in un formato "grezzo" (specialmente con Gutenberg). Per ottenere la versione migliore possibile del testo, potresti voler applicare alcuni filtri standard di WordPress, ma con cautela:

- **Invece di `apply_filters('the_content', $content)`**: Questo filtro spesso aggiunge bottoni social, pubblicità o script di altri plugin che sporcherebbero la memoria dell'AI.
- **Usa `wpautop($content)`**: Se il post è stato scritto con l'editor classico, questa funzione aggiunge i tag `<p>` mancanti che aiutano il convertitore a identificare i paragrafi.

### Perché questa configurazione è ottimale per lo Cheshire Cat?

L'obiettivo della memoria dichiarativa non è avere una copia carbone dell'HTML, ma avere **semantica pura**. Una struttura Markdown pulita (`#`, `##`, `**`, `-`) permette al Gatto di:

1. Risparmiare **Token** (l'HTML è molto più voluminoso del Markdown).
2. Migliorare il **Retrieval** (la ricerca nel database vettoriale è più precisa se il testo non è interrotto da classi CSS o tag `<div>`).
3. Citare le fonti in modo più naturale.