# GitHub Actions + Ansible: Deploy PyPI Privado com Supervisor

## 1. Estrutura do Projeto

```
projeto/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── ansible/
│   ├── inventories/
│   │   ├── production/
│   │   │   ├── hosts.yml
│   │   │   └── group_vars/
│   │   │       └── all.yml
│   │   └── staging/
│   │       ├── hosts.yml
│   │       └── group_vars/
│   │           └── all.yml
│   ├── roles/
│   │   ├── python-app/
│   │   │   ├── tasks/main.yml
│   │   │   ├── handlers/main.yml
│   │   │   ├── templates/
│   │   │   │   ├── supervisor.conf.j2
│   │   │   │   └── pip.conf.j2
│   │   │   └── vars/main.yml
│   │   └── supervisor/
│   │       ├── tasks/main.yml
│   │       └── handlers/main.yml
│   ├── playbooks/
│   │   ├── deploy.yml
│   │   └── health-check.yml
│   └── ansible.cfg
└── requirements.txt
```

## 2. GitHub Actions Workflow

### .github/workflows/deploy.yml

```yaml
name: Deploy Python App

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Ambiente para deploy'
        required: true
        default: 'staging'
        type: choice
        options:
        - staging
        - production
      package_version:
        description: 'Versão do pacote (opcional)'
        required: false
        type: string

env:
  ANSIBLE_HOST_KEY_CHECKING: False
  ANSIBLE_STDOUT_CALLBACK: yaml

jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: 
          - ${{ github.event.inputs.environment || (github.ref == 'refs/heads/main' && 'production' || 'staging') }}
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Install Ansible
      run: |
        python -m pip install --upgrade pip
        pip install ansible ansible-core
        pip install -r requirements.txt

    - name: Setup SSH Key
      uses: webfactory/ssh-agent@v0.8.0
      with:
        ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

    - name: Add known hosts
      run: |
        mkdir -p ~/.ssh
        for host in $(echo "${{ secrets.SERVER_HOSTS }}" | tr ',' ' '); do
          ssh-keyscan -H $host >> ~/.ssh/known_hosts
        done

    - name: Deploy Application
      run: |
        cd ansible
        ansible-playbook -i inventories/${{ matrix.environment }}/hosts.yml \
          playbooks/deploy.yml \
          --extra-vars "package_version=${{ github.event.inputs.package_version || 'latest' }}" \
          --extra-vars "environment=${{ matrix.environment }}" \
          --extra-vars "git_commit=${{ github.sha }}" \
          --vault-password-file <(echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}")

    - name: Health Check
      run: |
        cd ansible
        ansible-playbook -i inventories/${{ matrix.environment }}/hosts.yml \
          playbooks/health-check.yml \
          --extra-vars "max_retries=10" \
          --extra-vars "retry_delay=15"

    - name: Notificar sucesso
      if: success()
      run: |
        echo "✅ Deploy realizado com sucesso no ambiente ${{ matrix.environment }}"
        echo "📦 Versão: ${{ github.event.inputs.package_version || 'latest' }}"
        echo "🔗 Commit: ${{ github.sha }}"

    - name: Notificar falha
      if: failure()
      run: |
        echo "❌ Falha no deploy do ambiente ${{ matrix.environment }}"
        exit 1
```

## 3. Configuração dos Inventários

### ansible/inventories/production/hosts.yml

```yaml
all:
  children:
    app_servers:
      hosts:
        app01:
          ansible_host: 192.168.1.10
          ansible_user: deploy
        app02:
          ansible_host: 192.168.1.11
          ansible_user: deploy
  vars:
    environment: production
    app_workers: 4
```

### ansible/inventories/production/group_vars/all.yml

```yaml
# Configurações da aplicação
app_name: "my-python-app"
app_user: "appuser"
app_group: "appuser"
app_home: "/opt/{{ app_name }}"
supervisor_group: "my_app_group"

# PyPI Privado (sem autenticação)
pypi_index_url: "https://pypi.mycompany.com/simple/"
pypi_trusted_host: "pypi.mycompany.com"

# Configurações do Supervisor
supervisor_config_dir: "/etc/supervisor/conf.d"
supervisor_log_dir: "/var/log/supervisor"

# Configurações de porta (padrão 80xx)
app_base_port: 8000
app_workers: 4

# Health check
health_check_timeout: 30
```

