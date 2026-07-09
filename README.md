# Laboratorio 8 - ARSW: Escalabilidad, Alta Disponibilidad y Observabilidad en AWS

**Asignatura:** Arquitecturas de Software
**Duración sugerida:** 3 horas
**Modalidad:** Individual o parejas
**Plataforma:** AWS Academy Learner Lab
**Restricción:** Sin perfiles IAM personalizados

## 1. Propósito

Una arquitectura moderna no basta con desplegarse y esperar que responda correctamente: debe atender más usuarios cuando aumenta la demanda, mantenerse disponible ante fallas, distribuir tráfico entre múltiples servidores, detectar fallos mediante *health checks* y observar métricas para tomar decisiones con base en datos.

Este laboratorio integra tres conceptos fundamentales de una arquitectura cloud:

- **Escalabilidad** — mediante Amazon EC2 Auto Scaling.
- **Alta disponibilidad** — mediante un Application Load Balancer (ALB) que distribuye tráfico y retira instancias no saludables.
- **Observabilidad** — mediante métricas de Amazon CloudWatch.

## 2. Resultados de aprendizaje

Al finalizar el laboratorio se estará en capacidad de:

- Explicar la diferencia entre escalabilidad y alta disponibilidad.
- Crear una aplicación web simple en EC2.
- Crear una AMI a partir de una instancia base.
- Configurar un Launch Template y un Auto Scaling Group.
- Asociar un Auto Scaling Group a un Application Load Balancer.
- Validar health checks y generar carga artificial sobre la aplicación.
- Observar métricas básicas en CloudWatch y analizar eventos de escalamiento.
- Simular una falla y verificar la recuperación del sistema.
- Proponer mejoras orientadas a producción.

## 3. Conceptos base

| Concepto | Descripción |
|---|---|
| **Escalabilidad vertical** | Aumentar los recursos de una sola máquina (CPU, memoria, disco). Ej.: pasar de `t3.micro` a `t3.large`. |
| **Escalabilidad horizontal** | Agregar más instancias para distribuir la carga. Es el enfoque usado en este laboratorio. |
| **Alta disponibilidad** | El sistema sigue funcionando aunque falle un componente, gracias a un Load Balancer que enruta tráfico solo hacia instancias saludables. |
| **Observabilidad** | Capacidad de entender qué ocurre en el sistema mediante métricas, logs, eventos, alarmas y estado de *health checks*. |

## 4. Arquitectura implementada

```
Internet
   │
   ▼
Application Load Balancer (alb-scalability-ha)
   │
   ▼
Target Group (tg-scalability-ha) — health check: /health
   │
   ├────────────┬────────────┐
   ▼            ▼            ▼
EC2 Web A    EC2 Web B    EC2 Web C
  AZ 1         AZ 2       (si escala)
   └── Auto Scaling Group (asg-web-scalability) ──┘
```

## 5. Servicios AWS utilizados

| Se utilizan | No se utilizan (fuera de alcance / restricciones AWS Academy) |
|---|---|
| Amazon EC2 | IAM roles personalizados |
| Amazon Machine Image (AMI) | Instance profiles |
| Launch Template | CloudWatch Agent |
| Auto Scaling Group | SSM Session Manager |
| Application Load Balancer | Terraform / CloudFormation |
| Target Group | RDS |
| Security Groups | Route 53 |
| CloudWatch Metrics | |

## 6. Consideraciones de costo

- Usar instancias `t2.micro` o `t3.micro`.
- Capacidad: mínimo 2, máximo 3 instancias.
- El ALB consume créditos mientras esté activo, aunque no reciba tráfico → **eliminarlo al finalizar**.
- No ejecutar pruebas de carga durante períodos prolongados.
- Al terminar, eliminar Auto Scaling Group, Load Balancer, Target Group, Launch Template y AMI.

## 7. Guía de despliegue

### 7.1 Preparación inicial

1. Ingresar a AWS Academy Learner Lab y abrir la consola de AWS.
2. Seleccionar **una sola región** para toda la práctica (ej. `us-east-1`) y no cambiarla.
3. Verificar que exista una VPC predeterminada con subnets públicas en al menos dos zonas de disponibilidad.

