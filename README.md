# Terraform + Ansible + AWS

Provisionamento de infraestrutura web na AWS com Terraform e configuração automatizada com Ansible, utilizando state remoto S3 com DynamoDB para lock, workspaces (dev/prod), e Ansible Vault para dados sensíveis.

---

# Requisitos atendidos

- VPC com subnet pública, Internet Gateway e route table
- Security Group com SSH restrito ao IP do aluno e porta 3000 aberta para a aplicação
- Instância EC2 com Amazon Linux 2023 (AMI dinâmica)
- Ansible responsável por instalar Docker e executar container `getting-started-app`
- Integração Terraform → Ansible via `null_resource` com `local-exec`
- State remoto com backend S3 (bucket versionado) e DynamoDB para lock
- Workspaces dev e prod com instance_type variável (t2.micro em dev, t3.micro em prod)
- Tags consistentes: Name, Curso, Ambiente
- Código formatado e validado (`terraform fmt -check`, `terraform validate`)
- Nenhuma credencial commitada
- Variável sensível protegida com `ansible-vault`

---

# Pré-requisitos

### Pré-requisitos
- Terraform >= 1.0 instalado
- Ansible >= 2.9 instalado
- AWS CLI configurada com credenciais (ou variáveis de ambiente)

---
## Automação com Makefile

Simplifica a execução de tarefas complexas. Comandos como make setup, make apply-dev, e make destroy agora orquestram todo o fluxo, desde a criação da infraestrutura base até a destruição dos recursos.

Scripts Modulares:

    bootstrap.sh: Cria a infraestrutura base (bucket S3, tabela DynamoDB e chave SSH), que é um pré-requisito para o Terraform.

    destroy-all.sh: Garante a remoção completa e segura de todos os recursos, incluindo a limpeza de versões do bucket S3, resolvendo o problema comum de BucketNotEmpty.

    generate-tfvars.sh: Automatiza a criação do arquivo terraform.tfvars, preenchendo automaticamente seu IP público e o nome do Key Pair, reduzindo erros manuais.

Fluxo de Trabalho Robusto: A combinação do Makefile com os scripts cria um fluxo de trabalho padronizado, testável e fácil de replicar para uma primeira criação de ambiente.

## Aplicação Deployada

