# Bootstrapping a H/A etcd cluster

In this lab you will bootstrap a 3 node etcd cluster. The following machines will be used:

* controller0
* controller1
* controller2

## Why

All Kubernetes components are stateless which greatly simplifies managing a Kubernetes cluster. All state is stored
in etcd, which is a database and must be treated specially. To limit the number of compute resource to complete this lab etcd is being installed on the Kubernetes controller nodes. In production environments etcd should be run on a dedicated set of machines for the 
following reasons:

* The etcd lifecycle is not tied to Kubernetes. We should be able to upgrade etcd independently of Kubernetes.
* Scaling out etcd is different than scaling out the Kubernetes Control Plane.
* Prevent other applications from taking up resources (CPU, Memory, I/O) required by etcd.

## Provision the etcd Cluster

Run the following commands on `controller0`, `controller1`, `controller2`:

### TLS Certificates

The TLS certificates created in the [Setting up a CA and TLS Cert Generation](02-certificate-authority.md) lab will be used to secure communication between the Kubernetes API server and the etcd cluster. The TLS certificates will also be used to limit access to the etcd cluster using TLS client authentication. Only clients with a TLS certificate signed by a trusted CA will be able to access the etcd cluster.

Copy the TLS certificates to the etcd configuration directory:

```
sudo mkdir -p /etc/etcd/
```

```
cd $HOME/kubernetes
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

### Download and Install the etcd binaries

At the time of this writing etcd for ARM was not supported and no downloads were available, so I had to build it from source on a of the Raspberry Pi.
To save you the time and trouble, I'm providing the version I built as part of the repository hosting this tutorial.

```
wget https://raw.githubusercontent.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/master/etcd/etcd-3.1.0-rc.1-arm.tar.gz
```

Extract and install the `etcd` server binary and the `etcdctl` command line client: 

```
tar -xvf etcd-3.1.0-rc.1-arm.tar.gz
```

```
sudo mv etcd-3.1.0-rc.1-arm/etcd* /usr/bin/
rm -rf etcd-3.1.0-rc.1-arm*
```

All etcd data is stored under the etcd data directory. In a production cluster the data directory should be backed by a persistent disk. Create the etcd data directory:

```
sudo mkdir -p /var/lib/etcd
```

The etcd server will be started and managed by systemd. Create the etcd systemd unit file:

Make sure the IP addresses in the **--initial-cluster** match your environment.

```
cat > etcd.service <<"EOF"
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm
Type=notify
ExecStart=/usr/bin/etcd --name ETCD_NAME \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --initial-advertise-peer-urls https://INTERNAL_IP:2380 \
  --listen-peer-urls https://INTERNAL_IP:2380 \
  --listen-client-urls https://INTERNAL_IP:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://INTERNAL_IP:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller0=https://10.0.1.94:2380,controller1=https://10.0.1.95:2380,controller2=https://10.0.1.96:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

The ARM architecture is currently not supported, so we need to tell etcd that we want to use an unsupported architecture.
This is done by providing the environment variable **ETCD_UNSUPPORTED_ARCH=arm**


### Set The Internal IP Address

The internal IP address will be used by etcd to serve client requests and communicate with other etcd peers.

Each etcd member must have a unique name within an etcd cluster. Set the etcd name:

```
ETCD_NAME=$(hostname)

INTERNAL_IP=$(echo "$(ifconfig eth0 | awk '/\<inet addr\>/ { print substr( $2, 6)}')")

```
Notice that I'm using **eth0** or LAN connection to find the IP Address to replace in the Unit file. If you using Wi-Fi connection, you will probably need to change this to **wlan0**
The following command will display the interfaces with their IP Addresses:

```
ip a
```

Substitute the etcd name and internal IP address:

```
sed -i s/INTERNAL_IP/${INTERNAL_IP}/g etcd.service
```

```
sed -i s/ETCD_NAME/${ETCD_NAME}/g etcd.service
```

Once the etcd systemd unit file is ready, move it to the systemd system directory:

```
sudo mv etcd.service /etc/systemd/system/
```

Start the etcd server:

```
sudo systemctl daemon-reload
```
```
sudo systemctl enable etcd
```
```
sudo systemctl start etcd
```


### Verification

```
sudo systemctl status etcd --no-pager
```

> Remember to run these steps on `controller0`, `controller1`, and `controller2`

## Verification

Once all 3 etcd nodes have been bootstrapped verify the etcd cluster is healthy:

* On one of the controller nodes run the following command:

```
etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
```

At first all cluster members were reporting unhealthy for some reason. It took a while (10+ minutes) for them to become healthy.
In any event, I was able to continue with the installation of Kubernetes just fine.


```
member 4df25a7f2ac49e88 is healthy: got healthy result from https://10.0.1.94:2379
member 94230559abaa2df2 is healthy: got healthy result from https://10.0.1.95:2379
member f6ae1308cea6198a is healthy: got healthy result from https://10.0.1.96:2379
cluster is healthy
```
