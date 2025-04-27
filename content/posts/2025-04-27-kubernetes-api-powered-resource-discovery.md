---
author: João Antunes
date: 2025-04-27 19:40:00+01:00
layout: post
title: Kubernetes API powered resource discovery (feat. K8s C# client)
summary: Have you ever felt the need to automate the discovery of some information within a distributed system? Turns out, the Kubernetes API makes it surprisingly simple to do so.
images:
  - /images/2025/04/27/kubernetes-api-powered-resource-discovery.png
categories:
  - csharp
  - dotnet
tags:
  - kubernetes
slug: kubernetes-api-powered-resource-discovery
---

## Intro

When building distributed applications, have you ever felt the need to automate the discovery of some information within the overall system? For instance, you might want to know when a new instance of a service with certain metadata information comes online, in order to trigger some action. A great example of this in action is [Traefik](https://traefik.io). When using [Traefik with Docker](https://doc.traefik.io/traefik/providers/docker/), we can assign labels to our service containers, allowing Traefik to dynamically configure request routing to them. 

Like Traefik, many other applications operate this way, so I knew it was possible. What I didn’t know, was how easy or difficult it would be to implement this kind of functionality. As it turns out, if you're deploying applications to Kubernetes, its API makes the process surprisingly simple. Add to that the available client libraries - being the C# one the focus of this post - it becomes even more straightforward to get things going.
## Example scenario

Let's use a simple example to see this Kubernetes discovery going. We'll create a .NET service which will use the [Kubernetes C# client](https://github.com/kubernetes-client/csharp) in a background service to keep track of Kubernetes services exposing a predefined annotation.

Just to see things in action, we'll deploy a couple of [nginx](https://nginx.org/) services, annotating them in a way that out monitoring service can find the relevant information.

You can check out the full Kubernetes manifest files in the [source repository](https://github.com/joaofbantunes/DiscoveryViaKubernetesApiSample), but I'll add an nginx service definition here, so we can see how the annotations that we'll be reading look like.

```yaml
apiVersion: v1  
kind: Service  
metadata:  
  name: sample-nginx-service-1  
  annotations:  
    some-sample-annotation: "this is sample nginx service 1"  
spec:  
  selector:  
    app: sample-nginx-1  
  ports:  
    - protocol: TCP  
      port: 6060  
      targetPort: 80
```

As we can see, nothing fancy, just a simple annotation named `some-sample-annotation`, set as part of the service metadata.

Note that I'm using a service resource in this example, but we could apply the same kind of logic to other kinds of resources, it's not a service only thing.

## Working with the Kubernetes C# client

Now, for the interesting part, that ends up being much simpler than I expected 😅.

Created a quick C# project with a `KubernetesMonitor` class which inherits from `BackgroundService`. The `MonitorAsync` method is the interesting bit, so let's get right into it.

```csharp
private async Task MonitorAsync(CancellationToken ct)  
{  
    var config = KubernetesClientConfiguration.InClusterConfig();  
    var client = new Kubernetes(config);  
    var response = client.CoreV1.ListServiceForAllNamespacesWithHttpMessagesAsync(  
        watch: true,  
        cancellationToken: ct);  
  
    await foreach (var (type, service) in response.WatchAsync<V1Service, V1ServiceList>(cancellationToken: ct))  
    {
	    logger.LogInformation(
		    """
		    {Type}: service "{Service}" (namespace "{Namespace}"), with annotation "{Annotation}"
		    """,
		    type,
			service.Metadata.Name,  
            service.Metadata.Namespace(),  
            GetAnnotation(service));  
    }
}
```

We start by creating an instance of the `Kubernetes` class, using a default configuration. This class allows us to interact with the Kubernetes API.

We then use `ListServiceForAllNamespacesWithHttpMessagesAsync`, setting the `watch` flag as `true`, so we are informed of changes in the cluster as they happen. A couple of notes on using this method:

- Normally we would probably use `ListNamespacedServiceWithHttpMessagesAsync`instead, to watch only resources in a given namespace, but for this example, we're just watching everything.
- If you start watching after relevant resource changes, not to worry, this method will start by returning the current state of things, before new updates (though this can be disabled)
- We can pass some parameters that allow us to filter what resources we get notified about (e.g. we may only be interested in updates from resources we certain labels applied to them)

With the result of `ListServiceForAllNamespacesWithHttpMessagesAsync` in hand (of type `Task<HttpOperationResponse<V1ServiceList>>`), we can use the `WatchAsync` extension method to get an `IAsyncEnumerable` that makes it super clean to process the cluster changes as they're returned to us by the API.

I'm not doing anything useful with the change details in the demo, just logging it. For reference, here is the `GetAnnotation` function, which extracts the annotations we saw in the previous section from the service metadata.

```csharp
static string GetAnnotation(V1Service service)  
{  
	if (service.Metadata?.Annotations is null) return "N/A";
    
    service
	    .Metadata
	    .Annotations
	    .TryGetValue("some-sample-annotation", out var maybeAnnotation);  
    
    return !string.IsNullOrWhiteSpace(maybeAnnotation)
	    ? maybeAnnotation
	    : "N/A";  
}
```

## Deploying & seeing things in action

I'm not going to describe how to deploy things to Kubernetes, as it's not the point of the post (for running the sample, instructions are on the repo's readme), but it's worth just mentioning what needs to be different if we want to access the Kubernetes API.

To access the Kubernetes API from the pod, we need to give it permissions to do so. This can be done by creating a role or a cluster role with the required permissions (in this case we'll use a cluster role for our monitoring service to be able to access any namespace).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitor-role
rules:
  - apiGroups:
      - "*"
    resources:
      - "*"
    verbs:
      - get
      - list
      - watch
```

Note that we can (and probably should) limit more the permissions than I did, but given it's a demo, I just wanted to get it to work 😅.

With all the code and Kubernetes manifests ready, we can test our monitoring service.

If we start the monitor and then one of our sample nginx deployments, we'll get a log like `Added: service "sample-nginx-service-1" (namespace "default"), with annotation "this is sample nginx service 1"`.

If we then update the annotation, we get something like `Modified: service "sample-nginx-service-1" (namespace "default"), with annotation "this is sample nginx service 1 (updated)"`.

If we delete the nginx deployment, we'll get `Deleted: service "sample-nginx-service-1" (namespace "default"), with annotation "this is sample nginx service 1 (updated)"`.

We could also invert the order we do things, starting nginx first, then our monitoring service. When the monitoring service starts, it'll log `Added: service "sample-nginx-service-1" (namespace "default"), with annotation "this is sample nginx service 1"`.
## Outro

That does it for this quick look at using the Kubernetes API to discover information about the resources in a cluster.

We saw how to use the Kubernetes C# client library to implement the somewhat simple task of discovering services in our cluster, to read custom annotations we added to them.

I guess this isn't a task to do often, as most of the times infrastructure components handle these kinds of needs, but I've had a couple of scenarios having these capabilities could be interesting (granted, I was implementing infrastructure components 😅).

Relevant links:

- [Source code](https://github.com/joaofbantunes/DiscoveryViaKubernetesApiSample)
- [Kubernetes C# Client](https://github.com/kubernetes-client/csharp)
- [Kubernetes API](https://kubernetes.io/docs/reference/using-api/)
- [Traefik](https://traefik.io)
- [Traefik Docker docs](https://doc.traefik.io/traefik/providers/docker/)

Thanks for stopping by, cyaz! 👋
