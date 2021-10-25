# Terraform Packager

Terraform Packager é uma coleção de scripts para empacotar código Terrraform.

O objetivo é criar um artefato que seja autosuficiente e personalizável.

Uma imagem criada usa o conceito de Stacks e contém essencialmente:

- **Terraform**: binário em uma versão específica do Terraform
- **HCL Code**: Código Terraform usado para criar recursos
- **Providers**: Os Terraform Providers serão baixados somente durante o build da imagem.

Esse conceito procura seguir a fisolofia de "build once", ou seja, o build do artefato ocorre apenas uma vez e o mesmo artefato pode ser usado para criar várias instâncias da Stack usando diferentes credenciais.

## Dependências

1. Você precisará [instalar](https://github.com/smsilva/linux/blob/master/scripts/utilities/yq/install.sh) o `yq` utilitário para leitura de arquivos yaml.
2. Docker
3. Estes comandos foram testados no `Ubuntu 20.04`.

## Como usar

Se preferir ver um vídeo curto: [Terraform Packager: empacotando código Terraform](https://youtu.be/DDpqmtHY0Aw)

Para empacotar um código Terraform com o `terraform-packager` você precisa garantir que o diretório root do seu Módulo Terraform possua um arquivo `stack.yaml` com as seguintes variáveis:

```yaml
name: azure-nome-do-seu-modulo
version: 0.1.0
terraform:
  version: 1.0.9
```

### Executando

O exemplo usado considera que você possua variáveis de ambiente do **Azure Resource Manager** configuradas:

```bash
ARM_SUBSCRIPTION_ID................: ID_DE_UMA_SUBSCRIPTION_NA_AZURE
ARM_TENANT_ID......................: ID_DO_TENANT_DA_SUBSCRIPTION
ARM_CLIENT_ID......................: ID_DE_UMA_SERVICE_PRINCIPAL_CRIADA_PARA_USO_COM_TERRAFORM
ARM_CLIENT_SECRET..................: SECRET_DA_SERVICE_PRINCIPAL_ACIMA
ARM_STORAGE_ACCOUNT_NAME...........: NOME_DO_STORAGE_ACCOUNT_QUE_SERA_USADO_PARA_ARMAZENAR_O_TFSTATE
ARM_STORAGE_ACCOUNT_CONTAINER_NAME.: NOME_DO_STORAGE_ACCOUNT_CONTAINER
ARM_SAS_TOKEN......................: UM_TOKEN_TEMPORARIO_USADO_PARA_ACESSAR_A_STORAGE_ACCOUNTS
```

### 1. Azure CLI e Variáveis de Ambiente do Azure Resource Manager (ARM)

Confira se as variáveis de ambiente estão configuradas antes de prosseguir:

```bash
scripts/show_arm_variables
```

```bash
ARM_SUBSCRIPTION_ID................: 7f65403e-eec5-45c4-a4ac-1eafc74192ca
ARM_TENANT_ID......................: 16aaff56-7bcd-474f-a975-fe20562ee656
ARM_CLIENT_ID......................: 7dcc3fa0-2794-4ed7-b84f-cd31e244d34d
ARM_CLIENT_SECRET..................: 36
ARM_STORAGE_ACCOUNT_NAME...........: acmestorage
ARM_STORAGE_ACCOUNT_CONTAINER_NAME.: terraform
ARM_SAS_TOKEN......................: 116
```

### 2. Obtendo um SAS Token

O script `scripts/security/arm_sas_token_set_env_variable` obtém um SAS Token usando o Azure CLI.

O SAS Token expira em 1 hora e será usado para acessar o Storage Account que armazenará o State File do Terraform.

```bash
`scripts/security/arm_sas_token_set_env_variable`
```

Confira se as variáveis de ambiente estão configuradas antes de prosseguir:

```bash
scripts/show_arm_variables
```

```bash
ARM_SUBSCRIPTION_ID................: 7f65403e-eec5-45c4-a4ac-1eafc74192ca
ARM_TENANT_ID......................: 16aaff56-7bcd-474f-a975-fe20562ee656
ARM_CLIENT_ID......................: 7dcc3fa0-2794-4ed7-b84f-cd31e244d34d
ARM_CLIENT_SECRET..................: 36
ARM_STORAGE_ACCOUNT_NAME...........: acmestorage
ARM_STORAGE_ACCOUNT_CONTAINER_NAME.: terraform
ARM_SAS_TOKEN......................: 0
```

### 3. Empacotando o Projeto de Exemplo

```bash
./pack "${PWD}/example/azure-fake-module"
```

### 4. Executando o Container usando os valores padrão

```bash
scripts/stackrun azure-fake-module:latest plan
```

```bash
scripts/stackrun azure-fake-module:latest apply
```

### 5. Executando o Container usando arquivos de variáveis personalizadas

Embora não recomendável por ferir o princípio de que um artefato deveria sempre produzir o mesmo resultado, é possível passar um arquivo tfvars através de um volume para o container e usá-lo nos comandos Terraform.

```bash
export LOCAL_TERRAFORM_VARIABLES_DIRECTORY="${PWD}/example/azure-fake-module/environments/sandbox"

scripts/stackrun azure-fake-module:latest plan -var-file=/opt/variables/terraform.tfvars
```

```bash
scripts/stackrun azure-fake-module:latest apply -var-file=/opt/variables/terraform.tfvars -auto-approve
```

### 6. Criando Imagens Personalizadas

#### Sandbox

```bash
docker build --rm \
  --tag azure-fake-module/sandbox:latest "examples/azure-fake-module/environments/sandbox"
```

```bash
scripts/azrun azure-fake-module/sandbox:latest plan
```

```bash
scripts/azrun azure-fake-module/sandbox:latest apply
```
