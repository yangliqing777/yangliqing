[toc]

# Container networking

```
podman network create --interface-name slurm slurm
```

---



# SLURM

## Download

```bash
mkdir -p ${HOME}/scratch/slurm
time axel https://download.schedmd.com/slurm/slurm-25.05.2.tar.bz2 -o ${HOME}/scratch/slurm
```

## Start Container

```
elver=8

podman run --rm --tty --interactive --detach \
--replace --stop-timeout=0 \
--name slurms.build.r${elver} \
--hostname slurms-build-r${elver} \
--volume ${HOME}/scratch/slurm/r${elver}:/root/rpmbuild \
quay.io/rockylinux/rockylinux:${elver}-ubi \
sleep INFINITY

podman exec -ti slurms.build.r${elver} bash
```

## In-Container

```
time dnf install -y bash-completion yum-utils
time dnf install -y epel-release
time dnf install -y rpm-build

dnf config-manager --enable devel

cd ${HOME}/rpmbuild
tar xf slurm-25.05.2.tar.bz2

time dnf builddep ${HOME}/rpmbuild/slurm-25.05.2/slurm.spec -y

elver=8
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel${elver}/x86_64/cuda-rhel${elver}.repo
dnf install -y cuda-nvml-devel-12-9

time dnf install -y {dbus,freeipmi,gtk2,hdf5,http-parser,hwloc,json-c,libcurl,libjwt,librdkafka,libselinux,libyaml,lua,lz4,munge,mariadb,numactl,pam,pmix,rdma-core,rocm-smi,s2n-tls,ucx}-devel

time rpmbuild -ta ${HOME}/rpmbuild/slurm-25.05.2.tar.bz2 \
  --define "_with_nvml --with-nvml=/usr/local/cuda-12.9" \
  --with cray_shasta \
  --with slurmrestd \
  --with multiple_slurmd \
  --with pmix \
  --with ucx \
  --with hwloc \
  --with hdf5 \
  --with lua \
  --with numa \
  --with jwt \
  --with yaml \
  --with freeipmi \
  --with selinux \
  --with shared_libslurm \
  --with debug \
  --with munge \
  --with pam \
  --with x11
#  --with libcurl \
```

---



# Create Slurm images

## Installation

### Start shell

```bash
elver=8
BASE_IMAGE=mr${elver}

podman run --rm --tty --interactive --detach \
--replace --stop-timeout=0 \
--name slurms.install.r${elver} \
--hostname slurms-install-r${elver} \
--volume ${HOME}/scratch/slurm/r${elver}:/root/rpmbuild \
${BASE_IMAGE} \
sleep INFINITY

podman exec -ti slurms.install.r${elver} bash
```

### Repo file

```
cat >/etc/yum.repos.d/slurm.repo<<EndOfFile
[slurm]
name=slurm
baseurl=file:///root/rpmbuild/RPMS/x86_64
gpgcheck=0
enabled=1
EndOfFile
```

## Dockerfile

```dockerfile
mkdir -p /root/dockerfile/slurm
cd /root/dockerfile/slurm
tee Dockerfile << 'EndOfFile'
ARG rockyver="9"
FROM quay.io/rockylinux/rockylinux:${rockyver}-ubi-init

ARG SLURM_PACKAGES=""
ARG SLURM_SERVICES=""

RUN echo '[slurm]' > /etc/yum.repos.d/slurm.repo && \
    echo 'name=slurm' >> /etc/yum.repos.d/slurm.repo && \
    echo 'baseurl=file:///root/rpmbuild/RPMS/x86_64' >> /etc/yum.repos.d/slurm.repo && \
    echo 'gpgcheck=0' >> /etc/yum.repos.d/slurm.repo && \
    echo 'priority=1' >> /etc/yum.repos.d/slurm.repo && \
    echo 'enabled=1' >> /etc/yum.repos.d/slurm.repo

RUN dnf install -y epel-release yum-utils && crb enable || true
RUN dnf install -y epel-release yum-utils && dnf config-manager --enable powertools || true

RUN chmod 755 /etc
RUN groupadd -g 64030 slurm && \
    useradd -u 64030 -g 64030 -r -M -s /sbin/nologin slurm

RUN groupadd -g 888 munge && \
    useradd -u 888 -g 888 -r -M -s /sbin/nologin munge
    
RUN mkdir -p /var/{spool,lib,log}/slurm && \
    chown -R slurm:slurm /var/{spool,lib,log}/slurm

RUN if [ -n "$SLURM_PACKAGES" ]; then \
        dnf install -y $SLURM_PACKAGES && \
        dnf clean all && \
        rm -rf /var/cache/dnf; \
    fi
RUN rm -f /etc/yum.repos.d/slurm.repo

RUN echo SLURMD_OPTIONS="--conf-server ctld" > /etc/sysconfig/slurmd

RUN if [ -n "$SLURM_SERVICES" ]; then \
		for i in $SLURM_SERVICES; do systemctl enable $i; done \
    fi

EndOfFile
```

