# nfs-csi-provisioner

Describes how to setup the nfs csi provisioner in tkgs to provision PV & PVC using an existing NFS server share. 

Background doc
https://docs.vmware.com/en/VMware-Telco-Cloud-Platform---5G-Edition/1.0/telco-cloud-platform-5G-edition-reference-architecture-guide-10/GUID-0D85BBFA-37F4-49C3-8943-13F9B0BA278C.html

There was the recommendation " If an NSF server exists, use the CSI NFS Provisioner instead.
 
To setup the CSI NFS Provisioner we followed the instructions on this page https://github.com/kubernetes-csi/csi-driver-nfs


Install the driver:
```
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/install-driver.sh | bash -s master --
```

Check the status of the driver (make sure the pods are running)
```
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
```

We then created an example PV and PVC

PV
https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/pv-nfs-csi.yaml

PVC
https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/pvc-nfs-csi-static.yaml

For the PV we modified the volumeAttributes server and share to point to our server and share path. ie. 
```
csi:
    driver: nfs.csi.k8s.io
    readOnly: false
    volumeHandle: unique-volumeid  # make sure it's a unique id in the cluster
    volumeAttributes:
      server: 192.168.1.84
      share: /opt/sfw
```
Check the status of the PV, PVC
```
kubectl get pv,pvc | grep nfs

persistentvolume/pv-nfs       10Gi       RWX            Retain           Bound     default/pvc-nfs-static                                45m
persistentvolumeclaim/pvc-nfs-static                                     Bound     pv-nfs                                     10Gi       RWX
```

Finally we created a pod which uses the PVC

```
apiVersion: v1
kind: Pod
metadata:
  name: task1-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: pvc-nfs-static
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

and then exec'd a shell into the pod to verify we can access the volume, and read/write files in the volume share. We also repeated this with a second pod to verify the ReadWriteMany access mode.

```
kubectl exec --stdin --tty task1-pv-pod -- /bin/bash

root@task-pv-pod:/# cd /usr/share/nginx/html/

root@task-pv-pod:/usr/share/nginx/html# ls
abc  foo.txt  xyz
root@task-pv-pod:/usr/share/nginx/html# date > abc
root@task-pv-pod:/usr/share/nginx/html# cat abc
Thu Mar 25 15:25:20 UTC 2021
root@task-pv-pod:/usr/share/nginx/html#
```
