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
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57300
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;kiteci.pro.			IN	NS

;; ANSWER SECTION:
kiteci.pro.		21599	IN	NS	ns-109.awsdns-13.com.
kiteci.pro.		21599	IN	NS	ns-1404.awsdns-47.org.
kiteci.pro.		21599	IN	NS	ns-1623.awsdns-10.co.uk.
kiteci.pro.		21599	IN	NS	ns-697.awsdns-23.net.

;; Query time: 58 msec
;; SERVER: 127.0.1.1#53(127.0.1.1)
;; WHEN: Mon Nov 27 13:40:03 PST 2017
;; MSG SIZE  rcvd: 179
```