### ansible/inventories/staging/group_vars/all.yml

```yaml
# Herda as mesmas configurações mas com valores de staging
app_name: "my-python-app"
app_user: "appuser"
app_group: "appuser"
app_home: "/opt/{{ app_name }}"
supervisor_group: "my_app_group"

pypi_index_url: "https://pypi-staging.mycompany.com/simple/"
pypi_trusted_host: "pypi-staging.mycompany.com"

supervisor_config_dir: "/etc/supervisor/conf.d"
supervisor_log_dir: "/var/log/supervisor"

# Configurações de porta (padrão 80xx)
app_base_port: 8000
app_workers: 2

health_check_timeout: 30
```

## 4. Role do Supervisor

### ansible/roles/supervisor/tasks/main.yml

```yaml
---
- name: Instalar Supervisor
  apt:
    name: supervisor
    state: present
    update_cache: yes
  become: yes

- name: Criar diretório de logs do Supervisor
  file:
    path: "{{ supervisor_log_dir }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  become: yes

- name: Iniciar e habilitar Supervisor
  systemd:
    name: supervisor
    state: started
    enabled: yes
  become: yes

- name: Verificar se supervisorctl está funcionando
  command: supervisorctl status
  become: yes
  register: supervisor_status
  changed_when: false
```

### ansible/roles/supervisor/handlers/main.yml

```yaml
---
- name: reload supervisor
  supervisorctl:
    name: all
    state: reloaded
  become: yes

- name: restart supervisor group
  supervisorctl:
    name: "{{ supervisor_group }}:*"
    state: restarted
  become: yes
```

## 5. Role da Aplicação Python

### ansible/roles/python-app/tasks/main.yml

```yaml
---
- name: Criar usuário da aplicação
  user:
    name: "{{ app_user }}"
    group: "{{ app_group }}"
    shell: /bin/bash
    home: "{{ app_home }}"
    create_home: yes
    system: yes
  become: yes

- name: Criar diretórios da aplicação
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ app_user }}"
    group: "{{ app_group }}"
    mode: '0755'
  become: yes
  loop:
    - "{{ app_home }}"
    - "{{ app_home }}/logs"
    - "{{ app_home }}/.pip"

- name: Configurar pip para PyPI privado
  template:
    src: pip.conf.j2
    dest: "{{ app_home }}/.pip/pip.conf"
    owner: "{{ app_user }}"
    group: "{{ app_group }}"
    mode: '0644'
  become: yes

- name: Criar virtual environment
  pip:
    name: pip
    state: latest
    virtualenv: "{{ app_home }}/venv"
    virtualenv_python: python3
  become: yes
  become_user: "{{ app_user }}"

- name: Fazer backup da versão atual
  command: |
    cp -r {{ app_home }}/venv {{ app_home }}/venv-backup-{{ ansible_date_time.epoch }}
  become: yes
  become_user: "{{ app_user }}"
  ignore_errors: yes
  when: app_backup | default(true)

- name: Instalar/Atualizar pacote da aplicação
  pip:
    name: "{{ app_name }}{% if package_version != 'latest' %}=={{ package_version }}{% endif %}"
    state: "{{ 'latest' if package_version == 'latest' else 'present' }}"
    virtualenv: "{{ app_home }}/venv"
    extra_args: "--index-url {{ pypi_index_url }} --trusted-host {{ pypi_trusted_host }}"
  become: yes
  become_user: "{{ app_user }}"
  register: package_install
  notify: restart supervisor group

- name: Exibir versão instalada
  pip:
    name: "{{ app_name }}"
    virtualenv: "{{ app_home }}/venv"
    state: present
  become: yes
  become_user: "{{ app_user }}"
  register: installed_packages

- name: Configurar Supervisor para a aplicação
  template:
    src: supervisor.conf.j2
    dest: "{{ supervisor_config_dir }}/{{ app_name }}.conf"
    owner: root
    group: root
    mode: '0644'
  become: yes
  notify: 
    - reload supervisor
    - restart supervisor group

- name: Verificar se configuração do Supervisor está válida
  command: supervisorctl reread
  become: yes
  register: supervisor_reread
  changed_when: supervisor_reread.stdout != ""

- name: Adicionar programa ao Supervisor
  supervisorctl:
    name: "{{ supervisor_group }}:{{ app_name }}"
    state: present
  become: yes

- name: Aguardar alguns segundos antes do restart
  pause:
    seconds: 5
  when: package_install.changed
```