## Build

```bash
for elver in 9 8; do
for config in \
  "client:slurm slurm-sackd slurm-torque mpitests-openmpi:sackd munge" \
  "slurmctld:slurm-slurmctld slurm-example-configs:slurmctld munge" \
  "slurmd:slurm-slurmd mpitests-openmpi openssh-server:slurmd munge" \
  "slurmdbd:slurm-slurmdbd:slurmdbd munge" \
  "slurmrestd:slurm-slurmrestd:slurmrestd munge"
do
  IFS=':' read -r name packages services <<< "$config"
  podman build . \
    --volume ${HOME}/scratch/slurm/r${elver}:/root/rpmbuild \
    --build-arg rockyver="$elver" \
    --build-arg SLURM_PACKAGES="$packages" \
    --build-arg SLURM_SERVICES="$services" \
    -t "slurm-$name-r$elver"
done; done
```

---



# Munge key

```
elver=9
mkdir -p /root/scratch/slurm/r$elver/etc/munge

podman run --rm --tty --interactive \
--volume ${HOME}/scratch/slurm/r${elver}/etc/munge:/etc/munge \
slurm-slurmctld-r$elver \
create-munge-key -f
```

---



# Slurmctld

## Minimal configuration

```
elver=8
mkdir -p /root/scratch/slurm/r$elver/etc/slurm

tee /root/scratch/slurm/r$elver/etc/slurm/slurm.conf << 'EndOfFile'
SlurmctldParameters=enable_configless
ControlMachine=ctld
ClusterName=Rocky9
SlurmUser=slurm

SlurmdSpoolDir=/var/spool/slurm
StateSaveLocation=/var/lib/slurm
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmctldDebug=verbose
SlurmdDebug=verbose

SlurmctldPidFile=/var/run/slurm/slurmctld.pid
SlurmdPidFile=/var/run/slurm/slurmd.pid

SlurmSchedLogLevel=1
SlurmSchedLogFile=/var/log/slurm/slurmsched.log

GresTypes=gpu

AccountingStorageEnforce=1
AccountingStorageHost=slurmdbd

AccountingStorageType=accounting_storage/slurmdbd
AccountingStoreFlags=job_comment,job_env,job_extra,job_script
ProctrackType=proctrack/cgroup

SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

NodeName=cn[1-4] Boards=1 SocketsPerBoard=1 CoresPerSocket=16 ThreadsPerCore=2 RealMemory=64144 Gres=gpu:nvidia State=UNKNOWN
PartitionName=ordinary Nodes=ALL Default=YES MaxTime=INFINITE OverSubscribe=EXCLUSIVE State=UP

ReturnToService=2
TaskPlugin=task/affinity,task/cgroup
UsePAM=1
InactiveLimit=0
KillWait=3
MinJobAge=30
SlurmctldTimeout=12
SlurmdTimeout=30
Waittime=0
SchedulerType=sched/backfill

JobCompHost=slurmdb
JobCompPass=duangduang
JobCompType=jobcomp/mysql
JobCompUser=biubiu
JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/cgroup

EndOfFile
```

## Cgroup example config

```
elver=8
podman run --rm --tty --interactive --detach \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm:/data \
slurm-slurmctld-r$elver \
cp /etc/slurm/cgroup.conf.example /data/cgroup.conf
```

## gres

```bash
elver=8
tee /root/scratch/slurm/r$elver/etc/slurm/gres.conf << 'EndOfFile'
AutoDetect=nvidia
EndOfFile
```

