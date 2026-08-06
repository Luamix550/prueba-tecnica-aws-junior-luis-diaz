# Prueba Técnica: Soporte & Troubleshooting AWS (Junior Level)

Repositorio de entrega para la prueba técnica de Soporte y Troubleshooting en AWS.

## Contenido

1. [Módulo 1: Resolución de Escenarios (Troubleshooting)](#módulo-1-resolución-de-escenarios-troubleshooting)
   - [Escenario A: Laravel / EC2 / RDS - Connection Timed Out](#escenario-a-laravel--ec2--rds---connection-timed-out)
   - [Escenario B: S3 / Frontend - Errores 403 y CORS](#escenario-b-s3--frontend---errores-403-y-cors)
2. [Módulo 2: Operaciones, Docker y Monitoreo](#módulo-2-operaciones-docker-y-monitoreo)
   - [Optimización de Dockerfile](#optimización-de-dockerfile)
   - [Estrategia de Alertas en CloudWatch](#estrategia-de-alertas-en-cloudwatch)

## Estructura del repositorio

```
prueba-tecnica-aws-junior-luis-diaz/
├── README.md
├── /docker/
│   └── Dockerfile
└── /docs/
```

---

## Módulo 1: Resolución de Escenarios (Troubleshooting)

### Escenario A (Laravel / EC2 / RDS - Connection Timed Out)

#### Paso a paso de diagnóstico: los 3 chequeos clave

1. **Security Group de RDS (regla de entrada / inbound):** confirmar que exista una regla que permita el puerto de la base de datos (3306 para MySQL) desde el origen correcto, idealmente el Security Group de la instancia EC2 y no una IP fija. Este es el chequeo más importante: los Security Groups son *stateful*, así que el tráfico de salida (outbound) de EC2 casi siempre viene permitido por defecto y casi nunca es la causa. La única regla que alguien tiene que configurar manualmente, y que suele faltar u olvidarse, es el inbound de RDS.
2. **Prueba de conectividad desde la instancia EC2:** conectarse por SSH a la instancia y probar la conexión hacia el endpoint de RDS con un comando como `telnet <endpoint-rds> 3306` o `nc -zv <endpoint-rds> 3306`. Esto confirma si el problema es de red/Security Group o si es otra cosa (credenciales, motor caído, etc.).
3. **Estado de la instancia RDS en la consola** (pestaña de estado y de *Events*): verificar que esté `Available` y no en `Storage-full`, en reinicio o en failover, y revisar las métricas de CloudWatch (`FreeStorageSpace`, `DatabaseConnections`) para descartar que esté saturada o sin espacio en disco.

---

## Módulo 2: Operaciones, Docker y Monitoreo

_(pendiente)_
