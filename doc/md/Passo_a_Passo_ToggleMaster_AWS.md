# ToggleMaster — Passo a passo de instalação e execução na AWS

Este documento descreve a infraestrutura criada para executar a aplicação ToggleMaster em uma instância Amazon EC2 e armazenar seus dados em um banco Amazon RDS PostgreSQL.

## 1. Arquitetura implantada

- **VPC:** `togglemaster-vpc` — CIDR `10.40.0.0/16`
- **Subnet pública da aplicação:** `subnet-app-pub-a` — `10.40.30.0/24`
- **Subnet privada do banco:** `subnet-bd-priv-a` — `10.40.31.0/24`
- **Internet Gateway:** `gateway-togglemaster-public`
- **EC2:** Amazon Linux, tipo `t2.micro`, IP privado `10.40.30.164`, Elastic IP `54.81.198.105`, volume `gp3` de 8 GiB
- **Security Group da aplicação:** `sg-togglemaster-app`
- **RDS:** identificador `togglemaster-database`, classe `db.t3.micro`, 20 GiB, criptografia habilitada, Single-AZ
- **AWS Secrets Manager:** segredos de configuração da aplicação e credenciais do banco

## 2. Criar a VPC

1. No console AWS, abra **VPC > Your VPCs > Create VPC**.
2. Crie a VPC com CIDR `10.40.0.0/16`.
3. Adicione a tag para facilitar o gerenciamento de custos:
   - `Name`: `togglemaster-vpc`
4. Confirme a criação.

## 3. Criar as sub-redes

Na VPC `togglemaster-vpc`, crie as sub-redes:

| Nome | CIDR | Uso | Região |
|---|---|---|---|
| `subnet-app-pub-a` | `10.40.30.0/24` | EC2 da aplicação | `us-east-1a` |
| `subnet-bd-priv-a` | `10.40.31.0/24` | RDS PostgreSQL | `us-east-1a` |
| `subnet-bd-priv-b` | `10.40.100.0/28` | RDS PostgreSQL 2 | `us-east-1b` |

A sub-rede do RDS deve permanecer privada e sem rota direta para um Internet Gateway.

## 4. Criar e associar o Internet Gateway

1. Em **VPC > Internet Gateways**, crie `gateway-togglemaster-public`.
2. Adicione a tag `Name` com o valor `gateway-togglemaster-public`.
3. Associe o gateway à VPC `togglemaster-vpc`.
4. Crie a primeira tabela de rotas, associando à sub-rede pública e adicione a rota:
| Destino | Alvo |
|---|---|
| `10.40.0.0/16` | local |
| `0.0.0.0/0` | `igw-publica` |
5. Associe essa tabela de rotas à `subnet-app-pub-a`.
7. Crie a segunda tabela de rotas, associando a sub-rede privada e adicione as rotas:
| Destino | Alvo |
|---|---|
| `10.40.0.0/16` | local |
8. Associe essa tabela de rotas às subredes `subnet-bd-priv-a` e `subnet-bd-priv-b`.
9. Garanta que a instância EC2 receba um endereço público ou Elastic IP.

**Observação:** criar um Internet Gateway e associá-lo à VPC, por si só, não torna uma sub-rede pública. A rota `0.0.0.0/0` para o gateway e a associação da tabela de rotas são necessárias. A sub-rede privada do RDS não deve usar essa rota.

## 5. Criar o Security Group da EC2

Crie o Security Group `sg-togglemaster-app` na VPC `togglemaster-vpc`.

### Regras de entrada

| Protocolo | Porta | Origem | Observação |
|---|---:|---|---|
| TCP | 80 | `0.0.0.0/0` | HTTP |
| TCP | 443 | `0.0.0.0/0` | HTTPS |
| TCP | 22 | Meu IP (`myip`) | SSH administrativo restrito ao IP de origem |
| TCP | 5000 | `0.0.0.0/0` | Porta da aplicação, conforme configuração informada |

### Regra de saída

- Todo o tráfego para `0.0.0.0/0`.

## 6. Criar a instância EC2

