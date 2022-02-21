# User creation in Kubernetes

This assumes you are a user which has the ability to create and approve signing requests.  These may be used to create users in your system.

You will need OpenSSL for these examples and scripts.

1. Create private key

    ```
    openssl genrsa -out alice.key 4096
    ```

    Do not share the key, only you should have it.

1. Create a CSR
    First, create the config:


     ```
     cat <<EOF > alice.csr.cnf
     [ req ]
     default_bits = 2048
     prompt = no
     default_md = sha256
     distinguished_name = dn
     [ dn ]
     CN = alice
     O = developers
     [ v3_ext ]
     authorityKeyIdentifier=keyid,issuer:always
     basicConstraints=CA:FALSE                
     keyUsage=keyEncipherment,dataEncipherment
     extendedKeyUsage=clientAuth
     EOF
     ```
 
    Now, use that config to create a signing request:

    ```
    openssl req -config alice.csr.cnf -new -key alice.key -nodes -out alice.csr
    ```

1.  Get and submit the CSR in a Kubernetes object

    Now, submit the CSR to Kubernetes.  We will need to
    approve the request before the certificate is created.

    Note that by default there is no expiration time.  This means
    so long as the user is bound to any roles, access will
    continue to be granted.

    It is better to set an expiration time.  Here, we set it to
    one day.

    ```
    req=$(cat alice.csr | base64 | tr -d '\n')

    cat <<EOF | kubectl apply -f -
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
      name: alice
    spec:
      request: $req
      signerName: kubernetes.io/kube-apiserver-client
      expirationSeconds: 86400
      usages:
      - client auth
    EOF
    ```

1.  Approve the request

    ```
    kubectl certificate approve alice
    ```

1.  Obtain the signed certificate

    ```
    kubectl get csr alice -o jsonpath='{.status.certificate}'| base64 -d > alice.crt
    ```

1.  Add it to kubectl configuration

    First, add the credentials

    ```
    kubectl config set-credentials alice --client-key=alice.key --client-certificate=alice.crt --embed-certs=true
    ```

    Then, create a context for the user to a specific cluster.
    Note that you need to replace `kubernetes` with an existing
    cluster name, or add a new one.

    ```
    kubectl config set-context alice --cluster=kubernetes --user=alice
    ```

1. Add any roles or rolebindings you wish for this user.

    By default, they won't be able to do anything useful.  You
    will need to add a `Role` or `ClusterRole`, and then either
    a `RoleBinding` or `ClusterRoleBinding` as needed.  Be careful
    with the cluster-wide ones, they are powerful.

    Roles and RoleBindings are namespaced.  Here, the default
    namespace is used.  The only permissions allowed to alice
    is to list pods in this one namesapce, by creating a
    role called `viewer` and then binding the user to that role.
    Adjust as needed.

    An identity can have many role bindings, granting many permissions.  Permissions are additive, so the identity
    will be granted access based on merging all applicable
    bindings and roles.

    ```
    kubectl create role viewer --verb=get --verb=list --resource=pods

    kubectl create rolebinding viewer-binding-alice --role=viewer --user=alice
    ```

# Using the new context

To use the newly added context, you can either set it as the
current context, or add `--context alice` to the kubectl
command line each time.

```
kubectl --context alice get pods
```

# Cleanup

As mentioned, unless an expiration is set on the CSR sent
to Kubernetes, the certificate lifetime will be long.  When
I created one in a Google cluster, it had a 5 year expiry.

The CSR can be deleted at any time without affecting the
usage of the certificate.

```
kubectl delete csr alice
```

To remove the access set up above, delete the role binding
and if it's not used by others, the role.

```
kubectl delete rolebinding viewer-binding-alice

kubectl delete role viewer
```

Lastly, if you are done with the certificate, delete it from your
kubectl config.