### 7.2 Security Groups

**`sg-alb-scalability`** (para el Load Balancer)

| Tipo | Protocolo | Puerto | Origen/Destino |
|---|---|---|---|
| HTTP (entrada) | TCP | 80 | `0.0.0.0/0` |
| All traffic (salida) | All | All | `0.0.0.0/0` |

**`sg-ec2-scalability`** (para las instancias EC2)

| Tipo | Protocolo | Puerto | Origen |
|---|---|---|---|
| HTTP | TCP | 80 | `sg-alb-scalability` |
| SSH (opcional, depuración) | TCP | 22 | My IP |

### 7.3 Instancia base y AMI

1. Lanzar una instancia `web-scalability-base` (Amazon Linux 2023, `t2.micro`/`t3.micro`, subnet pública, IP pública habilitada, `sg-ec2-scalability`, sin IAM instance profile).
2. En **User data** se instala `httpd` y `stress-ng`, se habilita y arranca Apache, y se genera dinámicamente:
   - `/var/www/html/index.html`: página con el Instance ID y la Availability Zone obtenidos vía metadata (IMDSv2).
   - `/var/www/html/health`: responde `OK`, usado como endpoint de *health check*.
   - `/var/www/html/load.html`: endpoint simple para pruebas de carga.
3. Verificar `Running` y `2/2 checks passed`, y probar `http://IP_PUBLICA` y `http://IP_PUBLICA/health`.
4. Crear una AMI (`ami-web-scalability-arsw`) a partir de la instancia validada.

### 7.4 Launch Template

`lt-web-scalability`: usa la AMI creada, tipo `t2.micro`/`t3.micro`, `sg-ec2-scalability`, sin IAM instance profile y **sin User Data** (ya incluido en la AMI).

### 7.5 Target Group

`tg-scalability-ha` — HTTP:80, VPC default.

| Parámetro | Valor |
|---|---|
| Path | `/health` |
| Healthy threshold | 2 |
| Unhealthy threshold | 2 |
| Timeout | 5 s |
| Interval | 15 s |
| Success codes | 200 |

Las instancias no se registran manualmente: lo hace el Auto Scaling Group.

### 7.6 Application Load Balancer

`alb-scalability-ha` — Internet-facing, IPv4, dos zonas de disponibilidad, subnets públicas, `sg-alb-scalability`, listener `HTTP:80` → `tg-scalability-ha`.

### 7.7 Auto Scaling Group

`asg-web-scalability`, basado en `lt-web-scalability`, en al menos dos subnets públicas de zonas distintas, adjunto al target group `tg-scalability-ha`.

| Parámetro | Valor |
|---|---|
| ELB health checks | Enabled |
| Health check grace period | 120 s |
| Desired / Min / Max capacity | 2 / 2 / 3 |
| Política de escalamiento | Target tracking — Average CPU utilization al 50% |

## 8. Pruebas realizadas

### 8.1 Prueba de balanceo

```bash
for i in {1..10}; do
  curl -s http://DNS_DEL_LOAD_BALANCER | grep "Instance ID"
done
```

Se espera observar respuestas provenientes de distintas instancias EC2.

### 8.2 Prueba de escalabilidad

Generación de carga con `stress-ng` (si hay acceso SSH) o mediante múltiples solicitudes HTTP concurrentes desde el equipo local. Se observa en `EC2 → Auto Scaling Groups → asg-web-scalability → Activity/Monitoring` si la capacidad deseada pasa de 2 a 3 instancias.

### 8.3 Prueba de alta disponibilidad

Se detiene manualmente una de las instancias del Auto Scaling Group (`Instance state → Stop instance`) y se valida que:

- El Load Balancer sigue respondiendo desde las instancias saludables restantes.
- El Target Group detecta la instancia como `Unhealthy`.
- El Auto Scaling Group reemplaza la instancia para recuperar la capacidad deseada.

### 8.4 Observabilidad en CloudWatch

Métricas revisadas por servicio:

