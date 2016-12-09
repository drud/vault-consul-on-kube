# Vault/Consul recovery playbook

## What you must have to start with:

* Vault Unseal key(s)
* A root token
* A consul snapshot file (in the case of complete loss)

## Background

Vault is simply a front-end to consul as a storage mechanism. It has no state of its own.

When a vault server has access to a working consul cluster, it can be unsealed and access secrets.

The Consul cluster is a set of 3 servers that sync the full set of information among themselves. They can continue functioning as long as they have a quorum (2 servers) but cease to function otherwise.

#### Basic useful skills, diagnostics, resources

* Familiarity with kubectl "get" and "logs"
* `kubectl proxy` and explore the cluster using http://localhost:8001
* `kubectl port-forward consul-X* 8500` and use the consul web ui at http://localhost:8500/ui
* Know how to unseal and auth on the vault servers
* `kubectl exec -it consul-X* /bin/sh` and use the consul command-line tool.
* Using the Google Logs/Stackdriver logs UI to filter and review container logs
* [Consul outage recovery](https://www.consul.io/docs/guides/outage.html), [consul snapshot](https://www.consul.io/docs/commands/snapshot.html), [consul docs](https://www.consul.io/docs/index.html), [vault docs](https://www.vaultproject.io/docs/)

## Scenario: Vault is sealed on both vault servers

In this situation, both vault pods will show as unready in
`kubectl get po -l app=vault`
and
`vault status `
will show "Mode: sealed"

#### Response: Unseal the vault servers

```
$ kubectl -it vault-1<*> /bin/sh
$ vault unseal
Key (will be hidden): <unseal key>
$ vault unseal
Key (will be hidden): <key 2>
...
```

Repeat same process on vault-2*

This is done on the pod itself because you otherwise might be hitting the load balancer and landing at different vault servers each time you add an unseal key.

#### Success indication

If you have been successful, one of the vault pods will now show "ready" on `kubectl get po -l app=vault`, `vault status` using the external load balancer ip/dns should show active, and if you auth you should be able to access keys.

#### Why this might happen

When a vault server is recreated, it comes up sealed. Each server which may be destined for service must be manually unsealed by someone with the unseal keys. So any failure of nodes or containers can result in this problem.

## Scenario: One or more vault or consul servers is being restarted or is not coming up to ready state

#### Response: Look at logs for deployment/replicaset/pod and determine what's happening

See previous logs: `kubectl logs -p vault-1-*`

Explore replicaset and deployment for problems:
`kubectl proxy` and then explore each at http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

Note that there will always be just one vault server in ready state, because only one vault server can be active.

Suggestions:

* Recreate the deployment, for example `kubectl apply -f deployments/consul-1.yaml` or `kubectl delete deployment consul-1 && kubectl create -f deployments consul-1.yaml`

#### Success indication

All servers came up, vault servers can be unsealed.


## Scenario: Consul servers are unable to elect a leader

In this situation, you see in the logs lots of negotiation, but no leader. It is most likely a result of them not being successful in talking with each other.

You will most likely need to review the logs to try to figure out what has happened.

If the pods don't show "ready", then the services that route traffic to them won't be routing it, and so they can't talk their gossip.

You may have to rebuild a new app if you can't figure it out.

## Scenario: Complete loss of consul cluster (but consul volumes are intact)

In this situation, due to loss of all nodes, or due to some other event, we have the consul-1, consul-2 and consul-3 disk volumes, but nothing else.

We can either use the existing disk volumes, or start with them and then load a snapshot. The snapshot guarantees consistency at a point in time, the disk volumes might not.

#### Response: Bring up a new set of consul servers using the same configuration file

If your existing Kubernetes services and secrets (consul-config and vault-consul-key) remain in place, you can just use them. Otherwise, you'll need to recreate them from scratch using the README.md. If your services do not have the same IP addresses they did before, recovery will be more complex, probably in line with total loss instructions.


Consul servers and cluster membership are tightly coupled to IP addresses, so if you do not have the original services available, the recovery is more complex.

This process assumes that the services and secrets already exist - otherwise they need to be created using the technique in the full catastrophic loss restoration process.

Prerequisites:

1. `kubectl get service` shows all the appropriate services, with IP addresses.
2. `kubectl get secret` shows the consul-config, vault-consul-key, and vaulttls secrets

Process:

1. Bring up the consul servers with `kubectl apply -f deployments/consul-1.yaml -f deployments/consul-2.yaml -f deployments/consul-3.yaml`
2. Watch the resulting logs using `kubectl logs -f consul-1*`
3. If you see a leader elected, and you see familiar secret keys under Key/Value->Vault->Logical-><Hash>-> then things are close to working.
4. Check status on the vault servers and unseal them.

#### Success indication

All servers came up, vault servers can be unsealed.

## Scenario: Loss of a single server (probably loss of its disk)

In this situation, the consul cluster is functional but fragile, as one of the 3 servers is gone.

It's discussed under "Failure of a Server in a Multi-Server Cluster" on https://www.consul.io/docs/guides/outage.html - It's possible that some of the advanced techniques mentioned there would need to be done.

Demonstrate this with:
1. kill -9 the consul process on <consul-3*> and immediately delete the consul-3 deployment
2. Delete and recreate the consul-3 disk

#### Response: Recreate the server

1. `kubectl create -f deployments/<consul-?>.yaml`
2. `kubectl logs -f deployments/<consul-?>.yaml`

#### Success indication

* The server should be accepted back into the cluster.
* Vault servers can be unsealed and secrets read

## Scenario: Millions of logfile complaints about not being able to reach a consul server by UDP

This problem is apparently a Linux/docker bug which is triggered when the consul container restarts. See [docker issue](https://github.com/docker/docker/issues/8795). It does not seem to affect cluster functionality, but you'll have to filter logs to see anything but this one log.

```
[WARN] memberlist: Was able to reach consul-3 via TCP but not UDP, network may be misconfigured and not allowing bidirectional UDP
```

#### Response: UGLY and involves downtime: Delete the deployments, reboot the nodes

1. Capture a consul snapshot and transfer it to local.
2. Delete consul deployments: `kubectl delete deployment consul-1 consul-2 consul-3`
3. Reboot (one at a time preferably) each node in cluster (`kubectl get no`) (can be done with web UI or gcloud compute ssh): `gcloud compute ssh gke-vault-freshstart-default-pool-c003fbba-fnhl -- sudo reboot`
4. Recreate the consul deployments: `kubectl apply -f deployments/consul-1.yaml -f deployments/consul-2.yaml -f deployments/consul-3.yaml`
5. After things have come back, unseal the vault servers

#### Success indication

* Ability to unseal vault servers
* No more nasty logs.


## Complete loss and rebuild with recovery using a consul snapshot

In this scenario, all disks, deployments, and services have been lost, and we need to restore a snapshot to a brand-new environment.

1. Follow the full README.md process to set up the clusters.
2. Get the consul snapshot you need up to a consul server. We'll use consul-1: `uuencode fullsnap201612071509.snap fullsnap201612071509.snap | kubectl exec -it consul-1-3058537447-fnlt1 uudecode`
3. On consul-1* find the acl_master_token in /etc/consul/consul_config.json and use it to load the snapshot: `consul snapshot restore -token=8F2383EF-5199-4ED8-B20C-EF34D23FF109 fullsnap201612071509.snap`
4. Unseal the vault servers.
5. Go forth and prosper. Don't get in this situation again :)
