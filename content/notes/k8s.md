+++
title = 'Kubernetes Cheat Sheet'
+++
- Create a Job which prints output
```sh
$ kubectl create job hello --image=busybox:1.28 -- echo "Hello World"
```
- Create a CronJob that prints `Hello World` every minute
```sh
kubectl create cronjob hello --image=busybox:1.28   --schedule="*/1 * * * *" -- echo "Hello World"
```
- Get all running pods in the namespace
```sh
kubectl get pods --field-selector=status.phase=Running
```
- Check which nodes are ready
```sh
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"
```
- List Events sorted by timestamp
```sh
kubectl get events --sort-by=.metadata.creationTimestamp
```
- Update Resource
```sh
kubectl set image deployment/frontend www=image:v2               # Rolling update "www" containers of "frontend" deployment, updating the image
kubectl rollout history deployment/frontend                      # Check the history of deployments including the revision
kubectl rollout undo deployment/frontend                         # Rollback to the previous deployment
kubectl rollout undo deployment/frontend --to-revision=2         # Rollback to a specific revision
kubectl rollout status -w deployment/frontend                    # Watch rolling update status of "frontend" deployment until completion
kubectl rollout restart deployment/frontend                      # Rolling restart of the "frontend" deployment
```
- Partially update a node
```sh
kubectl patch node k8s-node-1 -p '{"spec":{"unschedulable":true}}'
```
- Update a container's image; spec.containers[*].name is required because it's a merge key
```sh
kubectl patch pod valid-pod -p '{"spec":{"containers":[{"name":"kubernetes-serve-hostname","image":"new image"}]}}'
```
- Update a deployment's replica count by patching its scale subresource
```sh
kubectl patch deployment nginx-deployment --subresource='scale' --type='merge' -p '{"spec":{"replicas":2}}'
```
- Interacting with running Pods
```sh
kubectl run -i --tty busybox --image=busybox:1.28 -- sh             # Run pod as interactive shell
kubectl run nginx --image=nginx -n mynamespace                      # Start a single instance of nginx pod in the namespace of mynamespace
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml # Generate spec for running pod nginx and write it into a file called pod.yaml
kubectl attach my-pod -i                                            # Attach to Running Container
kubectl port-forward my-pod 5000:6000                               # Listen on port 5000 on the local machine and forward to port 6000 on my-pod
```
- Interacting with Nodes and cluster
```sh
kubectl top pod POD_NAME --containers               # Show metrics for a given pod and its containers
kubectl top pod POD_NAME --sort-by=cpu              # Show metrics for a given pod and sort it by 'cpu' or 'memory'
kubectl cp my-namespace/my-pod:/tmp/foo /tmp/bar       # Copy /tmp/foo from a remote pod to /tmp/bar locally
kubectl cluster-info dump --output-directory=/path/to/cluster-state   # Dump current cluster state to /path/to/cluster-state
```
- Resource 
```sh
kubectl api-resources
```