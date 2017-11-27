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
Deploy Voyager operator following instructions here: https://github.com/appscode/voyager/blob/master/docs/install.md

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
In this tutorial, I am going to use `kiteci.pro` domain that was purchased on namecheap.com . Now, go to your AWS Route53 console and create a hosted zone for this domain.

![create-hosted-zone](/create-hosted-zone.png)

Once the hosted zone is created, you can see the list of name servers in AWS console.

![ns-servers](/ns-servers.png)

Now, go to the website of your domain registrar and update the list of name servers.

![domain-registrar](/domain-registrar.png)

Give time to propagate the updated DNS records. You can use the following command to confirm that the name server records has been updated.

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
