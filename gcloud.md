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

Se non specifico `action` allora la richiesta vien efatta all'URL in cui mi trovo, altrimenti all'URL specificato da action  .
Utile se ho piu' form nella stess apagina.
```html
<html>
    <body>
        <form method="POST" action="/subscription">
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
        cform = Classeform(request.form)    # per inizializzare i campi del form con i valori isneriti nel forntend    
        if cform.validate():
            object_dao.add_color(cform.name.data, cform.red.data)
            ...
            
    element = object_dao.get_element_by_name(PARAM)
    if request.method == 'GET':
        cform=Classeform(obj=Struct(**element))     # per inizializzare i campi del form con valori che scelgo io
        return render_template("home.html", myform=cform)

    return render_template("home.html", myform=Classeform())    # per creare un form vuoto

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
gcloud functions deploy nomeURL --runtime python310 --trigger-http --allow-unauthenticated --entry-point get_trend_v2 --no-gen2
```
Dopo functions specifico il nome dell'URL
Dopo --entry-point specifico il nome della funzione nel file .py

Esempio di URL: `https://us-central1-my-function-test-terra.cloudfunctions.net/nomeURL/secondo`

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
Per gestire un meccnaismo PubSub bsigona
- creare un topic
- il publisher puo' scrivere sul topic direttamente
- il subbscriber si deve iscrivere ad una subscription connessa ad un topic

## Focus su PubSub di tipo pull (subscriber fa pull, no webhook)
### 1. Includere la librerie nei requirements e installarla
Nei requirements.txt
```txt
google-cloud-pubsub==2.34.0
```
### 2. Creare il topic
Per creare il topic:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/credentials.json"
export TOPIC=cpu_temperature
gcloud pubsub topics create ${TOPIC}
```

Per controllare i topics attivi:
```bash
gcloud pubsub topics list
gcloud pubsub topics describe ${TOPIC}
```

### 3. Creare la subscription di tipo pull
```bash
export SUBSCRIPTION_NAME=cpu_sub
gcloud pubsub subscriptions create ${SUBSCRIPTION_NAME} --topic ${TOPIC}
```

Per testare la subscription:
* Pubblicare
    ```bash
    gcloud pubsub topics publish ${TOPIC} --attribute=from="cli" --message="Test Message"
    ```
    ```bash
    Leggere facendo pull
    gcloud pubsub subscriptions pull ${SUBSCRIPTION_NAME}
    ```

### 4. Creare il codice del publisher
```python
import json
import os
from google.cloud import pubsub_v1

update_interval=5.0
# Prendere variabili TOPIC e PROJECT_ID per creare il path del topic
topic_name=os.environ['TOPIC'] if 'TOPIC' in os.environ.keys() else '...'
project_id=os.environ['PROJECT_ID'] if 'PROJECT_ID' in os.environ.keys()  else '...'

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_name)

# Pubblicare sul topic, una volta oppure in modo ciclico
if __name__=='__main__':
    while True:
        now = datetime.now()
        data={'timestamp': now.timestamp(), 'time': str(now), 'temperature': None}
        res = publisher.publish(topic_path, json.dumps(data).encode('utf-8'))
        print(data, res.result())
        
        time.sleep(update_interval)
```

### 5. Creare il codice del subscriber

```python
import os
import json
import datetime
from google.cloud import pubsub_v1
from google.cloud import firestore

# Prendere variabili SUBSCRIPTION_NAME e PROJECT_ID per creare il path della subscription
subscription_name=os.environ['SUBSCRIPTION_NAME'] if 'SUBSCRIPTION_NAME' in os.environ.keys() else '...'
project_id=os.environ['PROJECT_ID'] if 'PROJECT_ID' in os.environ.keys()  else '...'

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_name)

# Funzione per salvare i dati nel db (opzionale), riceve come input un dizionario
db = firestore.Client()
def save_temperature(data):
        dt=datetime.datetime.fromtimestamp(data['timestamp'])
        docname=dt.strftime('%Y%m%d')
        print(docname, data['temperature'])
        db.collection('temperature')
		        .document(docname)
		        .set({str(data['timestamp']): data['temperature']}, merge=True)

# Funzione eseguita su ogni messaggio ricevuto
def callback(message):
    message.ack()
    try:
        save_temperature(json.loads(message.data.decode('utf-8')))
    except:
        pass    

if __name__=='__main__': 
    # Creare la subscription
    pull = subscriber.subscribe(subscription_path, callback=callback)
    try:
        # Mettersi in ascolto
        pull.result(timeout=30)
    except:
        pull.cancel()
```

**Focus su codifica dei dati**
Il publisher codifica i dati nel seguente modo prima di inviarli:
```python
data={'timestamp': now.timestamp(), 'time': str(now), 'temperature': None}
data_encoded = json.dumps(data).encode('utf-8')
res = publisher.publish(topic_path, data_encoded)
```

Il subscriber decodifica i dati nel seguente modo appena li riceve:
```python
def callback(message):
    message.ack()
    decoded_data = json.loads(message.data.decode('utf-8'))
    try:
        save_temperature(decoded_data)
    except:
        pass  
```

## Focus si PubSub di tipo push (subscriber e' esposto e riceve trigger http quando arriva un messaggio sul topic)
### 1. Includere la librerie nei requirements e installarla
Nei requirements.txt
```txt
google-cloud-pubsub==2.34.0
```
### 2. Creare il topic
Per creare il topic:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="$(pwd)/credentials.json"
export TOPIC=cpu_temperature
gcloud pubsub topics create ${TOPIC}
```

