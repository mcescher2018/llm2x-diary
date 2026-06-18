# Muovere il primo passo

Ne *Il Ritorno di Endymion*, il capitolo finale del grandissimo ciclo di Hyperion di Dan Simmons, Colei‑Che‑Insegna dice, a proposito del come accedere al Vuoto‑Che‑Lega, questa frase criptica: “Il primo passo è… muovere il primo passo”.

Nel piccolo, il mio problema era simile. Quello che volevo fare — senza aver mai sentito nemmeno citare il termine LLM2SQL — era connettere un LLM a un database.  
E quindi la prima domanda che mi è sorta è stata: “Dove lo trovo un LLM che posso installare in locale, e come posso colloquiarci?”.

La risposta è stata abbastanza semplice: servono **Ollama + un modello**.

> Disclaimer: a volte dirò cose banali, ma l’idea è davvero quella di partire da zero, perché è quello che ho fatto nel mio percorso.

Ollama è sostanzialmente un’applicazione che tiene internamente uno o più modelli fra cui scegliere e permette di colloquiarci tramite chiamate API come questa:

```bash
curl http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:8b",
    "messages": [
      {
        "role": "user",
        "content": "Come stai?"
      }
    ],
    "stream": false
  }'
```

Un modello è un file binario che, in prima approssimazione, contiene parametri e organizzazione della rete neurale con cui è stato realizzato il sistema linguistico: il programma che magicamente è in grado di rispondere a una frase in linguaggio naturale tipo “Come stai?” con un’altra apparentemente sensata come “Sono un’intelligenza artificiale…”.

Lavorando di norma su Windows, i primi tentativi sono stati con quel sistema operativo e con strumenti GUI semi‑pacchettizzati come GPT4ALL o AnythingLLM.
Ho ottenuto diversi problemi di instabilità e, cercando in rete, ho trovato che passare a Linux poteva essere una buona idea.

Mi sono creato una macchina virtuale Ubuntu Server 24.02 text‑only, raffinando così un minimo l’idea iniziale: realizzare la parte server su Linux e usare Windows come semplice client web.

Ho installato Ollama, attivato il modello llama3.1:8b e ottenuto una risposta al mio comando CURL “Ciao come stai?” in tempi accettabili anche su hardware domestico, tipo questa:

```json
{
    "model":"llama3.1:8b",
    "created_at":"2026-06-12T15:11:50.883974128Z",
    "message":
        {
            "role":"assistant",
            "content":"Sto bene, grazie! Sono un'intelligenza artificiale, quindi non ho emozioni o sentimenti come gli esseri umani, ma sono qui per aiutarti e rispondere alle tue domande in ogni momento. Come posso aiutarti oggi?"
        },
    "done":true,
    "done_reason":"stop",
    "total_duration":24766987268,
    "load_duration":14924923125,
    "prompt_eval_count":14,
    "prompt_eval_duration":1250159482,
    "eval_count":63,
    "eval_duration":8535631136
}
```

Confesso di essermi emozionato.

---

[← Torna all’Index](../index.md) · [Post successivo →](2026-06-18-il-genio-della-lampada.md)
