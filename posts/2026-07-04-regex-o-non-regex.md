---
layout: post
title: "Regex o non regex"
---

Che il problema principale fosse nell'approccio con AST, a dire il vero, non mi è stato chiaro fin da subito.

E forse è stato un bene, perché proprio da tentativi di miglioramento laterali rispetto all'NL Parser è nato il classificatore *machine learning*, che sarebbe divenuto uno dei cardini dell'applicazione finale.

In prima battuta, infatti, avevo pensato che la colpa principale del funzionamento non soddisfacente fosse nella parte di classificazione degli intent utente. 

E, oggettivamente, il componente dedicato era molto fragile, perché basato su semplici espressioni regolari. 

La parte del codice che cercava di capire se il messaggio fosse o meno una richiesta in linguaggio naturale aveva, infatti, una parte dichiarativa consistente in elenchi di verbi e nomi che dovevano essere riconosciuti dall'applicazione, come questi:

```javascript
const NL_VERBS = [
    "elenca",
    "elencami",
    "mostra",
    "mostrami",
    "dammi"
];

const DATA_NOUNS = [
    "clienti",
    "cliente",
    "ordini",
    "ordine",
    "prodotti",
    "prodotto",
    "articolo",
    "articoli"
];
```

Già questa non era elegantissima, costringendo a inserire un esempio per ogni variazione del verbo o del sostantivo.

Il vero problema, però, arrivava con la seguente parte di logica:

```javascript
export default function detectNaturalDataRequest(lower) {
    console.log("[NL Request Detector] Checking for NL data request:", lower);

    let hasVerb = false;
    for (let i = 0; i < NL_VERBS.length; i++) {
        const v = NL_VERBS[i];
        if (lower.startsWith(v) || lower.includes(" " + v)) {
            console.log("[NL Request Detector] Verb match:", v);
            hasVerb = true;
            break;
        }
    }
    if (!hasVerb) {
        console.log("[NL Request Detector] No request verb found.");
        return false;
    }

    for (let j = 0; j < DATA_NOUNS.length; j++) {
        if (lower.includes(DATA_NOUNS[j])) {
            console.log("[NL Request Detector] Data noun match:", DATA_NOUNS[j]);
            return true;
        }
    }

    console.log("[NL Request Detector] No data noun matched.");
    return false;
}
```

Tutto ruotava, dunque, intorno a degli `includes` o `startsWith` e, come potete facilmente intuire, bastava un niente per perdersi un input utente perfettamente lecito.

La nuova versione del software provò a risolvere il problema introducendo una tecnica che non usavo da tempo ma che mi aveva sempre affascinato: la classificazione di frasi tramite il confronto di *embedding*.

Questo era il cuore del nuovo modulo, al solito import a parte:

```javascript
for (const intent of Object.keys(INTENT_PROTOTYPES)) {
    for (const proto of INTENT_PROTOTYPES[intent]) {
        const protoEmbedding = await embed(proto);
        const score = cosineSimilarity(textEmbedding, protoEmbedding);

        debug.push({ intent, proto, score });

        if (score > bestScore) {
            bestScore = score;
            bestIntent = intent;
        }
    }
}
```

Gli *intent prototypes* avevano un nome pomposo ma erano semplicemente degli elenchi di frasi esemplificative dei vari intenti, come ad esempio queste:

```javascript
nl_query: [
    "mostrami i clienti",
    "dammi gli ordini",
    "mostrami i prodotti",
    "quanti clienti ho",
    "elenca gli ordini",
    "fammi vedere i clienti attivi",
    "mostrami i prodotti venduti",
    "dammi la lista dei clienti",
    "mostrami gli ordini recenti",
    "quali prodotti hai",
    "mostrami i clienti nuovi",
    "dammi i prodotti disponibili",
    "fammi vedere gli ordini aperti",
    "mostrami i clienti per regione",
    "mostrami i prodotti in magazzino",
    "dammi gli ordini dell'ultimo mese",
    "mostrami i clienti premium"
]
```

La funzione `embed` provvedeva a vettorializzare le frasi, ossia trasformarle in sequenze di numeri, e la `cosineSimilarity` era semplicemente una distanza. 

Con questo approccio non c'era più bisogno di includere fra gli esempi le esatte parole e la classificazione diventava molto più efficiente e robusta rispetto alla precedente, basata solo su funzioni stringa.

Rimaneva, appunto, la parte del parsing del linguaggio naturale.

Ma lì la soluzione doveva essere ancora più drastica, come vedremo meglio più avanti.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*