1. Abra **EC2 > Instances > Launch instances**.
2. Escolha uma imagem Amazon Linux.
3. Selecione o tipo `t2.micro`.
4. Configure a rede:
   - VPC: `togglemaster-vpc`
   - Sub-rede: `subnet-app-pub-a`
5. Crie ou selecione um par de chaves para SSH e armazene a chave privada com segurança.
6. Associe o Security Group `sg-togglemaster-app`.
7. Configure o armazenamento como `gp3`, 8 GiB.
8. Inicie a instância.
9. Associe o Elastic IP (no meu caso `54.81.198.105`) à instância ou à interface de rede correta.
10. Registre o ID da instância e confirme que o status passou para **Running** e as verificações de status foram aprovadas.

## 7. Atribuir a função/permissão IAM

A instância precisa de credenciais AWS para consultar o Secrets Manager. Associe à EC2 o perfil de instância (IAM Instance Profile) autorizado pelo ambiente do laboratório, referido nas anotações como **Lab Role**.

A política deve permitir somente as ações necessárias, principalmente `secretsmanager:GetSecretValue` nos segredos específicos utilizados e, se os segredos usam uma chave KMS gerenciada pelo cliente, a permissão de descriptografia necessária.

## 8. Conectar-se à EC2 via SSH

A partir de uma máquina cujo IP esteja permitido no Security Group:

```bash
chmod 400 /caminho/para/sua-chave.pem
ssh -i /caminho/para/sua-chave.pem ec2-user@54.81.198.105
```

## 9. Atualizar o sistema e instalar Docker

Na instância Amazon Linux:

```bash
sudo yum update -y
sudo dnf install -y docker jq awscli
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

A instalação de `jq` e AWS CLI é necessária para o script descrito adiante, caso esses utilitários não estejam presentes na imagem.

Para aplicar a associação do usuário ao grupo `docker`, encerre a sessão SSH e conecte-se novamente. Como alternativa para a sessão atual:

```bash
newgrp docker
```

Confirme a instalação:

```bash
docker --version
docker ps
aws --version
jq --version
```

### Instalar os plugins Buildx e Compose

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins

sudo curl -fL \
  https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-amd64 \
  -o /usr/local/lib/docker/cli-plugins/docker-buildx

sudo curl -fL \
  https://github.com/docker/compose/releases/download/v2.29.7/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose

sudo chmod +x \
  /usr/local/lib/docker/cli-plugins/docker-buildx \
  /usr/local/lib/docker/cli-plugins/docker-compose
```

Valide:

```bash
docker buildx version
docker compose version
```

## 10. Transferir o código para a EC2

Baixe o código e utilize a branch: release/TCL01
Transfira o código para a instância por SFTP, conforme realizado pelo grupo. Exemplo a partir da máquina local:

```bash
sftp -i /caminho/para/sua-chave.pem ec2-user@54.81.198.105
```

Dentro da sessão SFTP, envie o diretório/arquivo do projeto para a pasta definida na EC2, por exemplo `~/toggle-master`. Depois, na EC2:

```bash
cd ~/toggle-master
ls -la
```

## 11. Criar o banco Amazon RDS PostgreSQL

No console AWS, acesse **RDS > Databases > Create database** e configure, conforme a implantação realizada:

- Identificador: `togglemaster-database`
- Mecanismo: PostgreSQL
- Classe: `db.t3.micro` (2 vCPU, 1 GiB de RAM conforme os dados fornecidos)
- Armazenamento: SSD de uso geral, 20 GiB
- Criptografia: habilitada com a chave KMS indicada (`aws/rds`, conforme anotação)
- Multi-AZ: desabilitado (Single-AZ)
- Usuário principal: `togglemaster_app`
- Autenticação IAM do banco: desabilitada
- Acesso público: desabilitado
- VPC: `togglemaster-vpc`
- Sub-redes: `subnet-bd-priv-a` | `subnet-bd-priv-b`
- Credenciais de acesso: Armazenar em Aws Secrets Manager

### Subnet group do RDS

O Amazon RDS normalmente exige um **DB subnet group** com sub-redes em pelo menos duas zonas de disponibilidade.

### Configurar acesso de rede ao RDS

