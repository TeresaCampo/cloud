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
# DD-MM-YYYY -> transform to YYYY-MM-DD
def from_UIdata_to_DBdata(data_document):
    if data_document.get("data") and data_document["data"][2]=='-':
        data_components = data_document["data"].split('-')
        data_document["data"] = f"{data_components[-1]}-{data_components[1]}-{data_components[0]}"
    return data_document

# YYYY-MM-DD-> transform to DD-MM-YYYY
def from_DBdata_to_UIdata(data_document):
    if data_document.get("data") and data_document["data"][2]!='-':
        data_components = data_document["data"].split('-')
        data_document["data"] = f"{data_components[-1]}-{data_components[1]}-{data_components[0]}"
    return data_document
```