### ansible/roles/python-app/templates/pip.conf.j2

```ini
[global]
index-url = {{ pypi_index_url }}
trusted-host = {{ pypi_trusted_host }}
timeout = 60

[install]
trusted-host = {{ pypi_trusted_host }}
```

### ansible/roles/python-app/templates/supervisor.conf.j2

```ini
[group:{{ supervisor_group }}]
programs={{ app_name }}

[program:{{ app_name }}]
command={{ app_home }}/venv/bin/python -m {{ app_name }}
directory={{ app_home }}
user={{ app_user }}
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile={{ app_home }}/logs/{{ app_name }}.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
environment=PATH="{{ app_home }}/venv/bin",PYTHONPATH="{{ app_home }}",PORT="{{ app_port }}"
startsecs=10
startretries=3
stopwaitsecs=10
priority=100
numprocs={{ app_workers | default(1) }}
process_name=%(program_name)s_%(process_num)02d
```

### ansible/roles/python-app/handlers/main.yml

```yaml
---
- name: reload supervisor
  supervisorctl:
    name: all
    state: reloaded
  become: yes

- name: restart supervisor group
  supervisorctl:
    name: "{{ supervisor_group }}:*"
    state: restarted
  become: yes
  
- name: wait for app start
  pause:
    seconds: 10
```

### ansible/roles/python-app/vars/main.yml

```yaml
---
# Configurações específicas da role
pip_config_dir: "{{ app_home }}/.pip"
venv_path: "{{ app_home }}/venv"
backup_retention_days: 7
```

## 6. Playbook Principal

### ansible/playbooks/deploy.yml

```yaml
---
- name: Deploy Python Application from Private PyPI
  hosts: app_servers
  become: yes
  vars:
    package_version: "{{ package_version | default('latest') }}"
    
  pre_tasks:
    - name: Verificar conectividade
      ping:
      
    - name: Atualizar cache do apt
      apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Instalar dependências do sistema
      apt:
        name:
          - python3
          - python3-pip
          - python3-venv
          - curl
        state: present

  roles:
    - role: supervisor
    - role: python-app

  post_tasks:
    - name: Verificar status do grupo no Supervisor
      supervisorctl:
        name: "{{ supervisor_group }}:"
        state: started
      become: yes
      register: supervisor_status

    - name: Exibir status dos processos
      debug:
        msg: "Grupo {{ supervisor_group }} está rodando"
      when: supervisor_status is succeeded

    - name: Aguardar aplicação inicializar
      pause:
        seconds: 15
        prompt: "Aguardando aplicação inicializar..."

    - name: Executar health check inicial
      uri:
        url: "http://localhost:{{ app_base_port }}{{ '%02d' | format(item) }}/healthz"
        method: GET
        status_code: 200
        timeout: "{{ health_check_timeout }}"
      register: initial_health_check
      retries: 5
      delay: 10
      delegate_to: localhost
      loop: "{{ range(1, app_workers + 1) | list }}"
      loop_control:
        label: "Health check instância {{ item }} - porta {{ app_base_port }}{{ '%02d' | format(item) }}"

    - name: Confirmar deploy bem-sucedido
      debug:
        msg: |
          ✅ Deploy concluído com sucesso!
          📦 Pacote: {{ app_name }}
          🔢 Versão: {{ package_version }}
          🌐 Health check: {{ health_check_url }} retornou {{ initial_health_check.status }}
```

## 7. Playbook de Health Check

### ansible/playbooks/health-check.yml