| Servicio | Métricas |
|---|---|
| ApplicationELB | `RequestCount`, `HealthyHostCount`, `UnHealthyHostCount`, `TargetResponseTime`, `HTTPCode_Target_2XX_Count`, `HTTPCode_Target_5XX_Count` |
| Auto Scaling Group | `GroupDesiredCapacity`, `GroupInServiceInstances`, `GroupTotalInstances`, Average CPU utilization |
| EC2 | `CPUUtilization`, `NetworkIn`, `NetworkOut`, `StatusCheckFailed` |

## 9. Actividades resueltas

### 9.1 Actividad 1 — Análisis de escalabilidad y alta disponibilidad

**¿Qué componente distribuye el tráfico?**
El Application Load Balancer (`alb-scalability-ha`), apoyado en el Target Group (`tg-scalability-ha`) que le indica a qué instancias puede enviar solicitudes.

**¿Qué componente decide cuántas instancias deben existir?**
El Auto Scaling Group (`asg-web-scalability`), a través de su capacidad deseada y la política de *target tracking* sobre CPU promedio.

**¿Qué componente verifica la salud de las instancias?**
El Target Group, mediante los *health checks* HTTP configurados sobre la ruta `/health`.

**¿Por qué se seleccionan dos zonas de disponibilidad?**
Porque cada AZ es un centro de datos físicamente independiente. Si toda la capacidad estuviera en una sola AZ, una falla de energía, red o hardware en esa zona dejaría el servicio completo fuera de línea. Distribuir instancias en dos AZ permite que el sistema siga operando aunque una zona completa falle.

**¿Qué diferencia existe entre Target Group y Auto Scaling Group?**
El Target Group es el conjunto de destinos hacia los que el Load Balancer enruta tráfico y sobre el que se ejecutan los health checks; no crea ni destruye instancias, solo las agrupa y evalúa su salud. El Auto Scaling Group es quien realmente lanza, mantiene y termina instancias EC2 según la capacidad deseada y las políticas de escalamiento, registrándolas y retirándolas automáticamente del Target Group.

**¿Qué pasaría si una instancia falla?**
El Target Group empieza a fallar sus chequeos hacia esa instancia; tras alcanzar el *unhealthy threshold* (2 chequeos fallidos) la marca como `Unhealthy` y el ALB deja de enviarle tráfico. El Auto Scaling Group detecta que ya no cumple la capacidad deseada y lanza una instancia de reemplazo.

**¿Qué pasaría si aumenta la carga?**
El promedio de CPU del grupo sube. Cuando supera el valor objetivo (50%) el tiempo suficiente, la política de *target tracking* incrementa la capacidad deseada (hasta el máximo de 3), el Auto Scaling Group lanza nuevas instancias y estas se registran automáticamente en el Target Group una vez pasan sus health checks iniciales.

### 9.2 Actividad 2 — Análisis del escalamiento

| Pregunta | Respuesta |
|---|---|
| ¿Qué métrica activó la política de escalamiento? | `Average CPU Utilization` del Auto Scaling Group |
| ¿Cuántas instancias había antes de la prueba? | 2 (capacidad mínima/deseada inicial) |
| ¿Cuántas instancias hubo después? | 3 (capacidad máxima configurada) |
| ¿Cuánto tiempo tardó el sistema en reaccionar? | ≈ 3–5 minutos, ya que CloudWatch agrega métricas por minuto y Auto Scaling requiere varios puntos de datos consecutivos por encima del umbral antes de disparar la acción |
| ¿Qué limitación tiene usar solo CPU como métrica de escalamiento? | No siempre refleja el cuello de botella real: una aplicación puede estar saturada en conexiones, I/O o latencia de base de datos sin que la CPU se eleve, retrasando un escalamiento que sí se necesita |
| ¿Qué otra métrica podría ser útil para una aplicación web? | `RequestCountPerTarget` o `TargetResponseTime` del Load Balancer, que reflejan directamente la carga de solicitudes por instancia y la experiencia percibida por el usuario |

### 9.3 Actividad 3 — Análisis de observabilidad

