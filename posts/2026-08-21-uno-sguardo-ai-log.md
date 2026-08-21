---
layout: post
title: "Uno sguardo ai log"
---

Una frase che mi ripeto spesso è qualcosa tipo "se i log sono buoni allora l'applicazione è fatta".

Buoni vuol dire chiari, completi, non sovrabbondanti né troppo stringati e con una minima dose di *ASCII art* che li rende gradevoli.

Non so se sia un principio generale ma direi che, nel mio caso, abbia trovato varie conferme.

Per questo motivo, volevo dedicare un post ai log di LLMSQLPROD, che si possono vedere da terminale nella sessione in cui è stato avviato il `server.js` del backend. 

Cominciamo col dire che sono gestiti con un *logger* centralizzato chiamato `logSection`.

La conseguenza più importante è che, volendoli dirigere verso un file o un database anzichè a video, basterebbe un piccolo intervento all'interno di quella funzione e il gioco sarebbe fatto.

Per l'analisi, iniziamo con un caso semplice: quello in cui digitiamo il messaggio "Ciao, come va?". 

Questa è la parte essenziale del log:

```javascript
[LLMSQLPROD][Server/REQUEST] Incoming message Ciao, come va?
[LLMSQLPROD][Router/SIMPLE_CLASSIFY] Start { message: 'Ciao, come va?' }
[LLMSQLPROD][Router/ML] ML classifier invoked Ciao, come va?
[LLMSQLPROD][Router/ML] ML result { intent: 'smalltalk', score: 0.9360242273189124 }
[LLMSQLPROD][Server/CLASSIFYONLY] Classify Only result { intent: 'smalltalk', score: 0.9360242273189124 }
[LLMSQLPROD][Server/STREAM] Smalltalk detected → streaming mode
```

Come si vede, è scattata la classificazione preliminare che ha identificato con alto punteggio l'intent `smalltalk`, dato che "ciao come va" (testo quasi uguale all'input anche se non esattamente) è uno dei prototipi con cui è stato addestrato il classificatore. 

La *pipeline* del router viene praticamente saltata, mentre viene attivata la modalità streaming, con la chiamata diretta ad Ollama e la restituzione del messaggio nella UI in *streaming token*.

Passiamo a qualcosa di leggermente più complicato:

```javascript
[LLMSQLPROD][Server/STATE] Input Pending Question È corretta l'interpretazione?
[LLMSQLPROD][Router/SIMPLE_CLASSIFY] Start { message: 'Parlami dei Golden State Warriors' }
[LLMSQLPROD][Router/ML] ML classifier invoked Parlami dei Golden State Warriors
[LLMSQLPROD][Router/ML] Low confidence → UNKNOWN { score: 0.4643392859667115 }
[LLMSQLPROD][Router/ML] ML result { intent: 'unknown', score: 0.4643392859667115 }
[LLMSQLPROD][Server/CLASSIFYONLY] Classify Only result { intent: 'unknown', score: 0.4643392859667115 }
[LLMSQLPROD][Server/OUTOFSCOPE] Intent Undefined/Unknown or Score under 0.5
```

Qui il classificatore non ha trovato un prototipo da associare con sicurezza, quindi ha tenuto lo score al 46% e ha sovrascritto la sua ipotesi di intent con `UNKNOWN`. 

Il punteggio è risultato inferiore anche alla soglia di intervento del *disambiguator* (che per input completamente sballati farebbe domande altrettanto assurde) e quindi è scattato l'early return per "Out of scope". 

C'è da dire che se lo score fosse stato di poco superiore, ad esempio il 51%, il fuori ambito non sarebbe scattato e la *pipeline* avrebbe seguito un percorso completamente diverso: quello con la disambiguazione appunto. 

Questa latente instabilità è sicuramente un altro punto di miglioramento per gli sviluppi futuri.

È chiaro che non possiamo analizzare tutti i casi possibili, ma credo valga la pena illustrare anche un caso di dialogo a due turni in cui il *disambiguator* è intervenuto.

Il log, in questo caso, è piuttosto lungo (anche cercando di mantenere le sole parti più importanti) e si presenta così:

