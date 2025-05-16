# Guía completa: CI/CD con GitHub Actions y AWS EC2

## Índice
1. [Introducción](#introducción)
2. [Conceptos básicos](#conceptos-básicos)
3. [Estructura del pipeline creado](#estructura-del-pipeline-creado)
4. [Configuración de GitHub Actions](#configuración-de-github-actions)
5. [Configuración de AWS](#configuración-de-aws)
6. [Configuración de secretos en GitHub](#configuración-de-secretos-en-github)
7. [Prueba y verificación del pipeline](#prueba-y-verificación-del-pipeline)
8. [Solución de problemas comunes](#solución-de-problemas-comunes)
9. [Recursos adicionales](#recursos-adicionales)

## Introducción

Esta guía te ayudará a implementar un pipeline completo de CI/CD (Integración Continua/Entrega Continua) utilizando GitHub Actions y AWS EC2. El objetivo es automatizar el proceso de testing, build y despliegue de una aplicación backend cuando se hacen cambios en una rama con un Pull Request abierto.

El pipeline que implementaremos seguirá estos pasos:
1. Ejecutar tests del backend
2. Generar un build del backend
3. Desplegar el backend en una instancia EC2 de AWS

## Conceptos básicos

Antes de entrar en detalle, es importante entender algunos conceptos clave:

**GitHub Actions**: Es la plataforma de CI/CD integrada en GitHub que permite automatizar flujos de trabajo como pruebas, build y despliegue.

**Workflow**: Un archivo YAML (en este caso `.github/workflows/pipeline.yml`) que define tu flujo de trabajo automatizado.

**Eventos (Triggers)**: Determinan cuándo se ejecuta tu workflow. En nuestro caso, necesitamos que se active cuando se haga push a una rama con un PR abierto.

**Jobs**: Cada workflow contiene uno o más trabajos que se ejecutan en paralelo por defecto.

**Steps**: Pasos individuales dentro de un job que se ejecutan secuencialmente.

**AWS EC2**: Servicio de Amazon Web Services que proporciona capacidad informática en la nube. Es donde desplegaremos nuestra aplicación.

**PM2**: Gestor de procesos para aplicaciones Node.js en producción, que usaremos para gestionar nuestra aplicación en el servidor.

## Estructura del pipeline creado

El archivo `.github/workflows/pipeline.yml` que hemos creado tiene la siguiente estructura:

```yaml
name: Backend CI/CD Pipeline

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main  # o master, según la rama principal de tu repositorio
  push:
    branches:
      - '**'  # Activar en cualquier rama donde se haga push

jobs:
  test:
    # Configuración del job de tests
    ...
  
  build:
    # Configuración del job de build
    ...
  
  deploy:
    # Configuración del job de despliegue
    ...
```

### Explicación de los triggers

El pipeline se activará en dos situaciones:
1. Cuando se abra, sincronice o reabra un Pull Request dirigido a la rama `main`
2. Cuando se haga push a cualquier rama

Sin embargo, hemos añadido una condición `if: github.event_name == 'pull_request' || github.event.pull_request` para que los jobs solo se ejecuten si hay un PR abierto, como requiere el ejercicio.

### Job: Test

Este job se encarga de ejecutar los tests de backend:

1. Hace checkout del código
2. Configura Node.js v18
3. Instala las dependencias con `npm ci` (instalación limpia)
4. Ejecuta los tests con `npm test`

### Job: Build

Este job se ejecuta solo si los tests han sido exitosos:

1. Hace checkout del código
2. Configura Node.js v18
3. Instala las dependencias
4. Genera el build con `npm run build`
5. Sube los artefactos generados (carpeta `dist`) para usarlos en el siguiente job

### Job: Deploy

Este job se encarga del despliegue en EC2:

1. Descarga los artefactos del build
2. Configura las credenciales de AWS
3. Conecta por SSH a la instancia EC2
4. Transfiere los archivos necesarios
5. Despliega la aplicación usando PM2

## Configuración de GitHub Actions

GitHub Actions ya está incluido en tu repositorio de GitHub, por lo que no necesitas instalarlo. Solo necesitas crear el archivo de workflow en la ubicación correcta.

### Paso 1: Crear el directorio de workflows

El archivo de workflow debe estar en la ruta `.github/workflows/` de tu repositorio. Si este directorio no existe, debes crearlo:

```bash
mkdir -p .github/workflows
```

### Paso 2: Crear el archivo pipeline.yml

Ya hemos creado este archivo con la configuración necesaria.

## Configuración de AWS

Para poder desplegar en AWS EC2, necesitas configurar varios elementos:

### Paso 1: Crear una cuenta de AWS

Si no tienes una cuenta de AWS, necesitas crear una en [aws.amazon.com](https://aws.amazon.com/). AWS ofrece una capa gratuita para nuevos usuarios que puedes utilizar para este ejercicio.

### Paso 2: Crear un usuario IAM con acceso programático

Para que GitHub Actions pueda interactuar con AWS, necesitas un usuario con credenciales programáticas:

1. Ve a la consola de AWS y navega a IAM (Identity and Access Management)
2. Crea un nuevo usuario con acceso programático
3. Asigna los permisos necesarios (por ejemplo, AmazonEC2FullAccess)
4. Guarda el Access Key ID y Secret Access Key que se generan

### Paso 3: Lanzar una instancia EC2

Necesitas una instancia EC2 donde desplegar tu aplicación:

1. Ve al servicio EC2 en la consola de AWS
2. Haz clic en "Launch Instance"
3. Selecciona una AMI (Amazon Machine Image) - recomendamos Amazon Linux 2 o Ubuntu
4. Elige un tipo de instancia (t2.micro está en la capa gratuita)
5. Configura los detalles de la instancia según tus necesidades
6. Configura el grupo de seguridad para permitir tráfico en los puertos necesarios (SSH - 22, HTTP - 80, HTTPS - 443, y el puerto de tu aplicación)
7. Crea o selecciona un par de claves (key pair) para SSH
8. Lanza la instancia

### Paso 4: Configurar la instancia EC2

Una vez que la instancia está en funcionamiento, necesitas preparar el entorno:

1. Conéctate a la instancia por SSH:
   ```bash
   ssh -i tu-key.pem ec2-user@tu-instancia-ec2-dns
   ```

2. Instala Node.js:
   ```bash
   # Para Amazon Linux 2
   curl -sL https://rpm.nodesource.com/setup_18.x | sudo bash -
   sudo yum install -y nodejs

   # Para Ubuntu
   curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
   sudo apt-get install -y nodejs
   ```

3. Instala PM2 globalmente:
   ```bash
   sudo npm install -g pm2
   ```

4. Crea un directorio para la aplicación:
   ```bash
   mkdir -p ~/app
   ```

## Configuración de secretos en GitHub

Para que el workflow funcione correctamente, necesitas configurar los siguientes secretos en tu repositorio de GitHub:

1. Ve a tu repositorio en GitHub
2. Ve a "Settings" > "Secrets and variables" > "Actions"
3. Haz clic en "New repository secret"
4. Agrega los siguientes secretos:

- **AWS_ACCESS_KEY_ID**: Tu Access Key de AWS
- **AWS_SECRET_ACCESS_KEY**: Tu Secret Access Key de AWS
- **AWS_REGION**: La región donde se encuentra tu instancia EC2 (ej. us-east-1)
- **EC2_SSH_PRIVATE_KEY**: El contenido de tu archivo de clave privada (.pem)
- **EC2_HOST**: La dirección IP pública o DNS de tu instancia EC2
- **EC2_USERNAME**: El nombre de usuario para SSH (ej. ec2-user para Amazon Linux, ubuntu para Ubuntu)

### Importante: Formato de EC2_SSH_PRIVATE_KEY

Para el secreto EC2_SSH_PRIVATE_KEY, debes incluir el contenido completo del archivo .pem, incluyendo las líneas de inicio y fin:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA...
...
-----END RSA PRIVATE KEY-----
```

## Prueba y verificación del pipeline

Una vez que has configurado todo, es hora de probar el pipeline:

1. Crea una nueva rama en tu repositorio:
   ```bash
   git checkout -b feature/test-pipeline
   ```

2. Haz algún cambio en el código

3. Haz commit y push de tus cambios:
   ```bash
   git add .
   git commit -m "Test pipeline"
   git push origin feature/test-pipeline
   ```

4. Crea un Pull Request desde la rama feature/test-pipeline a main

5. Verifica que el workflow se active en la pestaña "Actions" de tu repositorio

6. Si todo está configurado correctamente, verás que se ejecutan los tres jobs: test, build y deploy

7. Una vez completado el despliegue, verifica que tu aplicación esté funcionando en la instancia EC2

## Solución de problemas comunes

### El workflow no se activa

- Verifica que el archivo .github/workflows/pipeline.yml esté en la rama correcta
- Asegúrate de que los eventos configurados en el workflow coincidan con tus acciones

### Fallos en los tests

- Verifica la salida de los tests para entender qué está fallando
- Asegúrate de que todas las dependencias estén correctamente instaladas

### Fallos en el build

- Verifica que el comando de build esté configurado correctamente en package.json
- Asegúrate de que todas las dependencias necesarias estén instaladas

### Fallos en el despliegue

- Verifica que los secretos estén configurados correctamente
- Asegúrate de que la instancia EC2 esté en funcionamiento y accesible
- Verifica que los grupos de seguridad permitan conexiones SSH
- Asegúrate de que PM2 esté instalado en la instancia

## Recursos adicionales

- [Documentación oficial de GitHub Actions](https://docs.github.com/es/actions)
- [Documentación de AWS EC2](https://docs.aws.amazon.com/ec2/)
- [Guía de PM2](https://pm2.keymetrics.io/docs/usage/quick-start/)

---

Con esta guía deberías poder configurar correctamente un pipeline CI/CD con GitHub Actions y AWS EC2. Recuerda que este es un proceso complejo y puede requerir ajustes según las especificidades de tu proyecto. 