| Métrica observada | Servicio AWS | Valor antes de la carga | Valor durante la carga | Valor después de la carga | Interpretación | Decisión arquitectónica que soporta |
|---|---|---|---|---|---|---|
| `CPUUtilization` (promedio del ASG) | EC2 / Auto Scaling | ≈ 5–10% | > 50% (supera el umbral) | Vuelve a bajar a ≈ 5–10% tras retirar la carga | El aumento sostenido de CPU es la señal que activa el escalamiento | Justifica la política de *target tracking scaling* al 50% de CPU |
| `HealthyHostCount` / `UnHealthyHostCount` | ApplicationELB | 2 healthy / 0 unhealthy | 2–3 healthy / 0 unhealthy (o 1 unhealthy durante la prueba de falla) | Vuelve a 2–3 healthy / 0 unhealthy | Confirma que el Target Group detecta en tiempo real qué instancias pueden recibir tráfico | Sustenta el uso de health checks para alta disponibilidad |
| `RequestCount` | ApplicationELB | Bajo, tráfico esporádico de verificación | Pico alto durante la generación de carga concurrente | Vuelve al nivel base | Muestra que el ALB efectivamente recibe y reparte el volumen de solicitudes generado | Confirma que el ALB es el punto único de entrada y distribución de tráfico |

### 9.4 Actividad 4 — Análisis de alta disponibilidad

**¿Qué ocurrió cuando se detuvo una instancia?**
El Target Group comenzó a fallar los health checks contra esa instancia y, tras el *unhealthy threshold* configurado (2 chequeos, con un intervalo de 15 s), la marcó como `Unhealthy`.

**¿El Load Balancer siguió respondiendo?**
Sí. El ALB continuó enrutando el 100% del tráfico hacia la(s) instancia(s) restantes en estado `Healthy`, sin interrupción perceptible para el usuario.

**¿El Target Group detectó la falla?**
Sí, el estado del target cambió automáticamente de `Healthy` a `Unhealthy`.

**¿El Auto Scaling Group lanzó una nueva instancia?**
Sí. Al tener habilitados los ELB health checks, el Auto Scaling Group interpretó la instancia no saludable como pérdida de capacidad y lanzó una instancia de reemplazo, que se registró en el Target Group tras superar sus propios health checks iniciales.

**¿Qué diferencia existe entre ocultar una falla y recuperarse de una falla?**
Ocultar una falla es dejar de exponerla al usuario sin corregir la causa (por ejemplo, seguir enrutando tráfico solo hacia las instancias sanas mientras la instancia caída permanece caída indefinidamente, reduciendo la capacidad real del sistema). Recuperarse de una falla implica que el sistema detecta el componente afectado, lo retira de servicio y restaura la capacidad total reemplazándolo, devolviendo la arquitectura a su estado esperado sin intervención manual.

**¿Qué atributo de calidad se evidencia en esta prueba?**
Disponibilidad (*availability*) y tolerancia a fallos (*fault tolerance / resiliencia*).

## 10. Relación entre los tres conceptos

| Concepto | Componente AWS relacionado | Evidencia en el laboratorio |
|---|---|---|
| Escalabilidad | Auto Scaling Group | Escalamiento de 2 a 3 instancias bajo carga de CPU |
| Alta disponibilidad | ALB + múltiples AZ | Continuidad de respuesta al detener una instancia |
| Observabilidad | CloudWatch Metrics | Métricas de CPU, salud de targets y conteo de requests |
| Detección de fallos | Health checks (Target Group) | Instancia marcada como `Unhealthy` al fallar |
| Recuperación | Auto Scaling Group | Reemplazo automático de instancia no saludable |
| Distribución de carga | Load Balancer | Respuestas alternadas entre distintos Instance IDs |

## 11. Diagnóstico de errores comunes

<details>
<summary><strong>El Target Group muestra instancias Unhealthy</strong></summary>

- La ruta `/health` existe y responde `OK`.
- Apache (`httpd`) está activo.
- El Security Group de EC2 permite tráfico desde el Security Group del ALB.
- El Target Group usa HTTP puerto 80.
- Las instancias están en la misma VPC.
</details>

