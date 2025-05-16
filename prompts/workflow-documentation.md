# Documentación del Pipeline de CI/CD para Backend

Este documento detalla el flujo de trabajo (workflow) de GitHub Actions configurado para implementar un pipeline de CI/CD que se activa con un push a una rama que tiene un Pull Request abierto.

## Estructura del Pipeline

El pipeline está compuesto por tres trabajos principales:

1. **Test**: Ejecuta pruebas en el código del backend
2. **Build**: Genera un build del backend
3. **Deploy**: Despliega el backend en una instancia EC2 de AWS

## Trigger del Pipeline

El pipeline se activa cuando:

- Se realiza un push a cualquier rama (excepto `main`)
- El push incluye cambios en la carpeta `backend/`
- La rama tiene un Pull Request abierto

## Trabajo 1: Test

Este trabajo realiza las siguientes acciones:

1. Checkout del código
2. Configuración de Node.js v18
3. Instalación de dependencias con `npm ci`
4. Ejecución de pruebas con `npm test`

## Trabajo 2: Build

Este trabajo se ejecuta después de que el trabajo de pruebas se complete con éxito y realiza:

1. Checkout del código
2. Configuración de Node.js v18
3. Instalación de dependencias con `npm ci`
4. Construcción del backend con `npm run build`
5. Carga del artefacto de construcción (carpeta `dist/`) para su uso en el paso de despliegue

## Trabajo 3: Deploy

Este trabajo se ejecuta después de que el trabajo de construcción se complete con éxito y realiza:

1. Descarga del artefacto de construcción
2. Configuración de credenciales de AWS
3. Despliegue en la instancia EC2 mediante SSH:
   - Compresión del código construido
   - Transferencia de archivos a la instancia EC2
   - Descompresión y configuración en el servidor
   - Reinicio de la aplicación mediante PM2

## Variables de Entorno y Secretos

El pipeline utiliza los siguientes secretos configurados en el entorno `backend_deplyment`:

- `AWS_ACCESS_ID`: ID de acceso para AWS
- `AWS_ACCESS_KEY`: Clave de acceso para AWS
- `EC2_INSTANCE`: Dirección IP o nombre de dominio de la instancia EC2
- `EC2_SSH_KEY`: Clave SSH privada para conectarse a la instancia EC2

## Flujo Completo

1. Un desarrollador crea un Pull Request o realiza un push a una rama con un PR abierto
2. GitHub Actions detecta los cambios y ejecuta el trabajo de pruebas
3. Si las pruebas pasan, se ejecuta el trabajo de construcción
4. Si la construcción es exitosa, se ejecuta el trabajo de despliegue
5. La aplicación queda desplegada en la instancia EC2

Este pipeline garantiza que solo el código que pasa las pruebas y se construye correctamente será desplegado en el entorno, manteniendo la calidad y la estabilidad del servicio. 