Crie/associe um Security Group para o banco, por exemplo `sg-togglemaster-db`, com entrada:

| Protocolo | Porta | Origem |
|---|---:|---|
| TCP | 5432 | Security Group `sg-togglemaster-app` |

Não use `0.0.0.0/0` como origem para a porta do PostgreSQL. O grupo do RDS deve permitir conexão somente a partir do Security Group da EC2.

Após o RDS ficar disponível, copie o **endpoint DNS** e confirme a porta `5432`. Coloque esses valores no segredo de configuração; não use o IP do banco, pois o endpoint pode mudar.

## 12. Armazenar configuração e credenciais no Secrets Manager

As anotações indicam dois segredos: um de configuração da aplicação, identificado como `dev/app/togglemaster`, e outro contendo usuário e senha do banco. Use os identificadores reais dos segredos na configuração do script.

### Segredo de configuração da aplicação

Exemplo de estrutura JSON (preencha os valores no Secrets Manager, não neste documento):

```json
{
  "TZ": "America/Sao_Paulo",
  "DB_NAME": "postgres",
  "DB_PORT": "5432",
  "DB_HOST": "<endpoint-rds>"
}
```

### Segredo das credenciais do banco

```json
{
  "username": "togglemaster_app",
  "password": "<senha armazenada no Secrets Manager>"
}
```

## 13. Preparar o script de deploy

No diretório do projeto, crie ou edite `scripts/02-deploy-app.sh`. O exemplo abaixo carrega os valores dos segredos sem imprimi-los. Substitua `idcredencial1` e `idcredencial2` pelos nomes/ARNs reais dos dois segredos e complete a linha que executa o container conforme o comando utilizado pelo projeto.

```bash
mkdir -p ./scripts
```

```bash
vim ./scripts/02-deploy-app.sh
```

```bash
set -euo pipefail
set +x
umask 077

REGION="us-east-1"
API_SECRET_ID="id_secrets_api" #substituir pelo id do secrets correspondente à aplicação
DB_SECRET_ID="id_secrets_rds" #substituir pelo id do secrets correspondente ao rds
CAMINHO_GIT="/path/to/toggle-master-monolith" #substituir pelo caminho do repo git

SECRET_DIR="/run/togglemaster-secrets"
sudo install -d -m 700 "$SECRET_DIR"

API_TMP="$(sudo mktemp "$SECRET_DIR/api.XXXXXX")"
DB_TMP="$(sudo mktemp "$SECRET_DIR/db.XXXXXX")"
PASS_TMP="$(sudo mktemp "$SECRET_DIR/pass.XXXXXX")"
trap 'sudo rm -f -- "$API_TMP" "$DB_TMP" "$PASS_TMP"' EXIT

sudo sh -c 'aws secretsmanager get-secret-value \
  --secret-id "$1" --region "$2" \
  --query SecretString --output text > "$3"' \
  sh "$API_SECRET_ID" "$REGION" "$API_TMP"

sudo sh -c 'aws secretsmanager get-secret-value \
  --secret-id "$1" --region "$2" \
  --query SecretString --output text > "$3"' \
  sh "$DB_SECRET_ID" "$REGION" "$DB_TMP"

sudo jq -e '.DB_HOST and .DB_NAME and .DB_PORT' "$API_TMP" >/dev/null
sudo jq -e '.username and .password' "$DB_TMP" >/dev/null
sudo sh -c 'jq -j ".password" "$1" > "$2"' sh "$DB_TMP" "$PASS_TMP"
sudo chmod 600 "$PASS_TMP"

export DB_HOST="$(sudo jq -r '.DB_HOST' "$API_TMP")"
export DB_NAME="$(sudo jq -r '.DB_NAME' "$API_TMP")"
export DB_PORT="$(sudo jq -r '.DB_PORT' "$API_TMP")"
export TZ="$(sudo jq -r '.TZ // "UTC"' "$API_TMP")"
export DB_USER="$(sudo jq -r '.username' "$DB_TMP")"

sudo mv -f -- "$PASS_TMP" "$SECRET_DIR/db_password"
export DB_PASSWORD_FILE="$SECRET_DIR/db_password"

unset API_JSON DB_JSON

cd $CAMINHO_GIT

docker compose up -d app --build

unset DB_HOST DB_NAME DB_PORT TZ DB_USER DB_PASSWORD
```


