# 装机日志（批量分发 · 环境安装 · SSH 免密 · 防火墙关闭 · 环境变量 · 编译与测试）

适用：AliOS/alinux 系环境；批量分发文件，安装 OFED / NVIDIA Driver / CUDA / Fabric Manager；配置 SSH 免密与防火墙；写入库路径；编译 NCCL / nccl-tests / SHARP；单机 & 多机 NCCL 基线

# 0. 清单与前置
Inventory 示例（hosts_510.ini）：
```
#hosts_510.ini

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
```
#push.yml

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
          if command -v dnf >/dev/null 2>&1; then
            dnf -y install rsync sshpass
          else
            yum -y install rsync sshpass
          fi
        fi
      changed_when: false

    - name: Launch parallel rsync jobs (fire-and-forget)
      vars:
        target_host: "{{ hostvars[item].ansible_host | default(item) }}"
        ssh_user: "{{ hostvars[item].ansible_user | default('root') }}"
        ssh_pass: "{{ hostvars[item].ansible_password | default('') }}"
      shell: >
        sshpass -p '{{ ssh_pass }}'
        rsync -az --delete --partial --omit-dir-times --progress
        -e "ssh {{ ssh_extra }}"
        {{ src_path }}
        {{ ssh_user }}@{{ target_host }}:{{ dst_path }}
        {% if rsync_path is defined %} --rsync-path='{{ rsync_path }}' {% endif %}
      loop: "{{ targets }}"
      loop_control:
        label: "{{ target_host }}"
      async: "{{ rsync_timeout_sec }}"
      poll: 0
      register: rsync_jobs

    - name: Wait for all rsync jobs to finish
      async_status:
        jid: "{{ item.ansible_job_id }}"
      loop: "{{ rsync_jobs.results }}"
      loop_control:
        label: "{{ (hostvars[item.item].ansible_host | default(item.item)) if (item.item is string) else item.item }}"
      register: rsync_wait
      until: rsync_wait.finished
      retries: 600          # 600 * 5s = 3000s
      delay: 5

    - name: Fail if any rsync job failed
      fail:
        msg: "Rsync to {{ item.item }} failed: rc={{ item.rc }}, stdout={{ item.stdout | default('') }}, stderr={{ item.stderr | default('') }}"
      when: item.rc is defined and item.rc != 0
      loop: "{{ rsync_wait.results }}"
```


执行：
```
ansible-playbook -i ./ini/hosts_510ini ./yml/push.yml -f 8
```

# 2. 环境安装（OFED → Driver → CUDA → FabricManager）
```
#suite.sh
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
### 主要工作：安装工具包、OFED、NVIDIA Driver、CUDA、fabircmanager
```
# run_suite.yml
- hosts: localhost
  gather_facts: false
  tasks:
    - name: 在本地创建日志目录
      file:
        path: /var/log/ansible/suite
        state: directory
        mode: "0755"

- hosts: all:!source
  strategy: free
  gather_facts: false
  become: true

  vars:
    workdir: /home/driver
    logdir_local: /var/log/ansible/suite
    async_timeout: 900          # 最长 15 分钟
    poll_interval: 10
    retries_max: "{{ (async_timeout // poll_interval) | int + 1 }}"

  tasks:
    - name: 启动 suite.sh（不等待，立刻返回 JID）
      shell: "chmod +x suite.sh && ./suite.sh"
      args: { chdir: "{{ workdir }}" }
      async: "{{ async_timeout }}"
      poll: 0
      register: job

    - name: 提交任务提示
      debug:
        msg: "submitted {{ inventory_hostname }} (jid={{ job.ansible_job_id }})"

    - name: 轮询直到完成或超时（最多15分钟）
      async_status:
        jid: "{{ job.ansible_job_id }}"
      register: r
      until: r.finished
      retries: "{{ retries_max }}"
      delay: "{{ poll_interval }}"
      failed_when: false   # 超时不算错误

    - name: 计算状态（OK/FAILED/TIMEOUT）
      set_fact:
        run_status: >-
          {% if not r.finished %}TIMEOUT{% elif r.rc is defined and r.rc|int == 0 %}OK{% else %}FAILED{% endif %}

    - name: 将每台输出写回本地
      delegate_to: localhost
      throttle: 64
      copy:
        dest: "{{ logdir_local }}/{{ inventory_hostname }}.log"
        mode: "0644"
        content: |
          ===== {{ inventory_hostname }} @ {{ lookup('pipe','date -Iseconds') }} =====
          STATUS={{ run_status }}
          RC={{ r.rc | default('N/A') }}

          --- STDOUT ---
          {{ r.stdout | default('') }}

          --- STDERR ---
          {{ r.stderr | default('') }}

    - name: 若 suite.sh 失败则报错
      fail:
        msg: "Host {{ inventory_hostname }} suite.sh FAILED RC={{ r.rc }}"
      when: run_status == 'FAILED'

```
执行:
```
ansible-playbook -i ./ini/hosts_510.ini ./yml/run_suite.yml -f 510
```


