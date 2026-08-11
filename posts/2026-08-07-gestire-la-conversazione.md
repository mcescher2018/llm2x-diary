---
layout: post
title: "Gestire la conversazione"
---

Confesso che introdurre i supporti conversazionali come *disambiguator*, *interpreter* e via dicendo, non era uno degli obiettivi iniziali.

L'ho fatto perché inizialmente mi era sembrata una cosa facile da ottenere e contemporaneamente *catchy*, dato che permetteva di far rientrare dalla finestra l'LLM che era stato sbattuto fuori dalla porta principale.

Ma ormai l'ho imparato: mai fidarsi delle "magnifiche sorti e progressive" che ti prospetta il genio.

Il motivo per cui avevo giudicato la cosa semplice era perché immaginavo i vari moduli conversazionali come delle chiamate ad Ollama con adeguati prompt.

Come ad esempio questo, relativo al *disambiguator*:

```javascript
const prompt = `
Sei un assistente che aiuta l'utente a chiarire cosa vuole chiedere.
Non generare SQL. Non proporre query. Non inventare dati.
Fai UNA sola domanda di chiarimento, breve e diretta.
Non proporre interpretazioni.

Messaggio utente:
"${message}"

Classificazione incerta. Chiedi una domanda utile per capire meglio.
`;
```

Il problema, però, non era realizzare i moduli, ma scrivere tutta la logica che gestisse la loro attivazione, nonché i relativi impatti nel flusso principale.

*Followup* ed *explainer* in questa implementazione aggiungono più o meno degli orpelli descrittivi ai dati generati dal sistema nel caso di `nl_query`. 

Questi sono stati effettivamente a basso impatto.

Ma solo perché ho scelto di lasciarli così, quasi come dei placeholder per implementazioni future.

In pratica, hanno comportato poca fatica ma non hanno aggiunto neanche valore tangibile.

Il *disambiguator* è stato più complesso, perché qui ha assunto il ruolo (importante) di intervenire nel caso di classificazioni incerte. 

Se faccio una domanda tipo "Mostrami l'efficienza raggiunta", la pipeline non ha elementi per costruire niente di sensato (quali entità, dimensioni, filtri): dunque deve prendere il controllo e provare a procurarsi quello che le manca. 

In pratica, è stato necessario introdurre una gestione di *early return* (conclusione anticipata del flusso principale, senza arrivare alle fasi di costruzione SQL ed esecuzione del comando) che risalisse la catena e presentasse all'utente una domanda tipo "Cosa intendi per efficienza?". 

La nuova risposta doveva sovrascrivere la precedente, *intent*, parametri e tutto il resto, come se fosse stata avviata una nuova conversazione. 

Infatti, il primo prompt era stato sostanzialmente identificato come rumore e, quindi, non aveva senso mescolarne le informazioni con quelle del secondo.

Ancora più complicata è stata la gestione degli *interpreter*, che in questa applicazione sono due: semantico e dedicato ai parametri. 

Il *semantic interpreter* interviene quando l'intento è chiaro ma la frase è vaga, come nel caso di "Mostrami il venduto". 

Perché è vaga? 

Perché non si sa se il venduto è generico o per una certa dimensione: questo, a livello di logica applicativa, è stato associato all'assenza di *marker* di alcuni tipi specifici come "totale", "per cliente", "più grande", "più piccolo". 

La funzione `isVague`, di importanza veramente elevata, è stata realizzata a regex, con un codice di questo tipo:

```javascript
export function isVague(message) {
    const text = message.toLowerCase();

    const hasGlobal = usesAggregationMarkers(message);
    const hasDimensionMarker = usesDimensionMarkers(message);
    const hasListMarker = usesListMarkers(message);
    const hasSuperlativeMarkers = usesSuperlativeMarkers(message);

    logSection("SENSE_CHECKER/IS_VAGUE", "Focus Indicators", { hasGlobal, hasDimensionMarker, hasListMarker, hasSuperlativeMarkers });

    const hasFocus = hasGlobal || hasDimensionMarker || hasListMarker || hasSuperlativeMarkers;

    return !hasFocus;
}
```

È sicuramente un punto di miglioramento da tener presente per il futuro. 

In ogni caso, qui, non c'è solo da ripassare la palla all'utente per chiedere chiarimento: occorre tenere in qualche modo in memoria l'analisi fatta sull'input iniziale e combinarla con il nuovo. 

Il router, l'elemento centrale della pipeline che agisce in base agli intenti, non può essere più *stateless* ma deve gestire un nuovo oggetto, una "pending interpretation", una struttura che conserva il contesto necessario per completare la richiesta.

Concettualmente è stato un salto importante, l'apertura di un fronte che non era mai stato testato in precedenza.

Il *params interpreter* genera anch'esso una pending interpretation, ma ha un meccanismo diverso di intervento. 

Entra in gioco quando la frase non è vaga ma manca di un parametro obbligatorio, come nel caso di "dammi il fatturato per oggetto". 

L'intent è `sales_by_dimension` ... ma qual è la dimensione richiesta? 

Ecco, quindi, che scatta una richiesta di chiarimento come nei casi precedenti, ma, stavolta, con un obiettivo molto più specifico: procurarsi il valore dei parametri mancanti.

Ma come fa il sistema a capire che manca un parametro obbligatorio? 

Lo ricava da un file di semantica, i parametri per intent, che nell'esempio in questione sarebbe utilizzato in questa parte:

```javascript
[INTENTS.SALES_BY_DIMENSION]: {
    requiresFact: true,
    requiresMeasure: true,
    requiresDimension: true
}
```

Tutto ciò, direttamente o attraverso sinonimi, con la dimensione che deve essere una di quelle previste dal *semantic model*.

L'aggiunta del supporto conversazionale mi ha costretto a scrivere molto nuovo codice.

Inizialmente dentro il router e poi in vari file accessori, corrispondenti a fasi, *helper* e altro, in modo da tornare ad una struttura leggibile.

Un *refactoring* cui ha contribuito anche l'insieme delle correzioni suggerite dai test, che vedremo con il prossimo post.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*