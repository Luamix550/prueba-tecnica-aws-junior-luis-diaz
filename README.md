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

#### Paso a paso de diagnóstico

**Pregunta:** ¿Cuáles son los 3 primeros chequeos que realizarías en la consola de AWS para identificar la causa?

**Respuesta:**

1. **Security Group de RDS (regla de entrada / inbound):** confirmar que exista una regla que permita el puerto de la base de datos (3306 para MySQL) desde el origen correcto, idealmente el Security Group de la instancia EC2 y no una IP fija. Este es el chequeo más importante: los Security Groups son *stateful*, así que el tráfico de salida (outbound) de EC2 casi siempre viene permitido por defecto y casi nunca es la causa. La única regla que alguien tiene que configurar manualmente, y que suele faltar u olvidarse, es el inbound de RDS.
2. **Prueba de conectividad desde la instancia EC2:** conectarse por SSH a la instancia y probar la conexión hacia el endpoint de RDS con un comando como `telnet <endpoint-rds> 3306` o `nc -zv <endpoint-rds> 3306`. Esto confirma si el problema es de red/Security Group o si es otra cosa (credenciales, motor caído, etc.).
3. **Estado de la instancia RDS en la consola** (pestaña de estado y de *Events*): verificar que esté `Available` y no en `Storage-full`, en reinicio o en failover, y revisar las métricas de CloudWatch (`FreeStorageSpace`, `DatabaseConnections`) para descartar que esté saturada o sin espacio en disco.

#### Solución de Red/Security Groups

**Pregunta:** Si descubres que la instancia RDS o EC2 cambió de IP o que el Security Group fue modificado, ¿cómo corregirías la configuración de red/seguridad para restaurar la conexión siguiendo buenas prácticas?

**Respuesta:** Usaría el DNS (endpoint) de RDS en vez de IPs fijas en la configuración de la aplicación, y en las reglas de entrada del Security Group referenciaría el Security Group de EC2 en lugar de una IP fija, aplicando el principio de mínimo privilegio.

#### Acción para Almacenamiento

**Pregunta:** Si la base de datos no tiene espacio en disco suficiente, ¿qué acción inmediata ejecutas en RDS para restablecer el servicio?

**Respuesta:** Aumentaría el almacenamiento asignado (Allocated Storage) de la instancia RDS desde Modify, no la clase de instancia, porque eso es lo que aumenta el disco y se puede aplicar de inmediato sin downtime.

---

### Escenario B (S3 / Frontend - Errores 403 y CORS)

#### Explicación técnica

**Pregunta:** Explica la diferencia entre un error por Permisos (IAM / S3 Bucket Policy) y un error de CORS.

**Respuesta:** IAM es un servicio de AWS que permite el acceso y asignación de roles a usuarios, y S3 Bucket Policy son las reglas que se le dan al propio bucket: quién tiene permitido usarlo, desde dónde y cómo. Ambos son permisos que decide AWS del lado del servidor. CORS es distinto: es un mecanismo de seguridad que implementa el navegador, no AWS ni el backend, para controlar qué orígenes pueden hacer peticiones hacia el backend. El error de CORS aparece cuando el servidor sí procesó la petición pero no incluyó los headers correctos (origen, métodos, headers permitidos) para que el navegador deje pasar esa respuesta.

#### Corrección de Permisos

**Pregunta:** ¿Qué verificación realizarías en la política de Bucket de S3 o en las credenciales de la aplicación para resolver el error 403 Forbidden?

**Respuesta:** En la Bucket Policy revisaría primero si hay algún `Deny` explícito bloqueando la acción, ya que un Deny siempre gana sobre un Allow. Luego confirmaría que exista un `Allow` para la acción necesaria (`s3:PutObject`) sobre el recurso correcto (`arn:aws:s3:::mi-bucket/*`), y revisaría el Block Public Access del bucket y de la cuenta, porque puede bloquear el acceso aunque la policy lo permita. En las credenciales de la aplicación, si el frontend sube directo a S3 con credenciales temporales (por ejemplo de un Cognito Identity Pool), verificaría que el rol IAM asociado tenga permiso `s3:PutObject` sobre ese bucket. Si se usa una URL pre-firmada generada por el backend, verificaría que no haya expirado y que el usuario/rol que la firmó tenga permiso sobre esa acción y ese bucket.

#### Configuración CORS

**Pregunta:** Muestra los pasos y la configuración JSON de cómo habilitar CORS en la consola de S3 para permitir peticiones desde `https://mi-app.com`.

**Respuesta:** Los pasos serían:

1. Entrar al bucket en la consola de S3.
2. Ir a la pestaña **Permissions**.
3. Bajar hasta la sección **Cross-origin resource sharing (CORS)**.
4. Dar click en **Edit**.
5. Pegar el JSON de configuración.
6. Dar **Save changes**.

```json
[
  {
    "AllowedHeaders": ["Authorization", "Content-Type"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://mi-app.com"],
    "ExposeHeaders": ["ETag"]
  }
]
```

---

## Módulo 2: Operaciones, Docker y Monitoreo

### Optimización de Dockerfile

**Pregunta:** ¿Cómo reordenarías las instrucciones del Dockerfile para aprovechar la caché de capas de Docker y optimizar el tiempo de compilación/despliegue?

Dockerfile optimizado (también está en [`/docker/Dockerfile`](docker/Dockerfile)):

```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

**Respuesta:** Docker guarda cada instrucción como una capa. Si copio todo el código antes de instalar las dependencias (como estaba el Dockerfile original), cualquier cambio en cualquier archivo obliga a repetir el `npm install` completo en cada build, aunque no haya cambiado ninguna dependencia. Por eso primero copio solo el `package.json`, corro `npm install`, y ya después copio el resto del código. Así, si solo cambio código y no dependencias, Docker reutiliza la instalación anterior en lugar de repetirla, y el build es mucho más rápido.
