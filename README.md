 **1) Building a sample web application in Go:**   
  「Gorilla web toolkit」を使用して、golang web appを作成。   
  （http://www.gorillatoolkit.org/pkg/mux　https://github.com/gorilla/mux）  
    
  mkdir go-kubernetes  
  cd go-kubernetes/  
  go mod init github.com/callicoder/go-kubernetes  
  c9 open main.go  

```
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gorilla/mux"
)

func handler(w http.ResponseWriter, r *http.Request) {
	query := r.URL.Query()
	name := query.Get("name")
	if name == "" {
		name = "Guest"
	}
	log.Printf("Received request for %s\n", name)
	w.Write([]byte(fmt.Sprintf("Hello, %s\n", name)))
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func readinessHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func main() {
	// Create Server and Route Handlers
	r := mux.NewRouter()

	r.HandleFunc("/", handler)
	r.HandleFunc("/health", healthHandler)
	r.HandleFunc("/readiness", readinessHandler)

	srv := &http.Server{
		Handler:      r,
		Addr:         ":8080",
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Start Server
	go func() {
		log.Println("Starting Server")
		if err := srv.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	// Graceful Shutdown
	waitForShutdown(srv)
}

func waitForShutdown(srv *http.Server) {
	interruptChan := make(chan os.Signal, 1)
	signal.Notify(interruptChan, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

	// Block until we receive our signal.
	<-interruptChan

	// Create a deadline to wait for.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	srv.Shutdown(ctx)

	log.Println("Shutting down")
	os.Exit(0)
}
```

  go build
  ls
  ./go-kubernetes 

 **2) Dockerizing the Go application:** 
  c9 open Dockerfile

```
# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from the latest golang base image
FROM golang:latest as builder

# Add Maintainer Info
LABEL maintainer="Liwei <uccblw@gmail.com>"

# Set the Current Working Directory inside the container
WORKDIR /app

# Copy go mod and sum files
COPY go.mod go.sum ./

# Download all dependancies. Dependencies will be cached if the go.mod and go.sum files are not changed
RUN go mod download

# Copy the source from the current directory to the Working Directory inside the container
COPY . .

# Build the Go app
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .


######## Start a new stage from scratch #######
FROM alpine:latest  

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy the Pre-built binary file from the previous stage
COPY --from=builder /app/main .

# Expose port 8080 to the outside world
EXPOSE 8080

# Command to run the executable
CMD ["./main"] 
```

  docker build -t go-kubernetes .
  docker tag go-kubernetes callicoder/go-hello-world:1.0.0
  docker images

 **3) Building and pushing the docker image to dockerhub/aws ECR:**   
  aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 443334274279.dkr.ecr.ap-northeast-1.amazonaws.com  
  docker tag 02e86642bd64  443334274279.dkr.ecr.ap-northeast-1.amazonaws.com/go-kubernetes  
  docker push 443334274279.dkr.ecr.ap-northeast-1.amazonaws.com/go-kubernetes  
  

 **4) Deploying the golang webapp on EKS cluster (Creating a Kubernetes Service & Scaling a Kubernetes deployment)**   
$ cat k8s-deployment.yaml  

```

apiVersion: apps/v1
kind: Deployment                 # Type of Kubernetes resource
metadata:
  name: go-hello-world           # Name of the Kubernetes resource
spec:
  replicas: 3                    # Number of pods to run at any given time
  selector:
    matchLabels:
      app: go-hello-world        # This deployment applies to any Pods matching the specified label
  template:                      # This deployment will create a set of pods using the configurations in this template
    metadata:
      labels:                    # The labels that will be applied to all of the pods in this deployment
        app: go-hello-world 
    spec:                        # Spec for the container which will run in the Pod
      containers:
      - name: go-hello-world
        image: 443334274279.dkr.ecr.ap-northeast-1.amazonaws.com/go-kubernetes:latest
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 8080  # Should match the port number that the Go application listens on
        livenessProbe:           # To check the health of the Pod
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 15
          timeoutSeconds: 5
        readinessProbe:          # To check if the Pod is ready to serve traffic or not
          httpGet:
            path: /readiness
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 1    
          
---
apiVersion: v1
kind: Service                    # Type of kubernetes resource
metadata:
  name: go-hello-world-service   # Name of the resource
spec:
  type: LoadBalancer                 # A port is opened on each node in your cluster via Kube proxy.
  ports:                         # Take incoming HTTP requests on port 9090 and forward them to the targetPort of 8080
  - name: http
    port: 80
    targetPort: 8080
  selector:
    app: go-hello-world         # Map any pod with label `app=go-hello-world` to this service
```


$ k apply -f k8s-deployment.yaml  
$ k get svc  
> NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP                                                                    PORT(S)        AGE
> go-hello-world-service   LoadBalancer   172.20.212.97   aad0dcf28ca5442e0b5ea2c1353dd28c-1763660814.ap-northeast-1.elb.amazonaws.com   80:31946/TCP   11m

動作確認  
>  $ curl aad0dcf28ca5442e0b5ea2c1353dd28c-1763660814.ap-northeast-1.elb.amazonaws.com  
> Hello, Guest  
>  $ curl aad0dcf28ca5442e0b5ea2c1353dd28c-1763660814.ap-northeast-1.elb.amazonaws.com?name=liwei  
> Hello, liwei


