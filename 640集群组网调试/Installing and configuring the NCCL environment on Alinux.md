装机日志（批量分发 · 环境安装 · SSH 免密 · 防火墙关闭 · 环境变量 · 编译与压测）

适用：AliOS/alinux 系环境；批量分发 /home/driver，安装 OFED / NVIDIA Driver / CUDA / Fabric Manager；配置 SSH 免密与防火墙；写入库路径；编译 NCCL / nccl-tests / SHARP；单机 & 多机 NCCL 基线

0. 清单与前置

1. 分发 /home/driver

2. 环境安装（OFED → Driver → CUDA → FabricManager）

3. SSH 免密（收集→合并→分发）

4. 关闭指纹校验 & 防火墙

5. 写入环境变量 LD_LIBRARY_PATH

6. 编译 NCCL / nccl-tests / SHARP

7. 单机 NCCL（对比 NVLS=1/0）

8. 多机 NCCL（OpenMPI）

0. 清单与前置

Inventory 示例（3.ini）：

[all]
10.19.5.168 ansible_user=root ansible_password=admin123

[source]
10.19.0.1 ansible_user=root ansible_password=admin123

[all:vars]
ansible_ssh_common_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
ansible_python_interpreter=/usr/bin/python3


source 组作为源机器，向 all:!source 的目标机器分发数据。

装机日志

1. 分发 /home/driver

yml/push.yml（核心任务节选）：

- hosts: source
  gather_facts: no
  vars:
    src_path: /home/driver/
    dst_path: /home/driver/
    ssh_extra: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5 -o ServerAliveInterval=30 -o ServerAliveCountMax=3"
    targets: "{{ groups['all'] | difference(groups['source']) }}"
    rsync_timeout_sec: 7200

  tasks:
    - name: Ensure rsync/sshpass exist on source (dnf/yum fallback)
      become: true
      raw: |
        set -e
        has() { command -v "$1" >/dev/null 2>&1; }
        if ! has rsync || ! has sshpass; then
          if command -v dnf >/dev/null 2>&1; then dnf -y install rsync sshpass
          else yum -y install rsync sshpass; fi
        fi
      changed_when: false

    - name: Launch parallel rsync jobs (fire-and-forget)
      vars:
        target_host: "{{ hostvars[item].ansible_host | default(item) }}"
        ssh_user: "{{ hostvars[item].ansible_user | default('root') }}"
        ssh_pass: "{{ hostvars[item].ansible_password | default('') }}"
      shell: >
        sshpass -p '{{ ssh_pass }}' rsync -az --delete --partial --omit-dir-times --progress
        -e "ssh {{ ssh_extra }}" {{ src_path }} {{ ssh_user }}@{{ target_host }}:{{ dst_path }}
        {% if rsync_path is defined %} --rsync-path='{{ rsync_path }}' {% endif %}
      loop: "{{ targets }}"
      async: "{{ rsync_timeout_sec }}"
      poll: 0
      register: rsync_jobs

    - name: Wait for all rsync jobs to finish
      async_status:
        jid: "{{ item.ansible_job_id }}"
      loop: "{{ rsync_jobs.results }}"
      register: rsync_wait
      until: rsync_wait.finished
      retries: 600
      delay: 5

    - name: Fail if any rsync job failed
      fail:
        msg: "Rsync to {{ item.item }} failed: rc={{ item.rc }}, stdout={{ item.stdout | default('') }}, stderr={{ item.stderr | default('') }}"
      when: item.rc is defined and item.rc != 0
      loop: "{{ rsync_wait.results }}"


上面片段对应并行 rsync + 异步等待 + 失败兜底的完整流程。

装机日志

 

装机日志

执行：

ansible-playbook -i ./ini/hosts_8back.ini ./yml/push.yml -f 8
# 或
ansible-playbook -i 6158.ini ./yml/push.yml


执行示例来自文档原文。

装机日志

2. 环境安装（OFED → Driver → CUDA → FabricManager）

suite.sh（核心段落）：

