# Teste inicial

Usei duas instâncias EC2 na AWS para um teste primário. Numa delas instalei o Ansible realizei os procedimentos de configuração a seguir, enquanto a outra serviu para receber a instalação via playbook.

## Ansible server

Preparação e instalação do serviço:

```shell
sudo apt update
sudo apt upgrade
sudo reboot
sudo apt install software-properties-common
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
```

[Referência](https://imasters.com.br/devsecops/automacao-e-provisionamento-agil-com-ansible)

## Inventário

Adicionando hosts que receberão a instalação:

```shell
sudo mv /etc/ansible/hosts /etc/ansible/hosts.bkp
sudo vim /etc/ansible/hosts
```

```vim
# Este é o inventário de hosts.

# Teste Instruct
[Instruct]
## Adicionar endereço IP do servidor.

# AWS fileserver
[AWS]
## Adicionar endereço IP do servidor.

[local]
127.0.0.1
```

[Referência](https://churrops.io/2018/02/04/primeiros-passos-com-ansible-configurando-e-gerenciando-nginx)

## Chaves de conexão

```shell
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@ENDERECO_IP
```

Testei a conexão com:

`ansible Instruct -m ping`

[Referência](https://hvops.com/articles/ansible-post-install)

## Atualizando hosts

A fim de testar e entender melhor o funcionamento do processo, inicialmente, criei um playbook `update.yml` com as seguintes instruções:

```vim
---
- name: "Atualização de pacotes do servidor."
- hosts: AWS
  become: true
  tasks:
    - name: Update apt repo and cache on all Debian/Ubuntu boxes
      apt: update_cache=yes force_apt_get=yes cache_valid_time=3600

    - name: Upgrade all packages on servers
      apt: upgrade=dist force_apt_get=yes
```

Executei chamando:

```shell
ansible-playbook -i hosts update.yml
```

[Referência](https://www.cyberciti.biz/faq/ansible-apt-update-all-packages-on-ubuntu-debian-linux)

# O projeto teste

## Criando a estrutura de diretórios

Executei o comando `ansible-galaxy install atosatto.minio` e a estrutura de diretórios da aplicação foi montada em `/home/.ansible/roles/atosatto.minio`.

Dentro de `tasks`, criei o playbook de instalação do MinIO.

```shell
vim minioserver.yml
```

Com as seguintes instruções:

```vim
---
- name: "Instalar MinIO"
  hosts: AWS
  become: true
  roles:
    - atosatto.minio
  vars:
    minio_server_datadirs:
      - "/S3bucket"
    minio_access_key: "inserir_chave_aqui"
    minio_secret_key: "inserir_chave_aqui"
```

Executei chamando:

```shell
ansible-playbook -i hosts minioserver.yml
```

## Verificando a aplicação

A aplicação subiu com sucesso e pude acessar usando http://ENDERECO_IP:9091  
Loguei usando as credenciais fornecidas no playbook e criei um bucket chamado `teste`.

# Problemas encontrados e construção interrompida

> A partir desse momento encontrei dificuldades para construir as demais ações do teste e não tive sucesso.

Criei o playbook de instalação das dependências do MinIO uploader.

```shell
vim miniouploaderdep.yml
```

Com as seguintes instruções:

```vim
---
- name: "Instalação de dependências do MinIO uploader."
  hosts: AWS
  become: true
  tasks:
    - name: "Dependências de pacotes apt."
      action: apt pkg={{ item }} state=installed
      with_items:
        - python3
        - python3-pip

    - name: "Dependências de pacotes pip."
      command: "pip3 install pipenv"

    - name: "Inicializando o virtualenv."
      command: "pipenv --python 3 install --system --deploy"
```

> Nesse momento entendi que o endpoint deveria ser compilado dentro de um Virtualenv usando os arquivos dentro do ZIP que foi compartilhado. Também alterei a sintaxe para `shell` ao invés de `command`.

Alterei as instruções para:

```vim
---
- name: "Instalação de dependências do MinIO uploader."
  hosts: AWS
  become: true
  tasks:
    - name: "Dependências de pacotes apt."
      action: apt pkg={{ item }} state=installed
      with_items:
        - python3
        - python3-pip

    - name: "Copiando arquivos do endpoint MinIO uploader"
      copy:
        src: /home/ubuntu/minio_uploader
        dest: /home/ubuntu

    - name: "Dependências de pacotes pip."
      shell: "pip3 install pipenv"

    - name: "Inicializando o virtualenv."
      shell: "pipenv --python 3 install --system --deploy"
```

> Mas também não tive sucesso. Os próximos passos não chegeram a ser testados, precisam ser refinados, mas seriam a continuidade do exercício.

Adicionando as variáveis de ambiente:

```shell
vim ~/.ansible/roles/atosatto.minio/vars/main.yml
```

```vim
---
go_arch_map:
  i386: '386'
  x86_64: 'amd64'
  aarch64: 'arm64'
  armv7l: 'arm'
  armv6l: 'arm6vl'

# Variáveis para MinIO uploader
S3_URL="http://ENDERECO_IP:9091"
S3_ACCESS_KEY="inserir_chave_aqui"
S3_SECRET_KEY="inserir_chave_aqui"
```

[Referência 1 - Página do projeto](https://galaxy.ansible.com/atosatto/minio)  
[Referência 2 - Guia de regras](https://docs.debops.org/en/master/ansible/roles/minio/deployment-guide.html)  
[Referência 3 - Problemas com pipenv no Ansible](https://github.com/pypa/pipenv/issues/363)  
[Referência 4 - Projeto consultado como referência](https://gig.tech/docs/minio-appliance-on-gig-edge-cloud-with-terraform-and-ansible)

## Automação do deploy usando Travis

Não cheguei a implementar.

[Referência - Implementando continous delivery](https://cloud.google.com/solutions/continuous-delivery-with-travis-ci)

Links que também foram úteis:  
[Referência 1 - Sintaxe SCP](http://www.hypexr.org/linux_scp_help.php)  
[Referência 2 - Regras da construção de playbooks](https://docs.ansible.com/ansible/2.3/playbooks_roles.html)  
[Referência 3 - Projeto exemplo no Git](https://github.com/lean-delivery/ansible-role-minio)  
[Referência 4 - Projeto exemplo da UNICAMP](https://www.ic.unicamp.br/~william/howto/ansible.html)

# Procedimento alternativo - execução manual

Com a finalidade de testar o funcionamento do endpoint, realizei a cópia de arquivos e construção do virtual env de maneira tradicional com os seguintes procedimentos:

## Cópia de arquivos

```shell
scp -r /home/ubuntu/minio_uploader ubuntu@ENDERECO_IP:/home/ubuntu
```

## Criando o ambiente

```shell
sudo apt install python3 python3-pip
pipenv --python 3 install --system --deploy
```

## Adicionando variáveis de ambiente

```shell
export S3_URL="http://ENDERECO_IP:9091"
export S3_ACCESS_KEY="inserir_chave_aqui"
export S3_SECRET_KEY="inserir_chave_aqui"
```

## Executando a aplicação

```shell
gunicorn --bind 0.0.0.0:5000 wsgi:app
```

## Verificando o funcionamento

Tive sucesso em acessar http://ENDERECO_IP:5000  
E tentei enviar um arquivo de teste para o bucket criado usando o comando abaixo:

```shell
curl -X POST -F "file=@/home/ubuntu/arquivoteste" http://ENDERECO_IP:5000/minio-upload/teste/arquivoteste
```

# Fim do exercício

Infelizmente não consegui concluir todo o processo e com a semana iniciando, dado minha atual ocupação profissional, não terei condições estudar e aplicar novas abordagens dentro do prazo esperado.
