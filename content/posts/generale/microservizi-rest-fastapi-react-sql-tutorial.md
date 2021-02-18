---
categories:
- tutorial
date: "2021-02-17T12:48:08+01:00"
description: Esempio minimale di microservizio web REST con lo stack FastAPI, SQL, React e Docker. 
slug: microservizi-rest-fastapi-react-sql-docker-tutorial
tags:
- progetti
- tutorial
- datascience
- microservizi
- web
title: "Introduzione ai microservizi REST con lo stack FastAPI/SQL/React (e Docker)"

---

* * * 
### [Intro](#intro)
### [Backend](#backend)
### [Frontend](#frontend)
### [Deploy](#deploy)
### [Conclusioni](#conclusioni)
* * * 

## Intro

In questo post racconto come craere un app di base utilizzando lo stack FastAPI (python), SQL(...lite) e React (javascript). Per quest'esempio utilizzo FastAPI ma i concetti possono essere estesi a qualsiasi altro framework, con qualche distinguo. Non è un tutorial step-by-step, ma una conoscenza basilare di python e javascript dovrebbe essere sufficiente a comprendere il codice (che come sempre è disponibile su [github](https://github.com/raphael2692/fullstack-bookmarks-manager)). Penso possa considerarsi un buon caso di studio per chi vuole infarinatura veloce su questi temi o per chi solitamente usa altri stack e non conosce quello proposto. 

L'applicazione di esempio, 'Bookmarks Manager', è il classico esempio di applicazione con funzionalità CRUD (giusto per non fare il solito esempio con i TODO...). L'applicazione è divisa in due servizi, */backend* che contiene la definizione degli endpoint e il database, e */frontend* che contiene una dashboard per visualizzare e modificare i record. Entrambe le componenti sono dockerizzate e vengono inizializzate dallo stesso file docker-compose.yml.

# Backend

### Definire l'entry point dell'applicazione
Per fare questa piccola app sono partito dal backend. Il primo step è definire l'entry-point dell'applicazione nello script main.py e istanziare classe base.

```python
# main.py
from fastapi import Depends, FastAPI

app = FastAPI()
```

### Connettere un database SQL

Per connettere l'applicazione al database si utilizza un ORM, sqlalchemy.  Un ORM (Object Relationship Management) è un'interfaccia software che standardizza diverse azioni e le traduce in delle query SQL. Nel file main.py viene definito l'engine e creata l'istanza (Session) che verrà passata alle altre funzioni del programma. Nel file database.py invece istanziamo l'engine e il modello base. 

```python
# main.py
from sqlalchemy.orm import Session
from .database import SessionLocal, engine

models.Base.metadata.create_all(bind=engine)

)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

```

```python
# database.py 

from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./bookmarks.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()
```

### Definire le entità del database
Successivamente definiamo le entità principali che dobbiamo manipolare e che conserveremo nel db. Solitamente, nel design di un database relazionale le entità corrispondone alle tabelle (*tables*).

```python
# database.py
from sqlalchemy import Boolean, Column, ForeignKey, Integer, String

from .database import Base

class Bookmark(Base):
    __tablename__ = "bookmarks"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    url = Column(String, index=True)
```
### Definire gli schemi (*models*)
Dopo è necessario associare alle entità create degli schemi di validazione utilizzati da FastAPI. In questo contesto, i modelli rappresentano le **interfacce** delle entità che creiamo in database e che manipoliamo attraverso l'applicazione web. Tipizzare le interfacce è fondamentale per definire per non incorrere in errori. In quanto classi inoltre possiamo trasferire gli attributi e creare relazioni sfruttando l'ereditarietà. 

🚀 **Pro-tip**: Non tutti i modelli che definiamo devono corrispondere a entità del database. Ad esempio qui abbiamo definito un modello "Transaction" utile a codificare in maniera personalizzata l'esito di un operazione. I modelli vengono definiti nel file 'models.py'. 

```python
from typing import List, Optional
from pydantic import BaseModel

class BookmarkBase(BaseModel):
    name: str
    url: str

class BookmarkCreate(BookmarkBase):
    pass

class Bookmark(BookmarkBase):
    id: int

    class Config:
        orm_mode = True

class Transaction(BaseModel):
    status: bool
    message: str
```

