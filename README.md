# Ansible Tutorial

## ■はじめに

AnsibleのBest Practicesを元にAnsibleを習得するためのチュートリアルです。

Best Practices: https://docs.ansible.com/ansible/2.8/user_guide/playbooks_best_practices.html#directory-layout

## ■初期ディレクトリ作成
```
mkdir group_vars
mkdir host_vars
mkdir roles
mkdir roles/common
mkdir roles/common/tasks
mkdir roles/web
mkdir roles/web/tasks
mkdir roles/web/handlers
mkdir roles/web/templates
mkdir roles/web/files
mkdir roles/web/vars
mkdir roles/web/meta
```

## ■hello, world

### commonロールのPlaybook作成
```
cat << EOF > roles/common/tasks/main.yml
---
- name: test
  debug:
    msg: "hello, common"
EOF
```
### webロールのPlaybook作成
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test
  debug:
    msg: "hello, world"
EOF
```

### webservers.yml作成
```
cat << EOF > webservers.yml
---
- hosts: webservers
  roles:
    - common
    - web
EOF
```

### site.yml作成
```
cat << EOF > site.yml
---
- import_playbook: webservers.yml
EOF
```

### inventory作成
```
cat << EOF > inventory.yml
---
all:
  children:
    webservers:
      hosts:
        host1:
          ansible_host: 127.0.0.1
          ansible_password: vagrant
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test] **************************************************************
ok: [host1] => {
    "msg": "hello, world"
}

PLAY RECAP *********************************************************************
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## ■パスワードの暗号化

### パスワードを変数に定義
```
cat << EOF > host_vars/host1.yml
---
password: vagrant
EOF
```

### inventoryからパスワードを参照
```
cat << EOF > inventory.yml
---
all:
  children:
    webservers:
      hosts:
        host1:
          ansible_host: 127.0.0.1
          ansible_password: "{{password}}"
EOF
```

### パスワードの暗号化
```
ansible-vault encrypt host_vars/host1.yml
```
パスワードを聞かれるので、「password」と入力する

### 暗号化後の変数確認
```
cat host_vars/host1.yml
```
```
$ANSIBLE_VAULT;1.1;AES256
35353963666439303563653630313238326262373961626663613731613836616133366332663735
3738626237656539393661343239386631336432353636340a323039393330323162663631323635
63386536303336373339353234353062643532653263333834333431323065303864636262646361
3562306430663235350a373762376564626137343164663139306337326338363765346430313337
34386264363235343132633932643465626564306163396434343561323234333035
```

### 実行
```
ansible-playbook -i inventory.yml site.yml
```

### 実行結果
```
PLAY [webservers] **************************************************************
ERROR! Attempting to decrypt but no vault secrets found
```
エラーとなる

### 実行
```
echo password > password.txt
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test] **************************************************************
ok: [host1] => {
    "msg": "hello, world"
}

PLAY RECAP *********************************************************************
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## ■変数の利用(グローバル)

### 変数を定義
```
cat << EOF > group_vars/all.yml
---
var1: TEST1
EOF
```

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test
  debug:
    msg: "hello, world, {{ var1 }}"
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test] **************************************************************
ok: [host1] => {
    "msg": "hello, world, TEST1"
}

PLAY RECAP *********************************************************************
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## ■ループ

### 変数を定義
```
cat << EOF > roles/web/vars/main.yml
---
var2: TEST2
list:
  - FOO
  - BAR
EOF
```

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test
  debug:
    msg: "hello, world, {{ var1 }}, {{ var2 }}, {{ item }}"
  loop: "{{ list }}"
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test] **************************************************************
ok: [host1] => (item=FOO) => {
    "msg": "hello, world, TEST1, TEST2, FOO"
}
ok: [host1] => (item=BAR) => {
    "msg": "hello, world, TEST1, TEST2, BAR"
}

PLAY RECAP *********************************************************************
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## ■handler

### handlerを作成
```
cat << EOF > roles/web/handlers/main.yml
---
- name: condition_handler
  debug:
    msg: "listned condition_handler"
  listen:
    - condition_handler