## Start Slurmctld

```bash
elver=8

podman run --rm --tty --interactive --detach \
--network slurm \
--security-opt seccomp=unconfined \
--volume ${HOME}/scratch/slurm/r${elver}/etc/munge:/etc/munge \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm/slurm.conf:/etc/slurm/slurm.conf \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm/cgroup.conf:/etc/slurm/cgroup.conf \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm/gres.conf:/etc/slurm/gres.conf \
--replace --stop-timeout=0 \
--name slurmctld \
--hostname ctld \
slurm-slurmctld-r$elver

podman exec slurmctld sinfo
```

## Commands

```bash
podman exec -ti slurmctld bash \
-c '
scontrol reconfigure
sinfo
scontrol show nodes
systemctl restart slurmctld
scontrol update NodeName=cn[1-4] State=resume'
```

---



# Slurmd

## Configless

```bash
echo SLURMD_OPTIONS="--conf-server ctld" > /etc/sysconfig/slurmd
```

## Start Slurmd

```bash
elver=9

mkdir -p ${HOME}/scratch/slurm/scratch

for i in {1..4}; do
podman run --rm --tty --interactive --detach \
--network slurm \
--security-opt seccomp=unconfined \
--cap-add=AUDIT_WRITE \
--cap-add=DAC_READ_SEARCH \
--cap-add=NET_ADMIN \
--cap-add=SYS_ADMIN \
--cap-add=SYS_NICE \
--cap-add=SYS_PTRACE \
-v ${HOME}/.ssh:/root/.ssh:ro \
--volume ${HOME}/scratch/slurm/scratch:/scratch \
--volume ${HOME}/scratch/slurm/r${elver}/etc/munge:/etc/munge \
--replace --stop-timeout=0 \
--name slurmd-r$elver-cn$i \
--hostname cn$i \
slurm-slurmd-r$elver
podman exec -ti slurmd-r$elver-cn$i slurmd -D -C -G -vvv
done
```

---



# Slurm-client

## Minimal configuration

```
elver=8
mkdir -p /root/scratch/slurm/r$elver/etc/slurm

tee /root/scratch/slurm/r$elver/etc/slurm/slurm.login.conf << 'EndOfFile'
ControlMachine=ctld
ClusterName=Rocky9
SlurmUser=slurm

NodeName=cn[1-4] State=UNKNOWN
EndOfFile
```

## Start client

```bash
elver=8

podman run --rm --tty --interactive --detach \
--network slurm \
--security-opt seccomp=unconfined \
--cap-add=AUDIT_WRITE \
-v ${HOME}/.ssh:/root/.ssh:ro \
--volume ${HOME}/scratch/slurm/scratch:/scratch \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm/slurm.login.conf:/etc/slurm/slurm.conf \
--volume ${HOME}/scratch/slurm/r${elver}/etc/munge:/etc/munge \
--replace --stop-timeout=0 \
--name slurm \
--hostname slurm \
slurm-client-r$elver
```

## Commands

```bash
podman exec -ti slurm bash -c '
scontrol reconfigure
sinfo
scontrol show nodes
scontrol update NodeName=cn[1-4] State=resume'
```

---



# Slurmdbd

## Minimal configuration

```bash
elver=8
mkdir -p /root/scratch/slurm/r$elver/etc/slurm

tee /root/scratch/slurm/r$elver/etc/slurm/slurmdbd.conf << 'EndOfFile'
StorageType=accounting_storage/mysql
StorageHost=slurmdb
StoragePass=duangduang
StorageUser=biubiu

AuthType=auth/munge
DbdHost=slurmdbd
SlurmUser=slurm
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd/slurmdbd.pid

DebugLevel=verbose
EndOfFile

chmod 600 /root/scratch/slurm/r$elver/etc/slurm/slurmdbd.conf
chown 64030.64030 /root/scratch/slurm/r$elver/etc/slurm/slurmdbd.conf
```

## Start slurmdbd

