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

## Deploy Voyager operator

```console
# install without RBAC
curl -fsSL https://raw.githubusercontent.com/appscode/voyager/5.0.0-rc.4/hack/deploy/voyager.sh \
  | bash -s -- aws

# run on master
kubectl patch deploy voyager-operator -n kube-system \
  --patch "$(curl -fsSL https://raw.githubusercontent.com/appscode/voyager/5.0.0-rc.4/hack/deploy/run-on-master.yaml)"
```

If you are trying this on a RBAC enabled cluster, pass the flag `--rbac` to installer script.
```console
# install without RBAC
curl -fsSL https://raw.githubusercontent.com/appscode/voyager/5.0.0-rc.4/hack/deploy/voyager.sh \
  | bash -s -- aws --rbac
```

## Configure Domain


```console
$ dig -t ns kiteci.pro

; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t ns kiteci.pro
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5237
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;kiteci.pro.			IN	NS

;; ANSWER SECTION:
kiteci.pro.		1800	IN	NS	dns1.registrar-servers.com.
kiteci.pro.		1800	IN	NS	dns2.registrar-servers.com.

;; ADDITIONAL SECTION:
dns1.registrar-servers.com. 1480 IN	A	216.87.155.33
dns1.registrar-servers.com. 1021 IN	AAAA	2620:74:19::33
dns2.registrar-servers.com. 141785 IN	A	216.87.152.33
dns2.registrar-servers.com. 1784 IN	AAAA	2001:502:cbe4::33

;; Query time: 24 msec
;; SERVER: 127.0.1.1#53(127.0.1.1)
;; WHEN: Mon Nov 27 13:37:04 PST 2017
;; MSG SIZE  rcvd: 186
```