```javascript
[LLMSQLPROD][Server/REQUEST] Incoming message Voglio un report delle attività della forza vendita
[LLMSQLPROD][Server/STATE] Input Pending Interpretation undefined
[LLMSQLPROD][Server/STATE] Input Pending Question undefined
[LLMSQLPROD][Router/SIMPLE_CLASSIFY] Start { message: 'Voglio un report delle attività della forza vendita' }
[LLMSQLPROD][Router/ML] ML classifier invoked Voglio un report delle attività della forza vendita
[LLMSQLPROD][Router/ML] ML result { intent: 'time_series_orders', score: 0.5402982702548201 }
[LLMSQLPROD][Server/CLASSIFYONLY] Classify Only result { intent: 'time_series_orders', score: 0.5402982702548201 }
[LLMSQLPROD][Server/REQUEST] Other cases → router mode
[LLMSQLPROD][Router/MAIN] Incoming message Voglio un report delle attività della forza vendita
[LLMSQLPROD][Router/MAIN] Incoming Context {}
[LLMSQLPROD][Router/PREPROCESS] Start { message: 'Voglio un report delle attività della forza vendita' }
[LLMSQLPROD][detect_mcp_like] Input: voglio un report delle attività della forza vendita
[LLMSQLPROD][detect_kpi_list] Checking: voglio un report delle attività della forza vendita
[LLMSQLPROD][detect_kpi_list] No 'kpi' keyword
[LLMSQLPROD][detect_mcp_like] No MCP-like intent detected
[LLMSQLPROD][detect_llm_stats] Checking: voglio un report delle attività della forza vendita
[LLMSQLPROD][detect_llm_stats] NO MATCH
[LLMSQLPROD][Router/SQLUTILS] Checking if direct SQL Voglio un report delle attività della forza vendita
[LLMSQLPROD][Router/PREPROCESS] End
[LLMSQLPROD][Router/MAIN] PreProcess Return Object {}
[LLMSQLPROD][Router/PRE-PROCESS] Handling Preprocess Early Return undefined
[LLMSQLPROD][Router/PRE-PROCESS] No Early Return to manage
[LLMSQLPROD][Router/CLASSIFY] Start { message: 'Voglio un report delle attività della forza vendita' }
[LLMSQLPROD][Router/ML] ML classifier invoked Voglio un report delle attività della forza vendita
[LLMSQLPROD][Router/ML] ML result { intent: 'time_series_orders', score: 0.5402982702548201 }
[LLMSQLPROD][Router/CLASSIFY] Disambiguation considerated
[LLMSQLPROD][LLM/DISAMBIGUATOR] Disambiguator Invoked
...
[LLMSQLPROD][SERVER/Validation] Type clarification
[LLMSQLPROD][SERVER/Validation] Payload Keys [ 'needsClarification', 'question' ]
[LLMSQLPROD][Server/RESPONSE] Final Frontend Message {
  type: 'clarification',
  kpis: undefined,
  sql: undefined,
  rows: undefined,
  schema: undefined,
  changelog: undefined,
  interpretation: undefined,
  question: 'Per aiutarti meglio, potresti fornirmi maggiori dettagli su cosa intendi con "report delle attività della forza vendita"? Vuoi includere dati relativi a:\n' +
    '\n' +
    '1. Le performance dei singoli venditori?\n' +
    "2. L'evoluzione del fatturato o degli ordini ricevuti nel tempo?\n" +
    '3. I punti di forza e di debolezza nella strategia commerciale attuale?\n' +
    '4. Qualcosa di diverso?\n' +
    '\n' +
    'Scegli una opzione per aiutarmi a capire meglio cosa stai cercando!',
  stats: undefined,
  error: undefined,
  intent: 'time_series_orders'
}
...
[LLMSQLPROD][Router/SIMPLE_CLASSIFY] Start { message: 'Voglio le vendite per cliente' }
[LLMSQLPROD][Router/ML] ML classifier invoked Voglio le vendite per cliente
[LLMSQLPROD][Router/ML] ML result { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Server/CLASSIFYONLY] Classify Only result { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Server/REQUEST] Other cases → router mode
[LLMSQLPROD][Router/MAIN] Incoming message Voglio le vendite per cliente
[LLMSQLPROD][Router/MAIN] Incoming Context { pendingInterpretation: undefined }
[LLMSQLPROD][Router/PREPROCESS] Start { message: 'Voglio le vendite per cliente' }
[LLMSQLPROD][detect_mcp_like] Input: voglio le vendite per cliente
[LLMSQLPROD][detect_kpi_list] Checking: voglio le vendite per cliente
[LLMSQLPROD][detect_kpi_list] No 'kpi' keyword
[LLMSQLPROD][detect_mcp_like] No MCP-like intent detected
[LLMSQLPROD][detect_llm_stats] Checking: voglio le vendite per cliente
[LLMSQLPROD][detect_llm_stats] NO MATCH
[LLMSQLPROD][Router/SQLUTILS] Checking if direct SQL Voglio le vendite per cliente
[LLMSQLPROD][Router/PREPROCESS] End
[LLMSQLPROD][Router/MAIN] PreProcess Return Object {}
[LLMSQLPROD][Router/PRE-PROCESS] Handling Preprocess Early Return undefined
[LLMSQLPROD][Router/PRE-PROCESS] No Early Return to manage
[LLMSQLPROD][Router/CLASSIFY] Start { message: 'Voglio le vendite per cliente' }
[LLMSQLPROD][Router/ML] ML classifier invoked Voglio le vendite per cliente
[LLMSQLPROD][Router/ML] ML result { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Router/CLASSIFY] Disambiguator skipped
[LLMSQLPROD][Router/CLASSIFY] Semantic Interepretation considerated
[LLMSQLPROD][LLM/SENSE_CHECKER] Invoked { intent: 'global_kpi_dimensions', llmMode: 'deterministic' }
[LLMSQLPROD][SENSE_CHECKER/IS_VAGUE] Focus Indicators {
  hasGlobal: false,
  hasDimensionMarker: true,
  hasListMarker: false,
  hasSuperlativeMarkers: false
}
[LLMSQLPROD][LLM/SENSE_CHECKER] Deterministic mode { vague: false }
[LLMSQLPROD][Router/CLASSIFY] Semantic Interpretation skipped
[LLMSQLPROD][Router/SAFETY] Validating intent global_kpi_dimensions
[LLMSQLPROD][Router/CLASSIFY] General classification { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Router/MAIN] Classification Return Object {
  intentObj: { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
}
[LLMSQLPROD][Router/CLASSIFY] Handling Classification Early Return undefined
[LLMSQLPROD][Router/MAIN] Input Intent for Exec Phase { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Router/EXEC-PHASE] Start { intent: 'global_kpi_dimensions' }
[LLMSQLPROD][Router/SQLUTILS] Checking if direct SQL Voglio le vendite per cliente
[LLMSQLPROD][Router/PRE-EXEC] Start {
  intent: 'global_kpi_dimensions',
  originalMessage: 'Voglio le vendite per cliente'
}
[LLMSQLPROD][Router/PARAMS] Extracting params {
  intent: 'global_kpi_dimensions',
  message: 'Voglio le vendite per cliente'
}
[LLMSQLPROD][Router/PARAMS] Final params {
  entity: 'customers',
  measure: 'count',
  filters: null,
  dimension: 'customer',
  time_granularity: undefined,
  time_range: undefined,
  limit: undefined,
  _detectedFact: 'orderlines',
  _detectedMeasure: 'sales_amount',
  isSufficient: true
}
[LLMSQLPROD][Router/PRE-EXEC] Extracted params {
  entity: 'customers',
  measure: 'count',
  filters: null,
  dimension: 'customer',
  time_granularity: undefined,
  time_range: undefined,
  limit: undefined,
  _detectedFact: 'orderlines',
  _detectedMeasure: 'sales_amount',
  isSufficient: true
}
[LLMSQLPROD][Router/PRE-EXEC] Initial intent global_kpi_dimensions
[LLMSQLPROD][Router/PRE-EXEC] Resolved intent sales_by_dimension
[LLMSQLPROD][Router/PRE-EXEC] Promoted params {
  entity: 'customers',
  measure: 'sales_amount',
  filters: null,
  dimension: 'customer',
  time_granularity: undefined,
  time_range: undefined,
  limit: undefined,
  _detectedFact: 'orderlines',
  _detectedMeasure: 'sales_amount',
  isSufficient: true,
  fact: 'orderlines'
}
[LLMSQLPROD][Router/PRE-EXEC] End
[LLMSQLPROD][LLM/PARAMS_INTERPRETER] Params interpreter invoked
[LLMSQLPROD][LLM/PARAMS_INTERPRETER] Skip: parameters sufficient
[LLMSQLPROD][Router/EXEC-PHASE] Interpretation Result null
[LLMSQLPROD][Router/EXECUTE] Executor invoked {
  intent: 'sales_by_dimension',
  originalMessage: 'Voglio le vendite per cliente'
}
[LLMSQLPROD][Router/EXECUTE] Building SQL
[LLMSQLPROD][SQL/SALES_BY_DIMENSION] Building SQL {
  entity: 'customers',
  measure: 'sales_amount',
  filters: null,
  dimension: 'customer',
  time_granularity: undefined,
  time_range: undefined,
  limit: undefined,
  _detectedFact: 'orderlines',
  _detectedMeasure: 'sales_amount',
  isSufficient: true,
  fact: 'orderlines'
}
[LLMSQLPROD][BUILDER_CORE] Join plan {
  tables: [ 'OrderLines', 'OrderHeaders', 'Customers' ],
  joins: [
    'OrderLines.order_id = OrderHeaders.id',
    'OrderHeaders.customer_id = Customers.id'
  ]
}
[LLMSQLPROD][SQL/SALES_BY_DIMENSION] Final SQL         SELECT
            Customers.name AS dimension,
  SUM(OrderLines.qty * OrderLines.unit_price) AS value
        FROM OrderLines
        JOIN OrderHeaders ON OrderLines.order_id = OrderHeaders.id
JOIN Customers ON OrderHeaders.customer_id = Customers.id

        GROUP BY Customers.name
        ORDER BY value DESC
        ;
[LLMSQLPROD][Router/EXECUTE] Validating SQL         SELECT
            Customers.name AS dimension,
  SUM(OrderLines.qty * OrderLines.unit_price) AS value
        FROM OrderLines
        JOIN OrderHeaders ON OrderLines.order_id = OrderHeaders.id
JOIN Customers ON OrderHeaders.customer_id = Customers.id

        GROUP BY Customers.name
        ORDER BY value DESC
        ;
...
[LLMSQLPROD][Router/EXECUTE] Executing SQL
[LLMSQLPROD][MCP/SQL] runQuery() called
[LLMSQLPROD][MCP/SQL] Query received:         SELECT
            Customers.name AS dimension,
  SUM(OrderLines.qty * OrderLines.unit_price) AS value
        FROM OrderLines
        JOIN OrderHeaders ON OrderLines.order_id = OrderHeaders.id
JOIN Customers ON OrderHeaders.customer_id = Customers.id

        GROUP BY Customers.name
        ORDER BY value DESC
        ;
[LLMSQLPROD][MCP/SQL] Query Response [
  { dimension: 'Rossi', value: 70 },
  { dimension: 'Bianchi', value: 25 }
]
[LLMSQLPROD][Router/EXEC-PHASE] Normal Pipeline Return
[LLMSQLPROD][Router/MAIN] Execute Return Object {
  refreshedIntent: {
    intent: 'sales_by_dimension',
    params: {
      entity: 'customers',
      measure: 'sales_amount',
      filters: null,
      dimension: 'customer',
      time_granularity: undefined,
      time_range: undefined,
      limit: undefined,
      _detectedFact: 'orderlines',
      _detectedMeasure: 'sales_amount',
      isSufficient: true,
      fact: 'orderlines'
    }
  },
  userWroteSQL: false,
  execResult: {
    sql: '        SELECT\n' +
      '            Customers.name AS dimension,\n' +
      '  SUM(OrderLines.qty * OrderLines.unit_price) AS value\n' +
      '        FROM OrderLines\n' +
      '        JOIN OrderHeaders ON OrderLines.order_id = OrderHeaders.id\n' +
      'JOIN Customers ON OrderHeaders.customer_id = Customers.id\n' +
      '        \n' +
      '        GROUP BY Customers.name\n' +
      '        ORDER BY value DESC\n' +
      '        ;',
    rows: [
      { dimension: 'Rossi', value: 70 },
      { dimension: 'Bianchi', value: 25 }
    ],
    error: undefined
  }
}
[LLMSQLPROD][Router/EXEC] Handling Classification Early Return undefined { intent: 'global_kpi_dimensions', score: 0.7134245540634061 }
[LLMSQLPROD][Router/EXEC] No Early Return, nothing to do
...
[LLMSQLPROD][LLM/EXPLAINER] Explainer invoked
[LLMSQLPROD][LLM/EXPLAINER] LLM explanation Il risultato della query indica i clienti più importanti per la nostra azienda in base ai loro acquisti. I dati mostrano due clienti: "Rossi" e "Bianchi". "Rossi" hanno avuto un totale di 70 unità di vendita, mentre "Bianchi" sono stati responsabili di una spesa totale di 25 unità di vendita.
[LLMSQLPROD][Router/POSTPROCESS] Followup skipped
[LLMSQLPROD][Router/POSTPROCESS] End
...
[LLMSQLPROD][Server/RESPONSE] Final Frontend Message {
  type: 'sql_result',
  sql: '        SELECT\n' +
    '            Customers.name AS dimension,\n' +
    '  SUM(OrderLines.qty * OrderLines.unit_price) AS value\n' +
    '        FROM OrderLines\n' +
    '        JOIN OrderHeaders ON OrderLines.order_id = OrderHeaders.id\n' +
    'JOIN Customers ON OrderHeaders.customer_id = Customers.id\n' +
    '        \n' +
    '        GROUP BY Customers.name\n' +
    '        ORDER BY value DESC\n' +
    '        ;',
  rows: [
    { dimension: 'Rossi', value: 70 },
    { dimension: 'Bianchi', value: 25 }
  ],
  explanation: 'Il risultato della query indica i clienti più importanti per la nostra azienda in base ai loro acquisti. I dati mostrano due clienti: "Rossi" e "Bianchi". "Rossi" hanno avuto un totale di 70 unità di vendita, mentre "Bianchi" sono stati responsabili di una spesa totale di 25 unità di vendita.',
  followup: null,
  userWroteSQL: false,
  originalMessage: 'Voglio le vendite per cliente'
}
[LLMSQLPROD][Server/STATE] Saved Pending Interpretation null
[LLMSQLPROD][Server/STATE] Saved Pending Question undefined
```

