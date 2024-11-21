# **Guía Práctica: Creación de un Clúster de Memcached con AWS ElastiCache usando la AWS CLI**

## **Objetivo:**
En esta práctica, aprenderás a crear y configurar un clúster de Memcached utilizando AWS ElastiCache con la AWS CLI. Configuraremos todos los recursos necesarios, como grupos de seguridad y grupos de subredes, para garantizar que el clúster esté correctamente configurado y sea accesible desde una máquina local.

![Diagrama de Arquitectura](diagrams\MemcachedClusterCLI.png)

---

## **Requisitos Previos**

1. **AWS CLI instalada** y configurada en tu máquina local. (Instrucciones [aquí](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)).
2. Conexión a Internet y tu **IP pública** identificada (puedes usar [WhatIsMyIP](https://whatismyipaddress.com/)).
3. Familiaridad básica con la CLI de AWS y los conceptos de redes en AWS.

---

## **Pasos del Laboratorio**

### **1. Crear un Grupo de Seguridad para el Clúster**

Un grupo de seguridad controla el tráfico que puede acceder al clúster. Aquí configuraremos reglas para:
- Permitir el acceso desde tu máquina local.
- Permitir conexiones desde recursos en la misma VPC.

#### **Comando para crear el grupo de seguridad:**
```bash
aws ec2 create-security-group \
    --group-name memcached-lab-sg \
    --description "Security group for Memcached lab" \
    --vpc-id $(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
```

El comando devolverá un `GroupId`, algo como `sg-0123456789abcdef0`. Guarda este valor para usarlo en los pasos siguientes.

#### **Configurar las reglas del grupo de seguridad:**

1. **Permitir acceso desde tu máquina local:**
   ```bash
   aws ec2 authorize-security-group-ingress \
       --group-id <GROUP_ID> \
       --protocol tcp \
       --port 11211 \
       --cidr <YOUR_PUBLIC_IP>/32
   ```

   Reemplaza:
   - `<GROUP_ID>` con el ID del grupo de seguridad.
   - `<YOUR_PUBLIC_IP>` con tu dirección IP pública. Utiliza `0.0.0.0/0` para habilitar todo el tráfico de entrada.

2. **Permitir acceso entre recursos en la VPC:**
   ```bash
   aws ec2 authorize-security-group-ingress \
       --group-id <GROUP_ID> \
       --protocol tcp \
       --port 11211 \
       --source-group <GROUP_ID>
   ```

---

### **2. Crear un Grupo de Subredes para ElastiCache**

El grupo de subredes permite a ElastiCache operar dentro de subredes específicas de la VPC.

#### **Crear el grupo de subredes:**
```bash
aws elasticache create-cache-subnet-group \
    --cache-subnet-group-name default-elasticache-group \
    --cache-subnet-group-description "Subnet group for Memcached lab with default VPC" \
    --subnet-ids $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)" --query "Subnets[*].SubnetId" --output text)
```

El comando se utiliza para listar las subredes de la VPC por defecto
```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)" --query "Subnets[*].SubnetId" --output text
```

---

### **3. Crear el Clúster de Memcached**

Con el grupo de seguridad y el grupo de subredes configurados, ahora crearemos el clúster.

#### **Comando para crear el clúster:**
```bash
aws elasticache create-cache-cluster \
    --cache-cluster-id memcached-lab-cluster \
    --engine memcached \
    --cache-node-type cache.t2.micro \
    --num-cache-nodes 2 \
    --region <YOUR_REGION> \
    --cache-subnet-group-name default-elasticache-group \
    --security-group-ids <GROUP_ID>
```

Reemplaza:
- `<YOUR_REGION>` con tu región (ejemplo: `us-east-1`).
- `<GROUP_ID>` con el ID del grupo de seguridad creado.

---

### **4. Verificar el Estado del Clúster**

Ejecuta el siguiente comando para verificar que el clúster esté activo:

```bash
aws elasticache describe-cache-clusters \
    --cache-cluster-id memcached-lab-cluster \
    --show-cache-node-info \
    --region us-east-1
```

Espera a que el estado sea `available`. El comando también mostrará las direcciones de los nodos del clúster.

---

### **5. Limpieza de Recursos**

Para evitar costos innecesarios, elimina los recursos creados al finalizar el laboratorio:

1. Eliminar el clúster:
   ```bash
   aws elasticache delete-cache-cluster --cache-cluster-id memcached-lab-cluster
   ```

2. Eliminar el grupo de subredes:
   ```bash
   aws elasticache delete-cache-subnet-group --cache-subnet-group-name default-elasticache-group
   ```

3. Eliminar el grupo de seguridad:
   ```bash
   aws ec2 delete-security-group --group-id <GROUP_ID>
   ```

---

¡Buena suerte! 🚀