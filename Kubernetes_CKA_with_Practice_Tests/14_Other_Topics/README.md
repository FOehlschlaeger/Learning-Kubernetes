# Other Topics

## JSON Path
See also https://github.com/FOehlschlaeger/Learning-Kubernetes/tree/main/JSON-Path

### Why JSON Path?
- large data sets with many nodes and Kubernetes ressources

### Examples
- Output of `kubectl` command in JSON format
```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx-pod",
    "namespace": "default"
  },
  "spec": {
    "containers": [
      {
        "image": "nginx:alpine",
        "name": "nginx"
      },
      {
        "image": "redis:alpine",
        "name": "redis-container"
      }
    ],
    "nodeName": "node01"
  },
  "status": {
    "conditions": [
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2019-06-13T05:34:09Z",
        "status": "True",
        "type": "Initialized"
      },
      {
        "lastProbeTime": null,
        "lastTransitionTime": "2019-06-13T05:34:09Z",
        "status": "True",
        "type": "PodScheduled"
      }
    ],
    "containerStatuses": [
      {
        "image": "nginx:alpine",
        "name": "nginx",
        "ready": false,
        "restartCount": 4,
        "state": {
          "waiting": {
            "reason": "ContainerCreating"
          }
        }
      },
      {
        "image": "redis:alpine",
        "name": "redis-container",
        "ready": false,
        "restartCount": 2,
        "state": {
          "waiting": {
            "reason": "ContainerCreating"
          }
        }
      }
    ],
    "hostIP": "172.17.0.75",
    "phase": "Pending",
    "qosClass": "BestEffort",
    "startTime": "2019-06-13T05:34:09Z"
  }
}
```
- Query to get the reason for the state of the container under the status section
```
$.status.containerStatuses[0].state.waiting.reason
```
- Query to get the restart count of redis-container under the status section
```
$.status.containerStatuses[?(@.name == "redis-container")].restartCount
```
```json
[
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "web-pod-1",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "image": "nginx:alpine",
          "name": "nginx"
        }
      ],
      "nodeName": "node01"
    }
  },
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "web-pod-2",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "image": "nginx:alpine",
          "name": "nginx"
        }
      ],
      "nodeName": "node02"
    }
  },
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "web-pod-3",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "image": "nginx:alpine",
          "name": "nginx"
        }
      ],
      "nodeName": "node01"
    }
  },
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "web-pod-4",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "image": "nginx:alpine",
          "name": "nginx"
        }
      ],
      "nodeName": "node01"
    }
  },
  {
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "db-pod-1",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "image": "mysql",
          "name": "mysql"
        }
      ],
      "nodeName": "node01"
    }
  }
]
```
- Query to get all pod names
```
$[?(@.kind == "Pod")].metadata.name
```


## Advanced Kubectl Commands
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

### Examples
- Use JSON Path query to retrieve the `osImages` of all the nodes and store it in the file `/opt/outputs/nodes_os.txt`
```
kubectl get nodes -o=jsonpath={'.items[*].status.nodeInfo.osImage'} > /opt/outputs/nodes_os.txt
```
- Get the usernames from a specific kube-config file at `root/my-kube-config` and store them in the file `/opt/outputs/users.txt`
```
kubectl config view --kubeconfig=/root/my-kube-config -o=jsonpath='{.users[*].name}' > /opt/outputs/users.txt
```
- sort existing persistent volumes based on their capacity and store results in the file `/opt/outputs/storage-capacity-sorted.txt`
```
kubectl get pv --sort-by=.spec.capacity.storage > /opt/outputs/storage-capacity-sorted.txt
```
- Retrieve just the first two colums as NAME and CAPACITY, while remaining the sorting by capacity of persistent volumes
```
kubectl get pv -o=custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage --sort-by=.spec.capacity.storage > /opt/outputs/pv-and-capacity-sorted.txt
```
- Use JSON Path to identify the context configured for the `aws-user` in the `/root/my-kube-config` file and store the result in `/opt/outputs/aws-context-name`
```
kubectl config --kubeconfig=/root/my-kube-config view -o=jsonpath='{.contexts[?(@.context.user == "aws-user")].name}' > /opt/outputs/aws-context-name
```
