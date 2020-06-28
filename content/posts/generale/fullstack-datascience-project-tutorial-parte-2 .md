---
categories:
- tutorial
date: "2020-06-28T12:48:08+01:00"
description: Come creare un progetto datascience fullstack. Deployare un API su cloud (Heroku), creare un db online (postgre, Heroku) e un client per automatizzare l'interrogazione all'API e l'upload dei dati (python, flask)

slug: fullstack-datascience-project-tutorial-parte-2
tags:
- progetti
- tutorial
- datascience
title: "Fullstack Datascience (parte II): Deployare un API su cloud, connetterla a un DB e automatizzare l'upload tramite interrogazione REST"

---

# Fullstack Datascience (parte II)
* * * 
## Indice
### [Introduzione](#Introduzione)
### [Deploy di libero-API su Heroku](#deploy-di-libero-api-su-heroku)
### [Creare un db postgre su Heroku](#creare-un-db-postgre-su-heroku)
### [Creare il client con flask](#creare-il-client-con-flask)
### [Nelle prossime puntate](#nelle-prossime-puntate)
* * * 

## Introduzione 

In questa parte di tutorial facciamo il deploy dell'[app di scraping creata precedentemente](https://www.raffaelespataro.it/posts/generale/fullstack-datascience-project-tutorial-parte-1/) sulla piattaforma Heroku. In questo modo la nostra app diventa una vera e propria web API, che risponderà con un JSON quando interrogata a un certo indirizzo. 

Ci sono molti modi in cui possiamo utilizzare un'app configurata in questo modo. Ad esempio potremmo costruire un nuovo 'front-end' del quotidiano Libero, magari da integrare in un app mobile di notizie (la maggiorparte di applicazioni di notizie su android e ios funzionano sotto la scocca in maniera analoga). 

Nel caso di un'applicazione datacentrica, sia che vogliamo creare visualizzazioni interessanti, sia che vogliamo creare un modello di machine learning, ci interessa avere a disposizioni più dati possibile. 

Per questo creeremo un databse online dove immagazzinare, con un un altro servizio, tutti i nostri dati. 

![banner heroku](/heroku_banner.png)


## Deploy di libero-API su Heroku

Heroku è una piattaforma ideale per questo tipo di esperimenti. Sia perché è gratis, sia perchè è molto semplice da usare. In pratica, se conoscete git, deployare un'app su Heroku coincide al 90% con il processo di pushare un repository su un host come github o gitlab. 

Per iniziare, bisogna innanzitutto creare un'utenza sul sito ufficiale di [Heroku](https://devcenter.heroku.com/). 

