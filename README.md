# Volume Snapshot Mover Performance Tester

This repository contains a basic golang script which enables a user to drive
the OADP data mover process as a separate operation from the Velero Backup
process. The intention is to simulate the performance results we will be able
to achieve in OADP 1.2 with the asynchronous plugin enhancement.

## Prerequisites
* Golang version 1.17+
* Modified OADP Installation with a custom data mover controller and a custom 
velero image

Example DPA:
```
    features:
      dataMover:
        enable: true
...
    unsupportedOverrides:
      veleroImageFqin: docker.io/dymurray/velero:nodelete
      dataMoverImageFqin: quay.io/konveyor/volume-snapshot-mover:perf
```

## Flags
This script supported customizable flags
* `namespaces` - This is a comma separated list of namespaces to include in the 
backup
* `restic-secret` - This is the name of the restic secret that gets created by 
the OADP operator when you enable the data mover. This contains the relevant 
volsync data to store the snapshots in s3. The name of this secret will be 
`<dpa-name>-volsync-restic`
  - *Note:* This is not the user-created data mover restic secret. This secret 
will be created by the operator in the OADP namespace.
* `kubeconfig` - Specify a path for a kubeconfig aside from the default one used 
by the current shell
* `concurrent` - Specifies the maximum number of running VolumeSnapshotBackups. 
Default is 12.

## Workflow

- Before starting, make sure all of the OADP data mover resources are cleaned from
any previous use:

- VolumeSnapshotBackups in the application namespace 
- ReplicationSource in the OADP namespace
- PVCs in the OADP namespace
- VolumeSnapshots in the application and OADP namespace 
- VolumeSnapshotContents

The script works by first creating a Velero backup setting `includedNamespaces`
to the comma separated list specified in the script arguments. This backup will
create a series of CSI snapshots for every PVC in those namespaces that is
provisioned by a CSI driver. The custom velero image used will preserve these
snapshots to be used by the volume snapshot mover.

Once the backup completes, we then will create a group of VolumeSnapshotBackups
limited to 12 (or whatever the `concurrent` flag is set to) at a time. These
VSBs will drive volsync to copy the snapshot into object storage.

When it completes, you can simply run `oc delete vsb --all -A` to clean up all
the resources created by the script.