L'input originale è "Voglio un report delle attività della forza vendita" che viene classificato come `time_series_order` (come succederebbe ad esempio per il testo "serie storica ordini"), con una confidenza bassa ma non bassissima, il 54%. 

In questo caso il *disambiguator* si attiva e produce la richiesta "Per aiutarti meglio, potresti fornirmi maggiori dettagli su cosa intendi con report delle attività della forza vendita?", che viene inoltrata al frontend. 

In questa simulazione l'utente ha inserito, nel secondo turno, il messaggio di chiarimento "Voglio le vendite per cliente".

Il nuovo input ha sovrascritto il precedente e, avendo tutte le caratteristiche per individuare con alta confidenza (71%) un intent semanticamente più convincente (`global_kpi_dimensions`, tipico di prompt come "numero ordini totali"), quest'ultimo viene associato all'insieme dei due input.

Notare che l'intent ricavato è stato nuovamente modificato dal sistema, perché la *fact table* associata alla parola vendite, combinata con la dimensione specificata dopo il *marker* "per", ha portato ad un *promoted intent* `sales_by_dimension`. 

È, infatti, quello il template invocato al termine del flusso, come possiamo verificare dall'SQL costruito. 

In questo caso è scattato anche l'*explainer*, perché la `SELECT` è stata giudicata complessa (due `JOIN` e una `GROUP BY`). 

L'LLM ha provato ad aggiungere un commento pertinente e, anche se pure qui i margini di miglioramento sarebbero ampi, ha portato a termine il suo compito.

Con questo pezzo, forse il più impegnativo da scrivere e probabilmente anche da leggere, il boss finale è battuto.

La partita è finita: diamo un'occhiata al nostro punteggio?

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*