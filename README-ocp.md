### Sigstore on OCP with OCP root CA

Following this will launch a Sigstore stack in OpenShift 4.12
You must have cluster-admin privileges to deploy the Sigstore stack.
Documentation about the components that comprise a keyless Sigstore system
is available in the [sigstore documentation](https://docs.sigstore.dev/).

The [Sigstore helm-charts](https://github.com/sigstore/helm-charts) will be used to deploy Sigstore components in OpenShift.

#### Create CA and key in fulcio-system namespace

```bash
oc extract cm/kube-root-ca.crt -n openshift-config
# ca.crt will be in current directory
oc create ns fulcio-system
oc -n fulcio-system create secret generic fulcio-secret-rh --from-file=private=file_ca_key.pem --from-file=public=file_ca_pub.pem --from-file=cert=fulcio-root.pem  --from-literal=password=fulcio --dry-run=client -o yaml | kubectl apply -f-
```

Follow the [RHEL 9 Virtual Machine setup document](./vm_rhel9.md) to spin up a VM with Libvirt.

#### Clone the Sigstore Ansible repository

```bash
git clone git@github.com:sabre1041/sigstore-ansible.git
cd sigstore-ansible
```

#### Edit the inventory file

Save the following content to `inventory` file

```bash
[sigstore]
<VM_IP_ADDRESS>

[sigstore:vars]
ansible_password=<ROOT_PASSWORD_VM>
ansible_user=redhat
remote_user=root
become=true
become_user=root
```

#### Edit your local `/etc/hosts` file

Add the below content.

```bash
<VM IP ADDRESS> keycloak.sigstore-dev.ez
<VM_IP_ADDRESS> fulcio.sigstore-dev.ez fulcio
<VM_IP_ADDRESS> rekor.sigstore-dev.ez rekor
<VM_IP_ADDRESS> tuf.sigstore-dev.ez tuf
```

#### Run the Playbook

You should now head over to the [ansible-setup](https://github.com/sabre1041/sigstore-ansible/blob/main/README.md). Follow the README to
download the prerequisites and run the playbook. Also, the ansible-setup describes how to add the self-signed certificates to your local trust-store. 

As an example, this is the command used for the VM created above.

```bash
# Run the playbook from your local system
 ansible-playbook -vv -i inventory playbooks/install.yml -e base_hostname=sigstore-dev.ez -K
```

Finally, [cosign](https://github.com/sigstore/cosign) can be used locally to sign and verify artifacts.
