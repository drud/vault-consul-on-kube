# Running Consul+Vault on Kubernetes

This process will bring up a 3-member consul cluster and a two vault servers running in an HA configuration.

Consul starter was from [Kelsey Hightower's Consul-on-Kubernetes](https://github.com/kelseyhightower/consul-on-kubernetes)
Thanks!

## Overview

A cluster of three [consul](https://github.com/hashicorp/consul) servers provides an HA back-end for two [vault](https://github.com/hashicorp/vault) servers.

Consul is not exposed outside the cluster. Vault is exposed on a
load-balancer service via https.

## What makes this work

- Services for each consul member and vault member
- Deployments for each (because they require some minor separate configuration)
- One service exposes the consul UI
- One load-balancer service exposes the vault servers to outside world

### Usage

* Clone this repo.
* Create services `kubectl apply -f services`

### Create Volumes

``` sh
$ kubectl apply -f ./pv
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

Create one deployment per consul member.

``` sh
$ kubectl apply -f deployments/consul-1.yaml -f deployments/consul-2.yaml -f deployments/consul-3.yaml
```

``` sh
$ kubectl get pods
```
``` sh
NAME                        READY     STATUS    RESTARTS   AGE
consul-1-3104874582-6o4n4   1/1       Running   0          1m
consul-2-3080431481-7mf6r   1/1       Running   0          1m
consul-3-3678840700-3bp3k   1/1       Running   0          1m
```


### Verification

``` sh
$ kubectl logs consul-1-3104874582-6o4n4
```
``` sh
==> WARNING: Expect Mode enabled, expecting 3 servers
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
           Version: 'v0.7.1'
         Node name: 'consul-1'
        Datacenter: 'us-west1-a'
            Server: true (bootstrap: false)
       Client Addr: 0.0.0.0 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 10.3.240.72 (LAN: 8301, WAN: 8302)
    Gossip encrypt: true, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

    2016/12/05 20:12:44 [INFO] raft: Initial configuration (index=0): []
    2016/12/05 20:12:44 [INFO] serf: EventMemberJoin: consul-1 10.3.240.72
    2016/12/05 20:12:44 [INFO] serf: EventMemberJoin: consul-1.us-west1-a 10.3.240.72
    2016/12/05 20:12:44 [INFO] raft: Node at 10.3.240.72:8300 [Follower] entering Follower state (Leader: "")
    2016/12/05 20:12:44 [INFO] consul: Adding LAN server consul-1 (Addr: tcp/10.3.240.72:8300) (DC: us-west1-a)
    2016/12/05 20:12:44 [INFO] consul: Adding WAN server consul-1.us-west1-a (Addr: tcp/10.3.240.72:8300) (DC: us-west1-a)
    2016/12/05 20:12:51 [ERR] agent: failed to sync remote state: No cluster leader
    2016/12/05 20:12:52 [WARN] raft: no known peers, aborting election
    2016/12/05 20:13:07 [ERR] agent: coordinate update error: No cluster leader
    2016/12/05 20:13:19 [ERR] agent: failed to sync remote state: No cluster leader
    2016/12/05 20:13:24 [INFO] serf: EventMemberJoin: consul-3 10.3.254.140
    2016/12/05 20:13:24 [INFO] consul: Adding LAN server consul-3 (Addr: tcp/10.3.254.140:8300) (DC: us-west1-a)
    2016/12/05 20:13:32 [INFO] serf: EventMemberJoin: consul-2 10.3.245.79
    2016/12/05 20:13:32 [INFO] consul: Adding LAN server consul-2 (Addr: tcp/10.3.245.79:8300) (DC: us-west1-a)
    2016/12/05 20:13:32 [INFO] consul: Existing Raft peers reported by consul-2, disabling bootstrap mode
    2016/12/05 20:13:33 [ERR] agent: coordinate update error: No cluster leader
    2016/12/05 20:13:38 [WARN] raft: Failed to get previous log: 1 log not found (last: 0)
    2016/12/05 20:13:38 [INFO] consul: New leader elected: consul-2
    2016/12/05 20:13:40 [INFO] agent: Synced service 'consul'
```

Log into consul-1:
``` sh
$ kubectl exec -it consul-1-117271-uw97q /bin/sh
```

Use the acl_master_token in your consul_config.json:
```
$ consul operator raft -list-peers -token=C4213989-B836-4A8F-A649-110803BCCDC3
Node      ID                 Address            State     Voter
consul-2  10.3.245.79:8300   10.3.245.79:8300   leader    true
consul-1  10.3.240.72:8300   10.3.240.72:8300   follower  true
consul-3  10.3.254.140:8300  10.3.254.140:8300  follower  true
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
key "vault/" { policy = "write" },
service "vault" {"policy"= "write"} 

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
You are now ready to deploy the vault instances:

``` sh
$ kubectl apply -f deployments/vault-1.yaml -f deployments/vault-2.yaml
```

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

$ vault write /secret/test1 value=1
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

Note that if a vault is sealed, its "READY" in `kubectl get po` will be 1/2, meaning
that although the logger container is ready, the vault container is not - it's not
considered ready until unsealed.
