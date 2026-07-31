---
layout: post
title: "Deterministico, anche troppo"
---

Come nella *march madness* del basket NCAA, anche per gli informatici esiste un mese di pazzia. 

Ed ecco così che questo intenso Luglio mi ha permesso di tornare a scrivere solo oggi, a diversi giorni dal post precedente.

L'idea è adesso, una volta chiarite tutte le premesse e chiusa la parentesi dedicata agli *embedding*, concentrarsi sugli aspetti più importanti del progetto pubblicato in GitHub e, sicuramente, uno di questi è la natura deterministica della sua pipeline.

L'LLM, da attore principale nelle implementazioni iniziali, già in quelle intermedie precedentemente trattate, era diventato protagonista solo nella gestione dell'intent *smalltalk*, mentre per gli altri ambiti (funzioni MCP e elaborazione del linguaggio naturale), finiva ai margini potendo intervenire solo come *fallback*. 

Tutto questo riduceva allucinazioni e imprevedibilità del sistema, più una cosa a cui non avevo pensato: i tempi di risposta.

Infatti, anche se il modello utilizzato era un `llama3.1:8b` molto leggero, una risposta di Ollama impiegava circa 5-10" per uscire sul mio hardware, mentre una risposta deterministica era pressoché istantanea.

Così, facendo i test sul primo nucleo del nuovo codice che stava nascendo, la valutazione di quanto il sistema usava l'LLM era molto facile.

La risposta mi arrivò chiara quanto inattesa: quasi per niente.

Dunque, ero finito in una situazione paradossale: a forza di migliorare la mia esercitazione/demo sugli LLM, avevo spinto l'LLM in una situazione di quasi inutilizzo. 

Anche l'idea dell'approccio deterministico andava rivista o, per meglio dire, resa più coerente con il risultato voluto.  

Di conseguenza per "prendere il meglio fra le soluzioni LLMREPORT (intent fissi, template SQL) e LLMSQLPLUS (semantic layer, AST etc...)" - ho messo la citazione perchè questa era proprio parte del prompt che sottoposi a Copilot - pensai a ripartire con una nuova soluzione, con l'idea che fosse la finale di questo percorso, chiamandola LLMSQLPROD appunto.

Questi i focus principali della sua progettazione:
- mantenere la pipeline deterministica e la struttura backend/frontend
- mantenere il classificatore basato su *embedding*
- eliminare AST e trasformare il semantic layer in un *semantic model*, più circoscritto
- tornare a dei template SQL abbandonando l'approccio SQL generico
- far rientrare l'LLM in moduli di supporto conversazionale chiamati *disambiguator*, *explainer*, *followup* e *interpreter*

L'ultima parte era un po' la vera novità e fu una cosa che il solito genio mi presentò come una robetta da niente.

Un sacco di nuove *lessons learned*, ma un altro bagno di sangue.

[← Torna all’Index](../index.md) · Post successivo → *in lavorazione*