🚀 **Pro-tip**: se volete sperimentare, non usate la vostra e-mail principale, ma createne una [usa e getta](https://temp-mail.org/it/). 

Una volta creato un account, dovete installare il client Heroku sul vostro PC. Si tratta di uno strumento a linea di comando, [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli). Heroku supporta tutte le piattaforme, ma con dei limiti. Infatti su windows, con alcuni framework, tipo flask, non funziona il comando `heroku local` che permette di simulare in locale il funzionamento dell'applicazione, per un deploy senza sorprese. Per questo suggerisco di usare linux, o seguire il tip di seguito.

🚀 **Pro-tip**: si tratta di un passaggio opzionale, ma se volete semplificarvi la vita nello sviluppo di piccoli progetti su windows, potete abilitare il [sottosistema linux per windows](https://docs.microsoft.com/it-it/windows/wsl/install-win10). Unito al [nuovo terminale windows](https://github.com/microsoft/terminal) (ci sono le schede, horray), è davvero comodo, e vi permetterà di sviluppare più velocemente. 

Una volta installata la heroku cli, per deployare la nostra app, ['libero-API'](https://www.raffaelespataro.it/posts/generale/fullstack-datascience-project-tutorial-parte-1/) non dobbiamo far altro che recarci nella directory del nostro progetto, inizializzare un repo e pusharlo su heroku:

```terminal

git init
git add .
git git commit -m "conoscete git, giusto? ;)"

heroku login

***
Vi verranno chieste le creadenziali
***

heroku create <nome-applicazione>
git push heroku master

***
A console apparirà il log di installazione della vostra app su Heroku. 
Se il processo va a buon fine leggerete il mitologico: BUILD SUCCESS e un URL alla vostra app
***

```

Aaand *that's it*,  libero API è pronto a distillare trollagine all'end-point '\<heroku-app-url>\/liberoquotidiano'. 

Per monitorare lo stato dell'app direttamente dal terminale, si può digitare `heroku console` dalla dir del progetto. 

## Creare un db postgre su Heroku

Per quest'esperimento basterà eseguire tutte le operazioni da web ed eseguire queste operazioni:

1. Connettersi al proprio account web Heroku
2. Creare una nuova applicazione web, ad es. "my-db"
3. Dirigersi a quest'URL (sostituire la stringa tra tag): `https://dashboard.heroku.com/apps/<my-db>/resources`
4. Nella bassa di ricerca degli add-on aggiungere "Heroku Postgre"
5. Una volta creato il db, lo troverete elencato a questo URL: https://data.heroku.com/datastores
6. Una volta selezionato "my-db" navigare sul pannello settings e copiarsi da qualche parte l'**URI**. L'URI continene in un'unica stringa l'indirizzo del vostro db e le credenziali di accesso. 


## Creare il client con flask

Ora che abbiamo un API e un database, non ci resta altro che creare un app che li metta in comunicazione. Potrebbe essere realizzata con qualasiasi linguaggio, 
ma in questo caso useremo python3 (3.7.x) e un framework chiamato flask. In questo modo oltre a risparmiarci molto 'boilerplate' e rendere più intuitivo il codice, potremo utilizzare strumenti familiari in ambito datascience, in primis pandas. 

Di seguito uno screen della dir di progetto:

![banner heroku](/screen_libero_data_uploader.png)

Il file app.py contiente tutto il codice a far partire la nostra web app, che è formata da un end-point minimale su "/" e dalla chiamata allo script di interrogazione dell'API/upload, 'datauploader.py'. Il file credenziali.txt contiene l'URI del db heroku che ho creato. Il file requirements.txt contiene tutte le dipendenze necessarie al progetto. 


```python

# app.py
from datauploader import DataUploader
from flask import Flask
import requests
import json

app = Flask(__name__)

@app.route('/')
def main():
    desc = "<h1>DB client v 0.1</h1>"
    return desc


@app.route('/updatedb')
def updatedb():
    db = DataUploader(db_url="<uri del vostro db>",resource_url="<url della vostra app Heroku>/liberoquotidiano")
    return db.upload_data()
    
if __name__ == '__main__':
    # Threaded option to enable multiple instances for multiple user access support
    app.run(threaded=True, port=5000)

```
🚀 **Pro-tip**: per evitare di incasinarvi il python di sistema, create sempre un virtual-enviroment nella vostra cartella di progetto con questi comandi (unix), ricordando di aggiungere al file .gitignore il path \<my-env\>:

```terminal
python3 -m venv <my-env>
source <my-env>/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
```

Lo script che esegue l'interrogazione e il caricamento dati è il file datauploader.py. Grazie a python e un pizzico di commenti il suo funzionamento è praticamente auto-esplicativo. L'unica nota che sottolineo è l'utilizzo di due livelli di astrazione per l'upload. Infatti invece di utilizzare una query SQL scritta manualmente, utilizzo un [ORM](https://www.fullstackpython.com/object-relational-mappers-orms.html)(Object-relational Mapper), sqlalchemy. Inoltre, invece di caricare strutture dati 'vanilla' utilizzo la struttura `DataFrame` di pandas. Inoltre, come sottolineato nell'articolo precedente, è molto importante wrappare tutto in un blocco try/except/finally che eviti che certe risposte o processi rimangano accesi in caso di errore. 

```python

# datauploaer.py

from sqlalchemy.types import Integer, Text, String, DateTime
import pandas as pd
import sys
import requests
from sqlalchemy import create_engine

class DataUploader:
    """
    Parametri: 
        db_url: url del database completo di credenziali d'accesso
        resource_url: url della risorsa/API (JSON) da richiedere sul web
    """

    def __init__(self, db_url, resource_url):
        self.db_url = db_url
        self.resource_url = resource_url
        pass

    def request_data(self):
        """
        Parametri:
            resource_url: url della risorsa json
        Output:
            File JSON
        """

        request = requests.get(self.resource_url)

        if request.ok:
            print("Connessione ok (200)")
            try:
                json_data = request.json()
                print(json_data)
                return json_data
            except BaseException as e:
                print("Errore, i dati non sono quelli attesi: " + e)
                print("Dati ricevuti:")
                print(request.content)
        else:
            print("Non c'è connessione")


    def upload_data(self):
        """
        Decrizione:
            Si connette a un db postgree su heroku e carica una tabella
        Output:
            Messaggio di transazione
        """
        try:
            engine = create_engine(self.db_url)  # fai una query SQL attraverso Pandas
            conn = engine.connect()

            json_data = self.request_data()
            new_data_df = pd.DataFrame(data=json_data['data'])
            
            new_data_df.to_sql("liberoQuotidiano",
                            con=engine,
                            if_exists="append",
                            dtype={
                                "id": Integer,
                                "text": Text,
                                "href": Text,
                                "source": String,
                                "date": DateTime
                            }
                            )
            # TEST
            sql_DF = pd.read_sql_table("liberoQuotidiano", con=engine)
            print(sql_DF)

            success_message = "Dati inseriti correttamente"
            return success_message

        except BaseException as e:
            error_message = "Errore di connessione al DB:"
            print(e)
            return error_message

        finally:
            conn.close()
            engine.dispose()

```

And...*that's it*, attivate l'app con il comando `flask run` e dirigendovi all'endopoint "/updatedb", dopo circa 30 sec dovreste ricevere una notifica di corretto inserimento dei dati nel vostro db online. Se volete automatizzare ulteriormente l'update ci sono due opzioni:

1. Caricare l'app su Heroku e configurare un add-on gratuito basato su cron. 
2. Mantenere l'app attiva in locale e automatizare la chiamata tramite un altro scrit python/shell o integrando in flask una chiamata particolare, come indicato [qui](https://stackoverflow.com/questions/21214270/how-to-schedule-a-function-to-run-every-hour-on-flask).

L'opzione 1, sebbene formalmente più corretta in un'ottica di architettura web, di fatto non funziona con questo particoalre progetto, in quanto il piano 'free' di Heroku non permette timeout maggiori di 30 sec sul webserver utilizzato da flask (gunicorn). Di fatto quindi, dato che lo scraping di libero si è rivelato un task abbastanza lungo, l'applicazione andrà in timeout error il più delle volte. 

## Nelle prossime puntate

E questo è tutto. Come sempre tutto il codice si trova disponibile su [Github](https://github.com/raphael2692/libero-datauploader). Nel prossimo tutorial mostrerò un esempio di applicazione che consumi i dati raccolti per farci qualcosa di...carino. 