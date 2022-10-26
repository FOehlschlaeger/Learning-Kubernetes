# Other Topics

## JSON Path
See also https://github.com/FOehlschlaeger/Learning-Kubernetes/tree/main/JSON-Path

### Why JSON Path?
- large data sets with many nodes and Kubernetes ressources

### JSON Path in Kubectl
- `kubectl` is Kubernetes CLI and sends requests to kube-apiserver 
- requests coming from kube-apiserver sent in JSON format
- for more details use option `-o wide`, but this will not list all available information from json format
- get specific information using JSON Path which no build-in `kubectl` commands can show

#### Steps to get specific formatted information
1. Identify the `kubectl` command containing the requested information (i.e. `kubectl get nodes`, `kubectl get pods` etc.)
2. Familiarize with command output in JSON format: `kubectl get pods -o json`
3. Form the JSON Path query to retrieve required information. **Important: "$" at beginning is added by Kubernetes**
4. Use JSON Path query with `kubectl` command: `kubectl get pods -0=jsonpath='{ .items[0].spec.containers[0].image }'`

#### Loops - Range
```
kubectl get nodes -o=jsonpath='{.items[0].metadata.name}{"\n"}{.items[*].status.capacity.cpu}'
```
More readable output using loops with `range` for Node Names and CPU capacity
```
kubectl get nodes -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\n"}{end}'
```

#### Custom columns
More table-like output structure, each column goes through the items
```
kubectl get nodes -o=custom-columns=NODE:metadata.name,CPU:.status.capacity.cpu
```

#### Sorting with JSON Path
```
kubectl get nodes --sort-by=.metadata.name
```
```
kubectl get nodes --sort-by=.status.capacity.cpu
```

## Advanced Kubectl Commands

