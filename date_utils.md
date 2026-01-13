## Ordinare una lista di date
Per ordinare una lista di date scritte come stringhe:
```python
from datetime import datetime, timedelta

sorted_ret_docs = sorted(ret_docs, key=lambda x: datetime.strptime(x, '%m-%d-%Y'))
```

## Creare data di N giorni prima o dopo
```python
from datetime import datetime, timedelta

earlier_date = datetime.strptime(data, '%d-%m-%Y') - timedelta(days=7)
```

## Creare data da stringa
```python
from datetime import datetime
def time_from_str(t):
    """converte ’HH:MM’ in oggetto time"""
    try:
        return datetime.strptime(t, '%H:%M').time()
    except:
        return None

```

## Creare stringa da data
```python
def date_from_str(d):
    # converte ’gg-mm-YYYY’ in oggetto datetime
    try:
        return datetime.strptime(d, '%d-%m-%Y')
    except:
        return None
```

## Creare cambiare il formato data, da dd-mm-YYYY -> YYYY-mm-dd
```python
def from_ddmmYYYY_to_YYYYmmdd(data_string):
    try: 
        input_date =  datetime.strptime(data_string, '%d-%m-%Y')
        return datetime.strptime(input_date, '%Y-%m-%d')
    except: 
        return data_string


def from_YYYmmdd_to_ddmmYYYY(data_string):
    try: 
        input_date =  datetime.strptime(data_string, '%Y-%m-%d')
        return datetime.strptime(input_date, '%d-%m-%Y')
    except: 
        return data_string
```

## Ordinare date nel database firestore
```python
from google.cloud.firestore_v1.field_path import FieldPath
from google.cloud import firestore

db.collection('letture').where(firestore.FieldPath.document_id(), "<", target_date_id).order_by(firestore.FieldPath.document_id(), direction=firestore.Query.DESCENDING).limit(1)
```
