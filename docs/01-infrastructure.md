# Infrastructure Provisioning

Kubernetes can be installed just about anywhere Linux runs on. It could be either physical or virtual machines. In this lab we are going to focus on [RASPBIAN](https://www.raspberrypi.org/downloads/raspbian/).

This lab will walk you through provisioning the Raspberry Pi software required for running a H/A Kubernetes cluster. 

# DISCLAIMER
The steps in this tutorial are "AS IS" without any warranties and support.
I am not responsible for any misconfiguration or damages to the Raspberry Pi equipment involved on this tutorial.


# OS Configuration

The OS for each Raspberry Pi is [RASPBIAN JESSIE LITE 2017-03-02](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-03-03/2017-03-02-raspbian-jessie-lite.zip) This is a small image with a nice size OS perfect for small servers.

A total of 5 Raspberry Pis will be configured. Here are their names and IP addresses:

| Hostname      | IP address    |
|:-------------:|:-------------:|
| controller0   | 10.0.1.90     |
| controller1   | 10.0.1.91     |
| controller2   | 10.0.1.92     |
| worker0       | 10.0.1.93     |
| worker1       | 10.0.1.94     |


## Memory and Swap

This GPU configuration has worked well for previous server setups I've done.

```
sudo sh -c 'echo "gpu_mem=16" >> /boot/config.txt'
```

### SWAP (optional):
I add 1GB of swap space.
Some people seem concerned that frequent writes to the MicroSD card will make it fail quickly. I therefore decided to set the swappiness such that it only uses the swap as a last resort.

```
sudo mkdir /swap
sudo dd if=/dev/zero of=/swap/swapfile bs=1M count=1024
sudo mkswap /swap/swapfile
sudo swapon /swap/swapfile
```

```
sudo sh -c 'echo "/swap/swapfile none swap sw 0 0" >> /etc/fstab' 
```

```
sudo sh -c 'echo "vm.swappiness = 1" >>/etc/sysctl.conf'
```

## Hostnames to IP Addresses

In my exprience, connectivity between each server would be highly improved if connected directly to LAN network instead of Wi-Fi.

```
sudo sh -c "cat >>/etc/hosts <<EOF
10.0.1.94       controller0
10.0.1.95       controller1
10.0.1.96       controller2
10.0.1.97       worker0
10.0.1.98       worker1
EOF
"
```

> Remember to run these steps on `controller0`, `controller1`, `controller2`, `worker0`, and `worker1`
