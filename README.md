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

## Setup Route53 Hosted Zone

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

## Configure IAM Permissions

To issue SSL certificate using Let's Encrypt, we have to prove that we own the `kiteci.pro` domain. The following AWS IAM policy document describes the permissions required for voyager operator to complete the DNS challenge. Replace <INSERT_YOUR_HOSTED_ZONE_ID_HERE> with the Route 53 zone ID of the domain you are authorizing.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:GetChange",
                "route53:ListHostedZonesByName"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/<INSERT_YOUR_HOSTED_ZONE_ID_HERE>"
            ]
        }
    ]
}
```

There are few different ways to grant these permissions to voyager operator pods.

### option #1: Using Instance IAM Role

When kops creates a cluster, it creates 2 IAM roles: one for the master and one for the nodes. You can grant these additional IAM permissions to the appropriate instance IAM role.

Here, we are running voyager operator pod on master node. So, we will grant these permissions to the role assigned to the master instance. To do this, take the following steps:

- Go to your EC2 dashboard and identify the IAM role for your master instance.

![master-iam-role](/master-iam-role.png)

- Go the [IAM roles console](https://console.aws.amazon.com/iam/home#/roles) and select the master IAM role for your cluster.

![master-role](/master-iam-role-console.png)

- Now add the custom inline policy show above.

![add-policy](/add-policy.png)


**NB:** _If you decide to run voyager operator on regular nodes, then you can grant these additional IAM permissions to the node IAM role for your cluster. Please note that this will allow any pods running on the nodes to perform these api calls._

### option #2: Create IAM User

If you are running cluster on cloud providers other than AWS but want to use Route53 as your DNS provider, this is your only option. You can also use this method for clusters running on AWS.

Here we will create a new IAM role called `voayegr` and grant it the necessary permissions. Then we wil issue an access key pair for this IAM role and pass this to voyager using a Kubernetes secret.

```console
aws iam create-user --user-name voyager
aws iam put-user-policy --user-name voyager --policy-name voyager --policy-document file://$PWD/voyager-policy.json
aws iam create-access-key --user-name voyager

kubectl create secret generic voyager-route53 --namespace default \
  --from-literal=AWS_ACCESS_KEY_ID=INSERT_YOUR_ACCESS_KEY_ID_HERE \
  --from-literal=AWS_SECRET_ACCESS_KEY=INSERT_YOUR_SECRET_ACCESS_KEY_HERE \
  --from-literal=AWS_HOSTED_ZONE_ID=INSERT_YOUR_HOSTED_ZONE_ID_HERE

kubectl get secret voyager-route53 -o yaml
apiVersion: v1
data:
  AWS_ACCESS_KEY_ID: SU5TRVJUX1lPVVJfQUNDRVNTX0tFWV9JRF9IRVJF
  AWS_HOSTED_ZONE_ID: SU5TRVJUX1lPVVJfSE9TVEVEX1pPTkVfSURfSEVSRQ==
  AWS_SECRET_ACCESS_KEY: SU5TRVJUX1lPVVJfU0VDUkVUX0FDQ0VTU19LRVlfSEVSRQ==
kind: Secret
metadata:
  creationTimestamp: 2017-11-27T23:17:31Z
  name: voyager-route53
  namespace: default
  resourceVersion: "16160"
  selfLink: /api/v1/namespaces/default/secrets/voyager-route53
  uid: 24949869-d3c9-11e7-98b3-08002787a1b5
type: Opaque
```

**NB**:

- Please make sure that you have updated the voyager-policy.json file to use the hosted zone id for your domain.
- The _`file://` prefix is required_, otherwise you will get an error like `An error occurred (MalformedPolicyDocument) when calling the PutUserPolicy operation: Syntax errors in policy.`
- The Kubernetes secret must be created in the same namespace where the `Certificate` object exists.
- `AWS_HOSTED_ZONE_ID` key is optional and will be automatically detected from DNS records, if not found in secret. If you are trying to issue a SANS certificate with multiple top level domains, do not set `AWS_HOSTED_ZONE_ID` in the secret.

## Create Certificate

Create a secret to provide ACME user email. Change the email to a valid email address and run the following command:

```console
kubectl create secret generic acme-account --from-literal=ACME_EMAIL=me@example.com
```

Create the Certificate CRD to issue TLS certificate from Let's Encrypt using DNS challenge.

```console
kubectl apply -f crt.yaml

apiVersion: voyager.appscode.com/v1beta1
kind: Certificate
metadata:
  name: kitecipro
  namespace: default
spec:
  domains:
  - kiteci.pro
  acmeUserSecretName: acme-account
  challengeProvider:
    dns:
      provider: route53
```

Now wait a bit and you should see a new secret named `tls-kitecipro`. This contains the `tls.crt` and `tls.key` .

```console
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
acme-account          Opaque                                3         20m
default-token-zj0wv   kubernetes.io/service-account-token   3         30m
tls-kitecipro         kubernetes.io/tls                     2         19m
```

```console
$ kubectl describe cert kitecipro
Name:		kitecicom
Namespace:	default
Labels:		<none>
Annotations:	kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"voyager.appscode.com/v1beta1","kind":"Certificate","metadata":{"annotations":{},"name":"kitecicom","namespace":"default"},"spec":{"acmeU...
API Version:	voyager.appscode.com/v1beta1
Kind:		Certificate
Metadata:
  Cluster Name:
  Creation Timestamp:			2017-10-29T22:07:45Z
  Deletion Grace Period Seconds:	<nil>
  Deletion Timestamp:			<nil>
  Resource Version:			1376
  Self Link:				/apis/voyager.appscode.com/v1beta1/namespaces/default/certificates/kitecicom
  UID:					97d91028-bcf5-11e7-bc3f-42010a800fd5
Spec:
  Acme User Secret Name:	acme-account
  Challenge Provider:
    Http:
      Ingress:
        API Version:	voyager.appscode.com/v1beta1
        Kind:		Ingress
        Name:		test-ingress
  Domains:
    kiteci.com
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason		Message
  ---------	--------	-----	----			-------------	--------	------		-------
  20m		20m		1	voyager operator			Normal		IssueSuccessful	Successfully issued certificate
```
