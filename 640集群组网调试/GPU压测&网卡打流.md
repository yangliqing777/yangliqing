# GPU压测
```
ansible-playbook -i hosts.ini  run_gpu_collect.yml -f 256 -e dur_sec=3600
```
-e dur_sec控制压测时间
最终输出的csv会在/home/driver/results_gpuburn目录下，每台机器一个csv里面有八行数据
```
host,gpu,seconds,pass,max_tempC,throttle_any,throttle_mask_seen,xid_count 
cucloud010019000135,0,49,1,55,1,0x0000000000000001,0 
cucloud010019000135,1,49,1,54,1,0x0000000000000001,0 
cucloud010019000135,2,49,1,55,1,0x0000000000000001,0 
cucloud010019000135,3,49,1,50,1,0x0000000000000001,0 
cucloud010019000135,4,49,1,51,1,0x0000000000000001,0 
cucloud010019000135,5,49,1,54,1,0x0000000000000001,0 
cucloud010019000135,6,49,1,53,1,0x0000000000000001,0 
cucloud010019000135,7,49,1,54,1,0x0000000000000001,0
```
```
- hosts: all:!source
  gather_facts: no
  vars:
    dur_sec: 10                 # 默认每台机器跑 10 秒；可在命令行 -e dur_sec=60 覆盖
    local_results: ./results_gpuburn
  pre_tasks:
    - name: Ensure local results dir
      delegate_to: localhost
      run_once: true
      file:
        path: "{{ local_results }}"
        state: directory
        mode: "0755"

  tasks:
    - name: Run gpu_burn_collect.sh on remote
      become: true
      shell: |
        set -e
        /home/driver/gpu_burn_collect.sh {{ dur_sec }}
      args:
        executable: /bin/bash
      register: run_out
      changed_when: true

    - name: Parse remote CSV path from stdout
      set_fact:
        remote_csv: "{{ (run_out.stdout_lines | select('search','^CSV_SAVED:') | list | last | regex_replace('^CSV_SAVED:\\s*','')) | default('') }}"

    - name: Fallback to latest CSV if not parsed
      when: remote_csv == ''
      shell: "ls -1t /var/log/gpu/gpu_burn_*.csv | head -n1"
      register: find_csv
      changed_when: false

    - name: Finalize remote CSV path
      set_fact:
        remote_csv_final: "{{ remote_csv if remote_csv!='' else (find_csv.stdout | trim) }}"

    - name: Fetch CSV to control host
      fetch:
        src: "{{ remote_csv_final }}"
        dest: "{{ local_results }}/"
        flat: yes

    - name: Show fetched file (per host)
      debug:
        msg: "Fetched: {{ inventory_hostname }} -> {{ local_results }}/{{ remote_csv_final | basename }}"
```


# 两台机器间打流
Server端（例：10.19.4.132）
```
i=0; for dev in /sys/class/infiniband/mlx5_*; do
  [[ $(cat $dev/ports/1/link_layer) != "InfiniBand" ]] && continue
  name=$(basename $dev); port=$((18515+i))
  ib_write_bw -d "$name" -i 1 -q 4 -s 8388608 -n 20000 --report_gbits -p $port >"/tmp/${name}_srv.log" 2>&1 &
  ((i++))
done
```
Client端
```
i=0; for dev in /sys/class/infiniband/mlx5_*; do
  [[ $(cat $dev/ports/1/link_layer) != "InfiniBand" ]] && continue
  name=$(basename $dev); port=$((18515+i))
  ib_write_bw -d "$name" -i 1 -q 4 -s 8388608 -n 20000 --report_gbits -p $port 10.19.4.132 | tee "/tmp/${name}_cli.log"
  ((i++))
done
```