#!/bin/bash
set -euo pipefail
echo "Start"
chmod +x ./*.sh ./*.run

# 远程 yum 源（替代本地 iso）
cat > /etc/yum.repos.d/Alios.repo << 'EOF'
[alios]
name=alios
baseurl=http://10.19.7.2/alios/
enabled=1
gpgcheck=0
EOF
yum --disablerepo="*" --enablerepo="alios" clean all
yum --disablerepo="*" --enablerepo="alios" makecache
yum --disablerepo="*" --enablerepo="alios" install -y make gcc gcc-c++
yum --disablerepo="*" --enablerepo="alios" install -y rpm-build kernel-rpm-macros elfutils-libelf-devel lsof libtool tk tcl

echo "MLNX_OFED Begin"
# ……（此处按你包名与参数执行 mlnxofedinstall）


源配置和安装顺序来自文档原文（Alios 源、工具链、OFED 起步）。

装机日志

3. SSH 免密（收集→合并→分发）

生成密钥与收集公钥：

- name: Ensure keypair exists on every node
  openssh_keypair:
    path: "{{ key_path }}"
    type: "{{ key_type }}"
    owner: "{{ target_user }}"
    group: "{{ target_user }}"
    mode: '0600'
    comment: "{{ target_user }}@{{ inventory_hostname }}"
    state: present
    force: false

- name: Fetch each node's public key to control node
  fetch:
    src: "{{ key_path }}.pub"
    dest: "/tmp/pubkeys/{{ inventory_hostname }}.pub"
    flat: yes


逐机生成 keypair 并抓取至控制节点。

装机日志

在控制节点合并 + 分发到所有节点：

- name: Merge all pubkeys into one file
  hosts: localhost
  gather_facts: false
  tasks:
    - shell: "cat /tmp/pubkeys/*.pub | sort -u > /tmp/all_authorized_keys"

- name: Distribute merged authorized_keys to all nodes
  hosts: "{{ target_group | default('all') }}"
  gather_facts: false
  become: true
  vars:
    target_user: root
    ssh_dir: "{{ '/root/.ssh' if target_user == 'root' else '/home/' + target_user + '/.ssh' }}"
  tasks:
    - name: Copy merged authorized_keys to each node
      copy:
        src: /tmp/all_authorized_keys
        dest: "{{ ssh_dir }}/authorized_keys"
        owner: "{{ target_user }}"
        group: "{{ target_user }}"
        mode: '0600'


三段式流程（收集→合并→分发）与关键 copy 任务。

装机日志

 

装机日志

执行：

ansible-playbook -i ./ini/hosts_510.ini ./yml/ssh.yml -f 510


来自原文示例。

装机日志

4. 关闭指纹校验 & 防火墙

yml/disable_checking_firewall.yml（节选）：

- hosts: all
  gather_facts: no
  become: true
  tasks:
    - name: 写入全局 SSH 配置，禁用 StrictHostKeyChecking
      copy:
        dest: /root/.ssh/config
        content: |
          Host *
             StrictHostKeyChecking no
             UserKnownHostsFile /dev/null
             LogLevel ERROR
        owner: root
        group: root
        mode: '0600'

    - name: Stop firewalld service (ignore errors if not present)
      shell: systemctl stop firewalld 2>/dev/null || true

    - name: Disable firewalld service (ignore errors if not present)
      shell: systemctl disable firewalld 2>/dev/null || true


执行：

ansible-playbook -i ./ini/hosts_8back.ini ./yml/disable_checking_firewall.yml -f 8


配置与执行命令来源于原文。

装机日志

 

装机日志

5. 写入环境变量 LD_LIBRARY_PATH

CUDA profile（建议放 /etc/profile.d/cuda.sh）：

cat > /etc/profile.d/cuda.sh <<'EOF'
export PATH=/usr/local/cuda-12.8/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:${LD_LIBRARY_PATH:-}
EOF
chmod 755 /etc/profile.d/cuda.sh
source /etc/profile.d/cuda.sh || true


该段来自装机脚本原文中的 CUDA 配置。

装机日志

6. 编译 NCCL / nccl-tests / SHARP

yml/build_nccl_all.yml（节选）：

- hosts: all
  gather_facts: no
  become: true
  vars:
    WDIR: /home/driver
    CUDA_HOME: /usr/local/cuda-12.8
    MPI_HOME: /usr/mpi/gcc/openmpi-4.1.7rc1

  tasks:
    - name: 编译 nccl-rdma-sharp-plugins
      shell: |
        set -e
        cd {{ WDIR }}/github/nccl-rdma-sharp-plugins
        sh autogen.sh
        ./configure --with-sharp=/opt/mellanox/sharp --with-cuda={{ CUDA_HOME }}
        time make -j
      args: { chdir: "{{ WDIR }}/github/nccl-rdma-sharp-plugins" }
      environment:
        PATH: "{{ CUDA_HOME }}/bin:{{ ansible_env.PATH }}"
        LD_LIBRARY_PATH: "{{ CUDA_HOME }}/lib64:{{ ansible_env.LD_LIBRARY_PATH | default('') }}"
        CUDA_HOME: "{{ CUDA_HOME }}"
        PKG_CONFIG_PATH: "/opt/mellanox/sharp/lib/pkgconfig:{{ ansible_env.PKG_CONFIG_PATH | default('') }}"

    - name: Build NCCL
      shell: |
        time make -C {{ WDIR }}/github/nccl -j src.build CUDA_HOME={{ CUDA_HOME }}
      args: { chdir: "{{ WDIR }}/github/nccl" }

    - name: Build NCCL-Tests
      shell: |
        make -C {{ WDIR }}/github/nccl-tests -j \
        MPI=1 MPI_HOME={{ MPI_HOME }} \
        CUDA_HOME={{ CUDA_HOME }} \
        NCCL_HOME={{ WDIR }}/github/nccl/build
      args: { chdir: "{{ WDIR }}/github/nccl-tests" }


执行：

ansible-playbook -i ./ini/hosts_8back.ini ./yml/build_nccl_all.yml -f 8
# 或
ansible-playbook -i 5168.ini ./yml/build_nccl_all.yml -f 8


变量定义、三段编译与执行命令均来自原文。

装机日志

 

装机日志

 

装机日志

 

装机日志

 

装机日志

7. 单机 NCCL（对比 NVLS=1/0）

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


该任务块来自原文，用于生成“NVLS=1”基线。

装机日志

（你可复制该任务再改 NCCL_NVLS_ENABLE=0 形成“NVLS=0”对照。）

8. 多机 NCCL（OpenMPI）

编译时已启用 MPI=1 和 MPI_HOME，多机运行可在 hostfile 或 -H 传入节点，示例命令建议与你的集群实践同步；此节在原 PDF 中仅给出编译参数与环境变量，未提供固定运行命令，因此这里不强行附带命令行以免误导。（如需我根据你当前的 OpenMPI 环境生成 2–4 节点的标准命令，请告诉我节点名/网卡/绑定策略即可。）
