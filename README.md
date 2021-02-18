# Blog personale

Generato con Hugo nella cartella /docs. 

Le categorie principali dovrebbero essere:

1. Generale (aggiornamenti/riflessioni su vita/lavoro)
2. Progetti (presentazione/aggiornamenti su progetti/applicazioni)
3. Tutorial


## Deploy

```shell

# Hugo, ad es su Debian
apt install hugo
git clone https://github.com/raphael2692/myblog.git
cd myblog 
# recuperare il tema
git clone https://github.com/vaga/hugo-theme-m10c.git themes/m10c

# per sviluppare:
hugo server

# per deploy:
hugo
git add . 
git commit -m "bla bla bla"
git push

```


