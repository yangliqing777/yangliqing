# NCCL操作记录

## 装cuda

```bash
# dnf module reset nvidia-driver -y
# dnf module enable nvidia-driver:560-open -y
# dnf install nvidia-driver -y
time dnf install @nvidia-driver:560-open -y
# real	2m20.820s
time dnf install cuda -y

# cat /etc/dnf/modules.d/nvidia-driver.module 
[nvidia-driver]
name=nvidia-driver
stream=560-open
profiles=
state=enabled
```

## 获取代码

```bash
WDIR=${HOME}
WDIR=/root/nccl
mkdir ${WDIR}/github -p
time for i in \
https://github.com/nvidia/nccl \
https://github.com/nvidia/nccl-tests \
https://github.com/Mellanox/nccl-rdma-sharp-plugins;
do
time git -C ${WDIR}/github clone $i
done

wget https://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-7.5.tar.gz
```

编译可执行文件

```bash
WDIR=${HOME}
WDIR=/root/nccl
time make -C ${WDIR}/github/nccl \
-j src.build CUDA_HOME=/usr/local/cuda-12.6
# real	3m14.621s
```

```bash
WDIR=${HOME}
WDIR=/root/nccl
time make -C ${WDIR}/github/nccl-tests -j \
MPI=1 MPI_HOME=/usr/mpi/gcc/openmpi-4.1.7a1 \
CUDA_HOME=/usr/local/cuda-12.6 \
NCCL_HOME=${WDIR}/github/nccl/build
# real	0m42.637s
```

```bash
WDIR=${HOME}
WDIR=/root/nccl
cd ${WDIR}/github/nccl-rdma-sharp-plugins
sh autogen.sh
./configure \
--with-sharp=/opt/mellanox/sharp \
--with-cuda=/usr/local/cuda-12.6
time make -j
```

OSU-Benchmark

```bash
WDIR=${HOME}
WDIR=/nfs/pengzhi/sharp
./configure \
--enable-cuda=yes --with-cuda=/usr/local/cuda-12.6 \
CC=/usr/mpi/gcc/openmpi-4.1.7a1/bin/mpicc \
CXX=/usr/mpi/gcc/openmpi-4.1.7a1/bin/mpicxx
```



## SHARP加速

### OpenSM + SHARP

```bash
/usr/sbin/opensm --priority 15
/opt/mellanox/sharp/bin/sharp_am --log_verbosity 4 --smx_enabled_protocols 5 --smx_protocol 1
```

```bash
/opt/mellanox/sharp/bin/sharp_hello -d mlx5_0:1
```

### NCCL-Tests

```
WDIR=${HOME}
WDIR=/root/nccl
/usr/mpi/gcc/openmpi-4.1.7a1/bin/mpirun \
-host g1,g2,g3,g4 -allow-run-as-root \
-map-by ppr:8:node -oversubscribe \
-x NCCL_SOCKET_IFNAME=eno1 \
-x NCCL_DEBUG=INFO \
-x NCCL_NET_GDR_LEVEL=PIX \
-x NCCL_COLLNET_ENABLE=0 \
-x NCCL_ALGO=CollnetChain \
-x UCX_NET_DEVICES=mlx5_0:1,mlx5_4:1 \
-x NCCL_IB_HCA=mlx5_0,mlx5_4 \
-x LD_LIBRARY_PATH=${WDIR}/github/nccl/build/lib:${WDIR}/github/nccl-rdma-sharp-plugins/src/.libs:/usr/mpi/gcc/openmpi-4.1.7a1/lib64 \
${WDIR}/github/nccl-tests/build/all_reduce_perf
```



