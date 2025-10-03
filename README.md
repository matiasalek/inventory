# Despliegue de Aplicación en AWS

Guía para desplegar una aplicación en AWS usando diferentes métodos: manual, CLI y Elastic Beanstalk.

## Prerrequisitos

- Cuenta de AWS activa
- AWS CLI instalado y configurado
- Aplicación Node.js preparada

## 1. Despliegue Manual en EC2

### 1.1 Creación Manual

**En la Consola AWS:**
1. EC2 → Launch Instance
2. **AMI**: Ubuntu Server 24.04 LTS
3. **Instance Type**: t3.micro
4. **Security Group**: Puertos 22 (SSH), 80 (HTTP), 3000 (App)
5. Crear/seleccionar Key Pair

**Instalación en el servidor:**
```bash
# Conectar
ssh -i "tu-key.pem" ubuntu@ip-publica

# Instalar dependencias
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs git nginx
sudo npm install -g pm2

# Desplegar aplicación
git clone https://github.com/matiasalek/inventory.git
cd inventory
npm install
pm2 start app.js --name "inventory"
```

### 1.2 Con User Data (Automatización)

En "Advanced Details" → "User data":
```bash
#!/bin/bash
apt update && apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs git nginx
npm install -g pm2

# Configurar aplicación
useradd -m appuser
cd /home/appuser
git clone https://github.com/tu-usuario/tu-repo.git app
chown -R appuser:appuser app
cd app
sudo -u appuser npm install

# Configurar Nginx proxy
cat > /etc/nginx/sites-available/default <<EOF
server {
    listen 80;
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host \$host;
    }
}
EOF

systemctl restart nginx
sudo -u appuser pm2 start app.js --name "mi-app"
```

## 2. Despliegue con AWS CLI

### 2.1 Configuración Inicial
```bash
# Configurar AWS CLI
aws configure

# Variables
REGION=us-east-1
KEY_NAME=mi-app-key
```

### 2.2 Crear Infraestructura
```bash
# Crear Security Group
aws ec2 create-security-group \
    --group-name mi-app-sg \
    --description "Security group para aplicacion"

SG_ID=$(aws ec2 describe-security-groups \
    --group-names mi-app-sg \
    --query 'SecurityGroups[0].GroupId' \
    --output text)

# Configurar reglas
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 3000 --cidr 0.0.0.0/0

# Crear Key Pair
aws ec2 create-key-pair --key-name $KEY_NAME --query 'KeyMaterial' --output text > $KEY_NAME.pem
chmod 400 $KEY_NAME.pem

# Crear User Data Script
cat > user-data.sh <<EOF
#!/bin/bash
apt update && apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs git nginx
npm install -g pm2
useradd -m appuser
cd /home/appuser
git clone https://github.com/tu-usuario/tu-repo.git app
chown -R appuser:appuser app
cd app && sudo -u appuser npm install
sudo -u appuser pm2 start app.js --name "mi-app"
EOF
```

### 2.3 Lanzar Instancia
```bash
# Crear instancia
aws ec2 run-instances \
    --image-id ami-0c02fb55956c7d316 \
    --count 1 \
    --instance-type t2.micro \
    --key-name $KEY_NAME \
    --security-group-ids $SG_ID \
    --user-data file://user-data.sh \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mi-app-cli}]'

# Obtener IP pública
PUBLIC_IP=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=mi-app-cli" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PublicIpAddress' \
    --output text)

echo "App disponible en: http://$PUBLIC_IP"
```

## 3. Despliegue con Elastic Beanstalk

### 3.1 Preparar Aplicación

**Estructura del proyecto:**
```
public/
├── app.js
├── sw.js 
├── index.html 
├── manifest.json 
└── .node_modules/
LICENSE
README.md
server.js
package-lock.json
package.json
```

**package.json:**
```json
{
  "name": "mi-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**.ebextensions/nodejs.config:**
```yaml
option_settings:
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
  aws:elasticbeanstalk:container:nodejs:environment:
    PORT: 8080
```

### 3.2 Desplegar con EB CLI
```bash
# Instalar EB CLI
pip install awsebcli --upgrade

# Inicializar y desplegar
cd mi-app
eb init  # Seguir configuración interactiva
eb create mi-app-prod  # Crear y desplegar

# Comandos útiles
eb status    # Ver estado
eb open      # Abrir en navegador
eb logs      # Ver logs
eb deploy    # Actualizar aplicación
eb terminate # Terminar ambiente
```

## 4. Comparación EC2 vs Elastic Beanstalk

| Aspecto | EC2 | Elastic Beanstalk |
|---------|-----|-------------------|
| **Control** | Total sobre infraestructura | Gestión automática |
| **Configuración** | Manual completa | Simplificada |
| **Escalabilidad** | Manual | Automática |
| **Tiempo setup** | Mayor | Menor |
| **Costos** | Potencialmente menor | Incluye servicios adicionales |
| **Complejidad** | Alta | Baja |

### Cuándo usar cada uno:
- **EC2**: Control total, configuraciones específicas, costos optimizados
- **Elastic Beanstalk**: Despliegue rápido, auto-scaling, gestión simplificada

## 5. Limpieza de Recursos

### EC2:
```bash
# Terminar instancia
aws ec2 terminate-instances --instance-ids i-xxxxxxxxx
# Eliminar Security Group y Key Pair
aws ec2 delete-security-group --group-id $SG_ID
aws ec2 delete-key-pair --key-name $KEY_NAME
```

### Elastic Beanstalk:
```bash
eb terminate mi-app-prod
```
## Conclusión

**Recomendaciones por caso de uso:**
- **Aprendizaje/Control**: EC2 manual
- **Automatización**: EC2 con CLI
- **Producción rápida**: Elastic Beanstalk
