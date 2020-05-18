---
title: Spring on Kubernetes Workshop
tags: Templates, Talk
description: This is a workshop that will show you how to run Spring apps on Kubernetes.
---

# Spring on Kubernetes!


Workshop Materials: https://hackmd.io/@ryanjbaxter/spring-on-k8s-workshop
Ryan Baxter, Spring Cloud Engineer, VMware
Dave Syer, Spring Engineer, VMware

---

## What You Will Do

* Create a basic Spring Boot app
* Build a Docker image for the app
* Push the app to a Docker repo
* Create deployment and service descriptors for Kubernetes
* Deploy and run the app on Kubernetes
* External configuration and service discovery
* Deploy the Spring PetClinic App with MySQL

---

### Prerequisites

* Basic knowledge of Spring and Kubernetes (we will not be giving an introduction to either)
* [JDK 8 or higher](https://openjdk.java.net/install/index.html)
    * **Please ensure you have a JDK installed and not just a JRE**
* [Docker](https://docs.docker.com/install/) installed
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed
* [Kustomize](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/INSTALL.md) installed
* [Skaffold](https://skaffold.dev/docs/install/) installed
* An IDE (IntelliJ, Eclipse, VSCode)
    * **Optional** [Cloud Code](https://cloud.google.com/code/) for IntelliJ and VSCode is nice. For VSCode the [Microsoft Kubernetes Extension](https://github.com/Azure/vscode-kubernetes-tools) is almost the same.


---

### Configure kubectl

* Depending on how you are doing this workshop will determine how you configure `kubectl`

#### Doing The Workshop On Your Own

* If you are doing this workshop on your own you will need to have your own Kubernetes cluster and Docker repo that the cluster can access
    * **Docker Desktop and Docker Hub** - Docker Desktop allows you to easily setup a local Kubernetes cluster ([Mac](https://docs.docker.com/docker-for-mac/#kubernetes), [Windows](https://docs.docker.com/docker-for-windows/#kubernetes)).  This in combination with [Docker Hub](https://hub.docker.com/) should allow you to easily run through this workshop.
    * **Hosted Kubernetes Clusters and Repos** - Various cloud providers such as Google and Amazon offer options for running Kubernetes clusters and repos in the cloud.  You will need to follow instructions from the cloud provider to provision the cluster and repo as well configuring `kubectl` to work with these clusters.


#### Doing The Workshop In Person

* Login To [https://gangway.workshop.demo.ryanjbaxter.com/](https://gangway.workshop.demo.ryanjbaxter.com/)
    * Use provided username and password
* Configuring `kubectl`
    * If you are on a unix based OS, you can copy the commands from the browser window and execute them in your terminal window to configure `kubectl`
    * If you are on Windows and using PowerShell copy and pasting the commands does not work well.  You can copy the commands into a text file and name is `configkubectl.sh` and run `./configkubectl.sh` in PowerShell to configure everything
* Run the below command to verify kubectl is configured correctly

```bash
$kubectl cluster-info
Kubernetes master is running at https://k8s.workshop.demo.ryanjbaxter.com:8443
CoreDNS is running at https://k8s.workshop.demo.ryanjbaxter.com:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use &#39;kubectl cluster-info dump&#39;.
```

---

## Create A Spring Boot App 

* Click [here](https://start.spring.io/starter.zip?type=maven-project&amp;language=java&amp;bootVersion=2.3.0.M4&amp;packaging=jar&amp;jvmVersion=1.8&amp;groupId=com.example&amp;artifactId=k8s-demo-app&amp;name=k8s-demo-app&amp;description=Demo%20project%20for%20Spring%20Boot%20and%20Kubernetes&amp;packageName=com.example.k8s-demo-app&amp;dependencies=web,actuator) to download a zip of the Spring Boot app
* Unzip the project to your desired workspace and open in your favorie IDE

---

## Add A RestController

Modify `K8sDemoApplication.java` and add a `@RestController`

:::warning
Be sure to add the `@RestController` annotation and not just the `@GetMapping`
:::

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class K8sDemoAppApplication {

  public static void main(String[] args) {
    SpringApplication.run(K8sDemoAppApplication.class, args);
  }

  @GetMapping(&#34;/&#34;)
  public String hello() {
    return &#34;Hello World&#34;;
  }
}
```

---

## Run The App

In a terminal window run

```bash
$ ./mvnw clean package &amp;&amp; java -jar ./target/*.jar
```

The app will start on port `8080`

---

## Test The App

Make an HTTP request to http://localhost:8080

```bash
$ curl http://localhost:8080; echo
Hello World
```

---

## Test Spring Boot Actuator

Spring Boot Actuator adds several other endpoints to our app as well

```bash
$ curl localhost:8080/actuator; echo
{
  &#34;_links&#34;: {
    &#34;self&#34;: {
      &#34;href&#34;: &#34;http://localhost:8080/actuator&#34;,
      &#34;templated&#34;: false
    },
    &#34;health&#34;: {
      &#34;href&#34;: &#34;http://localhost:8080/actuator/health&#34;,
      &#34;templated&#34;: false
    },
    &#34;info&#34;: {
      &#34;href&#34;: &#34;http://localhost:8080/actuator/info&#34;,
      &#34;templated&#34;: false
    }
}
```

:::warning
Be sure to stop the Java process before continuing on or else you might get port binding issues since Java is using port `8080`
:::

---

## Containerize The App

The first step in running the app on Kubernetes is producing a conainer for the app we can then deploy to Kubernetes

![](https://i.imgur.com/nM8G7ag.png)

&lt;!-- ---

### Creating A Dockerfile

* Used to describe how to build a container for your app
* Since Spring Boot just requires Java all you need is a JDK and your jar
* Create a file in the root of your app called `Dockerfile` with the below content

```Dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY target/k8s-demo-app-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT [&#34;java&#34;,&#34;-jar&#34;,&#34;/app.jar&#34;]
```

:exclamation: The above is *not* intended for general purpose production use. It is the quickest, dirtiest way to get a Spring Boot app into a container. See this [guide](https://spring.io/guides/topicals/spring-boot-docker) for more detail.

---

### Choosing A JDK

* JDK 8u192 introduced support for containers
* The JDK will be able to see the memory and CPUs available to the container as opposed to what the host has available

---

### Building A Container

* To build a container for our app we need to run `docker build`

```bash
$ docker build -t my-dockerfile-k8s-demo ./
```

* Running `docker images` will allow you to see the built container

```bash
$ docker images
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
my-dockerfile-k8s-demo                latest              ab449be57b9d        5 minutes ago       124MB
```

--- 
--&gt;
---

### Building A Container

* Spring Boot 2.3.x can build a container for you without the need for any additional plugins or files
* To do this use the Spring Boot Build plugin goal `build-image`

```bash
$ ./mvnw spring-boot:build-image
```

* Running `docker images` will allow you to see the built container

```bash
$ docker images
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
k8s-demo-app                          0.0.1-SNAPSHOT      ab449be57b9d        5 minutes ago       124MB
```

---

### Run The Container

```bash
$ docker run --name k8s-demo-app -p 8080:8080 k8s-demo-app:0.0.1-SNAPSHOT
```

---

### Test The App Responds

```bash
$ curl http://localhost:8080; echo
Hello World
```

:::warning
Be sure to stop the docker container before continuing.  You can stop the container and remove it by running `$ docker rm -f k8s-demo-app
`
:::

---

### Jib It!

* [Jib](https://github.com/GoogleContainerTools/jib) is a tool from by Google to make building Docker containers easier
* Runs as a Maven or Gradle plugin
* Does not require a Docker deamon
* Implements many Docker best practices automatically
![](https://i.imgur.com/fkzRqe2.png)
![](https://i.imgur.com/b4uolYy.png)

---

### Add Jib To Your POM

* Open the POM file of your app and add the following `plugin` to the `build` section

```xml
...
&lt;build&gt;
   &lt;plugins&gt;
      ...
      &lt;plugin&gt;
          &lt;groupId&gt;com.google.cloud.tools&lt;/groupId&gt;
          &lt;artifactId&gt;jib-maven-plugin&lt;/artifactId&gt;
          &lt;version&gt;1.8.0&lt;/version&gt;
          &lt;configuration&gt;
             &lt;from&gt;
                &lt;image&gt;openjdk:8u222-slim&lt;/image&gt;
             &lt;/from&gt;
             &lt;to&gt;
                &lt;image&gt;k8s-demo-app&lt;/image&gt;
             &lt;/to&gt;
             &lt;container&gt;
                &lt;user&gt;nobody:nogroup&lt;/user&gt;
             &lt;/container&gt;
          &lt;/configuration&gt;
       &lt;/plugin&gt;
       ...
   &lt;/plugins&gt;
&lt;/build&gt;
...
```

---

### Using Jib To Build A Container

* Now building a container is as simple as building our app

```bash
$ ./mvnw clean compile jib:dockerBuild
```

* NOTE: This still uses your local Docker dameon to build the container

---

### Putting The Container In A Registry

* Up until this point the container only lives on your machine
* It is useful to instead place the container in a registry
    * Allows others to use the container
* [Docker Hub](https://hub.docker.com/) is a popular public registry
* Private registries exist as well

---

### Using Jib To Push To A Registry

* By default [Jib will try and push the image to Docker Hub](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin#using-docker-hub-registry)
* We will push our container to a registry associated with our Kubernetes cluster
* Login to [https://harbor.workshop.demo.ryanjbaxter.com/](https://harbor.workshop.demo.ryanjbaxter.com/) with your supplied username and password
* Once logged in you will see two projects, one called `library` and the other with the same name as your user, we will use the one with the same name as your user

---

### Authenticate With The Registry

* Before Jib can push the container image to the registry we must authenticate

```bash
$ docker login harbor.workshop.demo.ryanjbaxter.com
```

* Enter your username and password when prompted

---

### Modify Jib Plugin Configuration

* Back in your POM file, modify the Jib plugin, specifically the image name
    * We also add an execution goal in order to do the container build and push whenever we run our Maven build
* **Be sure to replace `USERNAME` in the image name with your supplied user name**

```xml
&lt;plugin&gt;
    &lt;groupId&gt;com.google.cloud.tools&lt;/groupId&gt;
    &lt;artifactId&gt;jib-maven-plugin&lt;/artifactId&gt;
    &lt;version&gt;1.8.0&lt;/version&gt;
    &lt;configuration&gt;
        &lt;from&gt;
            &lt;image&gt;openjdk:8u222-slim&lt;/image&gt;
        &lt;/from&gt;
        &lt;to&gt;
            &lt;image&gt;harbor.workshop.demo.ryanjbaxter.com/[USERNAME]/k8s-demo-app&lt;/image&gt;
        &lt;/to&gt;
        &lt;container&gt;
            &lt;user&gt;nobody:nogroup&lt;/user&gt;
        &lt;/container&gt;
    &lt;/configuration&gt;
    &lt;executions&gt;
        &lt;execution&gt;
            &lt;phase&gt;package&lt;/phase&gt;
            &lt;goals&gt;
                &lt;goal&gt;build&lt;/goal&gt;
            &lt;/goals&gt;
        &lt;/execution&gt;
    &lt;/executions&gt;
&lt;/plugin&gt;
```

---

### Run The Build And Deploy The Container

* Now you should be able to run the Maven build which will deploy the container to your container registry

```bash
$ ./mvnw clean package
```

* Go back to Harbor and you should now see the image in your registry
![](https://i.imgur.com/Xj5JPVs.png)

---

## Deploying To Kubernetes

* With our container build and deployed to a registry we can now run this container on Kubernetes

---

### Deployment Descriptor

* Kubernetes uses YAML files to provide a way of describing how the app will be deployed to the platform
* You can write these by hand using the Kubernetes documentation as a reference
* Or you can have Kubernetes generate it for you using kubectl
* The `--dry-run` flag allows us to generate the YAML without actually deploying anything to Kubernetes
:::warning
Be sure to replace [USERNAME] with your username in the below `kubectl` command
:::


```bash
$ mkdir k8s
$ kubectl create deployment k8s-demo-app --image harbor.workshop.demo.ryanjbaxter.com/[USERNAME]/k8s-demo-app -o yaml --dry-run &gt; k8s/deployment.yaml
```

* The resulting `deployment.yaml` should look similar to this

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: k8s-demo-app
  name: k8s-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-demo-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: k8s-demo-app
    spec:
      containers:
      - image: harbor.workshop.demo.ryanjbaxter.com/user1/k8s-demo-app
        name: k8s-demo-app
        resources: {}
status: {}
```

---

### Service Descriptor

* A service acts as a load balancer for the pod created by the deployment descriptor
* If we want to be able to scale pods than we want to create a service for those pods

```bash
$ kubectl create service clusterip k8s-demo-app --tcp 80:8080 -o yaml --dry-run &gt; k8s/service.yaml
```

* The resulting `service.yaml` should look similar to this
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: k8s-demo-app
  name: k8s-demo-app
spec:
  ports:
  - name: 80-8080
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: k8s-demo-app
  type: ClusterIP
status:
  loadBalancer: {}
```

---

### Apply The Deployment and Service YAML

* The deployment and service descriptors have been created in the `/k8s` directory
* Apply these to get everything running
* If you have `watch` installed you can watch as the pods and services get created

```bash
$ watch -n 1 kubectl get all
```

* In a separate terminal window run

```bash
$ kubectl apply -f ./k8s
```

```bash
Every 1.0s: kubectl get all                                 Ryans-MacBook-Pro.local: Wed Jan 29 17:23:28 2020

NAME                               READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-d6dd4c4d4-7t8q5   1/1     Running   0          68m

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.243   &lt;none&gt;        80/TCP    68m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app   1/1     1            1           68m

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-d6dd4c4d4   1         1         1       68m
```

:::info
`watch` is a useful command line tool that you can install on [Linux](https://www.2daygeek.com/linux-watch-command-to-monitor-a-command/) and [OSX](https://osxdaily.com/2010/08/22/install-watch-command-on-os-x/).  All it does is continuously executes the command you pass it.  You can just run the `kubectl` command specified after the `watch` command but the output will be static as opposed to updating constantly.
:::

---

### Testing The App

* The service is assigned a cluster IP, which is only accessible from inside the cluster

```
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.243   &lt;none&gt;        80/TCP    68m
```

* To access the app we can use `kubectl port-forward`

```bash
$ kubectl port-forward service/k8s-demo-app 8080:80
```

* Now we can `curl` localhost:8080 and it will be forwarded to the service in the cluster

```bash
$ curl http://localhost:8080; echo
Hello World
```

:::success
**Congrats you have deployed your first app to Kubernetes 🎉**

&lt;iframe src=&#34;https://giphy.com/embed/msKNSs8rmJ5m&#34; width=&#34;480&#34; height=&#34;357&#34; frameBorder=&#34;0&#34; class=&#34;giphy-embed&#34; allowFullScreen&gt;&lt;/iframe&gt;&lt;p&gt;&lt;a href=&#34;https://giphy.com/gifs/day-subreddit-msKNSs8rmJ5m&#34;&gt;via GIPHY&lt;/a&gt;&lt;/p&gt;
:::

:::warning
Be sure to stop the `kubectl port-forward` process before moving on
:::

--- 

### Exposing The Service

&gt; NOTE: `LoadBalancer` features are platform specific. The visibility of your app after changing the service type might depend a lot on where it is deployed (e.g. per cloud provider).

* If we want to expose the service publically we can change the service type to `LoadBalancer`
* Open `k8s/service.yaml` and change `ClusterIp` to `LoadBalancer`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: k8s-demo-app
  name: k8s-demo-app
spec:
...
  type: LoadBalancer
...
```

* Now apply the updated `service.yaml`

```bash
$ kubectl apply -f ./k8s
```
---

### Testing The Public LoadBalancer

* Kubernetes will assign the service an external ip 
    * It may take a minute or so for Kubernetes to assign the service an external IP, until it is assigned you might see `&lt;pending&gt;` in the `EXTERNAL-IP` column

```bash
$ kubectl get service k8s-demo-app -w
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
k8s-demo-app   LoadBalancer   10.100.200.243   35.202.115.69   80:31428/TCP   85m
$ curl http://35.202.115.69; echo
Hello World
```

:::info
The `-w` option of `kubectl` lets you watch a single Kubernetes resource.
:::

---

## Best Practices

### Liveness and Readiness Probes

* Kubernetes uses two probes to determine if the app is ready to accept traffic and whether the app is alive
* If the readiness probe does not return a `200` no trafic will be routed to it
* If the liveness probe does not return a `200` kubernetes will restart the Pod
* Spring Boot has a build in set of endpoints from the [Actuator](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot) module that fit nicely into these use cases
    * The `/health/readiness` endpoint indicates if the application is healthy, this fits with the readiness proble
    * The `/health/liveness` endpoint serves application info, we can use this to make sure the application is &#34;alive&#34;

:::info
The `/health/readiness` and `/health/liveness` endpoints are only available in Spring Boot 2.3.x.
:::

---

### Add The Readiness Probe

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: k8s-demo-app
spec:
...
  template:
    ...
    spec:
      containers:
        ...
        readinessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/readiness
```

---

### Add The Liveness Probe

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: k8s-demo-app
spec:
...
  template:
    ...
    spec:
      containers:
        ...
        livenessProbe:
          httpGet:
            port: 8080
            path: /actuator/health/liveness
```

---

### Graceful Shutdown

* Due to the asyncronous way Kubnernetes shuts down applications there is a period of time when requests can be sent to the application while an application is being terminated.  
* To deal with this we can configure a pre-stop sleep to allow enough time for requests to stop being routed to the application before it is terminated.  
* Add a `preStop` command to the `podspec` of your `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
  name: k8s-demo-app
spec:
...
  template:
    ...
    spec:
      containers:
        ...
        lifecycle:
          preStop:
            exec:
              command: [&#34;sh&#34;, &#34;-c&#34;, &#34;sleep 10&#34;]
```

---
#### Handling In Flight Requests

* Our application could also be handling requests when it receives the notification that it need to shut down.  
* In order for us to finish processing those requests before the applicaiton shuts down we can configure a &#34;grace period&#34; in our Spring Boot applicaiton.  
* Open `application.properties` in `/src/main/resources` and add

```properties
server.shutdown.grace-period=30s
```

:::info
`server.shutdown.grace-period` is only available begining in Spring Boot 2.3.x
:::

---

### Update The Container &amp; Apply The Updated Deployment YAML

```bash
$ ./mvnw clean package
$ kubectl apply -f ./k8s
```

* An updated Pod will be created and started and the old one will be terminated
* If you use `watch -n 1 kubectl get all` to see all the Kubernetes resources you will be able to see this appen in real time

---

&lt;!-- ### Resource Constraints

* If your application is sharing a cluster with other applications and services it is usually a good idea to set up resource constraints.
* JVM processes generally show a spike of memory and CPU usage on startup, so you need to take that into account (or expect very slow startup).
* Kubernetes lets you set constraints as &#34;requests&#34; (the amount you start with) and &#34;limits&#34; (the maximum you are allowed to consume).

### Memory

In the `PodSpec` each container can have its own resources:

```yaml
spec:
  containers:
  - name: app
    image: ...
    resources:
      limits:
        memory: &#34;1024Mi&#34;
      requests:
        memory: &#34;1024Mi&#34;
```

N.B. The JVM hates to have `request&lt;limit`, so if you are setting either it&#39;s best to set both.

CNB images (from pack etc) calculate `JAVA_OPTS` at _runtime_, so you can change the manifest and not the image. Manually created images (from Dockefile etc) generally rely on JVM defaults, or you could manually modify with `JAVA_OPTS` if you build the image carefully.


### CPU

```yaml
spec:
  containers:
  - name: app
    image: ...
    resources:
      ...
      requests:
        cpu: &#34;1000m&#34; # 2000m or 4000m is better
```

It is best to set a `request` (if you set anything) and let the limit float. That way the app can use more CPU if it needs it and it is available.

Add the `resources` configuration to your deployment manifest and re-apply to the cluster. You will see the app restart, and probably notice it going a little slower because of the CPU request. --&gt;

### Cleaning Up

* Before we move on to the next section lets clean up everything we deployed

```bash
$ kubectl delete -f ./k8s
```

---

## Skaffold

* [Skaffold](https://github.com/GoogleContainerTools/skaffold) is a command line tool that facilitates continuous development for Kubernetes applications
* Simplifies the development process by combining multiple steps into one easy command
* Provides the building blocks for a CI/CD process
* Make sure you have Skaffold installed before continuing

```bash
$ skaffold version
v1.3.1
```

### Adding Skaffold YAML

* Skaffold is configured using...you guessed it...another YAML file
* We can easily initialize a YAML file using the `skaffold` command line

```bash
$ skaffold init --XXenableJibInit
apiVersion: skaffold/v2alpha1
kind: Config
metadata:
  name: k-s-demo-app--
build:
  artifacts:
  - image: harbor.workshop.demo.ryanjbaxter.com/rbaxter/k8s-demo-app
deploy:
  kubectl:
    manifests:
    - k8s/deployment.yaml
    - k8s/service.yaml

Do you want to write this configuration to skaffold.yaml? [y/n]: y
```

* This will create `skaffold.yaml` in the root of our project

---

### Development With Skaffold

* Skaffold makes some enhacements to our development workflow when using Kubernetes
* Skaffold will
    * Build the app (Maven)
    * Create the container (Jib)
    * Push the container to the registry (Jib)
    * Apply the deployment and service YAMLs
    * Stream the logs from the Pod to your terminal
    * Automatically setup port forwarding
```bash
$ skaffold dev --port-forward
```

---

### Testing Everything Out

* If you are `watch`ing your Kubernetes resources you will see the same resources created as before
* When running `skaffold dev --port-forward` you will see a line in your console that looks like

```
Port forwarding service/k8s-demo-app in namespace rbaxter, remote port 80 -&gt; address 127.0.0.1 port 4503
```

* In this case port `4503` will be forwarded to port `80` of the service

```bash
$ curl localhost:4503; echo
Hello World
```
---

### Make Changes To The Controller

* Skaffold is watching the project files for changes
* Open `K8sDemoApplication.java` and change the `hello` method to return `Hola World`
* Once you save the file you will notice Skaffold rebuild and redeploy everthing with the new change

```bash
$ curl localhost:4503; echo
Hola World
```

---

### Cleaning Everything Up

* Once we are done, if we kill the `skaffold` process, skaffold will remove the deployment and service resources
```bash
WARN[2086] exit status 1
Cleaning up...
 - deployment.apps &#34;k8s-demo-app&#34; deleted
 - service &#34;k8s-demo-app&#34; deleted
```

---

### Debugging With Skaffold

* Skaffold also makes it easy to attach a debugger to the container running in Kubernetes

```bash
$ skaffold debug --port-forward
...
Port forwarding service/k8s-demo-app in namespace rbaxter, remote port 80 -&gt; address 127.0.0.1 port 4503
Watching for changes...
Port forwarding pod/k8s-demo-app-75d4f4b664-2jqvx in namespace rbaxter, remote port 5005 -&gt; address 127.0.0.1 port 5005
...
```

* The `debug` command results in two ports being forwarded
    * The http port, `4503` in the above example
    * The remote debug port `5005` in the above example
* You can then setup the remote debug configuration in your IDE to attach to the process and set breakpoints just like you would if the app was running locally
* If you set a breakpoint where we return `Hola World` from the `hello` method in `K8sDemoApplication.java` and then issue our `curl` command to hit the endpoint you should be able to step through the code

![](https://i.imgur.com/4KK549f.png)

:::warning
Be sure to detach the debugger and kill the `skaffold` process before continuing
:::

---

## Kustomize

* [Kustomize](https://kustomize.io/) another tool we can use in our Kubernetes toolbox that allows us to customize deployments to different environments
* We can start with a base set of resources and then apply customizations on top of those
* Features
    * Allows easier deployments to different environments/providers
    * Allows you to keep all the common properties in one place
    * Generate configuration for specific environments
    * No templates, no placeholder spaghetti, no environment variable overload

---

### Getting Started With Kustomize

* Create a new directory in the root of our project called `kustomize/base`
* Move the `deployment.yaml` and `service.yaml` from the `k8s` directory into `kustomize/base`
* Delete the `k8s` directory

```bash
$ mkdir -p kustomize/base
$ mv k8s/* kustomize/base
$ rm -Rf k8s
```

---

### kustomize.yaml

* In `kustomize/base` create a new file called `kustomization.yaml` and add the following to it
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:  
- service.yaml
- deployment.yaml
```

&gt; NOTE: Optionally, you can now remove all the labels and annotations in the metadata of both objects and specs inside objects. Kustomize adds default values that link a service to a deployment. If there is only one of each in your manifest then it will pick something sensible.

---

### Customizing Our Deployment

* Lets imagine we want to deploy our app to a QA environment, but in this environment we want to have two instances of our app running
* Create a new directory called `qa` under the `kustomize` directory
* Create a new file in `kustomize/qa` called `update-replicas.yaml`, this is where we will provide customizations for our QA environment
* Add the following content to `kustomize/qa/update-replicas.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-app
spec:
  replicas: 2
```

* Create a new file called `kustomization.yaml` in `kustomize/qa` and add the following to it

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../base

patchesStrategicMerge:
- update-replicas.yaml
```

* Here we tell Kustomize that we want to patch the resources from the `base` directory with the `update-replicas.yaml` file
* Notice that in `update-replicas.yaml` we are just updating the properties we care about, in this case the `replicas`

---

### Running Kustomize

* You will need to have [Kustomize installed](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/INSTALL.md)

```bash
$ kustomize build ./kustomize/base
```

* This is our base deployment and service resources

```bash
$ kustomize build ./kustomize/qa
...
spec:
  replicas: 2
...
```

* Notice when we build the QA customization that the replicas property is updated to `2`

---

### Piping Kustomize Into Kubectl

* We can pipe the output from `kustomize` into `kubectl` in order to use the generated YAML to deploy the app to Kubernetes

```bash
$ kustomize build kustomize/qa | kubectl apply -f -
```

* If you are watching the pods in your Kubernetes namespace you will now see two pods created instead of one

```bash
Every 1.0s: kubectl get all                                 Ryans-MacBook-Pro.local: Mon Feb  3 12:00:04 2020

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-647b8d5b7b-r2999   1/1     Running   0          83s
pod/k8s-demo-app-647b8d5b7b-x4t54   1/1     Running   0          83s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-app   ClusterIP   10.100.200.200   &lt;none&gt;        80/TCP    84s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app   2/2     2            2           84s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-647b8d5b7b   2         2         2       84s
```

:::success
Our service `k8s-demo-app` will load balance requests between these two pods
:::

---

### Clean Up

* Before continuing clean up your Kubernetes environment

```bash
$ kustomize build kustomize/qa | kubectl delete -f -
```
---

### Using Kustomize With Skaffold

* Currently our Skaffold configuration uses `kubectl` to deploy our artifacts, but we can change that to use `kustomize`
* Change your `skaffold.yaml` to the following
:::warning
Make sure you replace `[USERNAME]` in the image property with your own username
:::
```yaml
apiVersion: skaffold/v2alpha3
kind: Config
metadata:
  name: k-s-demo-app
build:
  artifacts:
  - image: harbor.workshop.demo.ryanjbaxter.com/[USERNAME]/k8s-demo-app
    jib: {}
deploy:
  kustomize:
    paths: [&#34;kustomize/base&#34;]
profiles:
  - name: qa
    deploy:
      kustomize:
        paths: [&#34;kustomize/qa&#34;]
```

---

* Notice now the `deploy` property has been changed from `kubectl` to now use Kustomize
* Also notice that we have a new profiles section allowing us to deploy our QA configuration using Skaffold

---

### Testing Skaffold + Kustomize

* If you run your normal `skaffold` commands it will use the deployment configuration from `kustomize/base`

```bash
$ skaffold dev --port-forward
```

* If you want to test out the QA deployment run the following command to activate the QA profile

```bash
$ skaffold dev -p qa --port-forward
```

:::warning
Be sure to kill the `skaffold` process before continuing
:::
---

## Externalized Configuration

* One of the [12 factors for cloud native apps](https://12factor.net/config) is to externalize configuration
* Kubernetes provides support for externalizing configuration via config maps and secrets
* We can create a config map or secret easily using `kubectl`

```bash
$ kubectl create configmap log-level --from-literal=LOGGING_LEVEL_ORG_SPRINGFRAMEWORK=DEBUG
$ kubectl get configmap log-level -o yaml
apiVersion: v1
data:
  LOGGING_LEVEL_ORG_SPRINGFRAMEWORK: DEBUG
kind: ConfigMap
metadata:
  creationTimestamp: &#34;2020-02-04T15:51:03Z&#34;
  name: log-level
  namespace: rbaxter
  resourceVersion: &#34;2145271&#34;
  selfLink: /api/v1/namespaces/rbaxter/configmaps/log-level
  uid: 742f3d2a-ccd6-4de1-b2ba-d1514b223868
```

---

### Using Config Maps In Our Apps

* There are a number of ways to consume the data from config maps in our apps
* Perhaps the easiest is to use the data as environment variables
* To do this we need to change our `deployment.yaml` in `kustomize/base`
```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - image: harbor.workshop.demo.ryanjbaxter.com/rbaxter/k8s-demo-app
        name: k8s-demo-app
        envFrom:
          - configMapRef:
              name: log-level
        ...
```

* Add the `envFrom` properties above which reference our config map `log-level`
* Update the deployment by running `skaffold dev` (so we can stream the logs)
* If everything worked correctly you should see much more verbose logging in your console

---

### Removing The Config Map and Reverting The Deployment

* Before continuing lets remove our config map and revert the changes we made to `deployment.yaml`
* To delete the config map run the following command
```bash
$ kubectl delete configmap log-level
```
* In `kustomize/base/deployment.yaml` remove the `envFrom` properties we added
* Next we will use Kustomize to make generating config maps easier

---

### Config Maps and Spring Boot Application Configuration

* In Spring Boot we usually place our configuration values in application properties or YAML
* Config Maps in Kubernetes can be populated with values from files, like properties or YAML files
* We can do this via `kubectl`
```bash 
$ kubectl create configmap k8s-demo-app-config --from-file ./path/to/application.yaml
```
:::warning
No need to execute the above command, it is just an example, the following sections will show a better way
:::
* We can then mount this config map as a volume in our container at a directory Spring Boot knows about and Spring Boot will automatically recognize the file and use it

---

### Creating A Config Map With Kustomize

* Kustomize offers a way of generating config maps and secrets as part of our customizations
* Create a file called `application.yaml` in `kustomize/base` and add the following content

```yaml
logging:
  level:
    org:
      springframework: INFO
```

* We can now tell Kustomize to generate a config map from this file, in `kustomize/base/kustomization.yaml` by adding the following snippet to the end of the file

```yaml
configMapGenerator:
  - name: k8s-demo-app-config
    files:
      - application.yaml
```

* If you now run `$ kustomize build` you will see a config map resource is produced

```bash
$ kustomize build kustomize/base
apiVersion: v1
data:
  application.yaml: |-
    logging:
      level:
        org:
          springframework: INFO
kind: ConfigMap
metadata:
  name: k8s-demo-app-config-fcc4c2fmcd
```

:::info
By default `kustomize` generates a random name suffix for the `ConfigMap`.  Kustomize will take care of reconciling this when the `ConfigMap` is referenced in other places (ie in volumes). It does this to force a change to the `Deployment` and in turn force the app to be restarted by Kubernetes. This isn&#39;t always what you want, for instance if the `ConfigMap` and the `Deployment` are not in the same `Kustomization`. If you want to omit the random suffix, you can set `behavior=merge` (or `replace`) in the `configMapGenerator`.

:::

* Now edit `deployment.yaml` in `kustomize/base` to have kubernetes create a volume for that config map and mount that volume in the container

```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
   ...
  template:
    ...
    spec:
      containers:
      - image: harbor.workshop.demo.ryanjbaxter.com/rbaxter/k8s-demo-app
        name: k8s-demo-app
        volumeMounts:
          - name: config-volume
            mountPath: /config
        ...
      volumes:
        - name: config-volume
          configMap:
            name: k8s-demo-app-config
```

* In the above `deployment.yaml` we are creating a volume named `config-volume` from the config map named `k8s-demo-app-config`
* In the container we are mounting the volume named `config-volume` within the container at the path `/config`
* Spring Boot automatically looks in `./config` for application configuration and if present will use it (because the app is running in /)

---

### Testing Our New Deployment

* If you run `$ skaffold dev --port-forward` everything should deploy as normal
* Check that the config map was generated
```bash
$ kubectl get configmap
NAME                             DATA   AGE
k8s-demo-app-config-fcc4c2fmcd   1      18s
```

* Skaffold is watching our files for changes, go ahead and change `logging.level.org.springframework` from `INFO` to `DEBUG` and Skaffold will automatically create a new config map and restart the pod
* You should see a lot more logging in your terminal once the new pod starts

:::warning
Be sure to kill the `skaffold` process before continuing
:::

---

## Service Discovery

* Kubernetes makes it easy to make requests to other services
* Each service has a DNS entry in the container of the other services allowing you to make requests to that service using the service name
* For example, if there is a service called `k8s-workshop-name-service` deployed we could make a request from the `k8s-demo-app` just by making an HTTP request to `http://k8s-workshop-name-service`

---

### Deploying Another App

* In order to save time we will use an [existing app](https://github.com/ryanjbaxter/k8s-spring-workshop/tree/master/name-service) that returns a random name 
* The container for this service resides [in Docker Hub](https://hub.docker.com/repository/docker/ryanjbaxter/k8s-workshop-name-service) (a public container registery)
* To make things easier we placed a [Kustomization file](https://github.com/ryanjbaxter/k8s-spring-workshop/blob/master/name-service/kustomize/base/kustomization.yaml) in the GitHub repo that we can reference from our own Kustomization file to deploy the app to our cluster

### Modify `kustomization.yaml`

* In your k8s-demo-app&#39;s `kustomize/base/kustomization.yaml` add a new resource pointing to the new app&#39;s `kustomize` directory
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
  - deployment.yaml
  - https://github.com/ryanjbaxter/k8s-spring-workshop/name-service/kustomize/base

configMapGenerator:
  - name: k8s-demo-app-config
    files:
      - application.yaml
```

### Making A Request To The Service

* Modify the `hello` method of `K8sDemoApplication.java` to make a request to the new service
```java=
@SpringBootApplication
@RestController
public class K8sDemoAppApplication {

  private RestTemplate rest = new RestTemplateBuilder().build();

  public static void main(String[] args) {
    SpringApplication.run(K8sDemoAppApplication.class, args);
  }

  @GetMapping(&#34;/&#34;)
  public String hello() {
    String name = rest.getForObject(&#34;http://k8s-workshop-name-service&#34;, String.class);
    return &#34;Hola &#34; + name;
  }
}
```

* Notice the hostname of the request we are making matches the service name in [our `service.yaml` file](https://github.com/ryanjbaxter/k8s-spring-workshop/blob/master/name-service/kustomize/base/service.yaml#L7)


---

### Testing The App

* Deploy the apps using Skaffold
```bash
$ skaffold dev --port-forward
```

* This should deploy both the k8s-demo-app and the name-service app
```bash
$ kubectl get all
NAME                                             READY   STATUS    RESTARTS   AGE
pod/k8s-demo-app-5b957cf66d-w7r9d                1/1     Running   0          172m
pod/k8s-workshop-name-service-79475f456d-4lqgl   1/1     Running   0          173m

NAME                                TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)        AGE
service/k8s-demo-app                LoadBalancer   10.100.200.102   35.238.231.79   80:30068/TCP   173m
service/k8s-workshop-name-service   ClusterIP      10.100.200.16    &lt;none&gt;          80/TCP         173m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-app                1/1     1            1           173m
deployment.apps/k8s-workshop-name-service   1/1     1            1           173m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-app-5b957cf66d                1         1         1       172m
replicaset.apps/k8s-demo-app-fd497cdfd                 0         0         0       173m
replicaset.apps/k8s-workshop-name-service-79475f456d   1         1         1       173m
```

* Because we deployed two services and supplied the `-port-forward` flag Skaffold will forward two ports

```bash
Port forwarding service/k8s-demo-app in namespace user1, remote port 80 -&gt; address 127.0.0.1 port 4503
Port forwarding service/k8s-workshop-name-service in namespace user1, remote port 80 -&gt; address 127.0.0.1 port 4504
```

* Test the name service
```bash
$ curl localhost:4504; echo
John
```
:::info
Hitting the service multiple times will return a different name
:::

* Test the k8s-demo-app which should now make a request to the name-service
```bash
$ curl localhost:4503; echo
Hola John
```
:::info
Making multiple requests should result in different names coming from the name-service
:::

* Stop the Skaffold process to clean everything up before moving to the next step

---

## Running The PetClinic App

* The [PetClinic app](https://github.com/spring-projects/spring-petclinic) is a popular demo app which leverages several Spring technologies
    * Spring Data (using MySQL)
    * Spring Caching
    * Spring WebMVC

![](https://i.imgur.com/MCgAL4n.png)

---

### Deploying PetClinic

* We have a Kustomization that we can use to easily get it up and running

```bash
$ kustomize build https://github.com/dsyer/docker-services/layers/samples/petclinic | kubectl apply -f -
$ kubectl port-forward service/petclinic-app 8080:80
```

:::warning
The above `kustomize build` command may take some time to complete
:::

* Head to `http://localhost:8080` and you should see the &#34;Welcome&#34; page
* To use the app you can go to &#34;Find Owners&#34;, add yourself, and add your pets
* All this data will be stored in the MYSQL database

---

### Dissecting PetClinic

* Here&#39;s the `kustomization.yaml` that you just deployed:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
- ../../mysql
namePrefix: petclinic-
transformers:
- ../../mysql/transformer
- ../../actuator
- ../../prometheus
images:
  - name: dsyer/template
    newName: dsyer/petclinic
configMapGenerator:
  - name: env-config
    behavior: merge
    literals:
      - SPRING_CONFIG_LOCATION=classpath:/,file:///config/bindings/mysql/meta/
      - MANAGEMENT_ENDPOINTS_WEB_BASEPATH=/actuator
      - DATABASE=mysql
```

* The relative paths `../../*` are all relative to this file. Clone the repository to look at those: `git clone https://github.com/dsyer/docker-services` and look at `layers/samples`.
* Important features:
    * `base`: a generic `Deployment` and `Service` with a `Pod` listening on port 8080
    * `mysql`: a local MySQL `Deployment` and `Service`. Needs a `PersistentVolume` so only works on Kubernetes clusters that have a default volume provider
    * `transformers`: patches to the basic application deployment. The patches are generic and could be shared by multiple different applications.
    * `env-config`: the base layer uses this `ConfigMap` to expose environment variables for the application container. These entries are used to adapt the PetClinic to the Kubernetes environment.