### Definire le funzionalità applicative
In un file separato, 'crud.py' (giusto per capire di cosa si tratta...), definiamo le funzionalità vere e proprie dell'applicazione. In questo caso definiamo come leggere tutti i o un record dal db, cancellarne uno, eccetera. Notate che qualsiasi funzione python, con i giusti accertamenti può essere chiamata in questo contesto. Questo rende FastAPI **lo strumento ideale per deployare sul web modelli di ML e app basati su di essi**. Nota per i pignoli: non ho definito una funzione di update/edit dei record, perché mi sembrava che l'articolo fosse già tropo lungo...però liberissimi di farmi una pull request su github 😊. 

```python
from sqlalchemy.orm import Session
from . import models, schemas

def get_bookmarks(db: Session, skip: int = 0, limit: int = 100):
    return db.query(models.Bookmark).offset(skip).limit(limit).all()

def get_bookmark(db: Session, id:int):
    return db.query(models.Bookmark).get(id)

def create_bookmark(db: Session, item: schemas.BookmarkCreate):
    db_item = models.Bookmark(**item.dict())
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item

def delete_bookmark(db: Session, id:int):
    if db.query(models.Bookmark).get(id):
        db.query(models.Bookmark).filter_by(id=id).delete()
        db.commit()
        return {"status": True, "message":f"Record {id} deleted"}
    else:
        return {"status": False, "message":"No such record"}
```

### Definire gli endpoint (*path operations*)
Una volta definite entità e funzionalità, non rimane che mettere tutto insieme e creare un endpoint, vale a dire associare a una richiesta http la chiamata a una certa funzionalità che a sua volta richiede, restituisce e/o manipola una certa entità, o a sua volta richiama un'altra funzionalità, che a sua volta manipola un'altra entità...eccetera 😄. 

```python
# main.py

@app.post("/bookmarks/", response_model=schemas.Bookmark)
def create_bookmark(
    bookmark: schemas.BookmarkCreate, db: Session = Depends(get_db)
):
    return crud.create_bookmark(db=db, item=bookmark)


@app.get("/bookmarks/", response_model=List[schemas.Bookmark])
def read_bookmarks(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    items = crud.get_bookmarks(db, skip=skip, limit=limit)
    return items

@app.get("/bookmarks/{id}", response_model=schemas.Bookmark)
def read_bookmark(id:int, db: Session = Depends(get_db)):
    item = crud.get_bookmark(db=db, id=id)
    return item

@app.delete("/bookmarks/{id}", response_model=schemas.Transaction)
def delete_bookmark(id:int, db: Session = Depends(get_db)):
    msg = crud.delete_bookmark(db=db, id=id)
    if msg["status"]  == False:
        raise HTTPException(status_code=404, detail=msg["message"])
    return msg
```

### Definire i test
Nello sviluppo di applicazioni complesse (e qui a titolo esemplificativo) è fondamentale sviluppare dei test per ciascun endpoint. Di buona regola inoltre si procede gradualmente, scrivendo un endpoint, il relativo test, e così via. Esiste poi una metodologia chiamata TDD (Test Driven Development) per la quale è normativo scrivere prima il test e poi l'endpoint. Sicuramente si tratta di una strategia ottimale quando è necessario sviluppare funzionalità complesse perché complementare al principio 'divide et impera'. Per superare un problema complesso è utile scomporlo in sotto problemi, che diventano una serie di sotto test che vengono eseguiti in iterativamente fino alla soluzione, e successivamente sono utili come 'breakpoint' di controllo intermedi quando testiamo le suddette funzionalità complesse su diversi casi.  FastAPI è perfettamente integrata con il module [pytest](https://docs.pytest.org/en/stable/contents.html), quindi per definire i test basta creare un file con il prefisso "test_" e lanciare il comando "pytest", una volta installato il pacchetto. Il test è basato sugli statement [assert](https://www.w3schools.com/python/ref_keyword_assert.asp) di python. 