Salve e proteja o script:

```bash
chmod 700 ./scripts/02-deploy-app.sh
```

Execute a partir da raiz do projeto para que os caminhos relativos usados pelo script sejam resolvidos corretamente:

```bash
./scripts/02-deploy-app.sh
```

O comando `cd ./scripts` só deve ser usado se o script e seus caminhos relativos tiverem sido escritos para execução a partir desse diretório. Para o exemplo acima, execute a partir da raiz do repositório.

## 14. Validar a aplicação e a conexão com o banco

1. Confira se o container está em execução:

   ```bash
   docker ps
   ```

2. Consulte os logs sem exibir variáveis ou credenciais:

   ```bash
   docker logs --tail 100 togglemaster
   ```

3. Valide a API localmente na EC2, usando o endpoint/rota de saúde ou endpoint documentado pelo projeto:

   ```bash
   curl -i http://127.0.0.1:5000/
   ```

   Ajuste a rota e porta ao que a aplicação realmente disponibiliza.

4. A partir de uma máquina externa, valide pelo Elastic IP e endpoint configurados:

   ```bash
   curl -i http://ippublico:5000/
   ```

   Se o acesso for por HTTPS, utilize o domínio/certificado configurado. A regra TCP/443 no Security Group não configura TLS por si só.

5. Execute as operações CRUD da API (criar, consultar, atualizar e excluir uma *feature flag*) usando os endpoints documentados pelo projeto.
6. Confirme que os dados foram gravados e lidos no RDS.
7. Confirme que a porta 5432 do RDS só aceita conexões originadas do Security Group da EC2.
8. Verifique que nenhuma senha, conteúdo de segredo ou token aparece nos logs do container, histórico do shell ou evidências.

### Registro dos resultados

| Verificação | Resultado/evidência |
|---|---|
| EC2 acessível por SSH somente a partir do IP autorizado | [Aprovado] |
| Container iniciado e aplicação respondendo | [Aprovado] |
| Aplicação acessível pelo IP/DNS público | [Aprovado] |
| Operações CRUD concluídas | [Aprovado] |
| Conexão da aplicação com o RDS validada | [Aprovado] |
| Segredos recuperados sem impressão de valores | [Aprovado] |
| RDS sem acesso público e restrito ao SG da EC2 | [Aprovado] |

## 15. Operação e encerramento

- Para visualizar o estado: `docker ps`.
- Para consultar logs: `docker logs --tail 100 togglemaster`.
- Para parar a aplicação: `docker stop togglemaster`.
- Para iniciá-la novamente: `docker start togglemaster`.
- Para atualizar a imagem, siga o procedimento de build e deploy validado.
- Não remova a instância, o RDS, volumes ou segredos antes de confirmar se os dados e recursos precisam ser preservados.

## 16. Observações de segurança e produção

1. O Security Group fornecido abre a porta 5000 para toda a internet; restrinja-a se não for estritamente necessária.
2. SSH está restrito a `myip`; atualize a regra se o IP administrativo mudar.
3. Uma sub-rede pública requer rota para o Internet Gateway; uma sub-rede privada do RDS não deve ter essa rota.
4. Para acesso ao Secrets Manager sem saída pública na EC2, seria necessário configurar conectividade apropriada, como um VPC endpoint, ou uma rota de saída controlada. Verifique como o ambiente atual alcança o serviço.
5. `set +x` reduz o risco de impressão por tracing do Bash, mas variáveis exportadas ainda ficam disponíveis ao processo/container e a usuários com privilégios suficientes no host. Em produção, prefira entrega direta de segredos ao runtime e controle rigoroso de IAM e acesso ao host.
6. A EC2 `t2.micro` e o RDS `db.t3.micro`, Single-AZ, são configurações de baixo custo para laboratório/MVP e não demonstram alta disponibilidade.
7. O PostgreSQL 13.23 está fora do suporte comunitário. Planeje atualização para uma versão suportada e valide compatibilidade da aplicação antes da migração.