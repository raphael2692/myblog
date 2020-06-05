---
categories:
- tutorial
date: "2020-06-05T12:48:08+01:00"
description: Come creare un progetto datascience fullstack. Prima parte, creare un API personalizza con node, express e puppeteer! 
slug: fullstack-datascience-project-tutorial-parte-1
tags:
- progetti
- tutorial
- datascience
title: "Fullstack Datascience (parte I): Creare un API personalizza con node, express e puppeteer!"
---

# Fullstack Datascience (parte I)
* * * 
## Indice
### [Introduziione](#Introduzione)
### [Parte I: Creare una fonte di dati](#parte-i-creare-una-fonte-di-dati)

* * * 
## Introduziione 

### TL;DR
Questo è il primo tutorial che pubblico, vi prego di avere pietà e perdonarmi se suona un po' 'aranzullese'.
Se siete capitati qui googlando e vi serve solo il codice lo trovate su github.  🎉🎉🎉

### *Fullstack what*?
Ok, il termine è un po' pretenzioso. In pratica voglio condividere un progetto che comprenda queste funzioni:

1. Automatizzare il processo di scraping di un sito e trasformarlo in una 'Restful API' **(parte I)**
2. Creare un client che periodicamente richiami l'API e carichi i dati in un DB
3. Creare una web app che consumi i dati di questo DB e crei delle infografiche 
4. Creare un modello di machine learning che consumi la stessa fonte dati e la 'serva' in un'altra API
5. Deployare il tutto su una piattaforma cloud gratuita: [heroku](https://dashboard.heroku.com/login)

Ok questa lista è ancora più pretenziosa...😂

### Ma io voglio fare il *Datascientist*, perché dovrebbe interessarmi tutta 'sta roba?

Se come me lavori in questo settore da qualche tempo probabilmente già lo sai, altrimenti...è un modo (spero) divertente per familiarizzare con tecnologie che 
ti troverai a utilizzare tutti i giorni.  

### La mission del *Datascientist*: creare valore dai dati...

...e altri spot accalappiasghei che circolando sul web: cosa vogliono dire esattamente? Probabilmente nulla. Ma sedovessi scommettere tra assumere 
un dottorato in astrofisica per migliorare l'accuratezza di un modello open source dal 99,8% al 99,9% e assumerne un altro che mastichi un po' di backend preferirei sicuramente il secondo. Ok scherzo, in realtà scrivo così perché non ho un dottorato in astrofisica 😔...


## Parte I: Creare una fonte di dati

### Nella vita esistono diversi gradi di *swag*...

1. Lavorare su dati tabulari statici (ad es. il leggendario Iris Dataset)
2. Craere i propri dati tabulari a partire da fonti dinamiche (tipo una web API)
3. Fare scraping di un sito da un app locale qualora non esistesse un API appropriata (o qualora non potremmo permetterci il costo della chiave di attivazione...)
4. Fare scraping di un sito attraverso un API fatta da noi e usarla come 'microservizio' su tutti i nostri progetti presenti e futuri (o almeno, finchè non cambi la struttura del sito e possiamo buttare nel ... la nostra API)

Di seguito vi mostro come ho realizzato il punto 4. 

### Domande e riposte

- Con che tipo di dati voglio lavorare? Testuali
- Che tipo di testi posso cercare? Qualcosa di breve, ma nemmeno banale come i tag dei prodotti di amazon...i titoli di un giornale!
- Ok, quale giornale? Qualcosa di divertente, così mi annoio di meno durante i test...\**idea malvagia*\* **Libero**!


### Come funziona

![giornalismo di qualità](/articolo_libero.png)

Quest'artistico titolo e tutti i suoi fratellini si trovano da qualche parte nella pagina html del sito [liberoquotidiano.it](https://www.liberoquotidiano.it/). 


```html
<h2>
    <a href="https://www.liberoquotidiano.it/news/politica/23076003/giuseppe-conte-mes-resa-conti-senato-cazzata-crisi-politica-non-aggirabile.html">
    “È una caz***a”, l’ira grillina travolge Conte. Tutto deciso, c’è già la data (vicina) della “crisi politica”
    </a>
</h2>

```
La struttura del sito è molto semplice: tutti i titoli sono i valori di tag anchor all'interno di tag h2. Per tirarli fuori useremo [Puppeteer](https://github.com/puppeteer/puppeteer), una libreria di node.js che include una versione 'headless' del browser chromium (il fork open source di chrome). 

```javascript

// scraper.js

const puppeteer = require('puppeteer');
const userAgent = require('user-agents');

fetch = async (url) => {
    const startTime = Date.now();
    const scrapeDate = new Date();
    const browser = await puppeteer.launch({
        headless: true,
        defaultViewport: null,
        args: [
            '--no-sandbox',
            '--disable-setuid-sandbox',
        ]
    });
    try {
        const page = await browser.newPage();
        await page.setUserAgent(userAgent.toString());
        await page.goto(url, { waitUntil: 'networkidle2' });
        const html = await page.evaluate(
            function callback() {
                href_array = Array.from(document.querySelectorAll('h2>a'), element => element.getAttribute("href"));
                text_array = Array.from(document.querySelectorAll('h2>a'), element => element.innerHTML);
                return { href_array: href_array, text_array: text_array }
            }
        )
        // questa variabile serve come buffer per non inserire titoli duplicati
        const container = { text: {}, href: {} };
        // questo array serve come struttura finale da condividere
        const data = new Array;
        html.text_array.forEach(function (element, index) {
            // questo blocco filtra i risultati
            if (!(element in container.text) && (element.split(" ").length >= 3)) {
                container.text[index] = element
                container.href[index] = html.href_array[index]
                // questo blocco forma l'array finale
                data.push({
                    //l'id è formato preso dall'url dentro l'attributo href. NOTA: alcuni url sono 'invalidi' ma sono stati filtrati selezionando i text con grandezza > 3
                    id: html.href_array[index].match(/\(|\)|\d{8}/)[0],
                    text: element,
                    href: html.href_array[index],
                    source: "LiberoQuotidiano",
                    date: scrapeDate.toISOString()
                })
            }
        });
        //console.log(container)
        console.log({ data })
        await browser.close();
        console.log('Tempo impiegato: ', (Date.now() - startTime) / 1000, 's');
        // è importate restituire un oggetto
        return { data };
    } catch (error) {
        console.log("ERRORE: " + error)
    } finally {
        browser.close();
    }
}

// per importare le classi (funzioni) definite qui come moduli
module.exports.fetch = fetch

```
La logica del programma è racchiusa nella funzione ```fetch()``` che si occupa di prendere
tutti i seguenti dati:

- **id** (che fa parte dell'url di ogni articolo ed è estratto con una regex)
- **text** (il titolo vero e proprio)
- **href** (a ogni titolo corrisponde un url al relativo articolo...utile per esapendere in futuro la nostra api!)
- **source** (tag generico per identificare la fonte del titolo, nel caso un domani questi dati confluiscano in un DB con titoli da altre fonti)
- **date** (timestamp del momento in cui vengono raccolti i dati)

A complicare leggermente le cose è il fatto la struttura ```h2>a``` appartiene anche a elementi che non sono titoli, e che non ci interessano. 

Se non vi torna qualcosa di questo snippet probabilmente dovrete ripassare:

- i concetti base di programmazione asincrona con javascript (js)
- il concetto di callback
- l'oggetto Promise
- il concetto di deserializzazione

Perché javascript? Javascript è il linguaggio alla base dell'interazione tra browser web. L'obiettivo di questa guida è aprire qualche orizzonte rispetto al solito stack di strumenti da datascientist (python/R). Se per voi è nuovo, vale la pena perdere un po' di tempo per capire come funziona il web e javascript. I temi che ho elencato sopra sono trattati approfonditamente su [MDN (Mozzilla Developers Network)](https://developer.mozilla.org/it/). 

Notare l'utilizzo di una shotcut ```{data}``` per nestare tutto in un solo array, per evitare problemi di deserializzazione tra browser.

L'ultima riga serve a esportare questa funzione e renderla richiamabile da un altro script js. Il tutto è nestato in un blocco ```try/catch/finally``` per evitare memory leaks 
e debuggure gli errori più facilmente.

Una volta definita questa funzione, dobbiamo fare in modo che questa venga eseguita su richiesta. Grazie a express creare un web server è estramente intuitivo. La nostra funzione ```fetch()``` verrà eseguita ogni qualvolta un client (un browser) cercherà di andare all'indirizzo **https://localhost:3000/liberoquotidiano**.

```javascript

// app.js 

const express = require("express");
const scraper = require("./scraper");
const app = express();

app.use(express.static("public"))

app.get("/", function (req, res) {
    res.send("<h1> CUSTOM API (LiberoQuotidiano scraping) v 0.1 </h1>")
})

app.get('/liberoquotidiano', function (req, res) {
    scraper.fetch("https://www.liberoquotidiano.it/").then( function callback(something) {
    return res.json(something);
    })
  })

app.listen(process.env.PORT || 3000,
    () => console.log("Server is running..."));

```
E il risultato dell'interrogazione è il seguente (3 di 65 risultati):

```json

{"data":[
    {
    "id":"23076053","text":"\"Volano parole grosse\". L'accordo di Arcore è superato: Meloni-Salvini, contrasti sulle amministrative",
    "href":"https://www.liberoquotidiano.it/news/politica/23076053/meloni-salvini-contrasti-amministrative-volano-parole-grosse-accordo-arcore-superato.html","source":"LiberoQuotidiano",
    "date":"2020-06-05T15:06:34.487Z"
    },
    {
    "id":"23079879",
    "text":"\"Ha depositato nome e simbolo\". La voce dal Palazzo: il partito di Conte che fa tremare i 5 Stelle",
    "href":"https://www.liberoquotidiano.it/news/politica/23079879/giuseppe-conte-depositato-nome-simbolo-partito-tremano-5-stelle.html",
    "source":"LiberoQuotidiano",
    "date":"2020-06-05T15:06:34.487Z"
    },
    {
    "id":"23076003",
    "text":"“È una caz***a”, l’ira grillina travolge Conte. Tutto deciso, c’è già la data (vicina) della “crisi politica”",
    "href":"https://www.liberoquotidiano.it/news/politica/23076003/giuseppe-conte-mes-resa-conti-senato-cazzata-crisi-politica-non-aggirabile.html",
    "source":"LiberoQuotidiano",
    "date":"2020-06-05T15:06:34.487Z"
    }

```
## Nelle prossime puntate

E questo è quanto. Nella prossima mostrerò come creare un database relazionale ([postgreeSQL](https://www.postgresql.org/)) all'interno di un cloud gratuito (heroku) e come creare un client python che in automatico periodicamente aggiunga dati da questa API (o magari un'altra!). 
















