# Website estático com Docker, ECR e ECS

Imagem de laboratório que serve o template [Festava Live](https://github.com/edgarjose/festava_live) através de Apache. O Dockerfile está incluído; a criação do repositório ECR, das roles e do serviço ECS é feita separadamente.

## Testar localmente

Requisitos: Docker com suporte a containers Linux e internet durante o build.

```bash
docker build -t practical-static-site:lab .
docker run --rm -d --name practical-static-site -p 127.0.0.1:8080:80 practical-static-site:lab
curl -I http://127.0.0.1:8080/
docker stop practical-static-site
```

Abrir `http://127.0.0.1:8080/`. Esperar HTTP 200 e a página do template. O bind local impede que a porta de teste seja publicada em todas as interfaces do computador.

## Publicar no ECR

Usar Bash com AWS CLI v2 e credenciais configuradas. Escolher a conta e a região antes de criar recursos:

```bash
export AWS_REGION=us-east-1
ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
REGISTRY="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
aws ecr create-repository --repository-name practical-static-site --region "$AWS_REGION"
aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$REGISTRY"
docker buildx build --platform linux/amd64 -t "$REGISTRY/practical-static-site:lab" --push .
```

Se o repositório já existir, omitir a criação. Em máquinas ARM, o build para `linux/amd64` precisa de suporte a emulação ou de um builder compatível.

## Configurar ECS Fargate

1. Criar uma task definition Linux/x86_64 com rede `awsvpc`, a imagem ECR e porta de container `80`.
2. Usar uma task execution role com acesso de leitura ao ECR e entrega de logs.
3. Criar o serviço em sub-redes com saída para ECR e serviços AWS necessários.
4. Para acesso público, usar um ALB com target type `ip`; permitir a porta 80 da task apenas a partir do security group do ALB.
5. Confirmar a task RUNNING, o target saudável e uma resposta HTTP 200.

Não existem task definitions ou automação de ECS neste repositório. O [lab de descoberta de serviços](https://github.com/edgarjose/ecs-networking-service-discovery-lab) contém um exemplo distinto, com duas APIs privadas.

## Reprodutibilidade, atribuição e custos

A imagem base usa Amazon Linux 2023. O conteúdo descarregado da branch `main` pode mudar; fixar uma revisão e um digest da imagem base para builds estritamente reproduzíveis. O template mantém os direitos e termos dos seus autores.

O site é estático e não processa formulários ou pagamentos. Fargate, ALB, ECR, logs, NAT e IPv4 podem ter custos. Depois do teste, remover o serviço/tasks e os recursos dedicados; não deixar um NAT Gateway ou ALB ativo por engano.
