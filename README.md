# Running Consul and Vault(with Kata Containers) on Kubernetes

This process will bring up a 3-member consul cluster and a two vault servers running in an HA configuration. This is originally tested on GCE but can work in other public clouds such as AWS, Azure and private clouds using something like OpenStack.

This walkthrough is based off of [vault-consul-on-kube](https://github.com/drud/vault-consul-on-kube) (Thanks!) with minor modifications so that Vault works with [Kata Containers](https//katacontainers.io)

## Overview

A cluster of three [consul](https://github.com/hashicorp/consul) servers provides an HA back-end for two [vault](https://github.com/hashicorp/vault) servers.

The vault servers are running in [Kata Containers](https://katacontainers.io)

Consul is not exposed outside the cluster. Vault is exposed on a
load-balancer service via https.

Make sure you [install](https://github.com/kata-containers/documentation/blob/master/install/README.md) Kata Containers on all your Kuberentes nodes. Please note the Kata Containers works with [CRI-O](http://cri-o.io/) or the [Containerd CRI Shim](https://github.com/containerd/cri) (Not the dockershim yet)

## What makes this work

- Services for each consul member and vault member
- Deployments for each (because they require some minor separate configuration)
- One service exposes the consul UI
- One load-balancer service exposes the vault servers to outside world

### Usage

* Clone this repo.
* Create services `kubectl apply -f services`

### Create Volumes

* Use the size you deem appropriate

``` sh
$ gcloud compute disks create --size=50GB consul-1 consul-2 consul-3
```

### Create consul-config secret for consul configuration

Update example_config/consul_config.json to meet your needs:

`uuidgen` will create a new consul acl_master_token for you, which
you can plug into the consul_config.json.

`consul keygen` will generate the encryption
key for 'encrypt' in consul_config.json. You can utilize the Consul CLI locally for this.

``` sh
$ kubectl create secret generic consul-config --from-file=your_config/consul_config.json
```

### Consul Deployment

Create one deployment per consul member. The Consul version is 1.2.2 and this has not been tested with Kata Containers yet.

``` sh
$ kubectl apply -f deployments/consul-1.yaml -f deployments/consul-2.yaml -f deployments/consul-3.yaml
```

``` sh
$ kubectl get pods
```
``` sh
NAME                        READY     STATUS    RESTARTS   AGE
consul-1-9f84c7c9c-zq8c4    1/1       Running   0          1m
consul-2-7fcbb966cd-wghhg   1/1       Running   0          1m
consul-3-6fd4bc7f64-6r9fz   1/1       Running   0          1m
```


### Verification

``` sh
$ kubectl logs consul-1-9f84c7c9c-zq8c4
```
``` sh
WARNING: LAN keyring exists but -encrypt given, using keyring
WARNING: WAN keyring exists but -encrypt given, using keyring
bootstrap_expect > 0: expecting 3 servers
==> Starting Consul agent...
==> Consul agent running!
           Version: 'v1.2.2'
           Node ID: 'bef2c086-15c6-ec99-36a2-79a258adb9c5'
         Node name: 'consul-1'
        Datacenter: 'us-central1-c' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [0.0.0.0] (HTTP: 8500, HTTPS: -1, DNS: 8600)
      Cluster Addr: 10.106.237.83 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: true, TLS-Outgoing: false, TLS-Incoming: false

==> Log data will now stream in as it occurs:

    2018/08/19 23:30:14 [INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:bef2c086-15c6-ec99-36a2-79a258adb9c5 Address:10.106.237.83:8300} {Suffrage:Voter ID:31f9fc85-72ac-d5ca-b193-b559a2c322c2 Address:10.100.224.4:8300} {Suffrage:Voter ID:b570f248-ab94-de7d-a5c3-295cca24e6c6 Address:10.110.8.50:8300}]
    2018/08/19 23:30:14 [INFO] raft: Node at 10.106.237.83:8300 [Follower] entering Follower state (Leader: "")
    2018/08/19 23:30:14 [INFO] serf: EventMemberJoin: consul-1.us-central1-c 10.106.237.83
    2018/08/19 23:30:14 [INFO] serf: EventMemberJoin: consul-1 10.106.237.83
    2018/08/19 23:30:14 [INFO] serf: Attempting re-join to previously known node: consul-3.us-central1-c: 10.110.8.50:8302
    2018/08/19 23:30:14 [INFO] serf: Attempting re-join to previously known node: vault-2: 192.168.2.25:8301
    2018/08/19 23:30:14 [INFO] consul: Adding LAN server consul-1 (Addr: tcp/10.106.237.83:8300) (DC: us-central1-c)
    2018/08/19 23:30:14 [INFO] consul: Raft data found, disabling bootstrap mode
    2018/08/19 23:30:14 [INFO] consul: Handled member-join event for server "consul-1.us-central1-c" in area "wan"
    2018/08/19 23:30:14 [INFO] agent: Started DNS server 0.0.0.0:8600 (tcp)
    2018/08/19 23:30:14 [INFO] agent: Started DNS server 0.0.0.0:8600 (udp)
    2018/08/19 23:30:14 [INFO] agent: Started HTTP server on [::]:8500 (tcp)
    2018/08/19 23:30:14 [INFO] agent: started state syncer
    2018/08/19 23:30:14 [INFO] serf: EventMemberJoin: consul-3.us-central1-c 10.110.8.50
    2018/08/19 23:30:14 [INFO] serf: Re-joined to previously known node: consul-3.us-central1-c: 10.110.8.50:8302
    2018/08/19 23:30:14 [INFO] consul: Handled member-join event for server "consul-3.us-central1-c" in area "wan"
2018/08/19 23:30:21 [DEBUG] raft-net: 10.106.237.83:8300 accepted connection from: 192.168.5.19:44850
    2018/08/19 23:30:22 [ERR] agent: failed to sync remote state: No cluster leader
2018/08/19 23:30:22 [DEBUG] raft-net: 10.106.237.83:8300 accepted connection from: 192.168.5.19:44854
    2018/08/19 23:30:22 [INFO] serf: EventMemberJoin: consul-2.us-central1-c 10.100.224.4
    2018/08/19 23:30:22 [INFO] consul: Handled member-join event for server "consul-2.us-central1-c" in area "wan"
    2018/08/19 23:30:22 [INFO] serf: EventMemberJoin: consul-2 10.100.224.4
    2018/08/19 23:30:22 [INFO] consul: Adding LAN server consul-2 (Addr: tcp/10.100.224.4:8300) (DC: us-central1-c)
    2018/08/19 23:30:24 [INFO] serf: Attempting re-join to previously known node: vault-1: 192.168.4.23:8301
    2018/08/19 23:30:34 [INFO] serf: EventMemberJoin: consul-3 10.110.8.50
    2018/08/19 23:30:34 [INFO] consul: Adding LAN server consul-3 (Addr: tcp/10.110.8.50:8300) (DC: us-central1-c)
    2018/08/19 23:30:34 [INFO] serf: Attempting re-join to previously known node: consul-3: 10.110.8.50:8301
    2018/08/19 23:30:34 [INFO] serf: Re-joined to previously known node: consul-3: 10.110.8.50:8301
    2018/08/19 23:30:34 [INFO] consul: New leader elected: consul-3
    2018/08/19 23:30:35 [INFO] agent: Synced node info
```

Log into consul-1:
``` sh
$ kubectl exec -it consul-1-9f84c7c9c-zq8c4 /bin/sh
```

Use the acl_master_token in your consul_config.json:
```
consul operator raft list-peers -token=C4213989-B836-4A8F-A649-110803BCCDC3
Node      ID                                    Address             State     Voter  RaftProtocol
consul-1  bef2c086-15c6-ec99-36a2-79a258adb9c5  10.106.237.83:8300  follower  true   3
consul-2  31f9fc85-72ac-d5ca-b193-b559a2c322c2  10.100.224.4:8300   follower  true   3
consul-3  b570f248-ab94-de7d-a5c3-295cca24e6c6  10.110.8.50:8300    leader    true   3
```

### Create a key that vault will use to access consul (vault-consul-key)

We'll use the consul web UI to create this, which avoids all manner of
quote-escaping problems.

1. Port-forward port 8500 of <consul-1*> to local: `kubectl port-forward <consul-1*> 8500`
2. Hit http://localhost:8500/ui with browser.
3. Visit the settings page (gear icon) and enter your acl_master_token.
3. Click "ACL"
4. Add an ACL with name vault-token, type client, rules:
```
key "vault/" { 
  policy = "write"
}
service "vault" {
  "policy"= "write"
}

```
5. Capture the newly created vault-token and with it (example key here):
``` sh
$ kubectl create secret generic vault-consul-key --from-literal=consul-key=9f34ab90-965c-56c7-37e0-362da75bfad9
```

### Set the rules for the Anonymous Token

Still in the consul web ui, Hit http://localhost:8500/ui with browser

In ACL->Anonymous Token put defaults for anon token, allowing service
registration and locks:

```
key "lock/" {
  policy = "write"
}
service "" {
  policy = "write"
}
```

### TLS setup for exposed vault port

Get key and cert files for the domain vault will be exposed from. You can do this any way
that works for your deployment, including a [self-signed certificate](http://www.akadia.com/services/ssh_test_certificate.html), so long as you have a concatenated full certificate chain
in vaulttls.fullcert.pem and private key in vaulttls.key :

``` sh
$ kubectl create secret tls vaulttls --cert=vaulttls.fullcert.pem --key=vaulttls.key
```

### Provide DNS entry for the configured cert on external ip of the vault-lb service
You can run the following to determine the public IP address to use for your DNS record.

``` sh
$ kubectl get svc vault-lb
```

### Vault Deployment
This deploys Vault version 0.10.4. You are now ready to deploy the Vault instances:

``` sh
$ kubectl apply -f deployments/vault-1.yaml -f deployments/vault-2.yaml
```

You should see something like this:

``` sh
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
consul-1-9f84c7c9c-zq8c4    1/1       Running   0          24m
consul-2-7fcbb966cd-wghhg   1/1       Running   0          24m
consul-3-6fd4bc7f64-6r9fz   1/1       Running   0          24m
vault-1-6db7bbc6db-r4mdr    2/3       Running   0          21m
vault-2-db88cbcd-bj58b      2/3       Running   0          21m
```

Note that the "READY" status shows 2/3 because the vault is still
uninitialized and sealed.

### Vault Initialization

It's easiest to access the vault in its initial setup on the pod itself,
where HTTP port 9000 is exposed for access without https. You can decide
how many keys and the recovery threshold using args to `vault init`

``` sh
$ kubectl exec -it <vault-1*> /bin/sh

$ vault init
or
$ vault init -key-shares=1 -key-threshold=1

```

This provides the key(s) and initial auth token required.

Unseal with

``` sh
$ vault unseal
```
You should now see something like this:

``` sh
$ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
consul-1-9f84c7c9c-zq8c4    1/1       Running   0          26m
consul-2-7fcbb966cd-wghhg   1/1       Running   0          26m
consul-3-6fd4bc7f64-6r9fz   1/1       Running   0          26m
vault-1-6db7bbc6db-r4mdr    3/3       Running   0          23m
vault-2-db88cbcd-bj58b      3/3       Running   0          23m
```

(You should not generally use the form `vault unseal <key>` because it probably will leave traces of the key in shell history or elsewhere.)

and auth with
``` sh
$ vault auth
Token (will be hidden): <initial_root_token>
```

Then access <vault-2*> in the exact same way (`kubectl exec -it vault-2* /bin/sh`) and unseal it.
It will go into standby mode.

### Vault usage

On your local/client machine:

``` sh
$ export VAULT_ADDR=https://vault.example.com:8200
$ vault status
$ vault auth <root_or_other_token>

$ vault write /secret/test1 value=wsRsqs12$&
Success! Data written to: secret/test1

$ vault list /secret
Keys
----
junk
test1

$ vault read /secret/test1
Key             	Value
---             	-----
refresh_interval	768h0m0s
value           	1
```

### Vault failover testing

* Both vaults must be unsealed
* Restart active vault pod with kubectl delete pod <vault-1*>
* <vault-2*> should become leader "Mode: active"
* Unseal <vault-1*> - `vault status` will find it in "Mode: standby"
* Restart/kill <vault-2*> or kill the process
* <vault-1*> will become active

Note that if a vault is sealed, its "READY" in `kubectl get po` will be 2/3, meaning
that although the logger container is ready, the vault container is not - it's not
considered ready until unsealed.
