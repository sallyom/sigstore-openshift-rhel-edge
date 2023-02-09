### Sigstore on ROSA, OpenShift on AWS, deployed with AWS Security Token Service (STS)

[Running pods in OpenShift with AWS IAM Roles for service accounts (IRSA)](https://cloud.redhat.com/blog/running-pods-in-openshift-with-aws-iam-roles-for-service-accounts-aka-irsa)

Documentation about the components that comprise a keyless Sigstore system
is available in the [sigstore documentation](https://docs.sigstore.dev/).

This describes how to launch a Sigstore stack in OpenShift.
AWS STS is used to create workload identity tokens for service accounts.
Sigstore is configured to use the service account OIDC Identity Token to pass to Fulcio to authenticate requests.

You must have cluster-admin privileges to deploy the Sigstore stack.

The [Sigstore helm-charts](https://github.com/sigstore/helm-charts) will be used to deploy Sigstore components in OpenShift.

#### Create custom CA and key in fulcio-system namespace

Open [fulcio-create-CA script](./ROSA/fulcio-create-root-ca-openssl.sh) to check out the commands before running it.
The `openssl` commands are interactive.

```bash
./fulcio-create-root-ca-openssl.sh
oc create ns fulcio-system
oc -n fulcio-system create secret generic fulcio-secret-rh --from-file=private=file_ca_key.pem --from-file=public=file_ca_pub.pem --from-file=cert=fulcio-root.pem  --from-literal=password=secure --dry-run=client -o yaml | oc apply -f-
```

#### Edit the local [values.yaml](./ROSA/values.yaml) file

Run something like the following to view your cluster's cluster-domain

```bash
oc get dnsrecords -o yaml -n openshift-ingress-operator | grep dnsName
```
Edit the values.yaml file accordingly.

#### Install sigstore/scaffold helm chart and SCC anyuid script

Rather than use custom charts with added necessary SCCs, use [this script](./anyuid-scc.sh)

```bash
helm upgrade -i scaffolding sigstore/scaffold -f values.yaml && ./anyuid-scc.sh
```

Watch as the stack rolls out.
You might follow the logs from the fulcio-server deployment in `-n fulcio-system`.

#### Launch a test pod with cosign and an identity token created with AWS Security Token Service

[Understanding ROSA with STS](https://docs.openshift.com/rosa/rosa_getting_started/rosa-sts-getting-started-workflow.html)

```bash
oc new-project sigstoretest
oc apply -f sa.yaml
oc apply -f pod.yaml
```

Finally, [cosign](https://github.com/sigstore/cosign) can be used in the pod to sign and verify artifacts.
Currently I am entering the `signer-identity -n sigstoretest` pod and running the following commands:

```bash
cosign initialize --mirror=$TUF_URL --root=$TUF_URL/root.json
cosign login --username "sallyom" --password "+bpSxxxxxxxxxxx" quay.io
COSIGN_EXPERIMENTAL=1 cosign sign --force -y --fulcio-url=$FULCIO_URL --identity-token=$(cat $AWS_WEB_IDENTITY_TOKEN_FILE) --rekor-url=$REKOR_URL quay.io/sallyom/cosign:latest
```

TODO:
* Document plugging into a cluster build pipeline
* Integrate with a controller
* Document verifying cosign signed artifacts at the edge
