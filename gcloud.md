# Guida Google Cloud Platform


* [Creazione di un progetto](#creazione-di-un-progetto)
* [Creazione di un'app nel progetto](#creazione-di-unapp-nel-progetto)
* [Creazione del database Firestore](#creazione-del-database-firestore)
* [Creare RESTful API](#creare-restful-api)
* [Fare il deploy dell'app](#fare-il-deploy-dellapp)
* [Creazione applicazione Flask con frontend](#creazione-applicazione-flask-con-frontend)
* [Functions](#functions)
* [PubSub](#pubsub)
# Creazione di un progetto
### 1. **Creazione dell'ambiente virtuale**   
Creare .venv e attivarlo, inolte selezionarlo nell'IDE

```bash
python3 -m venv .venv

source .venv/bin/activate
```
### 2. **Creazione dei requirements**  
Creiamo il file **requirements.txt**, da aggiornare strada facendo...di solito contiene le seguenti librerie:
```
Flask==3.1.2
Flask-RESTful==0.3.10
google-cloud-firestore==2.22.0
WTForms==3.2.1
gunicorn==23.0.0
flask-cors==6.0.2
```
Installiamo i requirements nel .venv attivato
```bash
pip install -r requirements.txt
```

### 3. **Creazione del progetto su gcloud**  
Definiamo la variabile globale PROJECT_ID e BILLING_ACCOUNT_ID

```bash
export PROJECT_ID=
echo $PROJECT_ID
```

```bash
gcloud billing accounts list
export BILLING_ACCOUNT_ID=
echo $BILLING_ACCOUNT_ID
```

Creiamo il progetto e lo settiamo come default
```bash
gcloud projects create ${PROJECT_ID} --set-as-default
```
Per controllare che sia stato creato e settato come default
```bash
gcloud projects list
gcloud config get-value project
```

A questo punto colleghiamo il billign account (senza il quel non e' possibile fare null anel progetto)
```bash
gcloud billing projects link ${PROJECT_ID} --billing-account $BILLING_ACCOUNT_ID
```
Per verificare se il collegamento è stato eseguito con successo, eseguendo il seguente comando dovremmo avere per la voce `billingEnable == True`.
```bash
gcloud billing projects describe ${PROJECT_ID}
```

# Creazione di un'app nel progetto
### 1. Attiviamo tutti i services di cui avremo bisogno in seguito
```bash
gcloud services enable appengine.googleapis.com cloudbuild.googleapis.com storage.googleapis.com firestore.googleapis.com
```

### 2. Creiamo l'app
```bash
gcloud app create --project=${PROJECT_ID}
```
### 3. Creiamo il file .gcloudignore
Per essere sviluppata su gcloud un'app deve avere
* requirements.txt -> creato all'inizio, da aggiornare strada facendo
* .gcloudignore -> lo creiamo ora
* codice da eseguire 
* app.yaml  

Al momento quidni creiamo .gcloudignore, file che specifica file da non caricare su gcloud.
```
.git
.gitignore

# Python pycache:
__pycache__/

# virtual env
.venv/

# Ignored by the build system
/setup.cfg
credentials.json
```

# Creazione del database Firestore
### 1. Creiamo il database sulla cloud console 
Durante questo step ci dobbiamo ricordare che dobbiamo assegnare un nome al database, che deve essere in **modalita' nativa** e che dobbiamo sceglier euna regione in cui eseguirlo.
### 2. Creiamo un utente con credenziali gestite dal servizio iam e gli forniamo privilegio di owner su tutti i servizi, quindi anche sul database
Creiamo l'utente, gli diamo privilegi e salviamo le credenziali in un file locale .json  
Infine creiamo la variabile locale GOOGLE_APPLICATION_CREDENTIALS che indica al programma dove si trova credentials.json
```bash
export NAME=webuser
echo $NAME

gcloud iam service-accounts create ${NAME}

gcloud projects add-iam-policy-binding ${PROJECT_ID} --member "serviceAccount:${NAME}@${PROJECT_ID}.iam.gserviceaccount.com" --role "roles/owner"

touch credentials.json 
gcloud iam service-accounts keys create credentials.json --iam-account ${NAME}@${PROJECT_ID}.iam.gserviceaccount.com 

export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/credentials.json"
```
### 3. Creiamo la prima connessione con il database, per testare
```python
from google.cloud import firestore

db = firestore.Client(database="...")
```

### 4. Creiamo il file .json con file per inizializzar eild atabase
```json
[
	 {"name": "red", "red": 255, "green": 0, "blue": 0},
	 {"name": "green", "red": 0, "green": 255, "blue": 0},
	 {"name": "blue", "red": 0, "green": 0, "blue": 255},
]
```

### 5. Creiamo il file db_firestore.py
Di seguito un template
```python
from google.cloud import firestore
import json

class Classe_firestore(object):
    def __init__(self):
        self.db = firestore.Client(database="...")
        self.populate_db('db.json')
    
    def add_element(self, ..):
        self.db.collection('collection').document('document').set(...)

    def get_element_by_name(self, DOCUMENT_NAME):
        c = self.db.collection('collection').document("document").get()
        rv = c.to_dict() if c.exists else None
        return rv
    
    def delete_element_by_name(self, DOCUMENT_NAME):
        self.db.collection('collection').document("document").delete()

    def populate_db(self, filename):
        with open(filename) as f:
            data_list = json.load(f)
            for data in data_list:
                self.add_element(data)
```

Per selezionare file in base a campo contenuto
```python
docs = db.collection('collection').where('first', '==', 'Sheev').stream()
```

Per scorrere file di una collection
```python
for doc in db.collection('collection').stream():
    ...
```

Per ordinare file di una collection
```python
docs = db.collection('sith').order_by('first', direction = firestore.Query.DESCENDING).limit(2).stream()
```

# Creare RESTful API
Dalla lettura del file **dettagli_api.yaml** fornito, estraiamo i *metodi* da implementare, i *codici* ad esse associati, le *definizioni* delle resource ed i vari *path*.

### 1. Creiamo il file api.py
```python
from flask import Flask, request
from flask_restful import Resource, Api
from flask_cors import CORS

app = Flask(__name__)
CORS(app)
api=Api(app)

# Indicato all'interno di dettagli_api.yaml
basePath = '/api/v1'

class Object1Resource(Resource):
    def get(self, NOME_PARAMETRO):
        return obj, codice(200, 400)
    
    def post(self, NOME_PARAMETRO):
        param=request.json

        return None, codice(200, 400)

    def put(self, NOME_PARAMETRO):
        return None, codice(200, 400)

    def delete(self, NOME_PARAMETRO):
        return None, codice(200, 400)

    api.add_resource(ColorResource, f'{basePath}/path/<string:NOME_PARAMETRO>')

class Object2Resource(Resource):
    def get(self):
        return obj_list, 200

api.add_resource(ColorList, f'{basePath}/path')
```

### 2. Aggiornare l'host dello swagger
Se ho gia' fatto il deploy e voglio aggiornare l'host in un secondo moemtno, cerco l'URL su cui e' esposto il servizio con il seguente comando:
```bash
gcloud app browse --service="myService"
```

In generale il formato dell'URL e' `"<service-name>-dot-,<project-name>.appspot.com"`

# Fare il deploy dell'app
Per essere sviluppata su gcloud un'app deve avere
* requirements.txt -> creato all'inizio, da aggiornare strada facendo
* .gcloudignore -> creato prima, ricontrollare
* codice da eseguire -> appena creato
* app.yaml -> lo creiamo ora

### 1. Creiamo il file app.yaml
```yaml
# Versione di python
runtime: python313

# Limitare lo scaling
instance_class: F1
automatic_scaling:
  max_instances: 1

# Specifica quale app utilizzare per creare il servizio (es. api:app -> seleziona app da api.py)
entrypoint: gunicorn api:app
# Specifica il nome del servizio
service: default

# Definiscono come il server gestisce i metadata delle richieste HTTP
handlers:
- url: /static
  static_dir: static

- url: /.*
  secure: always
  script: auto
```

### 2. Lanciamo il comando per fare il deploy
```bash
gcloud app deploy app.yaml
```


# Creazione applicazione Flask con frontend
L'obiettivo è quello di creare un'interfaccia web per visualizzare i dati all'interno del database.  
Solitament ele web app con frontend hanno la seguente struttura:
* templates/    --> Cartella in cui raggruppiamo tutti i template HTML
* static/       --> Cartella in cui raggruppiamo tutti i file statici
* main.py       --> Usato per gestire tutta l'applicazione
* app.yaml      --> Usato per definire il deployment su gcloud

### 1. Creare main.py

```python
from flask import Flask, render_template, request, redirect
from file_firestore import *
from flask_cors import CORS

app = Flask(__name__)
CORS(app)
object_dao = Classe_firestore()

@app.route('/path', methods=['GET']) 
def nome_della_funzione()
    return render_template("TEMPLATE_HTML", PARAMETRI...)

@app.route('/path/<data>', methods=['GET']) 
def nome_della_funzione(data)
    return redirect("/path/" + cform.name.data, code=302)
```
### 2. Fare il deploy dell'app
Vedi su
## Focus su teplates html
### * Struttura generale del file (con import di bootstrap)
```html
<html>
    <head>
        <meta charset="utf-8">
        <title>Weekly report</title>
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.3.1/dist/css/bootstrap.min.css" integrity="sha384-ggOyR0iXCbMQv3Xipma34MD+dH/1fQ784/j6cY/iJTQUOhcWr7x9JvoRxT2MZw1T" crossorigin="anonymous">

    </head>
    <body>
       
        
    </body>
</html>
```

### * Tabella bootstrap
```html
<table class="table">
    <thead>
        <tr>
        <th scope="col">Asilo</th>
        <th scope="col">Numero pasti</th>
        </tr>
    </thead>

    <tbody>
        {% for record in list_records %}
            <tr>
                <td>{{record.asilo}}</td>
                <td>{{record.bambini}}</td>
            </tr>
        {% endfor %}            
    </tbody>
</table>
```
### zip operator for tables in jinja
```python
return render_template("home.html", zip = zip, monthyear_print = monthyear_print, monthyear_links = monthyear_links)
```

```html
{% for link, print in zip(monthyear_links,monthyear_print ) %}
	<tr>
		<td> <a href = "/{{link}}">{{print}}</a> </td>
	</tr>
{% endfor %}  
```
### * If in jinga
```html
{% if record.giorno == today %} <tr class="table-primary" > {% else %} <tr> {% endif %}
```

### * Color box
```html
<html>
    <body>
        <div class="color-box" style="background-color: rgb({{r}}, {{g}}, {{b}})">&nbsp</div>
    </body>
</html>
```

### * WTForm
Crea un WTForm con le label e i valori associati che sono specificati nella sua definizione (vedi dopo).
```html
<html>
    <body>
        <form method="POST">
            Create new color:
            <div>{{form.name.label}}: {{form.name}}</div>
            <div>{{form.red.label}}: {{form.red}}</div>
            <div>{{form.green.label}}: {{form.green}}</div>
            <div>{{form.blue.label}}: {{form.blue}}</div>
            <button>{{form.submit}}</button>
        </form>
    </body>
</html>
```

## Focus su WTForm nel main.py
Se dobbiamo utilizzare dei WTForm aggiungiamo, selezionando le tipologie di field di cui abbiamo bisogno tramite i seguenti imports:
```python
from wtforms import Form, StringField, IntegerField, SubmitField, validators
```
Per utilizzare i WTForm dichiariamo una classe che eredita da Form in cui spefichiamo i nomi e la tipologia di field e validators per i dati che saranno inseriti al loro interno  
La strucgcloud projects delete
gcloud projects delete
t serve quando vogliamo creare un oggetto Form a partire da un dizionario.
```python
class Classeform(Form):
    name =  StringField('Name', [validators.Length(min=1, max=255)])
    red =   IntegerField('Red', [validators.NumberRange(min=0, max=255)])
    submit= SubmitField('Submit')

    class Struct:
    def __init__(self, **entries):
        self.__dict__.update(entries)
```

```python
@app.route('/path', methods=['GET', 'POST'])
def nome_della_funzione():
    if request.method == 'POST':
        cform = Classeform(request.form)
        object_dao.add_color(cform.name.data, cform.red.data)
        ...
            
    element = object_dao.get_element_by_name(PARAM)
    if request.method == 'GET':
        cform=Classeform(obj=Struct(**element))
        cform.name.data = PARAM
        ...
```
# Staccare il billing di un progetto o eliminarlo
Per staccare il billing di un progetto
```bash
gcloud billing projects unlink ${PROJECT_ID}
```

Per eliminare un progetto
```bash
gcloud projects delete
```
# Functions 
Ci sono due tipi di funcitons
In ogni caso bisogna creare una nuova cartella e creare
- main.py
- requirement.txt
- credential.json

In particolar enei requirements di solito server
```
Flask==3.1.2
google-cloud-firestore==2.22.0
```
### 1. Functions che rispodnono a trigger http
```python
from flask import Flask, request
from google.cloud import firestore

def get_trend_v2(request):
    path = request.path.split('/')[-1]
    ...
    return f"Ritorna stringa"
```

```bash
gcloud functions deploy get_trend_Loree --runtime python310 --trigger-http --allow-unauthenticated --entry-point get_trend_v2 --no-gen2
```
Dopo functions specifico il nome dell'URL
Dopo --entry-point specifico il nome della funzione nel file .py

Esempio di URL: `https://us-central1-my-function-test-terra.cloudfunctions.net/get_trend_Loree/secondo`

### 2. Functions che rispodnono a trigger ad evento

```python
from flask import Flask, render_template, request, redirect
from google.cloud import firestore
from datetime import datetime, timedelta

...

def update_db(data, context): 
    db = firestore.Client(database="mensa")

    document_name = context.resource.split("/")[-1]

    new_value = data['value'] if len(data["value"]) != 0 else None
    old_value = data['oldValue'] if len(data["oldValue"]) !=0 else None
    if new_value and not old_value: # document added
        pass
    elif not new_value and old_value: # document removed
        pass
    else: # document updated
        pass
    

```

**In particolare il parametro data contiene:**
```json
{
  "oldValue": {}, 

	"value": {
	    "createTime": "2025-01-10T10:00:00.000Z",
	    "fields": {
	      "bambini": {
	        "integerValue": "25" 
	      },
	      "nome_asilo": {
	        "stringValue": "AsiloRossi"
	      }
	    },
	    "name": "projects/mensa-esame/databases/mensa/documents/prenotazioni/AsiloRossi_2025-10-10",
	    "updateTime": "2025-01-10T10:00:00.000Z"
  },

  "updateMask": {
    "fieldPaths": [ "bambini" ]
  }
}
```
Se old_value e' vuoto e value no --> creazioen documento
Se nessuno dei due e' vuoto --> aggiornamento
Se old_value e' pieno e value no --> cancellazione

**In particolare il parametro context contiene:**
- context.resource, ovvero quale risorsa ha scatenato l'evento.
  È una stringa che rappresenta il Full Path del documento su Firestore.
  Esempio: "projects/mensa-esame/databases/mensa/documents/prenotazioni/AsiloRossi_2025-10-10"
- context.event_id, ovvero id dell'evento. SI usa per permettere concetto di idempotenza ma noi non ci soffermiamo.
- context.event_type, indica il tipo di evento (wirte, delete, creation)
- context.timestamp, indica la data in cui e' stato generato l;evento ovvero quando e' scattato il trigger (non quando e' partita la funzioen)
  
Per sviluppare la funzioen si esegue:

```bash
gcloud functions deploy update_db \
--runtime python310 \
--trigger-event "providers/cloud.firestore/eventTypes/document.write" \
--trigger-resource "projects/mensa-esame/databases/mensa/documents/prenotazioni/{documentId}" \
--docker-registry=artifact-registry \
--no-gen2
```

# PubSub 
