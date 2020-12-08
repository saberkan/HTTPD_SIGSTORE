Filtering external registry images by signature in OpenShift 4

Introduction
Running a cloud container platform may imply a dependency with external registries. Some images can be used as-is, and others as source images for build configurations.
In order to achieve this need, developers will need to pull images from external sources. But for security reasons, administrators should verify different points:
 Whitelist a valid set of registries defined as secure sources.
Ensure images are not tempered.
Authorise only a set of images.

In order to achieve these requirements, a signature validating mechanism exists under RHEL hosts and OpenShift platform. In this article we will explain the mechanism and how to implement it using a RHEL host and OpenShift platform. We will go through different points:
How to sign an image?
How to deploy a web server signature store (SigStore)?
How to implement signature under OpenShift platform?

Remark: Red Hat provides SigStores for quay.io, registry.redhat.io and registry.access.redhat.com with signatures of all Red Hat official images. There is no need to configure a custom sigstore,  in this case. The following procedure can be found in the following link:
https://access.redhat.com/verify-images-ocp4

Meanwhile, In this article we will be using docker.io registry to demonstrate the use case with an external registry where we don’t have control on all the images. Administrators can replace this by a local enterprise registry in the rest of the procedure. 

Signing images
Step 1 - Generate the gpg key
In order to sign images we need to use a gpg2 key mechanisms, the following procedure shows how to achieve this in a RHEL host:

First we need to generate the gpg key and trust it:

$ gpg2 --gen-key
...
Real name: saberkan
Email address: saberkan@redhat.com
You selected this USER-ID:
    "saberkan <saberkan@redhat.com>"

Change (N)ame, (E)mail, or (O)kay/(Q)uit? O
...
pub   rsa2048 2020-12-05 [SC] [expires: 2022-12-05]
      ID_XXXXXXXXX
uid                      saberkan <saberkan@redhat.com>
sub   rsa2048 2020-12-05 [E] [expires: 2022-12-05]

# find ID from last output
$ gpg2 --edit-key ID_XXXXXXXXX trust
  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y
...
$ gpg> quit

Then we need to export the gpg public key to be used by consumers (in our case OpenShift):

$ mkdir -p /etc/pki/developers/
$ gpg2 --armor --export saberkan@redhat.com > /etc/pki/developers/signer-key.pub
Step 2 - Pull the image locally, or build the image locally
Let’s pull an image locally and sign it before the system-wide registries configuration:
 
$ podman pull docker.io/saberkan/ubi-minimal:latest
Trying to pull docker.io/saberkan/ubi-minimal:latest...
Getting image source signatures
Copying blob aebb8c556853 skipped: already exists
Copying blob 0fd3b5213a9b skipped: already exists
Copying config 28095021e5 done
Writing manifest to image destination
Storing signatures
28095021e526ad1dd5a65e11dc0fe4b34999ec398dbc60743f4b121d6bc9fc81

Step 3 - Configure a local SigStore for a registry in my host
Modify policy to check signatures with a local SigStore:

$ cat <<EOF > /etc/containers/policy.json
{
    "default": [
        {
            "type": "insecureAcceptAnything"
        }
    ],
    "transports":
        {
            "docker": {
              "docker.io": [
                {
                  "type": "signedBy",
                  "keyType": "GPGKeys",
                  "keyPath": "/etc/pki/developers/signer-key.pub"
                }
              ]
            },
            "docker-daemon":
                {
                    "": [{"type":"insecureAcceptAnything"}]
                }
        }
}
EOF

Add following in /etc/containers/registries.d/default.yaml:

$ cat <<EOF > /etc/containers/registries.d/default.yaml
docker:
  docker.io:
    sigstore: file:///var/lib/containers/sigstore
    sigstore-staging: file:///var/lib/containers/sigstore
EOF

Step 4 - Push the image and sign it, then check the local signature

$ podman push docker.io/saberkan/ubi-minimal:latest --sign-by saberkan@redhat.com 
Getting image source signatures
Copying blob 3485805ce47c done
Copying blob b0e2911c99f3 done
Copying config 28095021e5 done
Writing manifest to image destination
Signing manifest
Storing signatures
$ tree /var/lib/containers/sigstore/
/var/lib/containers/sigstore/
└── saberkan
    └── ubi-minimal@sha256=f19c5b5d417cad1452ced0d174bca363ac41554190406c9147488b58394e2c56
        └── signature-1


