# Ceph installation in isolated network, with cephadm

(work in progress)

Ceph version: v16.2.7

Example configuration, nodes:

- admin: 192.168.122.71 (to run admin commands and to host local registry)
- node01: 192.168.122.51
- node02: 192.168.122.52
- node03: 192.168.122.53
- node04: 192.168.122.54
- ...

## Download cephadm script and required images

With internet access...

```bash
wget https://raw.githubusercontent.com/ceph/ceph/v16.2.7/src/cephadm/cephadm
```

Get required images from `cephadm` source code:
```bash
cat cephadm | grep "_IMAGE ="
```

With internet access, pull docker images
`registry` image + required ceph images... and store them to files.

```bash
sudo docker pull registry:2
sudo docker pull quay.io/ceph/ceph:v16
sudo docker pull quay.io/prometheus/prometheus:v2.18.1
sudo docker pull quay.io/prometheus/node-exporter:v0.18.1
sudo docker pull quay.io/prometheus/alertmanager:v0.20.0
sudo docker pull quay.io/ceph/ceph-grafana:6.7.4
sudo docker pull docker.io/library/haproxy:2.3
sudo docker pull docker.io/arcts/keepalived
# check
sudo docker images
```

```bash
mkdir images
cd images

sudo docker save quay.io/ceph/ceph:v16 > ceph-v16.image
sudo docker save haproxy:2.3 > haproxy-2.3.image
sudo docker save registry:2 > registry-2.image
sudo docker save quay.io/ceph/ceph-grafana:6.7.4 > grafana-6.7.4.image
sudo docker save quay.io/prometheus/prometheus:v2.18.1 > prometheus-v2.18.1.image
sudo docker save quay.io/prometheus/alertmanager:v0.20.0 > alertmanager-v0.20.0.image
sudo docker save quay.io/prometheus/node-exporter:v0.18.1 > node-exporter-v0.18.1.image
sudo docker save arcts/keepalived > keepalived.image
# check
cd ..
find .
```

Transfer `cephadm` and all image files to admin node.

## setup admin node as local registry

```bash
sudo apt install -y docker.io
sudo docker load < registry-2.image
sudo docker run --privileged -d --name registry -p 5000:5000 -v /var/lib/registry:/var/lib/registry --restart=always registry:2
# check
sudo docker ps
```

Load images to local registry

```
sudo vi /etc/docker/daemon.json
{ "insecure-registries":["admin:5000"] }

sudo systemctl daemon-reload
sudo systemctl restart docker
```

```bash
sudo docker load < alertmanager-v0.20.0.image
sudo docker load < ceph-v16.image
sudo docker load < grafana-6.7.4.image
sudo docker load < haproxy-2.3.image
sudo docker load < keepalived.image
sudo docker load < node-exporter-v0.18.1.image
sudo docker load < prometheus-v2.18.1.image

sudo docker image tag quay.io/ceph/ceph:v16 admin:5000/ceph/ceph:v16
sudo docker image tag haproxy:2.3 admin:5000/haproxy:2.3
sudo docker image tag quay.io/ceph/ceph-grafana:6.7.4 admin:5000/ceph/ceph-grafana:6.7.4
sudo docker image tag quay.io/prometheus/prometheus:v2.18.1 admin:5000/prometheus/prometheus:v2.18.1
sudo docker image tag quay.io/prometheus/alertmanager:v0.20.0 admin:5000/prometheus/alertmanager:v0.20.0
sudo docker image tag quay.io/prometheus/node-exporter:v0.18.1 admin:5000/prometheus/node-exporter:v0.18.1
sudo docker image tag arcts/keepalived admin:5000/keepalived
# check
sudo docker images | grep admin

sudo docker push admin:5000/ceph/ceph:v16
sudo docker push admin:5000/haproxy:2.3
sudo docker push admin:5000/ceph/ceph-grafana:6.7.4
sudo docker push admin:5000/prometheus/prometheus:v2.18.1
sudo docker push admin:5000/prometheus/alertmanager:v0.20.0
sudo docker push admin:5000/prometheus/node-exporter:v0.18.1
sudo docker push admin:5000/keepalived
# check
curl admin:5000/v2/_catalog
curl admin:5000/v2/ceph/ceph/tags/list
curl admin:5000/v2/ceph/ceph-grafana/tags/list
# ...
```

## install cephadm

TODO...

Questions:

- Where does `cephadm` need to be installed, (admin, node01, ...)?

```bash
chmod +x cephadm
sudo ./cephadm install
# check
which cephadm
sudo cephadm --help
```

## configure ceph nodes

TODO...
This does not work yet.

On each node: node01, node02...
```bash
sudo apt install -y docker.io
sudo vi /etc/docker/daemon.json
# add... { "insecure-registries":["admin:5000"] }
sudo systemctl daemon-reload
sudo systemctl restart docker
```

