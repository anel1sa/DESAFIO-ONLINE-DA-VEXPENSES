# PROCESSO SELETIVO | ESTÁGIO EM DEVOPS

## Tarefa 1 
### Descrição Técnica
O código main.tf inicialmente cria e configura recurso na AWS. Define a região do provedor e cria variáveis que armazenam nomes.
```
 provider "aws" {
 region = "us-east-1"
 }
```
- Define a região como "us-east-1"
```
 variable "projeto" {
 description = "Nome do projeto"
type        = string
 default     = "VExpenses"
}

variable "candidato" {
description = "Nome do candidato"
type        = string
 default     = "SeuNome"
}
```
- Define as variáveis **_projeto_** e **_candidato_** que armazenam os nomes delas.

**Geração de chaves SSH**
```
resource "tls_private_key" "ec2_key" {
algorithm = "RSA"
rsa_bits  = 2048
}
```
- Gera chaves RSA e 2048 bits usando **_tls_private_key_** que é uma chave privada local
```
resource "aws_key_pair" "ec2_key_pair" {
key_name   = "${var.projeto}-${var.candidato}-key"
public_key = tls_private_key.ec2_key.public_key_openssh
}
```
- Cria um ``aws_key_pair``que associa a chave pública gerada. A chave privada é usada depois para acesso SSH na instância EC2

**VPC e Sub-rede**
```
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```
- Cria uma VPC com o bloco CIDR de ``10.0.0.0/16`` que permite criar sub-redes dentro do intevalo de IPs. A VPC suporta DNS e nomes de host pois foi configurada com ``true``.
```
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```
- Cria uma sub-rede dentro da VPC com CIDR ``10.0.1.0/24``

**Gateway de internet e Tabela de roteamento**
```
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```
- Cria um gateway para permitir que a VPC conecte com a internet.
```
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
```
- Cria uma tabela de rotas para a VPC e define uma rota de direção única ``0.0.0.0/0`` para o gateway de internet, isso permite que os recursos da VPC se comuniquem com a internet.
```
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id
}
```
- Associa a tabela de rotas à sub-rede, permitindo que eça use o gateway de internet para acessa-lá.

**Grupo de Segurança**
```
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
```
- Define um grupo de segurança que permite conexões SSH (porta 22) de qualquer origem IPv4 e IPv6 e todo o tráfego de saída é permitido

**Seleção de AMI**
```
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```
- Busca a AMI mais recente do Debian 12, usando friltro para que seja uma imagem com o nome certo e o tipo de visualização hvm. É usada para criar a instância EC2.

**Instância EC2**
```
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```
- Cria uma instância EC2 do tipo ``t2.micro`` com uma AMI do Debian 12. A instância é associada a sub-rede e grupo de segurança configurados acima e utiliza o ``user_data`` para atualizar o sistema quando inicializado. A instância é configurada para associar um IP público permitindo acesso direto da internet.

**Outputs**
```
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```
- Exibe a chave privada para acesso a instância EC2 e o endereço IP público da instância.

## Tarefa 2
### Melhorias Implementadas
#### Melhorias de Segurança
Foram realizadas as seguintes melhorias de segurança no código:

**Restrição do acesso SSH**
```
variable "allowed_ssh_ip" {
  description = "Endereço IP autorizado para acessar a instância via SSH"
  type        = string
  default     = "203.0.113.5/32" # Um IP específico para maior segurança
}
```
- O grupo de segurança foi modificado para restringir o acesso SSH a um IP de administrador. E a utilização de ``/32`` indica que apenas um único IP pode acessar a instância via SSH, aumentando a segurança.

**Segurança da chave SSH**
```
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```
- Geração da chave SSH usando ``rls_private_key`` garante que a chave privada não seja armazenada diretamente no código, evitando o risco de exposição.

**Automação da instalação do Ngnix**
```
user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF
```
- Esse script Bash é executado no primeiro boot da instância. Ele atualiza o sistema, instala o Nginx, inicia o serviço e habilita-o para iniciar automaticamente após reinicializações.

**Configuração do Bucket S3**
```
resource "aws_s3_bucket" "log_bucket" {
  bucket = "${var.projeto}-${var.candidato}-logs"
}
```
- O bucket foi criado para armazenar logs, ajudando na gestão e na recuperação de informações sobre a instância e o serviço.

**Gerenciamento de Tags**

- As tags foram padronizadas em todos os recursos para facilitar a identificação e o gerenciamento. Cada recurso é nomeado de acordo com o projeto e o candidato, melhorando a organização.

**Outputs sensíveis**
```
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}
```
- A chave privada é marcada como sensível nos outputs, garantindo que não seja mostrada em logs ou na interface de usuário do Terraform.