Per controllare i topics attivi:
```bash
gcloud pubsub topics list
gcloud pubsub topics describe ${TOPIC}
```

### 3. Creare la subscription di tipo push
```bash
export TOKEN="123token"
export SUBSCRIPTION_NAME2="cpu_alert_sub"

gcloud pubsub subscriptions create ${SUBSCRIPTION_NAME2} --topic ${TOPIC2} --push-endpoint "https://${PROJECT_ID}.appspot.com/pubsub/push?token=${TOKEN}" --ack-deadline 10
```

### 4. Creare il codice del publisher
Uguale a prima:
```python
import json
import os
from google.cloud import pubsub_v1

update_interval=5.0
# Prendere variabili TOPIC e PROJECT_ID per creare il path del topic
topic_name=os.environ['TOPIC'] if 'TOPIC' in os.environ.keys() else '...'
project_id=os.environ['PROJECT_ID'] if 'PROJECT_ID' in os.environ.keys()  else '...'

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_name)

# Pubblicare sul topic, una volta oppure in modo ciclico
if __name__=='__main__':
    while True:
        now = datetime.now()
        data={'timestamp': now.timestamp(), 'time': str(now), 'temperature': None}
        res = publisher.publish(topic_path, json.dumps(data).encode('utf-8'))
        print(data, res.result())
        
        time.sleep(update_interval)
```

### 5. Creare il codice del subscriber
```python
topic_name=os.environ['TOKEN'] if 'TOKEN' in os.environ.keys() else '123token'

@app.route('/pubsub/push', methods=['POST']) 
def pubsub_push(): 
    print('Received pubsub push') 
    if request.args.get('token', '') != app.config['PUBSUB_VERIFICATION_TOKEN']: 
        return 'Invalid request', 404 
    
    envelope = json.loads(request.data.decode('utf-8')) data=json.loads(base64.b64decode(envelope['message']['data'])) 
    ...
    return 'OK', 200

```

E come specificato prima devo esporre questa funzione sul seguente URL di gcloud ` --push-endpoint "https://${PROJECT_ID}.appspot.com/pubsub/push?token=${TOKEN}" `

### Focus su comunicazione asincrona con PubSub (visto in una prova d'esame)
Una persona puo' pubblicare su un topic per chiedere "a che persona devo fare il regalo". 
Vi e' un worker che risponde alla richiesta mandando la persona associata.

Codice della persona/client:
```python
import json
from google.cloud import pubsub_v1

# Topic richieste
project_id = "gcloud-160125"
topic_req = "secretsanta"
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, topic_req)

# Topic risposte, lo usero' per creare da codice la subscription(non devo farlo da terminale)
response_topic = "secretsanta-results"
subscriber = pubsub_v1.SubscriberClient()
resp_topic_path = publisher.topic_path(project_id, response_topic)

# Subscription al topic delle risposte
my_email = "otta@gmail.com"
sub_name = f"sub-res-{my_email.replace('@', '-').replace('.', '-')}"
sub_path = subscriber.subscription_path(project_id, sub_name)
try:
    filter_str = f'attributes.receiver_id = "{my_email}"'
    subscriber.create_subscription(
        request={
            "name": sub_path,
            "topic": resp_topic_path,
            "filter": filter_str,
        }
    )
    print(f"Subscription filtrata creata per {my_email}")
except Exception:
    print("Subscription già esistente o errore di permessi.")

# Funzione da chiamare per ogni messaggio ricevuto nella mia subscription
def callback(message):
    data = json.loads(message.data.decode('utf-8'))
    print(f"\n[SUCCESSO] Devi fare il regalo a: {data['assigned_target']}")
    message.ack()
    streaming_pull_future.cancel()

# 1. publish sul primo topic
publisher.publish(topic_path, json.dumps({"email": my_email}).encode('utf-8'))
print("Richiesta inviata. In attesa del filtro server...")

# 2. mi metto in ascolto sulla subscription del secondo topic
streaming_pull_future = subscriber.subscribe(sub_path, callback=callback)
try:
    streaming_pull_future.result(timeout=20)
except Exception:
    streaming_pull_future.cancel()
```

Codice del worker:
```python
import json
from datetime import datetime
from google.cloud import pubsub_v1

# topic risposte
project_id = "gcloud-160125"
response_topic = "secretsanta-results"
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(project_id, response_topic)

# topic richieste
request_sub = "secretsanta_sub"     # creato da terminale
subscriber = pubsub_v1.SubscriberClient()
sub_path = subscriber.subscription_path(project_id, request_sub)

# Funzione da chiamare su ogni messaggio in arrivo su topic richieste
def process_request(message):
    try:
        data = json.loads(message.data.decode('utf-8'))
        user_email = data.get('email')
        
        ...
        target_gift = "mario@gmail.com" 

        response_payload = {
            "assigned_target": target_gift,
            "timestamp": str(datetime.now())
        }
        
        # Pubblico su topic risposte
        # Il filtro lavorerà su 'receiver_id'
        publisher.publish(
            topic_path, 
            json.dumps(response_payload).encode('utf-8'),
            receiver_id=user_email
        )
        
        print(f"Sorteggio inviato a {user_email}")
        message.ack()
    except Exception as e:
        print(f"Errore: {e}")
        message.nack()

# 1. In ascolto su topic richieste
print("Worker attivo...")
streaming_pull_future = subscriber.subscribe(sub_path, callback=process_request)
with subscriber:
    try:
        streaming_pull_future.result()
    except KeyboardInterrupt:
        streaming_pull_future.cancel()
```