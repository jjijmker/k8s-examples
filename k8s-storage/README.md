# Share Pod

1. Log in to Minikube host:

```sh
minikube ssh
```

2. Then create a directory `pod-volume`:

```sh
mkdir pod-volume
```

3. Note the full path of the folder (`/home/docker/pod-volume`):

```sh
cd pod-volume
pwd
```

4. Now create the pod:

```sh
kubectl apply -f share-pod.yaml
```

5. Verify that the pod is running:

```sh
kubectl get pods -l app=share-pod
```

6. Now expose the pod using a `NodePort`:

```sh
kubectl expose pod share-pod --type=NodePort --port=80
```

7. Verify that the service and endpoints have been created:

```sh
kubectl get services,endpoints -l app=share-pod
```

8. Now open the `NodePort` in your browser:

```sh
minikube service share-pod
```

9. Verify that the browser window has the text "Introduction to Kubernetes", placed into the index file by the debian pod, and served up by the nginx pod.