# 3. SSH 免密（收集→合并→分发）
```
#ssh.yml

- name: Collect and distribute SSH authorized_keys from source host
  hosts: "{{ target_group | default('all') }}"
  gather_facts: false
  become: true

  vars:
    target_user: root
    target_group: all
    key_type: ed25519
    ssh_dir: "{{ '/root/.ssh' if target_user == 'root' else '/home/' + target_user + '/.ssh' }}"
    key_path: "{{ ssh_dir + '/id_' + key_type }}"
    merged_keys: "/tmp/all_authorized_keys"

  tasks:
    - name: Ensure {{ ssh_dir }} exists
      file:
        path: "{{ ssh_dir }}"
        state: directory
        owner: "{{ target_user }}"
        group: "{{ target_user }}"
        mode: '0700'

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

- name: Merge and distribute authorized_keys
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Merge all pubkeys into one file
      shell: "cat /tmp/pubkeys/*.pub | sort -u > /tmp/all_authorized_keys"

- name: Distribute merged authorized_keys to all nodes
  hosts: "{{ target_group | default('all') }}"
  gather_facts: false
  become: true

  vars:
    target_user: root
    target_group: all
    ssh_dir: "{{ '/root/.ssh' if target_user == 'root' else '/home/' + target_user + '/.ssh' }}"

  tasks:
    - name: Copy merged authorized_keys to each node
      copy:
        src: /tmp/all_authorized_keys
        dest: "{{ ssh_dir }}/authorized_keys"
        owner: "{{ target_user }}"
        group: "{{ target_user }}"
        mode: '0600'

```

 

执行：
```
ansible-playbook -i ./ini/hosts_510.ini ./yml/ssh.yml -f 510
```


# 4. 关闭指纹校验 & 防火墙

```
#diable.yml

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

```


执行：
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/disable.yml -f 8
```



# 5. 写入环境变量 LD_LIBRARY_PATH

```
#library.yml

- hosts: all
  gather_facts: no
  become: true

  vars:
    ld_library_path_line: 'export LD_LIBRARY_PATH=/home/driver/github/nccl/build/lib:/home/driver/github/nccl-rdma-sharp-plugins/src/.libs:/usr/mpi/gcc/openmpi-4.1.7rc1/lib64:$LD_LIBRARY_PATH'

  tasks:
    - name: 将 export 写入 /root/.profile
      lineinfile:
        path: /root/.profile
        regexp: '^export LD_LIBRARY_PATH='
        line: "{{ ld_library_path_line }}"
        create: yes
        state: present

    - name: 将 export 写入 /root/.bashrc
      lineinfile:
        path: /root/.bashrc
        regexp: '^export LD_LIBRARY_PATH='
        line: "{{ ld_library_path_line }}"
        create: yes
        state: present

```
执行：
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/library.yml -f 8
```


# 6. 编译 NCCL / nccl-tests / SHARP
```
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
        ./configure \
          --with-sharp=/opt/mellanox/sharp \
          --with-cuda={{ CUDA_HOME }}
        time make -j
      args:
        chdir: "{{ WDIR }}/github/nccl-rdma-sharp-plugins"
      environment:
        PATH: "{{ CUDA_HOME }}/bin:{{ ansible_env.PATH }}"
        LD_LIBRARY_PATH: "{{ CUDA_HOME }}/lib64:{{ ansible_env.LD_LIBRARY_PATH | default('') }}"
        CUDA_HOME: "{{ CUDA_HOME }}"
        PKG_CONFIG_PATH: "/opt/mellanox/sharp/lib/pkgconfig:{{ ansible_env.PKG_CONFIG_PATH | default('') }}"

    - name: Build NCCL
      shell: |
        time make -C {{ WDIR }}/github/nccl -j src.build CUDA_HOME={{ CUDA_HOME }}
      args:
        chdir: "{{ WDIR }}/github/nccl"

    - name: Build NCCL-Tests
      shell: |
        make -C {{ WDIR }}/github/nccl-tests -j \
        MPI=1 MPI_HOME={{ MPI_HOME }} \
        CUDA_HOME={{ CUDA_HOME }} \
        NCCL_HOME={{ WDIR }}/github/nccl/build
      args:
        chdir: "{{ WDIR }}/github/nccl-tests"
```


执行：
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/build_nccl_all.yml -f 8
```


# 7. 单机 NCCL（对比 NVLS=1/0）
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
### 主要工作：跑两遍NCCL，一遍开NVLS，一遍不开NVLS，输出日志写入/var/log/ansible/nccl
执行:
```
ansible-playbook -i ./ini/hosts_8back.ini ./yml/nccl_nvls.yml -f 8
```

# 8. 多机 NCCL（OpenMPI）
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