🚀 **Pro-tip**: Nelle applicazioni complesse, una delle fasi più delicate (e interessanti) è il design del database. Formalmente lo strumento miglire sono i diagrammi UML, facilmente creabili ad esempio con l'ottima app [drawio](https://app.diagrams.net/), ma senza dubbio il modo più divertente e ragionarci con una lavagnetta insieme a tutto il team di sviluppo  😊. 

```python
# test_main.py
from fastapi.testclient import TestClient
from .main import app

client = TestClient(app)

def test_read_bookmarks():
    response = client.get("/bookmarks/")
    response_body = response.json()
    print(response_body)
    assert response.status_code == 200
    assert type(response_body) ==  list
    if len(response_body) > 0:
            keys = []
            for i in response_body:
                for key in i.keys():
                    if key not in keys:
                        keys.append(key)
                assert len(keys) == 3
                assert "id" in keys
                assert "name" in keys
                assert "url" in keys

def test_read_bookmark():
    response = client.get("/bookmarks/")
    response_body = response.json()
    if len(response_body) > 0:
        response = client.get("/bookmarks/1")
        response_body = response.json()
        print(response)
        keys = []
        for key in response_body.keys():
            if key not in keys:
                keys.append(key)
        assert "id" in keys
        assert "name" in keys
        assert "url" in keys
```


### Testare l'integrabilità dell'applicazione
Uno dei motivi per consiglio sempre di utilizzare FastAPI, oltre la compatibilità con l'ecosistema python, è nativamente compatibile allo standard [OpenAPI](https://swagger.io/specification/). Tant'è che integra una pagina per i test di tutti gli endpoint creati, automaticamente, consultabile sia durante lo sviluppo che in fase di integrazione con servizi esterni o altre componenti del framework che state progettando. 

![screen_fastapi](/fastapi_screen.png)


## Frontend

### Creare il progetto

Per creare velocemente un progetto react consiglio di usare il boilerplate di 'react-create-apt' attraverso npx:

```shell
npx create-react-app frontend
```
Che permette di avere al volo un ambiente di sviluppo:

```shell
cd frontend
npm start
```
### Componenti principali
I componenti principali dell'applicazione sono due: PostBookmark che contiene i form per aggiungere un nuovo bookmark al database, e GetBookmark che contiene sia la view del metodo 'get("/bookmarks/ ...)' che una callback per cancellare un dato record. Per gestire alcuni aspetti ho usato gli hooks di react, ma non è fondamentale. Personalmente però trovo molto più comodo usarli ed evitare del tutto di scrivere delle classi. Per lo stile ho usato la libreria Material-UI che offre dei componenti intuitivamente utilizzabili al posto dei classici tag html, con tutti i benefici del caso (in particolare, passare le opzioni di stile attraverso le props). 

#### PostBookmark.js
```JSX

import { useState, useEffect } from "react";
import { useForm } from "../utils/useForm";

import Typography from "@material-ui/core/Typography";
import TextField from "@material-ui/core/TextField";
import Button from "@material-ui/core/Button";

function PostBookmark(props) {

  const [values, handleChange] = useForm({ formName: "", formUrl: "" });

  const postBookmark = (params) => {
    fetch("http://localhost/bookmarks/", {
      method: "POST",
      headers: {
        Accept: "application/json",
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        name: params.formName,
        url: params.formUrl,
      }),
    });
  };

  return (
    <div>
      <Typography variant="h5">components/PostBookmark</Typography>
      <form>
        <TextField
          label="name"
          type="text"
          name="formName"
          value={values.formName}
          onChange={handleChange}
        />
        <br />

        <TextField
          label="url"
          type="text"
          name="formUrl"
          value={values.formUrl}
          onChange={handleChange}
        />
        <br />
        <br />
        <Button
          variant="contained"
          type="submit"
          value="POST -> createBookmark"
          onClick={() => postBookmark(values)}
        >
          Add bookmark
        </Button>
      </form>
      <br />
      <br />
    </div>
  );
}

```

#### GetBookmark.js
```JSX
import React from "react";
import { useFetch } from "../utils/useFetch";
import { useState } from "react";

import Typography from "@material-ui/core/Typography";
import Button from "@material-ui/core/Button";

import Table from "@material-ui/core/Table";
import TableBody from "@material-ui/core/TableBody";
import TableCell from "@material-ui/core/TableCell";
import TableContainer from "@material-ui/core/TableContainer";
import TableHead from "@material-ui/core/TableHead";
import TableRow from "@material-ui/core/TableRow";
import Paper from "@material-ui/core/Paper";

const GetBookmarks = (params) => {

  const { data, loading } = useFetch(
    `http://localhost/bookmarks/?skip=0&limit=100`
  );

  async function deletePost(url) {
    await fetch(url, { method: "DELETE" }).then(window.location.reload());
  }

  const renderTr = (array) =>
    array.map((el) => (
      <TableRow key={el.id}>
        <TableCell>{el.id}</TableCell>
        <TableCell>
          {" "}
          <a href={el.url} target="_blank">
            {el.name}
          </a>
        </TableCell>
        <TableCell>
          <Button
            variant="contained"
            color="secondary"
            onClick={() => deletePost("http://localhost/bookmarks/" + el.id)}
          >
            DELETE
          </Button>
        </TableCell>
      </TableRow>
    ));

  if (data) {
    return (
      <div>
        <Typography variant="h5">components/GetBookmarks</Typography>
        <TableContainer component={Paper}>
          <Table className="BookmarksTable" aria-label="simple table">
            <TableHead>
              <TableRow>
                <TableCell>ID</TableCell>
                <TableCell>URL</TableCell>
                <TableCell></TableCell>
              </TableRow>
            </TableHead>
            <TableBody>{renderTr(data)}</TableBody>
          </Table>
        </TableContainer>
      </div>
    );
  }

  else {
    return "Loading data...";
  }
};