```bash
sudo cephadm --docker --image "admin:5000/ceph/ceph:v16" bootstrap --skip-monitoring-stack --mon-ip 192.168.122.51
```

Questions:

- Is the bootstrap process executed on admin or on the first mon node (node01)?
- There is currently an error in both cases.

## problem on admin

```
ubuntu@admin:~$ sudo cephadm --docker --image "admin:5000/ceph/ceph:v16" bootstrap --skip-monitoring-stack --mon-ip 192.168.122.51
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chrony.service is enabled and running
Repeating the final host check...
systemctl is present
lvcreate is present
Unit chrony.service is enabled and running
Host looks OK
Cluster fsid: 24ca549c-5f19-11ec-a20a-251110235af5
Verifying IP 192.168.122.51 port 3300 ...
Traceback (most recent call last):
  File "/usr/sbin/cephadm", line 6242, in <module>
    r = args.func()
  File "/usr/sbin/cephadm", line 1451, in _default_image
    return func()
  File "/usr/sbin/cephadm", line 2923, in command_bootstrap
    check_ip_port(args.mon_ip, 3300)
  File "/usr/sbin/cephadm", line 771, in check_ip_port
    attempt_bind(s, ip, port)
  File "/usr/sbin/cephadm", line 731, in attempt_bind
    raise e
  File "/usr/sbin/cephadm", line 724, in attempt_bind
    s.bind((address, port))
OSError: [Errno 99] Cannot assign requested address
```

## problem on node01

```
ubuntu@node01:~/ceph_items$ sudo cephadm --docker --image "admin:5000/ceph/ceph:v16" bootstrap --skip-monitoring-stack --mon-ip 192.168.122.51
Creating directory /etc/ceph for ceph.conf
Verifying podman|docker is present...
Verifying lvm2 is present...
Verifying time synchronization is in place...
Unit chrony.service is enabled and running
Repeating the final host check...
systemctl is present
lvcreate is present
Unit chrony.service is enabled and running
Host looks OK
Cluster fsid: da017daa-5f18-11ec-a05c-37b574681fc7
Verifying IP 192.168.122.51 port 3300 ...
Verifying IP 192.168.122.51 port 6789 ...
Mon IP 192.168.122.51 is in CIDR network 192.168.122.0/24
Pulling container image admin:5000/ceph/ceph:v16...
Extracting ceph user uid/gid from container image...
Creating initial keys...
Creating initial monmap...
Creating mon...
Waiting for mon to start...
Waiting for mon...
mon is available
Assimilating anything we can from ceph.conf...
Generating new minimal ceph.conf...
Restarting the monitor...
Setting mon public_network...
Creating mgr...
Verifying port 9283 ...
Wrote keyring to /etc/ceph/ceph.client.admin.keyring
Wrote config to /etc/ceph/ceph.conf
Waiting for mgr to start...
Waiting for mgr...
mgr not available, waiting (1/10)...
mgr not available, waiting (2/10)...
mgr not available, waiting (3/10)...
mgr not available, waiting (4/10)...
mgr not available, waiting (5/10)...
mgr is available
Enabling cephadm module...
Waiting for the mgr to restart...
Waiting for Mgr epoch 4...
Mgr epoch 4 is available
Setting orchestrator backend to cephadm...
Generating ssh key...
Wrote public SSH key to to /etc/ceph/ceph.pub
Adding key to root@localhost's authorized_keys...
Adding host node01...
Non-zero exit code 22 from /usr/bin/docker run --rm --ipc=host --net=host --entrypoint /usr/bin/ceph -e CONTAINER_IMAGE=admin:5000/ceph/ceph:v16 -e NODE_NAME=node01 -v /var/log/ceph/da017daa-5f18-11ec-a05c-37b574681fc7:/var/log/ceph:z -v /tmp/ceph-tmph9jxliaz:/etc/ceph/ceph.client.admin.keyring:z -v /tmp/ceph-tmpsecch5kc:/etc/ceph/ceph.conf:z admin:5000/ceph/ceph:v16 orch host add node01
/usr/bin/ceph: stderr Error EINVAL: Can not automatically resolve ip address of host where active mgr is running. Please explicitly provide the address.
ERROR: Failed to add host <node01>: Failed command: /usr/bin/docker run --rm --ipc=host --net=host --entrypoint /usr/bin/ceph -e CONTAINER_IMAGE=admin:5000/ceph/ceph:v16 -e NODE_NAME=node01 -v /var/log/ceph/da017daa-5f18-11ec-a05c-37b574681fc7:/var/log/ceph:z -v /tmp/ceph-tmph9jxliaz:/etc/ceph/ceph.client.admin.keyring:z -v /tmp/ceph-tmpsecch5kc:/etc/ceph/ceph.conf:z admin:5000/ceph/ceph:v16 orch host add node01
```

