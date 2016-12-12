# Setting up a Certificate Authority and TLS Cert Generation

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates.

In this lab you will generate a single set of TLS certificates that can be used to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet

> In production you should strongly consider generating individual TLS certificates for each component.

The steps can be followed on one of the Raspberry Pis, then the certificates can be distributed to the others. I'm using **controller0** for the steps and then I copy the certificates to the rest of the cluster.

After completing this lab you should have the following TLS keys and certificates:

```
ca-key.pem
ca.pem
kubernetes-key.pem
kubernetes.pem
```

## Working Directory

```
mkdir -p $HOME/kubernetes
cd $HOME/kubernetes
```

## Install CFSSL

This lab requires the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-arm
chmod +x cfssl_linux-arm
sudo mv cfssl_linux-arm /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-arm
chmod +x cfssljson_linux-arm
sudo mv cfssljson_linux-arm /usr/local/bin/cfssljson
```

## Setting up a Certificate Authority

### Create the CA configuration file

```
echo '{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}' > ca-config.json
```

### Generate the CA certificate and private key

Create the CA CSR:

```
echo '{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}' > ca-csr.json
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.csr
ca.pem
```

### Verification

```
openssl x509 -in ca.pem -text -noout
```

## Generate the single Kubernetes TLS Cert

In this section we will generate a TLS certificate that will be valid for all Kubernetes components. This is being done for ease of use. In production you should strongly consider generating individual TLS certificates for each component. (But all replicas of a given component must share the same certificate.)

**Before creating the `kubernetes-csr.json` make sure both the hostnames and IP addresses match your environment.**

Create the `kubernetes-csr.json` file:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "controller0",
    "controller1",
    "controller2",
    "worker0",
    "worker1",
    "10.0.1.94",
    "10.0.1.95",
    "10.0.1.96",
    "10.0.1.97",
    "10.0.1.98",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the Kubernetes certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.csr
kubernetes.pem
```

You may see the following warning, but in my case it didn't cause any problems.
 
```
[WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```


### Verification

```
openssl x509 -in kubernetes.pem -text -noout
```

## Copy TLS Certs

Set the list of Kubernetes hosts where the certs should be copied to:

```
KUBERNETES_HOSTS=(controller1 controller2 worker0 worker1)
```

```
for host in ${KUBERNETES_HOSTS[*]}; do
  ssh ${host} "mkdir -p ~/kubernetes"
  scp ca.pem kubernetes-key.pem kubernetes.pem ${host}:~/dev/servers/kubernetes
done
```


