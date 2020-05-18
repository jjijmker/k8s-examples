# README

Instructions:
https://hackmd.io/@ryanjbaxter/spring-on-k8s-workshop

Run with:
```sh
mvnw clean package
```

Uploads to:
https://us-east-2.console.aws.amazon.com/ecr/repositories?region=us-east-2#

```shell script
docker tag k8s-demo-app:latest 417177149200.dkr.ecr.us-east-2.amazonaws.com/k8s-demo-app:latest
docker push 417177149200.dkr.ecr.us-east-2.amazonaws.com/k8s-demo-app:latest
```

Create K8s deployment & service manifests:
```shell script
kubectl create deployment k8s-demo-app --image 417177149200.dkr.ecr.us-east-2.amazonaws.com/k8s-demo-app -o yaml --dry-run > k8s/deployment.yaml
kubectl create service clusterip k8s-demo-app --tcp 80:8080 -o yaml --dry-run > k8s/service.yaml
```