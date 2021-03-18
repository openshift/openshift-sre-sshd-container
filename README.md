# sre-ssh-proxy-container
This is a container that runs a _non-interactive_, unprivileged [OpenSSH](https://www.openssh.com/) daemon; primarily intended for tunneling over SSH to reach a [Kubernetes](https://kubernetes.io/) or [OpenShift](https://www.openshift.com/) cluster's [API server](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) in a context where the API server is not directly accessible.

The client can either [establish a SOCKS proxy with ssh](https://man.openbsd.org/ssh#D) and use the proxy to forward `kubectl` or `oc` commands by way of an `HTTPS_PROXY` environment variable, or directly configure the client host's route tables with a tool like [sshuttle](https://sshuttle.readthedocs.io/).

## Development

To build a local image: `make`

To push the image to a remote repository: `make push`

Use environment variables to override the default repository and image name:
- `IMAGE_REGISTRY` (default is `"quay.io"`)
- `IMAGE_REPOSITORY` (default is `"openshift-sre"`)
- `IMAGE_NAME` (default is `"sre-ssh-proxy"`)

To run the image locally: `make local`

This runs the OpenSSH daemon in a container, publishes its listening port to host port 2222, and mounts both a temporary RSA host key and your own `~/.ssh/authorized_keys` file.

## Usage

The container's entry point is `/opt/start-sshd.sh` and the OpenSSH daemon listens on port 2222<sup>[1](#footnote1)</sup>.  The startup script generates a single user named `sre-user`<sup>[1](#footnote1)</sup>.  The container requires at least one private host key file to be mounted, as well as a mounted directory for authorized keys files.  These locations are communicated to the container through environment variables:

#### SSH_HOST_*_KEY

Environment variable names of the form `SSH_HOST_*_KEY` (e.g. `SSH_HOST_RSA_KEY`) must point to the location of a private host key file within the container.  The startup script looks for these environment variables and adds a [HostKey](https://man.openbsd.org/sshd_config#HostKey) directive for each found variable to the [OpenSSH daemon's configuration file](https://man.openbsd.org/sshd_config) (e.g. `HostKey $SSH_HOST_RSA_KEY`).  The OpenSSH daemon requires at least one valid host key to start.

#### AUTHORIZED_KEYS_DIR

This variable must point to the directory within the container where authorized keys files are mounted.  The startup script verifies that this is a valid directory path before launching the OpenSSH daemon.

Note that multiple authorized keys files are allowed, even in subdirectories of `AUTHORIZED_KEYS_DIR`.  The OpenSSH daemon uses the [AuthorizedKeysCommand](https://man.openbsd.org/sshd_config#AuthorizedKeysCommand) directive instead of [AuthorizedKeysFile](https://man.openbsd.org/sshd_config#AuthorizedKeysFile).  The command, running as `sre-user`, recursively searches for and prints to standard output all regular files under `AUTHORIZED_KEYS_DIR` (with duplicate keys removed).

All files under `AUTHORIZED_KEYS_DIR` must adhere to the [authorized_keys file format](https://man.openbsd.org/sshd.8#AUTHORIZED_KEYS_FILE_FORMAT).

#### KEY SIGNING

The `signkey` command is used to generate a certificate for an authorized key signed by the cluster's csr-signer certificate, intended to be used for client auth when other authentication methods are unavailable.

When a user runs `signkey`, their corresponding authorized key is extracted from the `SSH_USER_AUTH` environment variable and compared against a list of `authorized_keys` files to determine group membership.

If a matching group is found, a certificate is generated with the authorized key fingerprint and group membership included in the DN.

By default, a new private key is generated and included in the output along with the certificate, which can be used for auth like so:

```
ssh sre-user@$HOST signkey > tls.pem
oc config set-credentials sre-user --client-certificate=tls.pem --client-key=tls.pem
```
To avoid generating a private key server-side, the user should send an optional `SIGN_KEY` environment variable (`ssh -o SendEnv=SIGN_KEY`) containing a base64 raw-encoded public key PEM block. An example script is included demonstrating this process.

To prevent unauthorized certificate generation via `oc exec` or similar, an agent must be forwarded for either of the above options to work. Agent-less authentication is only possible if `SIGN_KEY` matches the authorized key.

#### ELEVATION

Group membership cannot be changed when using x509 auth, so use impersonation to temporarily elevate:

```
oc --as backplane-cluster-admin ...
```

<sup><a name="footnote1">1</a> **TODO:** Consider making the port number and user name configurable through additional environment variables.</sup>