Este projeto implanta a aplicação **Cadastro de Nomes com GO & PostgreSQL**, cujo código-fonte está disponível no repositório: [dhuberto/docker](https://github.com/dhuberto/docker).

### Sobre a Aplicação

A aplicação é uma **aplicação web monolítica com renderização no servidor (SSR)**, desenvolvida para demonstrar na prática como construir, estruturar e containerizar um ecossistema focado em **Go** e banco de dados **PostgreSQL** utilizando as melhores práticas de Docker.

**Funcionalidades principais:**
- **Cadastro Simples:** Permite o envio de nomes através de um formulário web dinâmico.
- **Renderização no Servidor (SSR):** Lista em tempo real os nomes salvos no banco de dados, ordenados pelos mais recentes.
- **Monitoramento:** Exibe o status da conexão com o banco de dados diretamente na tela.

### Arquitetura da Aplicação no Projeto

A aplicação é executada em dois containers Docker orquestrados dentro da instância EC2 provisionada:

1. **Container `postgres-db`**: Banco de dados PostgreSQL, que persiste os dados em um volume Docker.
2. **Container `nodejs-app`**: Aplicação Go que se conecta ao banco de dados via rede interna do Docker.

A comunicação entre os containers é feita através de uma rede Docker interna chamada `app-network`, garantindo isolamento e resolução de nomes (o container da aplicação encontra o banco pelo hostname `postgres-db`).

### Como o código da aplicação chega na EC2

O fluxo de entrega do código é feito da seguinte forma:

1. **Repositório local**: O código da aplicação está no repositório [dhuberto/docker](https://github.com/dhuberto/docker).
2. **Clone local**: Durante a configuração do projeto, o repositório é clonado para dentro do diretório do projeto `terraform_ansible_aws`:
   ```bash
   git clone https://github.com/dhuberto/docker.git app-nodejs

---
## Estrutura do projeto

```
~/terraform_ansible_aws/
├── ansible/
│   ├── ansible.cfg                # Configurações globais do Ansible
│   ├── playbook.yml               # Playbook principal que configura a instância
│   └── vault.yml                  # Variáveis sensíveis (criptografadas com ansible-vault)
│
├── scripts/
│   ├── bootstrap.sh               # Cria bucket S3, DynamoDB e Key Pair via AWS CLI
│   ├── destroy-all.sh             # Remove todos os recursos (Terraform + AWS)
│   └── generate-tfvars.sh         # Gera o arquivo terraform.tfvars automaticamente
│
├── terraform/
│   ├── main.tf                    # Recursos AWS (VPC, subnet, EC2) + integração com Ansible
│   ├── variables.tf               # Declaração de todas as variáveis do projeto
│   ├── outputs.tf                 # Saídas (IP público, URL, etc.)
│   ├── providers.tf               # Configuração do provedor AWS e backend remoto S3
│   └── terraform.tfvars.example   # Exemplo de variáveis (copiar para .tfvars)
│
├── .gitignore                     # Arquivos ignorados pelo Git (state, .pem, .tfvars, etc.)
├── Makefile                       # Automação de tarefas (make setup, make apply-dev, etc.)
└── README.md                      # Documentação completa do projeto
```
---
## Arquitetura do projeto
```
+-----------------------------+
|                             |
|    (Control Node)           |
+-----------------------------+
|                             |
|  Terraform                  |
|  (make apply-dev/prod)      |
|  |                          |
|  +---> 1. Provisiona        |
|  |                          |
|  +---> 2. Executa local-exec|
|         (sleep 90)          |
|         (chama Ansible)     |
|                             |
|  Ansible                    |
|  (playbook.yml)             |
|  |                          |
|  +---> 3. Configura via SSH |
|                             |
+--------------+--------------+
               |
               | (SSH)
               v
+------------------------------------------------------+
|                    AMAZON WEB SERVICES (AWS)         |
+------------------------------------------------------+
|                                                      |
|  +------------------------------------------------+  |
|  |               VPC (10.0.0.0/16)                |  |
|  |                                                |  |
|  |  +------------------------------------------+  |  |
|  |  |        Subnet Pública (10.0.1.0/24)      |  |  |
|  |  |                                          |  |  |
|  |  |  +------------------------------------+  |  |  |
|  |  |  |      Security Group                |  |  |  |
|  |  |  |  - Porta 22 (SSH): SEU_IP/32       |  |  |  |
|  |  |  |  - Porta 3000 (App): 0.0.0.0/0     |  |  |  |
|  |  |  +------------------------------------+  |  |  |
|  |  |                                          |  |  |
|  |  |  +------------------------------------+  |  |  |
|  |  |  |      EC2 Instância (t3.micro)      |  |  |  |
|  |  |  |   - Amazon Linux 2023 (AMI)        |  |  |  |
|  |  |  |   - Key Pair: vockey               |  |  |  |
|  |  |  |   - user_data: Habilita SSH        |  |  |  |
|  |  |  |                                    |  |  |  |
|  |  |  |  +------------------------------+  |  |  |  |
|  |  |  |  |  DOCKER ENGINE (Instalado)   |  |  |  |  |
|  |  |  |  |  - Instalado pelo Ansible    |  |  |  |  |
|  |  |  |  |  - Executa o container       |  |  |  |  |
|  |  |  |  +------------------------------+  |  |  |  |
|  |  |  |                                    |  |  |  |
|  |  |  |  +------------------------------+  |  |  |  |
|  |  |  |  |  CONTAINER (getting-started) |  |  |  |  |
|  |  |  |  |  - Imagem: docker/getting-   |  |  |  |  |
|  |  |  |  |    started                   |  |  |  |  |
|  |  |  |  |  - Porta: 3000:80 (host:     |  |  |  |  |
|  |  |  |  |    container)                |  |  |  |  |
|  |  |  |  |  - Variável: ADMIN_PASSWORD  |  |  |  |  |
|  |  |  |  |    (do vault)                |  |  |  |  |
|  |  |  |  +------------------------------+  |  |  |  |
|  |  |  +------------------------------------+  |  |  |
|  |  +------------------------------------------+  |  |
|  +------------------------------------------------+  |
|                                                      |
|  +------------------------------------------------+  |
|  |    INTERNET GATEWAY (IGW)                      |  |
|  |    - Conecta a VPC à Internet                  |  |
|  +------------------------------------------------+  |
|                                                      |
|  +------------------------------------------------+  |
|  |    ROUTE TABLE (Pública)                       |  |
|  |    - Rota: 0.0.0.0/0 -> IGW                    |  |
|  +------------------------------------------------+  |
|                                                      |
|  +------------------------------------------------+  |
|  |    STATE REMOTO (S3 + DynamoDB)                |  |
|  |    - S3: danilo-terraform-backend-2026         |  |
|  |    - DynamoDB: terraform-locks                 |  |
|  |    - Versionamento e Lock para workspaces      |  |
|  +------------------------------------------------+  |
|                                                      |
+------------------------------------------------------+

+------------------------------------------------------+
|                INTERNET (USUÁRIO)                    |
+------------------------------------------------------+
|                                                      |
|  Acessa a aplicação via navegador:                   |
|  http://<IP_PUBLICO_EC2>:3000                        |
|                                                      |
+------------------------------------------------------+
```

---

## Preparação do Ambiente

## Linux (Debian/Ubuntu)
Comando: 
```bash 
sudo apt update
```
Comando: 
```bash 
sudo apt install -y git
```

## Instalação do Terraform
Comando: 
```bash 
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
```
Comando: 
```bash 
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```
Comando: 
```bash 
sudo apt update
```
Comando: 
```bash 
sudo apt install -y terraform
```
## Instalação do Ansible
```bash 
sudo apt update
sudo apt install -y ansible python3-pip
```

## Crie conta AWS em aws.amazon.com e Instale do AWS CLI v2
Comando: 
```bash 
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```
Comando: 
```bash 
unzip awscliv2.zip
```
Comando: 
```bash 
sudo ./aws/install
```

## Execute o aws configure para colocar as credenciais:
Comando: 
```bash 
aws configure
```

```bash

O comando vai pedir cinco informações, uma por vez:

# AWS Access Key ID [None]: <SUA_ACCESS_KEY_ID>
# AWS Secret Access Key [None]: <SUA_SECRET_ACCESS_KEY>
# AWS Session Token [None]: <SEU_SESSION_TOKEN>
# Default region name [None]: us-east-1
# Default output format [None]: json

# Serão criados os arquivos:
~/.aws/credentials
~/.aws/config
# São os tokens de login e senha aws
```
## Checklist final de verificação
Comandos: 
```bash 
git --version
```
```bash
terraform -version
```
```bash
ansible --version
```
```bash
aws --version
```
```bash
aws sts get-caller-identity
```

O último comando confirma que suas credenciais IAM estão configuradas corretamente e mostra qual
usuário está autenticado. Saída esperada:
```bash
{
"UserId": "AIDAEXAMPLE123456789",
"Account": "123456789012",
"Arn": "arn:aws:iam::123456789012:user/devops-iac-curso"
}
```


## Configuração inicial

### Clone o repositório
Baixar projeto, Comando: 
```bash
git clone https://github.com/dhuberto/terraform_ansible_aws.git
```
BAcessar o projeto, Comando: 
```bash
cd ~/terraform_ansible_aws
```
### Tornar scripts executáveis
Comando: 
```bash
chmod +x scripts/*.sh
```

### Baixar a aplicação que será deployada no conteiner no EC2
Comando: 
```bash
git clone https://github.com/dhuberto/docker.git app-go
```


### Configurar Ansible Vault
Comandos: 
```bash
cd ~/terraform_ansible_aws/ansible
echo "admin123" > vault_pass.txt
chmod 600 vault_pass.txt
echo "admin_password: 'admin123'" > vault.yml
echo "postgres_password: 'postgres123'" >> vault.yml
ansible-vault encrypt vault.yml

#Digite a senha: admin123
#repita: admin123

Senhas de exemplo;
```


### Executar configuração completa
Comando: 
```bash
cd ~/terraform_ansible_aws
make setup
```


### Inicializar
Comando: 
```bash
cd terraform && terraform init -reconfigure
```

### Validar
Comando: 
```bash
terraform validate
```

### voltar para o diretório principal
Comando: 
```bash
cd ~/terraform_ansible_aws
```

### Crie os workspaces dos ambientes separados e aplique:
### Levantar o ambiente DEV
Comando: 
```bash
make apply-dev
```

Output:

```text
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

application_url = "http://34.239.42.198:3000"
instance_public_dns = "ec2-34-239-42-198.compute-1.amazonaws.com"
instance_public_ip = "34.239.42.198"
security_group_id = "sg-01555b608ec8c9a5f"
workspace_atual = "dev"

```
<img width="1104" height="714" alt="{F403E928-0C32-4180-B777-C6B244FB3404}" src="https://github.com/user-attachments/assets/2e2427bc-581a-4e57-bf9c-3000a16114ef" />


### Levantar o ambiente PROD
Comando: 
```bash
make apply-prod
```

Output:

```text
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

application_url = "http://3.81.56.85:3000"
instance_public_dns = "ec2-3-81-56-85.compute-1.amazonaws.com"
instance_public_ip = "3.81.56.85"
security_group_id = "sg-063b8754e3c1ea49f"
workspace_atual = "prod"

```
<img width="1088" height="671" alt="{5D5DE59F-26E4-4B39-AF19-99BDBB8794D1}" src="https://github.com/user-attachments/assets/739cffe6-5a32-4a55-ba19-e911c85e9964" />

### O Resultado são os Output com os endereços de acesso de cada ambiente

## Caso queira Destruir (apagar tudo)
Ver ajuda:

Comando: 
```bash
cd ~/terraform_ansible_aws
```

```bash
make help
```

```markdown
## Comandos Makefile Disponíveis

| Comando | Descrição |
|---------|-----------|
| `make help` | Exibe todos os comandos disponíveis. |
| `make setup` | Configura o ambiente completo (bootstrap + tfvars). |
| `make apply-dev` | Aplica a infraestrutura no workspace `dev`. |
| `make apply-prod` | Aplica a infraestrutura no workspace `prod`. |
| `make destroy` | Remove todos os recursos da AWS e limpa o ambiente local. |
| `make clean` | Remove apenas os arquivos locais (state, .terraform). |

```



Comando para apagar tudo: 
```bash
make destroy
```
<small>
    
Output:

```text
ATENÇÃO: Isso vai destruir TODOS os recursos!
Digite 'sim' para confirmar: sim
==========================================
DESTRUINDO TODOS OS RECURSOS
==========================================

ATENCAO: Isso vai destruir TODOS os recursos da AWS!
Tem certeza? (digite 'sim' para confirmar): sim

Destruindo recursos do Terraform...
  - Destruindo workspace dev...
Switched to workspace "dev".
data.aws_ami.amazon_linux: Reading...
data.aws_availability_zones.available: Reading...
aws_vpc.main: Refreshing state... [id=vpc-092fcd60930d7820c]
data.aws_availability_zones.available: Read complete after 0s [id=us-east-1]
data.aws_ami.amazon_linux: Read complete after 1s [id=ami-02b3d83d84b07786d]
aws_subnet.public: Refreshing state... [id=subnet-0619a821070a041e9]
aws_internet_gateway.igw: Refreshing state... [id=igw-0464e854d6f7a2256]
aws_security_group.web_sg: Refreshing state... [id=sg-038144a3a93635f02]
aws_route_table.public: Refreshing state... [id=rtb-008ae45d3cb29870c]
aws_instance.web: Refreshing state... [id=i-09a2af576edbaea89]
aws_route_table_association.public: Refreshing state... [id=rtbassoc-03cad837c6efc60cb]
null_resource.run_ansible: Refreshing state... [id=4269767512034835537]
Plan: 0 to add, 0 to change, 8 to destroy.

Changes to Outputs:
  - application_url     = "http://35.173.196.218:3000" -> null
  - instance_public_dns = "ec2-35-173-196-218.compute-1.amazonaws.com" -> null
  - instance_public_ip  = "35.173.196.218" -> null
  - security_group_id   = "sg-038144a3a93635f02" -> null
  - workspace_atual     = "dev" -> null



Destroy complete! Resources: 8 destroyed.
  - Destruindo workspace prod...
Switched to workspace "prod".

Changes to Outputs:
  - application_url     = "http://54.173.176.87:3000" -> null
  - instance_public_dns = "ec2-54-173-176-87.compute-1.amazonaws.com" -> null
  - instance_public_ip  = "54.173.176.87" -> null
  - security_group_id   = "sg-0e5cd325b587de0a9" -> null
  - workspace_atual     = "prod" -> null
nul

Destroy complete! Resources: 8 destroyed.
Recursos do Terraform destruidos!

Removendo bucket S3...
Removendo todas as versoes do bucket...
{
    "Deleted": [
        {
            "Key": "env:/dev/terraform_ansible_aws/terraform.tfstate",
            "VersionId": "tEbBY1CKzJFPHiuqNtdhC8CU21gvGksI"
        },

remove_bucket: danilo-terraform-backend-2026
Bucket removido: danilo-terraform-backend-2026

Removendo tabela DynamoDB...

Removendo Key Pair da AWS...
{
    "Return": true,
    "KeyPairId": "key-07175a0377fa6d07a"
}
Key Pair removido: vockey

Removendo chave privada local...
Chave privada removida: /home/danilo/.ssh/vockey.pem

Removendo estado local do Terraform...
Estado local removido

==========================================
DESTRUICAO COMPLETA CONCLUIDA!
==========================================

```
</small>

### Apagar o Diretório do Projeto na maquina Local

Comando:
```bash
cd ~ && rm -rf terraform_ansible_aws
```
Tudo Limpo!

[![Download ZIP](https://img.shields.io/badge/Download-ZIP-blue?style=for-the-badge&logo=github)](https://github.com/dhuberto/terraform_ansible_aws/archive/main.zip)
