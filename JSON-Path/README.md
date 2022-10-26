# JSON-Path

This is an free course on [kodekloud.com](https://kodekloud.com/)

Kodekloud Login with Google Credentials

## References
- [KodeKloud: JSON Path Test - Free Course](https://kodekloud.com/topic/introduction-to-yaml-3/)

## Notes
### Dictionary (unordered)
```json
"car": {
  "color": "blue",
  "price": "20,000"
}
```

### List or Array (ordered)
```json
[
  "car", 
  "bus",
  "truck",
  "bike"
]
```
- Query elements from list (addressable via index due to ordered structure):
```
$[0] # first element ["car"]
$[0,3] # first and fourth element
```

### List of Dictionaries
```json
[
  "car": {
    "color": "blue",
    "price": "20,000",
    "wheels": [
      {
        "model": "A",
        "location": "front-right"
      },
      {
        "model": "B",
        "location": "front-left"
      },
      {
        "model": "C",
        "location": "rear-right"
      },
      {
        "model": "D",
        "location": "rear-left"
      }
    ]
  }, 
  "bus": {
      "color": "white",
      "price": "120,000",
    "wheels": [
      {
        "model": "A",
        "location": "front-right"
      },
      {
        "model": "B",
        "location": "front-left"
      },
      {
        "model": "C",
        "location": "rear-right"
      },
      {
        "model": "D",
        "location": "rear-left"
      }
    ]
  }
]
```
- Query element from list of dictionaries
```
# model of second wheel of car
$[0].wheels[1].model # "B"
```

### Root element (car and bus are items of root dictionary)
```json
{
  "vehicles": {
    "car": {
      "color": "blue",
      "price": "20,000"
    }, 
    "bus": {
      "color": "white",
      "price": "120,000"
    }
  }
}
```
- Query car details via `$.vehicles.car` provided in an array:
```json
[
  {
    "color": "blue",
    "price": "20,000"
  }
]
```

### Criteria
Create query based on criteria to filter an array
```
[
  12,
  13,
  15,
  19,
  20,
  16,
  17,
  18,
  14,
  11
]
```
- Query for list elements greater than 15
```
$[ ?( @ > 15 )] # other oerators: ==, !=, in [...], nin [...]
```
- Query car model of rear-right wheel of upper example
```
$[0].wheel[?(@.location == "rear-right")].model
```

### Wild Cards
