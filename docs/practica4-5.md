# Practica-4.5 - Terraform

## Ejercicio para crear la infraestructura de la Práctica 1.11 con un script de Terraform

### Comprobaciones previas
- **1. Configurar correctamente las credenciales de AWS en C:/Users/antonio/.aws/credentials.**
- **2. Ejecutar `terraform init` para inicializar el proyecto con Terraform.**
- **3. Ejecutar `terraform plan` para ver el plan de ejecución y confirmar que no hay errores.**
- **4. Ejecutar `terraform apply` para crear la infraestructura.**

**Archivo "main.tf" (Información relevante comentada en el código)"**
```tf
provider "aws" {
  region = var.region
}

# Definimos el grupo de seguridad para las instancias frontend, permitiendo reglas específicas de acceso.
resource "aws_security_group" "frontend" {
  name        = var.SG_FRONTEND
  description = "SG Frontend"
}

# Especificamos las reglas de entrada para el grupo de seguridad del frontend, 
# permitiendo tráfico en los puertos SSH (22), NFS (2049), HTTP (80), HTTPS (443) y MySQL (3306) desde cualquier dirección IP.
resource "aws_security_group_rule" "frontend_ingress" {
  count             = 5
  security_group_id = aws_security_group.frontend.id
  type              = "ingress"
  from_port         = [22, 2049, 80, 443, 3306][count.index]
  to_port           = [22, 2049, 80, 443, 3306][count.index]
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Definimos el grupo de seguridad para las instancias backend.
resource "aws_security_group" "backend" {
  name        = var.SG_BACKEND
  description = "SG Backend"
}

# Configuramos reglas de entrada para el backend, permitiendo únicamente acceso a SSH (22) y MySQL (3306) desde cualquier IP.
resource "aws_security_group_rule" "backend_ingress" {
  count             = 2
  security_group_id = aws_security_group.backend.id
  type              = "ingress"
  from_port         = [22, 3306][count.index]
  to_port           = [22, 3306][count.index]
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Definimos el grupo de seguridad para el loadbalancer de carga.
resource "aws_security_group" "loadbalancer" {
  name        = var.SG_LOADBALANCER
  description = "SG LoadBalancer"
}

# Permitimos tráfico en los puertos esenciales (SSH, MySQL, HTTP, HTTPS y NFS) en el loadbalancer.
resource "aws_security_group_rule" "loadbalancer_ingress" {
  count             = 5
  security_group_id = aws_security_group.loadbalancer.id
  type              = "ingress"
  from_port         = [22, 3306, 80, 443, 2049][count.index]
  to_port           = [22, 3306, 80, 443, 2049][count.index]
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Definimos el grupo de seguridad para el servidor NFS.
resource "aws_security_group" "nfs" {
  name        = var.SG_NFS
  description = "SG NFS"
}

# Permitimos tráfico en los puertos SSH (22) y NFS (2049) desde cualquier IP.
resource "aws_security_group_rule" "nfs_ingress" {
  count             = 2
  security_group_id = aws_security_group.nfs.id
  type              = "ingress"
  from_port         = [22, 2049][count.index]
  to_port           = [22, 2049][count.index]
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
}

# Creamos dos instancias EC2 para el frontend con las configuraciones definidas.
resource "aws_instance" "frontend" {
  count           = 2
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.frontend.name]

  tags = {
    Name = "frontend-${count.index + 1}"
  }
}

# Creamos una instancia EC2 para el backend.
resource "aws_instance" "backend" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.backend.name]

  tags = {
    Name = "backend"
  }
}

# Creamos una instancia EC2 para el loadbalancer de carga.
resource "aws_instance" "loadbalancer" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.loadbalancer.name]

  tags = {
    Name = "loadbalancer"
  }
}

# Creamos una instancia EC2 para el servidor NFS.
resource "aws_instance" "nfs" {
  ami             = var.ami_id
  instance_type   = var.instance_type
  key_name        = var.key_name
  security_groups = [aws_security_group.nfs.name]

  tags = {
    Name = "nfs"
  }
}

# Asignamos una IP elástica a cada una de las dos instancias frontend.
resource "aws_eip" "frontend" {
  count    = 2
  instance = aws_instance.frontend[count.index].id
}

# Asignamos una IP elástica a la instancia backend.
resource "aws_eip" "backend" {
  instance = aws_instance.backend.id
}

# Asignamos una IP elástica a la instancia loadbalancer.
resource "aws_eip" "loadbalancer" {
  instance = aws_instance.loadbalancer.id
}

# Asignamos una IP elástica a la instancia NFS.
resource "aws_eip" "nfs" {
  instance = aws_instance.nfs.id
}
```

