# External Load Balancer Config for etcd

**A sample config for pointing kube-apiserver to LB frontended etcd cluster**
<br><br>

This post follows from my [blog post located here](https://vrelevant.net/external-etcd-load-balancer-for-kubernetes/). For foundation on the etcd and self-signed certificate 
portions of this post, refer to [this post](https://vrelevant.net/install-etcd-cluster-with-tls/), and [this post](https://vrelevant.net/k8s-stacked-etcd-to-external-zero-downtime/).


I won't cover setting up Kubernetes or etcd clusters here, refer to the links above for that. Here I will cover configuring HAProxy, Keepalived, and the 
required certificates to frontend an etcd cluster with an L4 load balancer.

We can (and probably should) run the HAProxy and Keepalived on dedicated nodes, but I'll run them on the same nodes as my etcd endpoints to reduce the 
required infrastructure.

Perform these following steps on three etcd nodes:

### Install and Configure Keepalived (Each etcd node, or on dedicated nodes)

Keepalived manages a single virtual IP address between multiple nodes. This will be the IP address of our LB. If the active LB node becomes unavailable, 
the IP address will become active on another.

_As usual, this will be in Debian (Ubuuntu) format. Feel free to translate to RPM/YUM if that is your thing._

1. Choose an available IP address from your etcd node CIDR, this will be your LB address (I've used in 192.168.50.2)

2. Install Keepalived

```console
sudo apt install -y keepalived
```

3. Configure Keepalived

```console
export LB_IP=<available ip>
```

```console
export LB_IP_MASK=<e.g. 16>
```

```console
export INTERFACE=<Node active interface, e.g. ens34>
```

Leave below priority as 101 on the first node. On subsequent nodes, change priority to 100.

```console
cat <<EOF | sudo tee -a /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_ETCD
    enable_script_security
    max_auto_priority 100
}
vrrp_script check_haproxy {
  script "/usr/bin/pgrep haproxy"
  interval 3
  weight -2
  fall 2
  rise 2
}

vrrp_instance VI_1 {
    interface ${INTERFACE}
    virtual_router_id 53
    priority 101  #<<< Change this to 100 for subsequent nodes
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        ${LB_IP}/${LB_IP_MASK}
    }
    track_script {
        check_haproxy
    }
}
EOF
```

```console
sudo groupadd -r keepalived_script

sudo useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script

sudo chown -R keepalived_script:keepalived_script /etc/keepalived/

sudo chmod -R 664 /etc/keepalived/
```

### Install and Configure HAProxy (Each etcd node, or on dedicated nodes)

1. Install HAProxy

```console
sudo apt install -y haproxy
```

2. Set env vars for HAProxy config in step 3

```console
export ETCD1_IP=<IP of first etcd node>
```

```console
export ETCD2_IP=<IP of second etcd node>
```

```console
export ETCD3_IP=<IP of third etcd node>
```

3. Configure HAProxy

```console
sudo rm /etc/haproxy/haproxy.cfg
```

```console
cat <<EOF | sudo tee -a /etc/haproxy/haproxy.cfg
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot          /var/lib/haproxy
	stats socket    /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout   30s
	user            haproxy
	group           haproxy

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	                  global
	option	              httplog
	option	              dontlognull
        timeout connect       5000
        timeout client        50000
        timeout server        50000
	errorfile 400         /etc/haproxy/errors/400.http
	errorfile 403         /etc/haproxy/errors/403.http
	errorfile 408         /etc/haproxy/errors/408.http
	errorfile 500         /etc/haproxy/errors/500.http
	errorfile 502         /etc/haproxy/errors/502.http
	errorfile 503         /etc/haproxy/errors/503.http
	errorfile 504         /etc/haproxy/errors/504.http

frontend etcd-lb
    bind ${LB_IP}:2379
    mode tcp
    option tcplog
    default_backend etcd-server

backend etcd-server
    mode tcp
    balance roundrobin
      server etcd1 ${ETCD1_IP}:2379 check
      server etcd2 ${ETCD2_IP}:2379 check
      server etcd3 ${ETCD3_IP}:2379 check
EOF
```

4. Configure Ubuntu to allow HAProxy to start when the VIP is not bound to any interface

```console
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1
```

## Enable the HAProxy and Keepalived Services

```console
sudo systemctl enable haproxy --now
```

```console
sudo systemctl enable keepalived --now
```

```console
sudo systemctl status haproxy
```

```console
sudo systemctl status haproxy
```

## Conifugre etcd Client Certificate for LB

Because our kube-apiserver etcd client will be pointed to the LB, we need our etcd client TLS configured with certs that match the LB's IP. Follow 
[steps 1 - 6 here](https://github.com/n8sOrganization/Convert_K8s_Cluster_External_etcd/blob/main/README.md#create-external-etcd-cluster-certs). Use the 
LB IP address for the IP. Create a single cert/key pair only and copy to the directory storing those on each etcd node. Set the file permisssions appropriately.

Unlike in my previous post, where I used the same cert/key pair for client-to-node and peer-to-peer TLS, we'll use the LB cert/key pair for 
client-to-node TLS. We'll leave the peer-to-peer cert/key pair intact as the etcd nodes will continue to neogtiate TLS directly.

Your etcd TLS config will look something like the following. Be sure to update it on each eetcd node. Restart etcd one node at a time, allowing for it 
to come online before procedding with the next. 

Be aware that once you've completed this task, your kube-apiservers will not be able to communicate with the etcd cluster until their config has 
been updated to point to the LB address. I would suggest bringing the node that has the active Keepalived VIP online first, and then pointing the 
kube-apiserver configs to it. Then procedding to update the config on the other etcd node.

```console
ETCD_TRUSTED_CA_FILE="/etc/ssl/etcd/certificate/ca.crt"
ETCD_CERT_FILE="/etc/ssl/etcd/certificate/etcd-lb.pem"
ETCD_KEY_FILE="/etc/ssl/etcd/private/etcd-lb_key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/etc/ssl/etcd/certificate/ca.crt"
ETCD_PEER_CERT_FILE="/etc/ssl/etcd/certificate/etcd-1-dev_cert.pem"
ETCD_PEER_KEY_FILE="/etc/ssl/etcd/private/etcd-1-dev_key.pem"
ETCD_PEER_CLIENT_CERT_AUTH=true
```

## Update kube-apiserver Config

On your control plane nodes. edit the `/etc/kubernetes/manifests/kube-apiserver.yaml` and change the `--etcd-servers=https://192.168.50.2:2379` value 
to the IP of your LB.

That should do it. If all went well, you now have a three node HA LB configured for your etcd cluster. Go ahead and check your interface IPs and 
you'll see one of the nodes active with you LB IP. Test your connection to your etcd cluster and kube-apiserver. Stop the haproxy service on the node 
that currently has the VIP bound and check that your etcd and K8s access persists.
