# Guia Genérico para Criar e Configurar Ambientes AWS com CloudFormation e Elastic Beanstalk

Este guia fornece orientações gerais para a configuração de ambientes de desenvolvimento, homologação e produção na AWS. Ele utiliza **CloudFormation** para provisionar a infraestrutura e a **EB CLI** (Elastic Beanstalk Command Line Interface) para gerenciar e configurar os ambientes. O exemplo é totalmente personalizável e genérico para atender diferentes cenários.

---

## **Script Automatizado**

### **Arquivo: `create_infrastructure.sh`**

Este script verifica dependências, cria ou atualiza stacks CloudFormation, e aguarda a conclusão das operações.

```bash
#!/bin/bash

set -e

AWS_REGION="sa-east-1"
STACK_NAME="GenericInfraStack"
TEMPLATE_FILE="infra-template.yaml"
APPLICATION_NAME="generic-application"

# Verificar dependências
check_dependencies() {
  for cmd in aws jq; do
    if ! command -v $cmd &> /dev/null; then
      echo "Erro: o comando $cmd não está instalado. Instale e tente novamente."
      exit 1
    fi
  done
}

# Criar ou atualizar o stack CloudFormation
create_or_update_infrastructure() {
  echo "🔧 Verificando stack CloudFormation..."

  if aws cloudformation describe-stacks --stack-name $STACK_NAME --region $AWS_REGION &> /dev/null; then
    echo "🔄 Stack já existe. Atualizando $STACK_NAME..."
    aws cloudformation update-stack \
      --stack-name $STACK_NAME \
      --template-body file://$TEMPLATE_FILE \
      --parameters ParameterKey=ApplicationName,ParameterValue=$APPLICATION_NAME \
      --capabilities CAPABILITY_NAMED_IAM || {
      echo "⚠️ Nenhuma atualização necessária para $STACK_NAME."
    }
    echo "⏳ Aguardando a atualização do stack $STACK_NAME..."
    aws cloudformation wait stack-update-complete --stack-name $STACK_NAME --region $AWS_REGION
  else
    echo "🆕 Stack não encontrada. Criando nova stack: $STACK_NAME..."
    aws cloudformation create-stack \
      --stack-name $STACK_NAME \
      --template-body file://$TEMPLATE_FILE \
      --parameters ParameterKey=ApplicationName,ParameterValue=$APPLICATION_NAME \
      --capabilities CAPABILITY_NAMED_IAM
    echo "⏳ Aguardando a criação do stack $STACK_NAME..."
    aws cloudformation wait stack-create-complete --stack-name $STACK_NAME --region $AWS_REGION
  fi

  echo "✅ Infraestrutura configurada com sucesso."
}

# Fluxo principal
main() {
  check_dependencies
  create_or_update_infrastructure
}

main
```

---

### **Arquivo: `manage_environments.sh`**

Este script gerencia os ambientes Elastic Beanstalk, verificando se eles já existem, criando novos quando necessário e aplicando tags apropriadas.

