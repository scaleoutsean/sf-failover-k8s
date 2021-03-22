## What does sf-failover-k8s do?

It aims to provide a simple method to failover a Kubernetes cluster from one SolidFire cluster to another by redirecting iSCSI to a backup SolidFire cluster.

To do that it does the following:

1) Go through all the PVs in a Kubernetes cluster
2) Find only iSCSI vols
3) Find the volumes with PVs name on SolidFire cluster
4) Remove all volume pairs (sources of inbound replication) for these volumes
5) Change volume access mode from `failoverTarget` to `readWrite` for all these volumes
6) Update IQN and Target Portal for these PV and replaces PVs in Kubernetes

**NOTE:** this script **has not been tested** and it may not really work (in fact I am sure I can't make it work as-is). I would **not recommend** testing it out outside of a lab environment.

## Prerequisites

- All primary SolidFire Volumes (corresponiding to PVs in K8S) have a replicated volume on the secondary SolidFire with exactly the same name
- The secondary SolidFire cluster already has VAG defined and all volumes are already added to the corresponding VAG

## Post-failover steps

- After the script has completed, all pods must be restarted for new iSCSI connections to establish.

## How to use

- Clone repository

  `git clone https://github.com/kapilarora/sf-failover-k8s.git`

- Enter cloned repository
  
  `cd sf-failover-k8s`

- Setup your profile with Kubernetes and SolidFire details. Edit file named profile (remember to set `NO_EXECUTE=true` for a dry run)

- Create and activate (on subsequent only activate) a virtualenv:
  
  `virtualenv venv; source venv/bin/activate`

- Install dependencies

  `pip install -r requirements.txt`

- Setup Env variables using a profile file
  
  `source profile`

- Execute failover script:
 
  `python3 failover.py`

- Restart pods

- To failback, replicate SolidFire volumes to the original cluster and once they're in sync, stop pods, load the profile (just set new `SF_` variables) and run this script again

## Sample profile file

```raw
KUBECONFIG="/home/sean/.kube/config"
SF_IP="192.168.1.34"
SF_USERNAME="admin"
SF_PASSWORD="admin"
SF_TARGET_PORTAL="192.168.103.34"
NO_EXECUTE="true"
WAIT_TIME="60"
LOG_LEVEL="debug"
```

## Limitations

- The script is limited to exactly what it says it does in a simple "one Kubernetes, two SolidFire clusters" environment
- This script not designed to work in complex environments (for example, Kubernetes cluster connected to three SolidFire arrays)
- This script may requires the use of VAGs which latest Trident no longer recommends
  - Trident's SolidFire backend should be configured to use VAGs
  - If that is the case user must maintain VAGs if Trident's SolidFire backend does not use CHAP (please check the Trident documentation for VAG-related limitations)
- After switch-over, Trident backend configuration for SolidFire may need updating (in the case the new cluster has different cluster admin username or password, for example)

To make this script usable, we'd have to update it and test it with a Trident release v21.01 or better. Please submit results of your testing in Issues.
