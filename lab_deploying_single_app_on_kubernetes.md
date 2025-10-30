# LAB | Deploy a Simple Java App on Kubernetes

In this lab, you will learn how to **containerize** a simple Java application and run it on a **Kubernetes** cluster. By the end of this exercise, you’ll have a basic Java “Hello World” (or similar) application running as a **Deployment** in Kubernetes, accessible as a **Service**.

## Prerequisites

1. **Kubernetes Cluster**: You have access to a running Kubernetes cluster (local *Minikube*, Docker Desktop’s Kubernetes, or a remote cluster).
2. **kubectl**: Installed and configured to point to your cluster.
3. **Docker**: Installed locally for building container images.
4. **Java JDK** (optional if you want to compile the sample yourself).
5. **Text Editor or IDE**: For creating/editing files.

## Step 1: Create a Simple Java Application

For demonstration, we’ll create a very minimal Java web application that returns “Hello, Kubernetes!” via HTTP.

1. **Create a directory** for your app:

   ```bash
   mkdir java-k8s-lab
   cd java-k8s-lab
   ```

2. **Initialize a simple Java app** (e.g., using Maven or just a single .java file). Here’s a straightforward example using a basic HTTP server:

   **HelloController.java** (using a lightweight HTTP server)

   ```java
   import com.sun.net.httpserver.HttpServer;
   import com.sun.net.httpserver.HttpHandler;
   import com.sun.net.httpserver.HttpExchange;
   import java.io.IOException;
   import java.io.OutputStream;
   import java.net.InetSocketAddress;

   public class HelloController {
       public static void main(String[] args) throws IOException {
           HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
           server.createContext("/", new MyHandler());
           server.setExecutor(null);
           server.start();
           System.out.println("Server started on port 8080...");
       }

       static class MyHandler implements HttpHandler {
           @Override
           public void handle(HttpExchange t) throws IOException {
               String response = "Hello, Kubernetes!";
               t.sendResponseHeaders(200, response.length());
               try (OutputStream os = t.getResponseBody()) {
                   os.write(response.getBytes());
               }
           }
       }
   }
   ```

> **Note**: You can alternatively use Spring Boot or any other Java framework. The key idea is to have an application that listens on port **8080**.

## Step 2: Containerize the Java Application

### Create a Dockerfile

In your `java-k8s-lab` directory, create a file named **Dockerfile**:

```Dockerfile
# --------- Build Stage ---------
FROM adoptopenjdk:11 as build
WORKDIR /app

# Copy the Java source file into the container
COPY HelloController.java .

# Compile the Java source file
RUN javac HelloController.java

# Package all compiled classes (including inner classes) into an executable jar
RUN jar cfe app.jar HelloController *.class

# --------- Run Stage ---------
FROM adoptopenjdk:11-jre-hotspot
WORKDIR /app

# Copy the executable jar from the build stage
COPY --from=build /app/app.jar .

# Expose the port on which the app listens
EXPOSE 8080

# Run the jar file
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build and Tag the Docker Image

1. **Build** the Docker image locally:

   ```bash
   docker build -t my-java-app:1.0 .
   ```

   Here, **my-java-app:1.0** is the image name and tag.

2. **Verify** the image was created:

   ```bash
   docker images
   ```

   You should see **my-java-app** listed.

### Run and Test the Image Locally

Run the container locally to confirm it works:

```bash
docker run -d -p 8080:8080 --name test-java-app my-java-app:1.0
```

- **`-p 8080:8080`**: Maps local port 8080 to the container’s 8080.
- **`--name test-java-app`**: Container name for reference.

Open a browser or run:

```bash
curl http://localhost:8080
```

You should see:

```
Hello, Kubernetes!
```

If it looks good, **stop** and **remove** the container:

```bash
docker rm -f test-java-app
```

### Push the Image to a Registry

To deploy to Kubernetes, you typically need the image in a registry accessible to your cluster.

1. **Log in** to Docker Hub:

   ```bash
   docker login
   ```

2. **Tag** your image:

   ```bash
   docker tag my-java-app:1.0 <your-dockerhub-username>/my-java-app:1.0
   ```

3. **Push** the image:

   ```bash
   docker push <your-dockerhub-username>/my-java-app:1.0
   ```

## Step 3: Create a Kubernetes Deployment Manifest

Create a file named **deployment.yaml**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
      - name: java-app-container
        image: <your-dockerhub-username>/my-java-app:1.0
        ports:
        - containerPort: 8080
```

