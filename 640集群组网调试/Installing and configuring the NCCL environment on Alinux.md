装机日志（批量分发 · 环境安装 · SSH 免密 · 防火墙关闭 · 环境变量 · 编译与压测）

适用：AliOS/alinux 系环境；批量分发 /home/driver，安装 OFED / NVIDIA Driver / CUDA / Fabric Manager；配置 SSH 免密与防火墙；写入库路径；编译 NCCL / nccl-tests / SHARP；单机 & 多机 NCCL 基线

0. 清单与前置

1. 传输文件

2. 环境安装

3. 配置免密

4. 关闭指纹校验 & 防火墙

5. 配置环境变量

6. 编译 NCCL / nccl-tests / SHARP

7. 单机 NCCL（对比 NVLS=1/0）

8. 多机 NCCL（OpenMPI）

# 0. 清单与前置

Inventory 示例（hosts_510.ini）：
```
[all]
10.19.0.[1:64]
10.19.0.[129:191]
10.19.4.[129:192]
10.19.5.[1:64]
10.19.5.[129:192]
10.19.6.[1:63]
10.19.7.[1:64]
10.19.7.[129:192]

[source]
10.19.0.1

[all:vars]
ansible_ssh_common_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
ansible_python_interpreter=/usr/bin/python3
```


# 1. 传输文件

[push.yml](./yml/push.yml)：

执行：

ansible-playbook -i ./ini/hosts_510ini ./yml/push.yml -f 8


# 2. 环境安装（OFED → Driver → CUDA → FabricManager）
```
#!/bin/bash
set -euo pipefail

echo "Start"
chmod +x ./*.sh ./*.run

cat > /etc/yum.repos.d/Alios.repo << 'EOF'
[alios]
name=alios
baseurl=http://10.19.7.2/alios/
enabled=1
gpgcheck=0
EOF
yum --disablerepo="*" --enablerepo="alios" clean all
yum --disablerepo="*" --enablerepo="alios" makecache
#yum --disablerepo="*" --enablerepo="alios" groupinstall -y "Development Tools"
yum --disablerepo="*" --enablerepo="alios" install -y make gcc gcc-c++
yum --disablerepo="*" --enablerepo="alios" \
    install -y rpm-build kernel-rpm-macros elfutils-libelf-devel lsof libtool tk tcl

echo "MLNX_OFED Begin"

tar xf MLNX_OFED_LINUX-24.10-2.1.8.0-alinux3.2-x86_64.tgz
cd MLNX_OFED_LINUX-24.10-2.1.8.0-alinux3.2-x86_64
./mlnxofedinstall --add-kernel-support --skip-distro-check --skip-repo --without-mlnx-nvme --without-fw-update
dracut -f
cd ..

echo "MLNX_OFED Down, NVIDIA Driver Begin"

./NVIDIA-Linux-x86_64-570.133.20-20250704014111-5.10.134-18.al8.run \
    --silent \
    --accept-license \
    --disable-nouveau \
    --kernel-module-type=open \
    --no-install-compat32-libs \
    --no-opengl-files

echo "NVIDIA Driver Down, CUDA Begin"

./cuda_12.8.0_570.86.10_linux.run \
    --silent \
    --toolkit \
    --no-drm \
    --no-opengl-libs \
    --override
    
cat >/etc/profile.d/cuda.sh <<'EOF'
export PATH=/usr/local/cuda-12.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:${LD_LIBRARY_PATH:-}
EOF
chmod 755 /etc/profile.d/cuda.sh
source /etc/profile.d/cuda.sh || true

echo "CUDA Down, fabricmanager Begin"
yum --disablerepo="*" --enablerepo="alios" -y install nvidia-fabric-manager
systemctl enable --now nvidia-fabricmanager
echo "ALL Down"
```
主要工作：安装工具包、OFED、NVIDIA Driver、CUDA、fabircmanager
[run_suite.yml](./yml/run_suite.yml)
```
ansible-playbook -i ./ini/hosts_510.ini ./yml/run_suite.yml -f 510
```


#3. SSH 免密（收集→合并→分发）

[ssh.yml](./yml/ssh.yml)
 

执行：
```
ansible-playbook -i ./ini/hosts_510.ini ./yml/ssh.yml -f 510
```


#4. 关闭指纹校验 & 防火墙

[disable.yml](./yml/disable.yml)


执行：
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/disable.yml -f 8
```



#5. 写入环境变量 LD_LIBRARY_PATH

```
cat > /etc/profile.d/cuda.sh <<'EOF'
export PATH=/usr/local/cuda-12.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:${LD_LIBRARY_PATH:-}
EOF
chmod 755 /etc/profile.d/cuda.sh
source /etc/profile.d/cuda.sh || true
```



#6. 编译 NCCL / nccl-tests / SHARP

[build_nccl_all.yml](./yml/build_nccl_all.yml)


执行：
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/build_nccl_all.yml -f 8
```


#7. 单机 NCCL（对比 NVLS=1/0）
```
yml/nccl_nvls.yml（NVLS 开启）：

- hosts: all
  gather_facts: false
  become: false
  strategy: free
  vars:
    workdir: /home/driver/github/nccl-tests
    logdir_local: /var/log/ansible/nccl
    async_timeout: 600
    poll_interval: 5
    host_label: "{{ hostvars[inventory_hostname].ansible_host | default(inventory_hostname) }}"
    cuda_profile: /etc/profile.d/cuda.sh

  tasks:
    - name: 运行 all_reduce_perf（NVLS 开启）
      shell: |
        set -euo pipefail
        [ -f "{{ cuda_profile }}" ] && . "{{ cuda_profile }}" || true
        cd "{{ workdir }}"
        NCCL_COLLNET_ENABLE=0 NCCL_NVLS_ENABLE=1 NCCL_DEBUG=INFO \
        ./build/all_reduce_perf -b 8M -e 16G -f 2 -g 8 \
          | grep -E -A9999 "size\s+count" || true
      async: "{{ async_timeout }}"
      poll: "{{ poll_interval }}"
      register: run_nvls_on
```


#8. 多机 NCCL（OpenMPI）
```
time /usr/mpi/gcc/openmpi-4.1.7rc1/bin/mpirun \
-host 10.19.5.168,10.19.5.169 \
-map-by ppr:8:node -oversubscribe \
-x NCCL_SOCKET_IFNAME=bond0 \
-x NCCL_IB_QPS_PER_CONNECTION=1 \
-allow-run-as-root -x NCCL_DEBUG=VERSION \
-x NCCL_COLLNET_ENABLE=1 \
-x SHARP_COLL_ENABLE_PCI_RELAXED_ORDERING=1 \
-x NCCL_RAS_ENABLE=0 -x NCCL_MIN_CTAS=24 \
/home/driver/github/nccl-tests/build/all_reduce_perf -b 8M -e 16G -f 2 -g 1
```
