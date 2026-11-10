# Preentrega 7 · Plan de monitoreo y auditoría AWS
**Autora:** Ivanna Micaela Arzamendia · Cloud Computing AWS · 2026

Propuesta académica para la arquitectura de la preentrega 6: VPC 10.0.0.0/16 en sa-east-1, dos AZ, EC2 privadas, ALB, dos NAT, RDS PostgreSQL Multi AZ y S3 privado con CloudFront. No se realizó un despliegue ni se afirma que existan recursos reales.

## Archivos
- `PreEntrega7_Plan_Monitoreo_Ivanna_Arzamendia.docx`: plan completo, recursos, métricas, matriz de alarmas, descripción del dashboard, auditoría y validación.
- `dashboard/dashboard-template.json`: definición de panel con métricas, estado de alarmas y Logs Insights.
- `auditoria/eventbridge-red.json`: patrón de eventos críticos de red para EventBridge. No crea la regla ni su destino.

## Cómo revisar la entrega
El Word cubre todos los puntos de la consigna. La descripción del panel es el mock-up explicativo solicitado. El JSON es un complemento editable, no evidencia de configuración en una cuenta real. Para entregar en GitHub, subir esta carpeta descomprimida y sus subcarpetas al repositorio asignado.

## Preparación del dashboard
1. Trabajar en la cuenta autorizada y validar costos antes de crear recursos.
2. Sustituir todas las cadenas `REEMPLAZAR` del JSON por datos de la cuenta: IDs de EC2 A/B, NAT A/B, identificador de RDS, nombre real del bucket, ID de cuenta y ARNs de alarmas ya creadas.
3. `REEMPLAZAR_ALB_SUFFIX` es el valor de dimensión LoadBalancer (ejemplo: app/nombre/id), no un ARN. `REEMPLAZAR_TG_SUFFIX` es el valor targetgroup/nombre/id.
4. Crear el log group `/ivanna/prod/api`, enviar logs JSON por CloudWatch Agent y configurar el filtro y las alarmas descritos en el Word. Configurar SNS y confirmar la suscripción del email real de la administradora.
5. Habilitar Billing Alerts. Crear la alarma EstimatedCharges y el tópico SNS de costos en us-east-1. Los recursos operativos y su tópico están en sa-east-1.
6. En CloudWatch > Dashboards > Create dashboard, abrir Actions > View/edit source (según idioma de la consola) y pegar la plantilla editada. Alternativamente usar PutDashboard mediante CLI/API. Revisar mensajes de validación y datos antes de utilizar el panel.
7. Crear AWS Budgets con un presupuesto mensual de USD 10 y aviso FORECASTED al 100%. El enlace del widget abre Budgets; seleccionar allí el presupuesto del proyecto. El forecast no es una métrica nativa de este JSON.

## Qué muestran los widgets
| Recurso | Datos | Interpretación |
|---|---|---|
| EC2 | CPUUtilization, StatusCheckFailed, NetworkIn/Out | CPU %, fallo de estado y bytes por período, separados por instancia. |
| NAT | ErrorPortAllocation, BytesOutToDestination | Errores de puertos y tráfico de salida por gateway. |
| RDS | CPUUtilization, FreeStorageSpace | CPU y espacio libre. El panel convierte bytes a GiB con metric math. |
| S3 | BucketSizeBytes, NumberOfObjects | Tamaño de clase Standard y cantidad total, actualización diaria. Los widgets muestran 14 días. |
| ALB | HealthyHostCount, HTTPCode_Target_5XX_Count | Capacidad saludable y errores de la aplicación detrás del balanceador. |
| API | ErrorCount y Logs Insights | Total de status 500–599 y registros recientes que explican la falla. |
| Costos | EstimatedCharges | Gasto estimado acumulado, publicado en us-east-1. No es forecast. |

Las métricas operativas usan 300 segundos. BucketSizeBytes y NumberOfObjects usan 86400; EstimatedCharges usa 21600 y Maximum. La ventana general es de 24 h. Si cambia la clase de S3, agregar las series StorageType necesarias; StandardStorage no representa todo el bucket si hay otras clases.

## Alarmas y supuestos
Los umbrales y las acciones están en las matrices del documento. Los USD 10 son una advertencia temprana, no el precio calculado de la arquitectura. Para FreeStorageSpace se asumen 100 GiB aprovisionados: 15% equivale a 15 GiB (16.106.127.360 bytes); cambiar el umbral si se modifica la capacidad. S3 > 50 GiB señala crecimiento, no agotamiento del bucket. Crear una alarma por instancia o NAT, sin promediar las AZ.

No considerar la ausencia de datos como salud confirmada. Revisar recursos detenidos, dimensiones, permisos y publicación. Las alarmas de CPU, RDS y estado usan missing; los contadores esporádicos y ErrorCount usan notBreaching. SNS avisa cambios de estado, no cada evaluación.

## Logs y auditoría
CloudWatch Agent y rol IAM limitado para `/ivanna/prod/api` (30 días). JSON de logs con status numérico. Filtro: `{ $.status >= 500 && $.status < 600 }`, valor 1, namespace Ivanna/API, métrica ErrorCount, sin dimensiones y default 0. VPC Flow Logs: `/ivanna/prod/vpc-flow`, 14 días.

CloudTrail: trail multirregión, eventos de gestión, bucket de auditoría privado y cifrado, integridad activada, retención interna de 365 días incluyendo versiones no actuales. Los eventos de datos S3 son selectivos y se habilitan aparte.

AWS Config: recorder, rol IAM y delivery channel en sa-east-1; reglas `s3-bucket-public-read-prohibited`, `restricted-ssh` y `rds-instance-public-access-check`. El incumplimiento se notifica por EventBridge > SNS. Config detecta; no bloquea los cambios.

Importar el patrón `auditoria/eventbridge-red.json` en una regla EventBridge en sa-east-1 y asociar SNS con política que autorice EventBridge. Captura cambios e intentos críticos de SG, rutas y VPC. No contiene el despliegue de destinos ni permisos. Para otras regiones replicar la regla. La remediación inicial será manual: investigar identidad, evaluar impacto, corregir y verificar cumplimiento. SSM Automation puede incorporarse después para cambios controlados, como restaurar Block Public Access; nunca borrar recursos automáticamente como respuesta genérica.

## Pruebas propuestas
Verificar dimensiones y series; confirmar SNS; probar ALARM/OK en laboratorio; generar logs de error y validar el filtro; comprobar EstimatedCharges; probar un cambio inocuo de SG y revisar CloudTrail, EventBridge y Config. No borrar una VPC productiva para probar DeleteVpc. Un estado de alarma forzado comprueba notificación, no la recolección real de métricas.
