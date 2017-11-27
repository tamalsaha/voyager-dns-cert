# voyager-route53-cert

This tutorial shows how to issue free SSL certificate from Let's Encrypt using DNS challenge for domains using Route53 DNS service.

This article has been tested with a kops managed Kubernetes cluster on AWS.

```console
$ kops version
Version 1.7.1 (git-c69b811)

$ kubectl version --short
Client Version: v1.8.3
Server Version: v1.7.10
```