```bash
elver=8

podman run --rm --tty --interactive --detach \
--network slurm \
--security-opt seccomp=unconfined \
--volume ${HOME}/scratch/slurm/r${elver}/etc/munge:/etc/munge \
--volume ${HOME}/scratch/slurm/r${elver}/etc/slurm/slurmdbd.conf:/etc/slurm/slurmdbd.conf \
--replace --stop-timeout=0 \
--name slurmdbd \
--hostname slurmdbd \
slurm-slurmdbd-r$elver
```

## Commands

```bash
podman exec -ti slurmdbd bash \
-c '
scontrol reconfigure
sinfo
scontrol show nodes
scontrol update NodeName=cn[1-4] State=resume'
podman exec -ti slurmdbd journalctl -ef
```

---



# Slurmdb

## Start database

```
mkdir -p /root/scratch/slurm/slurmdb/mariadb

podman run --rm --tty --interactive --detach \
--network slurm \
--volume /root/scratch/slurm/slurmdb/mariadb:/var/lib/mysql \
--replace --stop-timeout=0 \
--name slurmdb \
--hostname slurmdb \
--env MARIADB_USER=biubiu \
--env MARIADB_PASSWORD=duangduang \
--env MARIADB_ROOT_PASSWORD=bluebiu \
quay.io/mariadb-foundation/mariadb-devel
```

## Configure database

[1. Slurm简介 — Slurm资源管理与作业调度系统安装配置 2021-12 文档 (ustc.edu.cn)](http://hmli.ustc.edu.cn/doc/linux/slurm-install/slurm-install.html#mariadb-mysql)

```bash
podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -uroot -pbluebiu -e \
"create database slurm_acct_db"
podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -uroot -pbluebiu -e \
"create database slurm_jobcomp_db"

podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -uroot -pbluebiu -e \
"create user 'biubiu'@'localhost'"
podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -uroot -pbluebiu -e \
"grant all on *.* to 'biubiu'@'%' identified by 'duangduang' with grant option"

podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -uroot -pbluebiu -e \
"show databases"
podman run --network slurm --rm quay.io/mariadb-foundation/mariadb-devel mariadb -hslurmdb -ubiubiu -pduangduang -e \
"show databases"
```

---



# OpenMPI-mpitests

```bash
podman exec -ti slurm bash

tee /scratch/mpi.sh << 'EndOfFile'
#!/bin/sh
/usr/lib64/openmpi/bin/mpirun \
-mca plm_rsh_args "-p 22 -o StrictHostKeyChecking=no" \
-host cn1,cn2,cn3,cn4 \
-allow-run-as-root \
-map-by ppr:1:node:PE=2 -oversubscribe \
-report-bindings \
/usr/lib64/openmpi/bin/mpitests-osu_allreduce -x 1
EndOfFile

cd /scratch
sbatch -N 4 mpi.sh
```

---



# Dependency, Compile, Spec configure

```
elver=9

dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel${elver}/x86_64/cuda-rhel${elver}.repo
dnf install -y cuda-nvml-devel-12-9

time dnf install -y {dbus,freeipmi,gtk2,hdf5,http-parser,hwloc,json-c,libcurl,libjwt,librdkafka,libselinux,libyaml,lua,lz4,munge,mysql,numactl,pam,pmix,rdma-core,rocm-smi,s2n-tls,ucx}-devel

./configure \
--prefix=/root/rpmbuild/fakeroot \
--with-nvml=/usr/local/cuda-12.9 \
--enable-cgroupv2 \
--disable-silent-rules \
--enable-pkgconfig \
--enable-pam \
--enable-x11 \
--enable-sview \
--enable-glibtest \
--enable-gtktest \
--enable-optimizations \
--enable-salloc-kill-cmd \
--enable-slurmrestd \
--enable-multiple-slurmd \
--without-rpath \
--with-mysql_config \
--with-aix-soname \
--with-gnu-ld \
--without-shared-libslurm \
--with-json \
--with-jwt \
--with-http-parser \
--with-yaml \
--with-ofed \
--with-hdf5 \
--with-lz4 \
--with-hwloc \
--with-rsmi \
--with-pmix \
--with-freeipmi \
--with-ucx \
--with-rdkafka \
--with-s2n \
--with-bpf \
--with-lua \
--without-readline \
--with-munge \
--with-libcurl \
--without-oneapi \
--without-hpe-slingshot
```

