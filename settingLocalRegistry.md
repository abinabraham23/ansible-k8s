# Setting up the local registry

1. Start the cluster and allow insecure registries 
```
minikube start --insecure-registry "10.0.0.0/24"
```

2. Tell minikube to start a registry inside a pod in the Kubernetes cluster 
```
minikube addons enable registry
```

3. Get the name of the registry pod, in my case it is registry-56mrm (the official docs didn't explain this)
```
kubectl get pods --namespace kube-system
```

4. Forward all traffic to the registry, run the below two commands in separate terminals (replace the registry pod name)
```
kubectl port-forward --namespace kube-system registry-56mrm 5000:5000

docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:$(minikube ip):5000"
or 
docker run --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:host.docker.internal:5000"
```

Note: With above docker command, we can (ab)use docker’s network configuration to instantiate a container on the docker’s host, and run socat there to redirect traffic going to the docker vm’s port 5000 to port 5000 on our host workstation.

6. To check the list of respositories in registry visit. http://localhost:5000/v2/_catalog


## Testing the Setup

1. Build the docker image
```
docker build -t eap72 .
```

2. Tag the newly built image  and prepare for push to our newly created registry 
```
docker tag eap72:latest localhost:5000/eap72:latest
```

3. Push the tagged image to local registry 
```
docker push localhost:5000/eap72:latest
```

4. Reference it in Kubernetes deployment manifest, for example:
```
containers:
  - name: "{{ app_name }}"
    image: localhost:5000/eap72
```

### In case of disk space issue, run below command with minikube
minikube ssh -- docker system prune -f
minikube ssh -- docker volume prune -f