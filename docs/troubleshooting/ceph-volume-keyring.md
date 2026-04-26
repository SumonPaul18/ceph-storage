The primary issue is that ceph-volume cannot find the mandatory bootstrap keyring because you are likely running it outside of the managed container environment where those keys reside. Additionally, the ceph orch command you tried failed because the --encrypted flag is not a valid direct argument for that specific subcommand. [1, 2] 
To fix this and successfully create your encrypted OSD, follow these steps:
## 1. Fix the ceph-volume Keyring Error [3] 
If you must use ceph-volume directly, you need to provide the bootstrap-osd keyring from your cluster. [2] 

* Extract the keyring:

ceph auth get client.bootstrap-osd > /var/lib/ceph/bootstrap-osd/ceph.keyring

* Retry the command:

ceph-volume lvm create --data /dev/vdd --dmcrypt

[2, 4] 

## 2. Correct ceph orch Encryption Syntax [2] 
The ceph orch daemon add osd command does not support a simple --encrypted flag. To deploy an encrypted OSD via the orchestrator, you should use a Service Specification (YAML) which is the recommended way to handle advanced configurations like DM-Crypt. [1, 5, 6, 7] 

   1. Create a file named osd_spec.yaml:
   
   service_type: osdservice_id: encrypted_osdsplacement:
     hosts:
       - ceph1data_devices:
     paths:
       - /dev/vddencryption: true
   
   2. Apply the specification:
   
   ceph orch apply -i osd_spec.yaml
   
   
## Why your previous commands failed:

* ceph-volume: It failed with [errno 13] RADOS permission denied because /var/lib/ceph/bootstrap-osd/ceph.keyring was missing. Without this key, the tool cannot authenticate with the Monitors to request a new OSD ID.
* ceph orch: The error unused arguments: ['--encrypted'] confirms that the CLI parser for daemon add only expects raw|lvm methods and basic host:device paths. It does not have a shorthand flag for encryption. [1, 2] 

Pro-tip: You can also enable OSD encryption through the Ceph Dashboard by navigating to Cluster > OSDs > Create and checking the Encryption box. [8] 
Would you like the steps to wipe the disk first to ensure it's clean for the new OSD?

[1] [https://docs.ceph.com](https://docs.ceph.com/en/latest/cephadm/services/osd/)
[2] [https://www.tunbury.org](https://www.tunbury.org/2025/11/03/cepfs-partition-setup/)
[3] [https://forum.proxmox.com](https://forum.proxmox.com/threads/unable-to-create-ceph-osd.56501/)
[4] [https://access.redhat.com](https://access.redhat.com/solutions/7002457)
[5] [https://docs.ceph.com](https://docs.ceph.com/en/reef/cephadm/services/osd/)
[6] [https://docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/5/html/operations_guide/management-of-osds-using-the-ceph-orchestrator)
[7] [https://docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_ceph_storage/6/html/operations_guide/management-of-osds-using-the-ceph-orchestrator)
[8] [https://kifarunix.com](https://kifarunix.com/how-to-encrypt-data-at-rest-on-ceph-cluster-osd/)
