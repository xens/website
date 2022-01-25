---
title: Use a SOCKS5 Proxy to Access the Kubernetes API
content_type: task
weight: 40
---

<!-- overview -->
This page shows how to use an SOCKS5 proxy to access a remote Kubernetes API.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

<!-- steps -->

## Setup big-picture

The following diagram represents what we're going to achieve in this tutorial. We've got a client machine from where we're going to create requests to talk to the Kubernetes API that is hosted on a remote machine. We leverage SSH to create a secure SOCKS5 tunnel between the local and the remote machine on top of which the HTTPS traffic between the client and the Kubernetes API will flow. In this example we're using SSH as the tunnel mechanism but it can be replaced by any other SOCKS5 proxy.

{{< mermaid >}}
graph LR;

  subgraph local[Local client machine]
  client([client])-- Local <br> traffic .->  local_ssh[Local SSH <br> SOCKS5 proxy];
  end
  local_ssh[SSH <br>SOCKS5 <br> proxy]-- SSH Tunnel -->SSHd
  
  subgraph remote[Remote server]
  SSHd[SSHd <br> server]-- Local traffic -->service1;
  end
  client([client])-. proxied HTTPs traffic <br> going through the proxy .->service1[Kubernetes API];

  classDef plain fill:#ddd,stroke:#fff,stroke-width:4px,color:#000;
  classDef k8s fill:#326ce5,stroke:#fff,stroke-width:4px,color:#fff;
  classDef cluster fill:#fff,stroke:#bbb,stroke-width:2px,color:#326ce5;
  class ingress,service1,service2,pod1,pod2,pod3,pod4 k8s;
  class client plain;
  class cluster cluster;
{{</ mermaid >}}

## Using ssh to create a SOCKS5 proxy

This command starts a SOCKS5 proxy between your client machine and the target where the Kubernetes API is listening:

    $ssh -D 8080 -q -N  username@kubernetes_server

* -D 8080: opens a SOCKS proxy on local port :8080.
* -q: quiet mode. Causes most warning and diagnostic messages to be suppressed.
* -N: Do not execute a remote command. Useful for just forwarding ports.
* username@kubernetes_server: the remote SSH server where the Kubernetes cluster is running.

## Client configuration

To explore the Kubernetes API you'll first need to instruct your clients to send their queries through the SOCKS5 proxy we created earlier:

for command-line tools we can export the following ENV variable:

    $export HTTPS_PROXY=socks5://localhost:8080

Commands like `cURL` will then route their traffic through the proxy:

    $curl -k -v https://localhost/api

In this example localhost will not be the localhost of the client machine but the localhost of the Kubernetes server.

To use the official Kubernetes client `kubectl` with a proxy you'll need to set it in your `~/.kube/config` file:

    - cluster:
          certificate-authority-data: xxx
          server: https://localhost:6443
          proxy-url: socks5://localhost:8080

Then you should be able to query the cluster through the proxy:

    $kubectl get pods
    NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
    kube-system   coredns-85cb69466-klwq8                  1/1     Running     0          5m46s