**Key points:**

- **replicas**: 1, meaning one Pod.
- **selector**: labels must match `app: java-app`.
- **image**: Use the correct Docker Hub image reference.
- **containerPort**: 8080.

## Step 4: Deploy to Kubernetes

### Apply the Deployment Manifest

```bash
kubectl apply -f deployment.yaml
```

### Verify the Deployment

```bash
kubectl get deployments
```

You should see **java-app-deployment** with `1/1` up-to-date and available.

Check Pods:

```bash
kubectl get pods
```

You should see a pod named something like **java-app-deployment-xxx** in `Running` state.

## Step 5: Expose the Application as a Service

Create a **Service** that maps traffic to the container’s port.

### Create a Service Manifest

Create **service.yaml**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  type: NodePort
  selector:
    app: java-app
  ports:
    - protocol: TCP
      port: 80         # The service port
      targetPort: 8080 # Container's port
      nodePort: 30080  # Node port (optional)
```

**Notes:**

- **type**: `NodePort` for simplicity. Use `LoadBalancer` on cloud.
- **selector**: Must match the pod's label (`app: java-app`).
- **ports**:
  - `port`: 80 inside cluster
  - `targetPort`: 8080 in container
  - `nodePort`: 30080 (optional)

### Apply the Service Manifest

```bash
kubectl apply -f service.yaml
```

### Test the Application

1. **Check** the service:

   ```bash
   kubectl get svc
   ```

   You should see **java-app-service** with **NodePort** type.

2. **Access** the app:

   - If using **Minikube**:

     ```bash
     minikube service java-app-service
     ```

     This opens the app in your browser.

   - Or access via `<NodeIP>:30080`. Use `localhost` or get Node IP:

     ```bash
     kubectl get nodes -o wide
     ```

   **Example:**

   ```bash
   curl http://<NodeIP>:30080
   ```

   Output:

   ```
   Hello, Kubernetes!
   ```

## Cleanup (Optional)

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

Optionally remove local Docker images:

```bash
docker rmi my-java-app:1.0
docker rmi <your-dockerhub-username>/my-java-app:1.0
```

## **Deliverables**

* Submit the following files:

  * `HelloController.java` — your Java application source code.
  * `Dockerfile` — used to build your Java app image.
  * `deployment.yaml` — Kubernetes Deployment manifest.
  * `service.yaml` — Kubernetes Service manifest.

* Provide **screenshots** or terminal output showing:

  * Successful creation of the Docker image (`docker build` and `docker run`).
  * Running Kubernetes Deployment and Service (`kubectl get deployments`, `kubectl get pods`, `kubectl get svc`).
  * Browser or `curl` output displaying:

    ```
    Hello, Kubernetes!
    ```

## **Submission**

Upon completion, add your deliverables to git. Then commit git and push your branch to the remote. Make a pull request and paste the PR link in the submission field in the Student Portal.

## Summary

In this lab, you:

1. **Created** a simple Java web app.
2. **Built** and **containerized** it.
3. **Pushed** it to Docker Hub.
4. **Deployed** it to Kubernetes as a **Deployment**.
5. **Exposed** it via a **Service**.

This covers the core steps to deploy a Java app on Kubernetes. Now you can explore scaling, CI/CD, volumes, and more.

*Happy containerizing and orchestrating in Kubernetes!*