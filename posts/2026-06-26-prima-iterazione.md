---
layout: post
title: "Prima Iterazione"
---

Qualcosa cominciava a definirsi.

Erano stati scelti i linguaggi con cui sviluppare le parti dell'applicazione, anche se per il momento volevo concentrarmi sul solo backend. 
Il database SQLite c'era e aveva già i suoi dati dimostrativi.

Decisioni essenziali, però, tipo quale doveva essere il ruolo dell'LLM e quale la modalità di interazione con il DB, non erano ancora state prese. 

Qui credo sia importante dire che l'architettura di LLMSQLPROD non è nata tutta insieme ma è cresciuta in modo molto meno lineare di come lo sto esponendo. 

Una testimonianza è anche il nome che il progetto ha assunto alla fine. 

Perché LLMSQLPROD? Perché è nato dopo una serie di altri tentativi LLM* più o meno ambiziosi o funzionanti. Il suffisso PROD non indicava un prodotto production‑ready, ma semplicemente che quella versione era la candidata a diventare la demo pubblicata con i documenti di supporto che state leggendo.

La prima applicazione della famiglia si chiamava LLMREPORT e credo sia quella che, pur con tutti i suoi limiti, ha tracciato la strada più di ogni altra. 

I primi commenti sul codice, quindi, si riferiranno proprio a lei.

Intanto, non c'era frontend ed il server consisteva in un main file con pochi moduli importati. 

Il dialogo con il modello linguistico avveniva con una funzione javascript molto simile al codice CURL precedentemente riportato:

```javascript
import ollama from 'ollama';

export async function askLLM(prompt) {
  const response = await ollama.chat({
    model: "llama3.1:8b",
    messages: [
      { role: "user", content: prompt }
    ]
  });

  return response.message.content;
}
```

La struttura del server, import esclusi, era questa:

```javascript
const app = Fastify();
app.post('/chat', async (req, reply) => {
  const text = req.body.text;
  const intent = await intentClassifier.classify(text);
  const report = reportRouter.get(intent.intent);
  const params = await paramParser.parse(text, report);
  const { sql, bindings } = queryBuilder.build(report.report_id, params);
  const rows = await sqlExecutor.execute(sql, bindings);
  return resultFormatter(rows, report);
});

app.listen({ port: 3000 });
```

La chiamata avveniva da linea di comando con una istruzione come la seguente:

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "text": "dammi il fatturato per cliente"
  }'
```

Questa, infine, era la risposta che si otteneva:

```bash
  id  name     revenue
----  -------  -------
1,00  Rossi      70,00
2,00  Bianchi    25,00
```

Poco? Forse. Ma avrei capito solo dopo quanto il piccolo fosse bello.

---

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*