```bash
#!/bin/bash

APPLICATION_NAME="generic-application"
REGION="sa-east-1"
PLATFORM="Docker"
ENVIRONMENTS=("dev" "staging" "production")
STACK_NAME="GenericInfraStack"

# Verifica se o ambiente existe
environment_exists() {
    local env_name=$1
    eb list | grep -q "^$env_name\$"
}

# Aplica tags ao ambiente
apply_tags() {
    local env_name=$1
    RESOURCE_ARN=$(aws elasticbeanstalk describe-environments \
        --environment-names "$env_name" \
        --query 'Environments[0].EnvironmentArn' \
        --output text)

    if [ -z "$RESOURCE_ARN" ]; then
        echo "⚠️ ARN do ambiente $env_name não encontrado. Pulando aplicação de tags."
        return
    fi

    echo "🔖 Aplicando tags para $env_name..."
    aws elasticbeanstalk update-tags-for-resource \
        --region "$REGION" \
        --resource-arn "$RESOURCE_ARN" \
        --tags-to-add "Key=Environment,Value=$env_name" "Key=Application,Value=$APPLICATION_NAME"
}

# Configura o ambiente Elastic Beanstalk
configure_environment() {
    local env_name=$1

    echo "🔧 Configurando ambiente $env_name..."

    if environment_exists "$env_name"; then
        echo "✅ Ambiente $env_name já existe. Atualizando..."
        eb deploy "$env_name" --staged
    else
        echo "🆕 Criando ambiente $env_name..."
        eb create "$env_name" \
            --region "$REGION" \
            --platform "$PLATFORM" \
            --vpc.id "$VPC_ID" \
            --vpc.publicip \
            --vpc.elbsubnets "$SUBNET_A,$SUBNET_B" \
            --vpc.ec2subnets "$SUBNET_A,$SUBNET_B" \
            --envvars "ENVIRONMENT=$env_name" || {
                echo "❌ Falha ao criar o ambiente $env_name."
                exit 1
            }
        
        # Aplicar tags após criação
        apply_tags "$env_name"
    fi
}

# Recupera IDs da VPC e Subnets
get_vpc_and_subnet_ids() {
    echo "🔍 Recuperando IDs da VPC e Subnets..."
    VPC_ID=$(aws cloudformation describe-stacks \
        --stack-name "$STACK_NAME" \
        --query "Stacks[0].Outputs[?OutputKey=='VPCId'].OutputValue" \
        --output text)

    SUBNET_A=$(aws cloudformation describe-stacks \
        --stack-name "$STACK_NAME" \
        --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetAId'].OutputValue" \
        --output text)

    SUBNET_B=$(aws cloudformation describe-stacks \
        --stack-name "$STACK_NAME" \
        --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetBId'].OutputValue" \
        --output text)

    if [ -z "$VPC_ID" ] || [ -z "$SUBNET_A" ] || [ -z "$SUBNET_B" ]; then
        echo "❌ Falha ao recuperar informações da infraestrutura. Verifique o CloudFormation."
        exit 1
    fi

    echo "🔍 VPC_ID: $VPC_ID"
    echo "🔍 SUBNET_A: $SUBNET_A"
    echo "🔍 SUBNET_B: $SUBNET_B"
}

# Configura a aplicação no Elastic Beanstalk
initialize_application() {
    echo "🔧 Inicializando aplicação $APPLICATION_NAME..."

    if ! eb status &>/dev/null; then
        echo "🆕 Aplicação não encontrada. Inicializando..."
        eb init "$APPLICATION_NAME" --region "$REGION" --platform "$PLATFORM"
    else
        echo "✅ Aplicação $APPLICATION_NAME já configurada."
    fi
}

# Fluxo principal
main() {
    echo "🚀 Gerenciando ambientes Elastic Beanstalk para $APPLICATION_NAME..."

    # Inicializar aplicação
    initialize_application

    # Obter IDs da VPC e Subnets
    get_vpc_and_subnet_ids

    # Configurar cada ambiente
    for env in "${ENVIRONMENTS[@]}"; do
        configure_environment "$APPLICATION_NAME-$env"
    done

    echo "✅ Todos os ambientes configurados com sucesso!"
}

main
```

---

### **Template CloudFormation**

O template provisiona recursos compartilhados como VPC, subnets e grupos de segurança, além de rotas para acesso público.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Template para provisionar infraestrutura genérica na AWS

Parameters:
  ApplicationName:
    Type: String
    Description: Nome da aplicação.

  CIDR:
    Type: String
    Default: "10.0.0.0/16"
    Description: CIDR para a VPC.

  SubnetACIDR:
    Type: String
    Default: "10.0.1.0/24"
    Description: CIDR para a Subnet Pública A.

  SubnetBCIDR:
    Type: String
    Default: "10.0.2.0/24"
    Description: CIDR para a Subnet Pública B.

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref CIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${ApplicationName}-VPC"

  # Demais recursos omitidos para simplificação...
```

Adapte os nomes e variáveis de acordo com o seu caso de uso!