Now only the signed image can be pulled, other images from the external registry cannot be pulled.
Pull the signed image:

$ podman pull docker.io/saberkan/ubi-minimal:latest
Trying to pull docker.io/saberkan/ubi-minimal:latest...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 438b5a18fd81 skipped: already exists
Copying blob 9d49c31ade4f skipped: already exists
Copying config 28095021e5 done
Writing manifest to image destination
Storing signatures
28095021e526ad1dd5a65e11dc0fe4b34999ec398dbc60743f4b121d6bc9fc81


Pull a not signed image:

$ podman pull docker.io/saberkan/s2i-tomcat7-jdk8-builder:latest
Trying to pull docker.io/saberkan/s2i-tomcat7-jdk8-builder:latest...
  A signature was required, but no signature exists
Error: error pulling image "docker.io/saberkan/s2i-tomcat7-jdk8-builder:latest": unable to pull docker.io/saberkan/s2i-tomcat7-jdk8-builder:latest: unable to pull image: Source image rejected: A signature was required, but no signature exists
Deploy a web server SigStore
Administrators can deploy a local http web server, in the same host that pushes and signs the image. The SigStore can be exposed then with an URL, and each time a signature occurs the SigStore is updated immediately.

It’s up to the administrator to choose the way of exposing the SigStore, in our case, it’s a simple httpd server where the sigstore was copied, and exposed under an URL: https://mywebserver/sigstore/

$ cp -r /var/lib/containers/sigstore /var/www/html/
Implementation on OpenShift 4
There are two different ways to configure /etc/containers/policy.json under openshift: 

Using machine config operator (MCO)
Using image.config.openshift.io/cluster CR
Meanwhile, the second option is limited and it only allows to limit the resources registries but doesn’t allow to configure SigStores.  Both options edit the same file /etc/containers/policy.json under OpenShift nodes. So, be aware to not use both MCO and image.config.openshift.io/cluster CR to whitelist registries with registrySources.allowedRegistries parameter, because the second configuration will erase the first one.

In order to achieve the configuration on OpenShift, we just need to configure MCO to use our SigStore for our registry, and limit the registries that can be used to pull images by users with image.config.openshift.io/cluster with allowedRegistriesForImport parameter, which doesn’t alter our policy.json file.

Remark: The following link describes the difference between registrySources.allowedRegistries and allowedRegistriesForImport parameters: :https://access.redhat.com/solutions/4931451

In our case, we will be using the following:

$ oc edit image.config.openshift.io/cluster
kind: Image
metadata:
  annotations:
  name: cluster
spec:
  allowedRegistriesForImport:
    - domainName: quay.io
    - domainName: registry.redhat.io
    - domainName: docker.io

We create configuration for the external registry, and convert everything including the gpg public key to base64 to be used with the machine configuration.

$ cat > policy.json <<EOF
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker": {
      "docker.io": [
        {
            "type": "signedBy",
            "keyType": "GPGKeys",
            "keyPath": "/etc/pki/developers/signer-key.pub"
        }
      ]
    },
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
EOF
$ cat <<EOF > docker.io.yaml
docker:
     docker.io:
         sigstore: https://mywebserver/sigstore/
EOF
$ export DOCKER_REG=$( cat docker.io.yaml | base64 -w0 )
$ export SIGNER_KEY=$(cat /etc/pki/developers/signer-key.pub | base64 -w0 )
$ export POLICY_CONFIG=$( cat policy.json | base64 -w0 )

Then we create and apply the machine configuration

$ cat > worker-custom-registry-trust.yaml <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: worker-custom-registry-trust
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${DOCKER_REG}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/containers/registries.d/docker.io.yaml
      - contents:
          source: data:text/plain;charset=utf-8;base64,${POLICY_CONFIG}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/containers/policy.json
      - contents:
          source: data:text/plain;charset=utf-8;base64,${SIGNER_KEY}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/pki/developers/signer-key.pub
  osImageURL: ""
EOF
$ oc apply -f worker-custom-registry-trust.yaml

