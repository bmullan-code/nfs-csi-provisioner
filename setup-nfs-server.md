# Setup VM Image as NFS Server

```
# /etc/exports (in this case sharing the directory /opt/sfw)
/opt/sfw *(insecure,rw,no_root_squash,sync,no_subtree_check)
```

```
exportfs -ra
systemctl start nfs-server.service
```

Test client side mount of share ... 

```
mkdir ~/sfw
mount -v -t nfs 192.168.1.84:/opt/sfw ~/sfw
```