**Archivo "output.tf" (Información relevante comentada en el código)"**
```tf
# Salida de las IPs públicas de las instancias frontend
output "frontend_elastic_ips" {
  value = [for i in aws_eip.frontend : i.public_ip]
}

# Salida de la IP pública de la instancia backend
output "backend_elastic_ip" {
  value = aws_eip.backend.public_ip
}

# Salida de la IP pública de la instancia del balanceador de carga
output "balanceador_elastic_ip" {
  value = aws_eip.loadbalancer.public_ip
}

# Salida de la IP pública de la instancia NFS
output "nfs_elastic_ip" {
  value = aws_eip.nfs.public_ip
}
```

**Archivo "variables.tf" (he dejado lo tuyo y he añadido los grupos de seguridad)"**
```tf
variable "region" {
  description = "Región de AWS donde se creará la instancia"
  type        = string
  default     = "us-east-1"
}

variable "allowed_ingress_ports" {
  description = "Puertos de entrada del grupo de seguridad"
  type        = list(number)
  default     = [22, 80, 443]
}

variable "sg_name" {
  description = "Nombre del grupo de seguridad"
  type        = string
  default     = "sg_ejemplo_06"
}

variable "sg_description" {
  description = "Descripción del grupo de seguridad"
  type        = string
  default     = "Grupo de seguridad para la instancia de ejemplo 06"
}

variable "ami_id" {
  description = "Identificador de la AMI"
  type        = string
  default     = "ami-00874d747dde814fa"
}

variable "instance_type" {
  description = "Tipo de instancia"
  type        = string
  default     = "t2.small"
}

variable "key_name" {
  description = "Nombre de la clave pública"
  type        = string
  default     = "vockey"
}

variable "instance_name" {
  description = "Nombre de la instancia"
  type        = string
  default     = "instancia_ejemplo_06"
}

# Variable que define el nombre del SG para el frontend
variable "SG_FRONTEND" {
  description = "Nombre del SG del frontend"
  type        = string
  default     = "sg_frontend"
}

# Variable que define el nombre del SG para el backend
variable "SG_BACKEND" {
  description = "Nombre del SG del backend"
  type        = string
  default     = "sg_backend"
}

# Variable que define el nombre del SG para el LoadBalancer
variable "SG_LOADBALANCER" {
  description = "Nombre del SG del LoadBalancer"
  type        = string
  default     = "sg_loadbalancer"
}

# Variable que define el nombre del SG para el servidor NFS
variable "SG_NFS" {
  description = "Nombre del SG del NFS Server"
  type        = string
  default     = "sg_nfs"
}
```

**COMPROBACIONES**
- **Salida de ejecución de terraform**
![Salida-Terraform](capturas/salida.png)
- **Creación de instancias**
![Instancias-Creadas](capturas/instancias.png)
- **Creación de Grupos de Seguridad**
![Grupos-Seguridad-Creados](capturas/grupos-seguridad.png)
- **Creación de IPs Elásticas**
![IPsElásticas](capturas/ip-elastica.png)
- **Grupos de seguridad asociados a instancias**
![IPsElásticas](capturas/instancias-asociadas.png)