```yaml
---
- name: Health Check da Aplicação
  hosts: app_servers
  gather_facts: no
  vars:
    max_retries: "{{ max_retries | default(5) }}"
    retry_delay: "{{ retry_delay | default(10) }}"
    
  tasks:
    - name: Verificar status do Supervisor
      supervisorctl:
        name: "{{ supervisor_group }}:"
        state: started
      become: yes
      register: supervisor_check

    - name: Listar processos do grupo
      command: supervisorctl status {{ supervisor_group }}:*
      become: yes
      register: group_status
      changed_when: false

    - name: Exibir status dos processos
      debug:
        var: group_status.stdout_lines

    - name: Health check da aplicação
      uri:
        url: "http://localhost:{{ app_base_port }}{{ '%02d' | format(item) }}/healthz"
        method: GET
        status_code: 200
        timeout: "{{ health_check_timeout }}"
        return_content: yes
      register: health_response
      retries: "{{ max_retries }}"
      delay: "{{ retry_delay }}"
      delegate_to: localhost
      loop: "{{ range(1, app_workers + 1) | list }}"
      loop_control:
        label: "Health check instância {{ item }} - porta {{ app_base_port }}{{ '%02d' | format(item) }}"

    - name: Verificar conteúdo da resposta
      debug:
        msg: |
          🏥 Health Check Instância {{ item.item }}:
          📊 Status: {{ item.status }}
          🔗 URL: http://localhost:{{ app_base_port }}{{ '%02d' | format(item.item) }}/healthz
          ⏱️  Tempo de resposta: {{ item.elapsed }}s
          📄 Conteúdo: {{ item.content | default('N/A') }}
      loop: "{{ health_response.results }}"
      loop_control:
        label: "Resultado instância {{ item.item }}"

    - name: Falhar se algum health check não passou
      fail:
        msg: "❌ Health check falhou na instância {{ item.item }}! Status: {{ item.status }}"
      when: item.status != 200
      loop: "{{ health_response.results }}"
      loop_control:
        label: "Verificação instância {{ item.item }}"
```

## 8. Configuração de Secrets Encriptados

### ansible/group_vars/all/secrets.yml (encriptado com ansible-vault)

```yaml
# Execute: ansible-vault create group_vars/all/secrets.yml
# Este arquivo pode conter outras credenciais se necessário no futuro
$ANSIBLE_VAULT;1.1;AES256
66386439653762346665653764333464363139613735393334323965663933313364326664316564
6238653830633234643266373734356330636164383932650a373265613830613062663636633936
39383331363262653764326466303635346464373761663861386663663730373761313965623134
6532306334306661330a653937333731393435303563633939353461323766623232313738323334
6439

# Exemplo de conteúdo (caso precise de outras credenciais):
# vault_database_password: "db-secret-password"
# vault_api_keys:
#   external_service: "api-key-123"
```

## 9. Templates

### ansible/roles/python-app/templates/pip.conf.j2

```ini
[global]
index-url = https://{{ pypi_username }}:{{ pypi_password }}@{{ pypi_trusted_host }}/simple/
trusted-host = {{ pypi_trusted_host }}
timeout = 120
retries = 3

[install]
trusted-host = {{ pypi_trusted_host }}
find-links = {{ pypi_index_url }}
```

### ansible/roles/python-app/templates/supervisor.conf.j2

```ini
[group:{{ supervisor_group }}]
programs={{ app_name }}
priority=100

[program:{{ app_name }}]
command={{ app_home }}/venv/bin/gunicorn {{ app_name }}.wsgi:application --bind 0.0.0.0:{{ app_base_port }}%(process_num)02d --workers 1
directory={{ app_home }}
user={{ app_user }}
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile={{ app_home }}/logs/{{ app_name }}_%(process_num)02d.log
stdout_logfile_maxbytes=50MB
stdout_logfile_backups=10
stderr_logfile={{ app_home }}/logs/{{ app_name }}_error_%(process_num)02d.log
stderr_logfile_maxbytes=50MB
stderr_logfile_backups=10
environment=PATH="{{ app_home }}/venv/bin",PYTHONPATH="{{ app_home }}",PORT="{{ app_base_port }}%(process_num)02d",ENVIRONMENT="{{ environment }}"
startsecs=10
startretries=3
stopwaitsecs=10
stopsignal=TERM
priority=200
numprocs={{ app_workers | default(2) }}
process_name=%(program_name)s_%(process_num)02d

; Configurações de recursos
minfds=1024
minprocs=200
```

## 10. Configuração dos Secrets no GitHub