EOF
```

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test handlers1
  command: /bin/true
  notify:
    - condition_handler
- name: test handlers2
  command: /bin/true
  notify:
    - condition_handler
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test handlers1] ****************************************************
changed: [host1]

TASK [web : test handlers2] ****************************************************
changed: [host1]

RUNNING HANDLER [web : condition_handler] **************************************
ok: [host1] => {
    "msg": "listned condition_handler"
}

PLAY RECAP *********************************************************************
host1                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

## ■template

### templateファイル作成
```
cat << EOF > roles/web/templates/sample.conf.j2
foo={{ var1 }}
EOF
```

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test template
  template:
    src: sample.conf.j2
    dest: /tmp/sample.conf
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test template] *****************************************************
ok: [host1]

PLAY RECAP *********************************************************************
host1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

### 出力ファイル確認
```
cat /tmp/sample.conf
```
```
foo=TEST1
```

## ■scriptモジュール

### シェルスクリプト作成
```
cat << "EOF" > roles/web/files/sample.sh
#!/bin/bash
cp -p "$1" "$1.bak"
EOF
```

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test template
  template:
    src: sample.conf.j2
    dest: /tmp/sample.conf
- name: test script
  script: sample.sh sample.conf
  args:
    chdir: /tmp
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test template] *****************************************************
ok: [host1]

TASK [web : test script] *******************************************************
changed: [host1]

PLAY RECAP *********************************************************************
host1                      : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### スクリプト実行結果確認
```
ls -ltr /tmp
```
```
-rw-rw-r-- 1 vagrant vagrant    10 Jul 16 02:29 sample.conf.bak
-rw-rw-r-- 1 vagrant vagrant    10 Jul 16 02:29 sample.conf
```

## ■meta

### 依存関係を定義
```
cat << EOF > roles/web/meta/main.yml
dependencies:
  - common
EOF
```

### webservers.ymlの修正
```
cat << EOF > webservers.yml
---
- hosts: webservers
  roles:
    - web
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"
}

TASK [web : test template] *****************************************************
ok: [host1]

TASK [web : test script] *******************************************************
changed: [host1]

PLAY RECAP *********************************************************************
host1                      : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

## ■完成

### Playbookの修正
```
cat << EOF > roles/web/tasks/main.yml
---
- name: test
  debug:
    msg: "hello, world: {{ var1 }}, {{ var2 }}, {{ item }}"
  loop: "{{ list }}"
- name: test register
  command: /bin/false
  register: result
  ignore_errors: True
- name: test condition
  debug:
    msg: "condition is false: {{ result.failed }}"
  when: result.failed == True
- name: test handlers1
  command: /bin/true
  notify:
    - condition_handler
- name: test handlers2
  command: /bin/true
  notify:
    - condition_handler
- name: test template
  template:
    src: sample.conf.j2
    dest: /tmp/sample.conf
- name: test script
  script: sample.sh sample.conf
  args:
    chdir: /tmp
EOF
```

### 実行
```
ansible-playbook -i inventory.yml site.yml --vault-password-file password.txt
```

### 実行結果
```
PLAY [webservers] **************************************************************

TASK [Gathering Facts] *********************************************************
[WARNING]: Platform linux on host host1 is using the discovered Python          
interpreter at /usr/bin/python3.12, but future installation of another Python   
interpreter could change the meaning of that path. See                          
https://docs.ansible.com/ansible-
core/devel/reference_appendices/interpreter_discovery.html for more
information.
ok: [host1]

TASK [common : test] ***********************************************************
ok: [host1] => {
    "msg": "hello, common"                                                      
}               

TASK [web : test] **************************************************************
ok: [host1] => (item=FOO) => {
    "msg": "hello, world: TEST1, TEST2, FOO"
}                                                                               
ok: [host1] => (item=BAR) => {
    "msg": "hello, world: TEST1, TEST2, BAR"
}                                                                               
TASK [web : test register] *****************************************************
fatal: [host1]: FAILED! => {"changed": true, "cmd": ["/bin/false"], "delta": "0:
00:00.009958", "end": "2024-07-16 03:27:28.242141", "msg": "non-zero return code
", "rc": 1, "start": "2024-07-16 03:27:28.232183", "stderr": "", "stderr_lines":
 [], "stdout": "", "stdout_lines": []}
...ignoring

TASK [web : test condition] ****************************************************
ok: [host1] => {
    "msg": "condition is false: True"
}

TASK [web : test handlers1] ****************************************************
changed: [host1]

TASK [web : test handlers2] ****************************************************
changed: [host1]

TASK [web : test template] *****************************************************
ok: [host1]

TASK [web : test script] *******************************************************
changed: [host1]

RUNNING HANDLER [web : condition_handler] **************************************
ok: [host1] => {
    "msg": "listned condition_handler"
}

PLAY RECAP *********************************************************************
host1                      : ok=10   changed=4    unreachable=0    failed=0    s
kipped=0    rescued=0    ignored=1
```
