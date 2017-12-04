# Issue Let's Encrypt certificate using Google Cloud DNS

This tutorial shows how to issue free SSL certificate from Let's Encrypt via DNS challenge for domains using Google Cloud DNS service.

This article has been tested with a GKE cluster.

```console
$ kubectl version --short
Client Version: v1.8.3
Server Version: v1.8.4-gke.0
```

## Deploy Voyager operator

Deploy Voyager operator following instructions here: https://github.com/appscode/voyager/blob/master/docs/install.md

```console
# install without RBAC
curl -fsSL https://raw.githubusercontent.com/appscode/voyager/5.0.0-rc.5/hack/deploy/voyager.sh \
  | bash -s -- gke
```

If you are trying this on a RBAC enabled cluster, pass the flag `--rbac` to installer script.

```console
# install without RBAC
curl -fsSL https://raw.githubusercontent.com/appscode/voyager/5.0.0-rc.5/hack/deploy/voyager.sh \
  | bash -s -- gke --rbac
```