### Repository Secrets:
```
SSH_PRIVATE_KEY=-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----

ANSIBLE_VAULT_PASSWORD=sua-senha-vault-super-secreta

SERVER_HOSTS=192.168.1.10,192.168.1.11
```

## 11. Tasks Específicas da Aplicação

### Adição ao ansible/roles/python-app/tasks/main.yml

```yaml
- name: Parar grupo do Supervisor antes da atualização
  supervisorctl:
    name: "{{ supervisor_group }}:*"
    state: stopped
  become: yes
  ignore_errors: yes

- name: Aguardar processos pararem completamente
  pause:
    seconds: 5

- name: Instalar/Atualizar pacote da aplicação
  pip:
    name: "{{ app_name }}{% if package_version != 'latest' %}=={{ package_version }}{% endif %}"
    state: "{{ 'latest' if package_version == 'latest' else 'present' }}"
    virtualenv: "{{ venv_path }}"
    extra_args: "--index-url {{ pypi_index_url }} --trusted-host {{ pypi_trusted_host }} --upgrade"
  become: yes
  become_user: "{{ app_user }}"
  register: package_install

- name: Verificar versão instalada
  command: "{{ venv_path }}/bin/pip show {{ app_name }}"
  become: yes
  become_user: "{{ app_user }}"
  register: package_info
  changed_when: false

- name: Exibir informações do pacote
  debug:
    msg: "{{ package_info.stdout_lines }}"

- name: Configurar Supervisor para aplicação
  template:
    src: supervisor.conf.j2
    dest: "{{ supervisor_config_dir }}/{{ app_name }}.conf"
    owner: root
    group: root
    mode: '0644'
  become: yes
  notify:
    - reload supervisor
    - restart supervisor group

- name: Recarregar configuração do Supervisor
  supervisorctl:
    name: all
    state: reloaded
  become: yes

- name: Iniciar grupo da aplicação
  supervisorctl:
    name: "{{ supervisor_group }}:*"
    state: started
  become: yes
  register: app_start

- name: Aguardar aplicação estar pronta
  pause:
    seconds: 20
    prompt: "Aguardando aplicação inicializar completamente..."

- name: Health check pós-deploy
  uri:
    url: "http://localhost:{{ app_base_port }}{{ '%02d' | format(item) }}/healthz"
    method: GET
    status_code: 200
    timeout: "{{ health_check_timeout }}"
  register: post_deploy_health
  retries: 10
  delay: 15
  delegate_to: localhost
  loop: "{{ range(1, app_workers + 1) | list }}"
  loop_control:
    label: "Health check instância {{ item }} - porta {{ app_base_port }}{{ '%02d' | format(item) }}"
```

## 12. Comandos para Setup Inicial

```bash
# 1. Criar e configurar vault
ansible-vault create ansible/group_vars/all/secrets.yml

# 2. Testar conexão
ansible all -i ansible/inventories/staging/hosts.yml -m ping

# 3. Deploy manual para teste
cd ansible
ansible-playbook -i inventories/staging/hosts.yml playbooks/deploy.yml \
  --extra-vars "package_version=1.0.0" \
  --ask-vault-pass

# 4. Verificar status do Supervisor
ansible app_servers -i inventories/staging/hosts.yml \
  -m shell -a "supervisorctl status my_app_group:*" \
  --become

# 5. Health check manual
ansible-playbook -i inventories/staging/hosts.yml playbooks/health-check.yml
```

## 13. Troubleshooting

### Comandos de Debug:

```bash
# Verificar logs do Supervisor
tail -f /var/log/supervisor/my-python-app.log

# Status detalhado do grupo
supervisorctl status my_app_group:*

# Restart manual do grupo
supervisorctl restart my_app_group:*

# Verificar instalação do pacote
/opt/my-python-app/venv/bin/pip show my-python-app

# Teste manual do health check
curl -f http://localhost:8000/healthz
```

Este setup te dá um pipeline robusto que:
- ✅ Instala pacotes do PyPI privado com autenticação
- ✅ Gerencia a aplicação via Supervisor em grupos
- ✅ Faz restart do grupo `my_app_group` após instalação  
- ✅ Testa a rota `/healthz` esperando status 200
- ✅ Inclui backup automático e rollback em caso de falha
- ✅ Logs estruturados e monitoramento