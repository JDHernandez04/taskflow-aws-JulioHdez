# Implementación de Despliegue Continuo (CI/CD) con AWS CodePipeline

**Autor:** Julio Daniel Hernández Medrano  
**Proyecto:** TaskFlow API (`taskflow-aws-JulioHdez`)

## 1. Resumen
Este documento detalla la arquitectura, configuración y ejecución de un pipeline de Integración y Despliegue Continuos (CI/CD) para la aplicación TaskFlow API, desarrollada en Spring Boot (Java 21). El diseño del pipeline asegura que cualquier integración de código en la rama `main` desencadene un flujo automatizado de compilación, empaquetado y despliegue en un entorno de Amazon Web Services (AWS), eliminando la necesidad de ejecución manual de comandos en el servidor.

## 2. Arquitectura y Componentes de AWS
La infraestructura se diseñó siguiendo los principios de mínimo privilegio y automatización de despliegues. Los servicios involucrados son:

*   **AWS IAM (Identity and Access Management):** Gestión estricta de permisos. Se crearon roles dedicados para EC2 (`taskflow-ec2-role`), CodeBuild y CodeDeploy, limitando el acceso exclusivamente a los recursos necesarios (por ejemplo, `AmazonS3ReadOnlyAccess` para la instancia).
*   **Amazon EC2 (Elastic Compute Cloud):** Servidor de alojamiento (instancia `t3.micro` con Amazon Linux 2023). Configuró un Security Group (`taskflow-ec2-sg`) permitiendo tráfico SSH (puerto 22) y HTTP (puerto 8080).
*   **Amazon S3 (Simple Storage Service):** Bucket privado (`taskflow-artefactos-JulioHdez`) con versionado habilitado, utilizado como almacén seguro para los artefactos de construcción.
*   **AWS CodeBuild:** Entorno de compilación efímero. Utiliza la imagen `amazonlinux-x86_64-standard` para ejecutar Maven, compilar el código fuente y empaquetar el ejecutable junto con los scripts de despliegue.
*   **AWS CodeDeploy:** Servicio de orquestación de despliegues en la instancia EC2. Se comunica con el servidor a través de un agente local (CodeDeploy Agent) y ejecuta las instrucciones definidas en el archivo de especificaciones.
*   **AWS CodePipeline:** Servicio integrador que coordina GitHub (origen), CodeBuild (construcción) y CodeDeploy (despliegue) en un único flujo de trabajo continuo.

## 3. Configuración del Repositorio e Infraestructura como Código (IaC)
Para permitir el control de CodeDeploy sobre la instancia, se integraron los siguientes archivos de configuración en la raíz del repositorio:

*   `buildspec.yml`: Define las fases de construcción. Especifica el entorno (`corretto21`), ejecuta la orden `mvn package`, renombra el archivo `.jar` resultante para estandarizar su uso y empaqueta los artefactos de salida junto con los archivos de configuración.
*   `appspec.yml`: Controla el ciclo de vida del despliegue en la instancia EC2 mediante hooks específicos:
    *   `ApplicationStop`: Detiene el servicio actual para liberar el puerto.
    *   `AfterInstall`: Ajusta permisos y traslada el archivo `.jar` y el servicio a sus directorios correspondientes (`/opt/taskflow`).
    *   `ApplicationStart`: Inicia el servicio de la API.
    *   `ValidateService`: Verifica la disponibilidad de la aplicación consultando el endpoint de salud (health check).
*   `taskflow.service`: Definición de la aplicación como un servicio de `systemd`. Esto reemplaza la ejecución temporal (`nohup`) y asegura que el sistema operativo demonice el proceso, reiniciándolo automáticamente en caso de fallos.
*   **Permisos de ejecución:** Se aplicó el modo `100755` (ejecutable) a los scripts bash mediante Git para evitar errores de permisos de ejecución en el servidor Linux durante el ciclo de vida del despliegue.

## 4. Orquestación del Pipeline
El flujo se configuró en AWS CodePipeline mediante tres etapas consecutivas:
1.  **Source:** Configurado mediante un Webhook hacia GitHub para detectar cambios en la rama `main` del repositorio `taskflow-aws-JulioHdez`.
2.  **Build:** CodeBuild descarga el código fuente, ejecuta el `buildspec.yml` y deposita el archivo `.zip` resultante (BuildArtifact) en el bucket S3.
3.  **Deploy:** CodeDeploy toma el BuildArtifact de S3 y, basándose en la etiqueta de la instancia (`Name: taskflow-ec2`), ejecuta secuencialmente los hooks del `appspec.yml`.

## 5. Resolución de Incidentes (Troubleshooting)
Durante la fase de integración, se documentó y resolvió el siguiente incidente:

*   **Incidente:** Fallo en la etapa de Build (Error: `AccessDenied s3:GetObject`).
*   **Causa Raíz:** El rol de servicio generado automáticamente para CodeBuild carecía de permisos de lectura y escritura sobre el bucket S3 personalizado (`taskflow-artefactos-JulioHdez`), ya que las políticas por defecto solo autorizan buckets creados nativamente por CodePipeline.
*   **Solución:** Se intervino la consola de IAM para modificar el rol `codebuild-taskflow-build-service-role`, adjuntando la política administrada `AmazonS3FullAccess`. Posteriormente se reintentó la ejecución de la etapa, la cual finalizó con éxito.

## 6. Resultados y Validación
La implementación cumplió con los criterios de aceptación (DoD). El pipeline completó su ejecución de forma exitosa (estado Success en todas las etapas). La disponibilidad de la aplicación fue validada externamente mediante una petición HTTP GET al endpoint `/info` por el puerto 8080 de la dirección IP pública de la instancia, retornando correctamente la carga útil JSON con la versión actual de la API. Todo el proceso operó sin necesidad de establecer una conexión SSH manual al servidor.
