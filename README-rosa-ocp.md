## Sigstore on OpenShift, with AWS Security Token Service (STS)

[Running pods in OpenShift with AWS IAM Roles for service accounts (IRSA)](https://cloud.redhat.com/blog/running-pods-in-openshift-with-aws-iam-roles-for-service-accounts-aka-irsa)

Documentation about the components that comprise a keyless Sigstore system
is available in the [sigstore documentation](https://docs.sigstore.dev/).

This describes how to launch a Sigstore stack in OpenShift.
AWS STS is used to create workload identity tokens for service accounts.
Sigstore is configured to use the service account OIDC Identity Token to pass to Fulcio to authenticate requests.

You must have cluster-admin privileges to deploy the Sigstore stack.

The [Sigstore helm-charts](https://github.com/sigstore/helm-charts) will be used to deploy Sigstore components in OpenShift.

### Create custom CA and key in fulcio-system namespace

Open [fulcio-create-CA script](./ROSA/fulcio-create-root-ca-openssl.sh) to check out the commands before running it.
The `openssl` commands are interactive.

```bash
./fulcio-create-root-ca-openssl.sh
oc create ns fulcio-system
oc -n fulcio-system create secret generic fulcio-secret-rh --from-file=private=file_ca_key.pem --from-file=public=file_ca_pub.pem --from-file=cert=fulcio-root.pem  --from-literal=password=secure --dry-run=client -o yaml | oc apply -f-
```

### Edit the local [values.yaml](./ROSA/values.yaml) file

Run something like the following to view your cluster's cluster-domain

```bash
oc get dnsrecords -o yaml -n openshift-ingress-operator | grep dnsName
```
Edit the values.yaml file accordingly.

### Install sigstore/scaffold helm chart and SCC anyuid script

Rather than use custom charts with added necessary SCCs, use [this script](./anyuid-scc.sh)

```bash
helm upgrade -i scaffolding sigstore/scaffold -f values.yaml && ./anyuid-scc.sh
```

Watch as the stack rolls out.
You might follow the logs from the fulcio-server deployment in `-n fulcio-system`.

## Launch pod with cosign container and an identity token created with AWS Security Token Service

[Understanding ROSA with STS](https://docs.openshift.com/rosa/rosa_getting_started/rosa-sts-getting-started-workflow.html)

```bash
oc new-project sigstore-test

# If using keyless signing, you do not need to generate a signing key, but the pod.yaml expects this secret so update accordingly
# For the signing key, I use the password 'secure', and COSIGN_PASSWORD is hard-coded in pod.yaml so update accordingly
cosign generate-key-pair k8s://sigstore-test/cosign-keypair

# To push signatures to registries, cosign requires push access
oc create secret generic -n sigstore-test --from-file /path/to/your/registry/auth/config.json cosign-login

# Inspect the pod.yaml and make any necessary changes
oc apply -f ROSA/sts-sa.yaml -n sigstore-test
oc apply -f ROSA/pod.yaml -n sigstore-test
```

Finally, [cosign](https://github.com/sigstore/cosign) can be used in the pod to sign and verify artifacts.
As a PoC, here we will exec into `signer-identity -n sigstore-test` pod and running the following:

```bash
oc exec -it signer-identity -n sigstore-test -- sh
```

Either use a cosign signing key or leverage fulcio's ability to generate ephemeral keys

### Option 1: Keyless, produce a signed certificate from fulcio and rekor log

```bash
cosign sign \
  --fulcio-url=$FULCIO_URL \
  --identity-token=$(cat $AWS_WEB_IDENTITY_TOKEN_FILE) \
  --rekor-url=$REKOR_URL quay.io/your/image
```

### Option 2: No Certificate, managed-key based signature with Rekor entry

```bash
cosign sign --rekor-url=$REKOR_URL --key /cosign/key/cosign.key quay.io/your/image
```

### Option 3: Provide self-managed key to Fulcio to issue a signed certificate

```bash
cosign sign \
  --fulcio-url=$FULCIO_URL \
  --identity-token=$(cat $AWS_WEB_IDENTITY_TOKEN_FILE) \
  --rekor-url=$REKOR_URL \
  --issue-certificate \
  --key /cosign/key/cosign.key quay.io/your/image
```

### Verify from local system

> **Note**
> If a self-managed key was passed to fulcio to issue a digital certificate, either of the following will verify the signing

```bash
# To verify against the cosign signing key (if not using keyless signing), extract the public key
oc extract --keys cosign.pub secret/cosign-keypair -n sigstore-test

# Verify with public key
cosign verify quay.io/your/image --key ./cosign.pub

# If a fulcio certificate was issued, either with ephemeral keys or a self-managed cosign-keypair
cosign verify quay.io/your/image \
  --certificate-oidc-issuer https://rh-oidc.s3.us-east-1.amazonaws.com/21o4ml1f5kchd6ee9nmh5dhc6efqba38 \
  --certificate-identity-regexp sigstore-test
```

TODO:
* Document plugging into a cluster build pipeline
* Integrate with a controller
