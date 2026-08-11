---
layout: post
title: "Il mondo dei test"
---

I test, un argomento divisivo.

L'esperienza personale mi ha portato a diffidare dei test automatizzati a livello utente (tipo RSAT in ambiente Dynamics 365) e a credere un po' di più in quelli scritti contestualmente al codice.

E il parere, altrettanto personale, è che possono essere utili ma vanno usati con consapevolezza.

La loro debolezza principale, secondo me, è legata a due aspetti.

Da un lato, tipicamente, si devono mockare alcune funzioni e quindi non possono riprodurre il vero scenario reale. Può essere anche un aspetto positivo (la famosa atomicità degli *unit test*) ma deve essere chiaro che stiamo facendo amichevoli pre-campionato, non scontri di playoff.

Dall'altro tendono a invecchiare molto presto, specie se gli scenari che testano non vengono aggiornati. Esempio classico: l'inserimento di un nuovo ordine in un esercizio che nel frattempo è stato chiuso.

A dire il vero i test hanno anche dei grossi pregi: permettono di controllare parti specifiche dell'applicazione quando questa non è ancora completa e, allo stesso modo, consentono facilmente di controllare il comportamento dell'applicazione in presenza di variazioni intorno ad un dato di base. Una cosa che, nel contesto di questa applicazione (che deve rispondere allo stesso modo a input tipo "Dammi il fatturato", "Voglio il fatturato", "Mostrami il totale del venduto" etc...) è decisiva.

Dunque, accanto a Backend e Frontend, ho inserito nell'albero del progetto una sezione per i test (automatizzati) del Backend.
I test utente li ho invece eseguiti interagendo manualmente con l'applicazione attraverso la finestra del browser. 

La cartella "test_backend" ha questa struttura:

```text
.
├── basic_tests
│   ├── mocks_builder.js
│   ├── test_embedding.js
│   ├── test_ml_classifier.js
│   ├── test_router_conversational.js
│   ├── test_router.js
│   └── test_sql_builder.js
└── e2e_test
    ├── index.js
    ├── tests_e2e.js
    └── utils_e2e.js
```

I file nella sezione basic sono di tipo puntuale. 

Per eseguirli è sufficiente scrivere da linea di comando una istruzione di questo tipo:

```bash
$ node test_embedding.js
Testing local embedding…
Embedding 'ciao': 384 [
  -0.08178652077913284,
  -0.019077906385064125,
  -0.03256573528051376,
  0.061043430119752884,
  -0.04521749168634415
]
```

Unico prerequisito, qui, è la presenza del servizio Ollama, attivo e running.

Il test_embedding sopra riportato è proprio il più semplice (e meno significativo). 

Per tutti gli altri una volta lanciati viene visualizzato a schermo il log dell'applicazione, con alla fine un riepilogo dei risultati, ad esempio:

```bash
$ node test_router.js
...
=== TEST REPORT ===
✔ Passed: 10
✘ Errors: 0
⚠ Exceptions: 0
```

Tutti questi file `test_*.js` contengono al loro interno, insieme alla logica per eseguirli e valutarne il successo, i prompt con cui il sistema viene testato.

Questi possono essere utili anche come spunto per inviare input uguali o simili al sistema *live*, verificando che si comporti come atteso.

Ciò è particolarmente vero per i test e2e, abbreviazione di End-To-End, chiamata così perché interagisce realmente con il backend dell'applicazione. 

In questo caso, oltre ad Ollama, anche il server.js deve essere avviato. 

Ecco il comando e un esempio di output relativo al test del prompt "Elencami i KPI" che fa parte della suite Sysmeta (Meta Intents e Intents di Sistema, come appunto quelli che coinvolgono l'MCP):

```bash
$ node index.js
...
────────────────────────────────────────────────────────────
SUITE: Sysmeta
TEST:  Elencami i KPI
────────────────────────────────────────────────────────────

System Response: {
  type: 'kpi_list',
  kpis: [
    {
      key: 'sales_amount',
      fact: 'orderlines',
      sql: 'SUM(OrderLines.qty * OrderLines.unit_price)',
      synonyms: [Array]
    },
    {
      key: 'order_count',
      fact: 'orderheaders',
      sql: 'COUNT(DISTINCT OrderHeaders.id)',
      synonyms: [Array]
    },
    { key: 'count', fact: null, sql: 'COUNT(*)', synonyms: [Array] }
  ],
  intent: 'kpi_list'
}

Contenuto MCP ricevuto:
{
  "type": "kpi_list",
  "kpis": [
    {
      "key": "sales_amount",
      "fact": "orderlines",
      "sql": "SUM(OrderLines.qty * OrderLines.unit_price)",
      "synonyms": [
        "vendite",
        "fatturato",
        "ricavi",
        "venduto",
        "venduti"
      ]
    },
    {
      "key": "order_count",
      "fact": "orderheaders",
      "sql": "COUNT(DISTINCT OrderHeaders.id)",
      "synonyms": [
        "ordini",
        "numero ordini",
        "conteggio ordini",
        "ordinato",
        "ordinati"
      ]
    },
    {
      "key": "count",
      "fact": null,
      "sql": "COUNT(*)",
      "synonyms": [
        "conteggio",
        "totale",
        "numero"
      ]
    }
  ],
  "intent": "kpi_list"
}
✔ OK
...
────────────────────────────────────────────────────────────
SUITE: Followup
TEST:  Voglio esclusivamente i prodotti top
────────────────────────────────────────────────────────────

System Response: {
  type: 'interpretation',
  interpretation: 'Vuoi mostrare solo i prodotti con valutazione più alta (5 stelle) e che siano stati acquistati di più dagli altri utenti.',
  question: "È corretta l'interpretazione?",
  intent: 'list_entities'
}
→ Richiesta Interpretazione : Per risultato completo eseguire test a due turni
✔ OK

========================================
              RISULTATO FINALE
========================================
   ✔ PASS: 60
   ✘ FAIL: 0
========================================
```

Notare che per questi test niente è stato mockato: le risposte e il codice SQL generati provengono dall’ambiente reale. 

Chiaramente, è stato possibile solo perché questa applicazione interroga semplicemente: non scrive dati nel DB o nel filesystem.

Nel corso di questo post, una buona parte degli *snippet* riportati contiene log che, insieme alla logica, sono la colonna portante di un buon sistema di test o debug.

È proprio ai log che dedicheremo il prossimo pezzo.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*