export default GetBookmarks;

```
### Risultato finale

Un po' spartano, ma l'importante è essere chiari:

![screen_frontend](/bookmarks_manager_front.png)

## Deploy 

Una volta sviluppate le funzionalità di base è utile definire un file [Docker](https://www.docker.com/) per deployare rapidamente la soluzione su qualsiasi piattaforma cloud o server Linux locale (io uso un'ottima distro Debian, [Pop OS!](https://pop.system76.com/), versione LTS 20.04). In questo caso ho creato due Dockerfile, uno per il backend e uno per il frontend. 

### Container Backend
```dockerfile
FROM tiangolo/uvicorn-gunicorn-fastapi:python3.8
COPY ./app /app 
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip install pip --upgrade
RUN pip install -r requirements.txt
COPY . /app
```
### Container Frontend
```dockerfile
FROM node:13.12.0-alpine
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json ./
COPY package-lock.json ./
RUN npm install --silent
RUN npm install react-scripts@3.4.1 -g --silent
COPY . ./
CMD ["npm", "start"]
```
### Orchestratore

Un altro vantaggio della piattaforma docker è di poter connettere rapidamente dei database dedicati alle nostre applicazioni (ad esempio invece che usare un'istanza SQLlite avremmo potuto connettere un container docker) e 'orchestrare' tutte le applicazioni del nostro framework attraverso un semplice file di configurazione ("docker-compose.yml"). Di seguito il contenuto del file e la struttura finale dell'applicazione:

```shell
version: '3.3'
services: 
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - 80:80
    volumes:
      - ./backend:/app
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - 3000:3000
    volumes:
      - ./frontend:/app
```

![screen](/bookmark_manager_dir.png)

## Conclusioni

Se ave trovato lo stack interessante potete leggere di più al riguardo sul sito di [FastAPI](https://fastapi.tiangolo.com/), che oltre documenta il codice in maniera approfondita ma assumendo che il lettore non sia un esperto. Per quanto riguarda il frontend, penso che React sia la soluzione più flessibile, ma esistono tante altre opzioni o framework, ad esempio, [Vue.js](https://vuejs.org/) se si vuole un framework javascript...con meno javascript o [Jinja2](https://jinja.palletsprojects.com/en/2.11.x/) (che può essere [servito direttamente da FastAPI](https://fastapi.tiangolo.com/advanced/templates/)) se preferiamo lavorare con template html. 

Seppur minimale quest'esempio costituisce lo scheletro di un'architettura a microservizi. Sul pezzo fondamentale che manca, un sistema di brokeraggio e load balancig penso che scriverò qualcosa nei prossimi post. 



