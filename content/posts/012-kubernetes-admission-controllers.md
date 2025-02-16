+++
title = "Kubernetes Admission Controllers: A Dive Into Policy Enforcement"
description = "A study on Kubernetes Admission Controllers and how to build, deploy and use one in a cluster."
image = "https://raw.githubusercontent.com/TaskMasterErnest/smol/master/images/tskmstr.jpg"
date = 2025-02-04T07:07:07+01:00
draft = true
categories = ["kubernetes"]
tags = ["admission controllers", "webhooks"]
+++

Kubernetes has revolutionized how we deploy and manage our applications.
Ensuring compliance, security, and operational best practices across a cluster requires robust guardrails.
That is where we have **Kubernetes Admission Controllers**—the gatekeepers of the cluster.
In here, we will explore how they work, why we need them, and how we can build some custom policies tailored to our organization's needs.

<!--more-->
---
## Introduction (From A Layman's POV)
Imagine you are the security guard at Area 51 (a restricted area - I wouldn't happen to know anything about that). You get to make the decision on who enters or not based off some factors you are looking out for.
You want to to make sure that:
- only authorized people can enter.
- people entering follow certain rules, like having an ID badge.
- visitors who have been granted access need some modification to their status, like assigning them a guest/visitor badge.

That is what Admission Controllers are. They are these security guards that stop and check requests coming into the cluster, before these requests are permanently turned into objects, ensuring that only authorized, valid and compliant requests go through.

---
## What are Admission Controllers?
Now that you have been primed on what Admission Controllers are, and before we dive into what they are, let us look at the way a request for creating or modifying an object flows through a cluster before an object is created or modified.

Kubernetes has an API Server which acts as the central location for all requests coming into the cluster. If you are thinking that the API server resides in the control plane of a Kubernetes cluster—you are right!

The API Server exposes itself via an HTTP REST endpoint where users/clients connect to an send their requests. These requests go through the following main stages: authentication ==> authorization ==> admission. 

Read all about the API Server and the request flow [here](https://taskmasterernest.github.io/posts/008-a-dive-into-the-kubernetes-api-server/#kubernetes-http-request-flow).

Before we get to what Admission Controllers actually are, let us refresh our memories on that Controllers are in Kubernetes.
A Controller is a control loop that observes the state of a cluster, compares it with the desired state, and takes action to reconcile the actual state with the desired state.

What an Admission Controller does it to intercept API requests made to the API server, makes sure the requests pass a certain criterium before being persisted in the etcd datastore.

Tying that into the Controller definition, the Admission Controller listens for requests being made to the API server, it intercepts these requests, checks if they match with the desired state of the cluster and then take some action depending on their findings.

---

## How Admission Controllers Work
The illustration below shows the request flow to the Kubernetes API server when we have Admission Controllers in the mix.
![image of request flow with admission controllers](/img/request-flow-with-admission-controllers.png "Kubernetes Request Flow with Admission Controllers Present")

From the illustration above, we can enumerate the stages through which requests to the API server flow.
1. the API Server receives an API call via its HTTP REST interface.
2. the request is authenticated and authorized.
3. the request payload is run through the Mutating Admission Controllers.
4. the resultant object schema is checked to see if it abides by Kubernetes object schema standards.
5. the object schema is then run through the Validation Admission Controllers.
6. finally, the objects are stored in the etcd cluster.

In the request flow diagram above, each Admission Controller utilizes webhooks—these webhooks are HTTP callbacks that receive and modify or validate requests that are being sent to the API server.

These webhooks are called Admission Webhooks and they are the juice of the Admission Controllers.

---

## Admission Webhooks - the marrow in the bone
Admission Webhooks are extensions of the Admission Controllers that do the work of checking whether a request matches the required schema specified for the cluster, before being validated and stored in an etcd cluster.

There are two types of admission webhooks, each paired with the specific admission controller; they are:
1. **mutating admission webhooks**: They contain custom code that modifies the request before it is applied to the cluster.
2. **validation admission webhooks**: They contain custom code that validates—not modify—a request before it is applied to a cluster.

The admission webhooks are executed in phases. The first phase sees the mutating admission webhook being executed, then the validating admission webhooks are executed in the second phase.

> Note that mutating admission webhooks modify the actual request that was issued. The issuer mya not know about the changes that have been implemented.
> For security and transparency, the best way to utilize webhooks is to use the validating admission webhook to reject a request and have the issuer fix the request.

---

## Creating Controllers with Custom Logic
In this section, we are going to build out a couple of custom admission webhooks for very specific examples that I have scoped out.

1. the first is a mutating admission webhook that will add a custom annotion to the Pod ie `PodModified: "true"`
2. the second is a validating webhook that will look up an annotation `team` and ensure it exists before allowing the request to pass.

This is the part where we get our hands dirty by writing some Go code that communicates using the kubernetes-client package to the Kubernetes API server.

We will do the following in order:
- write a web server that listens for requests
- decode the request, AdmissionReview objects, that are sent by the API server
- implement the mutation logic
- implement validation logic

Let us jump right into it!

---

### 1. Writing the Web Server
For writing the controller, I opted to use the Gorilla Mux package. You could say this is a preference of mine. It does contain some features that I find useful.

We define our server this way:

1. We define the configuration constraints that our web server should abide by:
```go
const (
	tlsDir        = "/etc/webhook/certs"
	tlsCertFile   = "tls.crt"
	tlsKeyFile    = "tls.key"
	webhookPort   = ":8443"
	readTimeout   = 10 * time.Second
	writeTimeout  = 10 * time.Second
	idleTimeout   = 30 * time.Second
	shutdownGrace = 5 * time.Second
)
```

- in here, we state the directory where the server should look for its TLS configuration, the ports the web server should run on and some timeouts for the web server.

2. We add in some global variables that are needed to work with Kubernetes objects, specifically for serializing and de-serializing data.
```go
var (
	scheme       = runtime.NewScheme()
	codecFactory = serializer.NewCodecFactory(scheme)
	deserializer = codecFactory.UniversalDeserializer()
	podResource  = metav1.GroupVersionResource{
		Group:    "",
		Version:  "v1",
		Resource: "pods",
	}
	allowedContent = "application/json"
)
```
- the `scheme` here is a registry that holds all the information about all the Kubernetes objects (Pods, Deployments, etc.) and how they are structured. 
- The `runtime` package here provides the tools that will be used to work with these Kubernetes objects ar runtime.
- the `codecFactory` is used to create codecs. A codec can be defined as the black box that performs the encoding/serialization and decoding/de-serialization of Kubernetes objects.
- the `serializer` package provides the tools that are needed to encode and/or decode Kubernetes objects.
- for the `podResource`, this is used to define the specific Kubernetes resource that we will be working with.

3. We also define a `Webhook` struct that contains the `http.Server{}` and `slog.Logger{}` structs. This is so we can use both structs when referencing our webhook server.
```go
type WebhookServer struct {
	server *http.Server
	logger *slog.Logger
}
```
- I have developed a liking for slog for structured logging. It feels much more intuitive for writing logging statements.

4. We then create our server with Mux and with the constraints we have defined.
```go
mux := http.NewServeMux()
whs := &WebhookServer{
  logger: logger,
  server: &http.Server{
    Addr:    webhookPort,
    Handler: mux,
    TLSConfig: &tls.Config{
      Certificates: []tls.Certificate{cert},
      MinVersion:   tls.VersionTLS13,
    },
    ReadTimeout:  readTimeout,
    WriteTimeout: writeTimeout,
    IdleTimeout:  idleTimeout,
  },
}
```

5. We register the endpoints where we will make a call to the custom webhook code to perform the admission job, either mutating or validating.
```go
mux.HandleFunc("/mutate", whs.handleRequest(whs.serveMutatingRequest))
mux.HandleFunc("/validate", whs.handleRequest(whs.serveValidatingRequest))
mux.HandleFunc("/healthz", whs.healthCheck)
```

6. To modularize things a bit, we create an `admissionHandler` type that takes in a a request via  an `admissionRequest` function and the returns a reponse via the `admissionResponse`.
```go
type admissionHandler func(*admissionv1.AdmissionRequest) (*admissionv1.AdmissionResponse, error)
```
- the `AdmissionRequest` in Kubernetes is what contains all of the information about a request that is being sent to the API server—and it identifies the request as being subject to admission control.
- the `AdmissionResponse` contains the response from the API server. It contains information as to whether a request has been allowed or denied. In the case of modification, it includes the new request schema to be sent to the API server.

7. We then create a `handleRequest` method that takes in the `admissionHandler` and returns a `handlerFunc` to our server.
```go
func (whs *WebhookServer) handleRequest(handler admissionHandler) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			whs.errorResponse(w, "Method Not Allowed", http.StatusMethodNotAllowed)
			return
		}

		if contentType := r.Header.Get("content-type"); contentType != allowedContent {
			whs.errorResponse(w, fmt.Sprintf("Unsupported Content Type: %s", contentType), http.StatusBadRequest)
			return
		}

		body, err := io.ReadAll(r.Body)
		if err != nil || len(body) == 0 {
			whs.errorResponse(w, "Empty or Unreadable Body", http.StatusBadRequest)
			return
		}

		var admissionReview admissionv1.AdmissionReview
		if _, _, err := deserializer.Decode(body, nil, &admissionReview); err != nil {
			whs.errorResponse(w, "Invalid Admission Review Request", http.StatusBadRequest)
			return
		}

		if admissionReview.Request == nil {
			whs.errorResponse(w, "Missing Admission Request", http.StatusBadRequest)
			return
		}

		response, err := handler(admissionReview.Request)
		if err != nil {
			whs.logger.Error("Admission review", "error", err)
			response = &admissionv1.AdmissionResponse{
				UID:     admissionReview.Request.UID,
				Allowed: false,
				Result: &metav1.Status{
					Message: err.Error(),
					Code:    http.StatusInternalServerError,
				},
			}
		}

		admissionReview.Response = response
		admissionReview.Response.UID = admissionReview.Request.UID

		res, err := json.Marshal(admissionReview)
		if err != nil {
			whs.errorResponse(w, "Error Encoding Response", http.StatusInternalServerError)
			return
		}

		w.Header().Set("Content-Type", allowedContent)
		if _, err := w.Write(res); err != nil {
			whs.logger.Error("Failed to write response", "error", err)
		}
	}
}
```

In this section, we have created the web server and the logic to handle the requests that are to be sent to the API server.

---

### 2. Implementing the Mutation Logic
In here is where we handle the request that was intercepted on its way to the API server, and then change its schema—to what is prescribed, and send it on its way again.

For our specific example of using the Mutation Logic on a Pod, here is the way we define this:

1. We check if the request coming in, in the `AdmissionRequest` is for a Pod resource.
```go
if req.Resource != podResource {
  return nil, fmt.Errorf("unsupported Resource: %s", req.Resource)
}
```
- if this is not a Pod resource, then we return an error indicating that it is unsupported.

2. We now attempt to deserialize (decode) the request object in the `AdmissionRequest` into a Kubernetes Pod struct.
```go
var pod corev1.Pod
if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
  return nil, fmt.Errorf("failed to unmarshal Pod: %w", err)
}
```

3. We initialize our patching—which is what we do to these requests as modification—struct and we apply the patches.
```go
var patches []patchOperation

if pod.Annotations == nil {
  patches = append(patches, patchOperation{
    Op:    "add",
    Path:  "/metadata/annotations",
    Value: make(map[string]string),
  })
}

patches = append(patches, patchOperation{
  Op:    "add",
  Path:  "/metadata/annotations/PodModified",
  Value: "true",
})
```
- in this case, we make sure that we initialize the pod annotations and specify that a path be constructed for us.
- then, we mutate the request and by adding the actual annotation patch that we need.

4. Then we marshal the new, mutated object schema and send a response back to the webhook server who will then forward the request along to the API server.
```go
patchBytes, err := json.Marshal(patches)
if err != nil {
  return nil, fmt.Errorf("failed to marshal patch: %w", err)
}

return &admissionv1.AdmissionResponse{
  UID:     req.UID,
  Allowed: true,
  Patch:   patchBytes,
  PatchType: func() *admissionv1.PatchType {
    pt := admissionv1.PatchTypeJSONPatch
    return &pt
  }(),
}, nil
```
- after this response is sent back to the webhook server handler, the request ID is matched against the response ID so that that webhook server knows which of the requests it needs to work on.

A simple visual of the process flow that utilizes the MutatingAdmissionWebhook is shown here:
![Visual of Mutating Admission Webhook flow](/img/request-flow-with-mutating-admission-webhook.png)

---

### 3. Implementing the Validation Logic