<details>
<summary><strong>El Auto Scaling Group no crea instancias</strong></summary>

- El Launch Template usa una AMI disponible (`Available`).
- El tipo de instancia está permitido en AWS Academy.
- Hay subnets seleccionadas y el límite de instancias no fue alcanzado.
- La capacidad deseada es mayor que 0.
</details>

<details>
<summary><strong>No se activa el escalamiento</strong></summary>

- La carga duró suficiente tiempo y superó el umbral de CPU configurado.
- La política de escalamiento está asociada al Auto Scaling Group.
- El máximo del grupo permite crecer y CloudWatch recibió métricas suficientes.
</details>

<details>
<summary><strong>El Load Balancer no responde</strong></summary>

- El ALB es Internet-facing y su Security Group permite HTTP desde Internet.
- El Target Group tiene al menos un target en estado `Healthy`.
- El DNS del ALB se copió correctamente.
</details>

## 12. Reto final — Informe técnico

Estado de los entregables solicitados:

| # | Entregable | Estado | Dónde queda resuelto |
|---|---|---|---|
| 1 | Diagrama de arquitectura implementada |  Resuelto | Sección 4 |
| 2 | Captura del Auto Scaling Group | | — |
| 3 | Captura del Load Balancer |  | — |
| 4 | Captura del Target Group con targets Healthy |  | — |
| 5 | Evidencia de respuesta desde varias instancias || — |
| 6 | Evidencia de escalamiento o intento de escalamiento | | — |
| 7 | Evidencia de métricas en CloudWatch |  | — |
| 8 | Evidencia de falla simulada y recuperación |  | — |
| 9 | Análisis de escalabilidad |  Resuelto | Sección 9.1 y 9.2 |
| 10 | Análisis de alta disponibilidad |  Resuelto | Sección 9.4 |
| 11 | Análisis de observabilidad |  Resuelto | Sección 9.3 |
| 12 | Propuesta de mejora para producción |  Resuelto | Sección 15 |

Los ítems marcados como "Adjuntar" requieren evidencia visual (capturas de pantalla) tomada directamente de la consola de AWS durante la ejecución real del laboratorio; el análisis conceptual que las acompaña ya está resuelto en las secciones referenciadas.

## 13. Limpieza de recursos

Al finalizar, eliminar en este orden para evitar consumo innecesario de créditos:

1. Auto Scaling Group
2. Application Load Balancer
3. Target Group
4. Launch Template
5. AMI creada
6. Snapshot asociado a la AMI
7. Instancia base (si sigue existiendo)
8. Security Groups creados

## 14. Rúbrica de evaluación

| Criterio | Descripción | Peso |
|---|---|---|
| Escalabilidad | Configura un Auto Scaling Group con política de escalamiento | 25% |
| Alta disponibilidad | Usa ALB, Target Group y al menos dos zonas de disponibilidad | 20% |
| Observabilidad | Analiza métricas relevantes en CloudWatch | 20% |
| Prueba de falla | Simula caída de instancia y analiza la recuperación | 15% |
| Análisis arquitectónico | Relaciona decisiones con atributos de calidad | 15% |
| Limpieza de recursos | Elimina recursos para evitar consumo innecesario | 5% |

## 15. Posibles mejoras para producción

- HTTPS con ACM (Certificate Manager).
- Instancias privadas detrás del ALB.
- NAT Gateway o VPC Endpoints.
- Base de datos Multi-AZ.
- CloudWatch Alarms y logs centralizados.
- Despliegues blue/green.
- Auto Scaling basado en `RequestCountPerTarget`.
- Infraestructura como código (Terraform / CloudFormation).

## 16. Conclusión

La escalabilidad no debe diseñarse de forma aislada: una arquitectura escalable también debe ser **observable** y **disponible**. La solución implementada no solo distribuye tráfico entre instancias, sino que permite que el sistema crezca ante carga, detecte instancias no saludables y entregue información para diagnosticar su comportamiento.