```
/root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/ompi/bin/mpirun \
-host g1,g2,g3,g4 -allow-run-as-root -map-by ppr:1:node -oversubscribe \
-x LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${WDIR}/github/nccl/build/lib:${WDIR}/github/nccl-rdma-sharp-plugins/src/.libs:/usr/mpi/gcc/openmpi-4.1.7a1/lib64 \
-x NCCL_SOCKET_IFNAME=eno1 \
-x NCCL_DEBUG=INFO \
-x NCCL_NET_GDR_LEVEL=SYS \
-x NCCL_COLLNET_ENABLE=1 \
-x NCCL_ALGO=CollnetChain \
-x NCCL_IB_HCA=mlx5_0,mlx5_4 \
-x SHARP_COLL_LOG_LEVEL=3 -mca coll_hcoll_enable 0 -x HCOLL_ENABLE_SHARP=1 -x SHARP_COLL_ENABLE_SAT=1 \
-mca coll_ucc_enable 1 -x OMPI_UCC_CL_BASIC_TLS=nccl -x UCC_TL_SHARP_TUNE=inf \
-x SHARP_COLL_ENABLE_CUDA=1 -x SHARP_COLL_ENABLE_GPU_DIRECT_RDMA=1 \
-mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1,mlx5_4:1 \
/root/osu-micro-benchmarks-7.5/c/mpi/collective/blocking/osu_allreduce -d cuda

```



### OSU-Benchmark

```
WDIR=${HOME}
/usr/mpi/gcc/openmpi-4.1.7a1/bin/mpirun \
-host g1,g2 -allow-run-as-root \
-map-by ppr:1:node -oversubscribe \
-mca coll_hcoll_enable 0 \
-x CUDA_VISIBLE_DEVICES=0 \
-mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1 \
/root/osu-micro-benchmarks-7.5/c/mpi/collective/blocking/osu_allreduce
```

```
module load /root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/modulefiles/hpcx

/root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/ompi/bin/mpirun \
-host g1,g2,g3,g4 -allow-run-as-root \
-map-by ppr:1:node -oversubscribe \
-x LD_LIBRARY_PATH \
-x SHARP_COLL_LOG_LEVEL=3 \
-mca coll_hcoll_enable 1 \
-x HCOLL_ENABLE_SHARP=1 \
-x SHARP_COLL_ENABLE_SAT=1 \
-mca coll_ucc_enable 0 \
-x OMPI_UCC_CL_BASIC_TLS=ucp,sharp \
-x UCC_TL_SHARP_TUNE=inf \
-mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1,mlx5_4:1 \
/root/osu-micro-benchmarks-7.5/c/mpi/collective/blocking/osu_allreduce
```



```
module load /root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/modulefiles/hpcx

/root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/ompi/bin/mpirun \
-host g1,g2,g3,g4 -allow-run-as-root \
-map-by ppr:2:node -oversubscribe \
-x LD_LIBRARY_PATH \
-x CUDA_VISIBLE_DEVICES=0,1 \
-x SHARP_COLL_LOG_LEVEL=3 \
-mca coll_hcoll_enable 1 \
-x HCOLL_ENABLE_SHARP=1 \
-x SHARP_COLL_ENABLE_SAT=1 \
-mca coll_ucc_enable=0 \
-x OMPI_UCC_CL_BASIC_TLS=ucp,sharp \
-x UCC_TL_SHARP_TUNE=inf \
-mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1,mlx5_4:1 \
/root/osu-micro-benchmarks-7.5/c/mpi/collective/blocking/osu_allreduce
```



```
module load /root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/modulefiles/hpcx

/root/hpcx-v2.21-gcc-doca_ofed-redhat9-cuda12-x86_64/ompi/bin/mpirun \
-host g1,g2,g3,g4 -allow-run-as-root \
-map-by ppr:2:node -oversubscribe \
-x LD_LIBRARY_PATH \
-x CUDA_VISIBLE_DEVICES=0,1 \
-x SHARP_COLL_LOG_LEVEL=3 \
-mca coll_hcoll_enable 0 \
-x HCOLL_ENABLE_SHARP=1 \
-x SHARP_COLL_ENABLE_SAT=1 \
-mca coll_ucc_enable=1 \
-x OMPI_UCC_CL_BASIC_TLS=ucp,sharp \
-x UCC_TL_SHARP_TUNE=inf \
-mca pml ucx -x UCX_NET_DEVICES=mlx5_0:1,mlx5_4:1 \
/root/osu-micro-benchmarks-7.5/c/mpi/collective/blocking/osu_allreduce -d cuda
```

