# week-3-lab-11

# Diseño AWS Cloud Architecture

# Parte 1: Diseñando la Infraestructura en la Nube

## 1. VPC (Virtual Private Cloud)
### VPC: 
* Necesitamos una VPC llamada ECommerceVPC con un rango CIDR de 10.0.0.0/16.

### Subnets:
* Public Subnet 1: 10.0.0.128/20 para instancias de aplicación y Fargate.
* Public Subnet 2: 10.0.0.144/20 para instancias de aplicación FE y Fargate, redundancia y balanceo de carga.
* Private Subnet 1: 10.0.0.0/19  para instancias de aplicación BE y Fargate
* Private Subnet 2: 10.0.32.0/19 para instancias de aplicación BE y Fargate y redundancia y balanceo de carga.

### Internet Gateway: 
* Asociado a la VPC para permitir el acceso a Internet.

### NAT Gateway: 
* Permite que las instancias en subredes privadas accedan a Internet sin ser accesibles desde el exterior, sumamente importante en seguridad, pues se evita accesos no autorizados y no se exponen las instancia hacía internet.

### Route Tables:
* Public Route Table: Asociada a las subredes públicas, con una ruta al Internet Gateway.
* Private Route Table: Asociada a las subredes privadas, con una ruta al NAT Gateway.

## 2. Compute (EC2 y Fargate)

### EC2 Instances:
### Web Servers: 
* Lanzar instancias EC2 (t2.micro) en las subredes públicas con Amazon Linux 2. Es donde vivirá en front end de la app.
  
### Security Groups: 
* Permitir tráfico HTTP (puerto 80) y HTTPS (puerto 443) desde cualquier origen, y SSH (puerto 22) solo desde direcciones IP específicas.Esto permite solo tener y permitir conexiones a puertos expecificos, lo cual reduce la superficie de exposición.

### Fargate:
### ECS Tasks: 
* Configurar servicios Fargate para ejecutar contenedores Docker en las subredes privadas, lo que permite una mayor flexibilidad y escalabilidad sin gestionar servidores.

### 3. Storage (S3)
### S3 Bucket: 
* Crear un bucket ecommerce-static-assets para almacenar archivos estáticos como imágenes y videos. Adicional se coloca otro bucket (ecommerce-static-resources) para servidor contenido estatico con el CDN de cloud front.
  
### Bucket Policy: 
* Configurar políticas de acceso para permitir la lectura pública de objetos, pero restringir la escritura solo a usuarios autorizados, para el bucket que sirve al CDN solo se debe permitir el consumo de objetos únicamente al grupo de seguridad asociado a la distribución cloudfront.


# Parte 2: Configuración de IAM
## IAM Roles and Policies
* Developer Role: Crear un rol DeveloperRole con permisos limitados para acceder a los recursos necesarios para el desarrollo y pruebas.
* Admin Role: Crear un rol AdminRole con permisos más amplios, pero limitados a tareas administrativas específicas. Nunca usar el usuario root asociado a la cuenta AWS.
* Application Server Role: Crear un rol AppServerRole para las instancias EC2 y tareas Fargate con permisos para acceder al bucket S3 (s3:GetObject, s3:PutObject). Se recomienda crear identidades para cada servicio, esto permite tener mayor control sobre los permisos de cada servicio utilizado.

# Parte 3: Estrategia de Gestión de Recursos
## Auto Scaling and Load Balancing
### Auto Scaling Group:
* Configurar un Auto Scaling Group para las instancias EC2, asegurando que siempre haya al menos 2 instancias y un máximo de 6, dependiendo de la carga. Para los contenedores corriendo en fargate, se administrarán con el HPA configurado, es decir, al ser serverless, con la configuración indicada se podrán escalar.

### Scaling Policies: 
* Basadas en métricas como el uso de CPU (escalar cuando el promedio supere el 70%). Si los EC2 demoran en arrancar, se puede bajar el umbral a un 50 o 60%, con el objetivo de tener un tiempo razonable para tener lista la siguiente instancia sin degradar los servicios.
### Elastic Load Balancer (ALB):
* Configurar un Application Load Balancer (ALB) para distribuir el tráfico entrante entre las instancias EC2 y tareas Fargate en las subredes públicas. Se recomienda evaluar si se deberá colocar un NLB (Network Loab Balancer) frente al ALB, ya que este permite tener una ip elastica que no cambia si hay reinicios.

### Target Groups: 
* Crear grupos de destino para registrar las instancias EC2 y las tareas Fargate, lambda y servicios que deban comunicarse, esto permite tener una politica de accesos robusta combinada con ACLs.

## Cost Management

## AWS Budgets:
* Configurar un presupuesto para monitorear los gastos mensuales y recibir alertas cuando el costo se acerque a los límites definidos. Si ya se cuentan con otras aplicaciones, cuentas aws con recursos, se recomienda hacer uso de recursos compartidos y consolidar la facturación, esto permite ahorrar costos de facturación.

# Parte 4: Implementación Teórica
## Flujo de Datos y Control
* Usuarios: Los usuarios acceden a la aplicación web a través de sus navegadores, enviando solicitudes HTTP/HTTPS que pasan por un WAF.
* Route 53: Gestiona los dominios y dirige el tráfico a CloudFront y este lo pasará al ALB.
* CloudFront: Distribuye contenido estático desde S3 para una entrega rápida y segura.
* WAF (Web Application Firewall): Filtra las solicitudes para proteger contra ataques comunes, como inyecciones SQL y cross-site scripting (XSS) entre otros ataques.
* ALB: El ALB distribuye las solicitudes entre las instancias EC2 y las tareas Fargate en las subredes públicas.
* EC2 Instances y Fargate Tasks: Procesan las solicitudes. Si necesitan acceder a archivos estáticos, consultan el bucket S3.
* S3 Bucket: Almacena y sirve archivos estáticos a los contenedores y  cloudfront. Los buckets que almacenan data de la aplicación, se les ha configurado un ciclo de vida, donde despues de 3 meses, los objetos se mueven a glacier y despues de 3 años estos se eliminan.
* RDS Aurora Serverless: Las instancias EC2 y las tareas Fargate se conectan a una base de datos RDS Aurora en las subredes privadas para operaciones de lectura/escritura, donde al ser serverless, no se tiene carga administrativa de la bd.
* Secret Manager: Almacena credenciales sensibles y otros secretos necesarios para la aplicación, accedidos por las instancias EC2 y tareas Fargate, por ejemplo cadenas de conexión a Aurora, Certificados, etc.
* Lambda: Ejecuta funciones bajo demanda para notificaciones u otras tareas event-driven. En este caso, envíar correos electronicos a los usuarios.
* CloudWatch: Monitorea y registra los logs y métricas de la aplicación para supervisión y alertas.
Cognito: Gestiona la autenticación y autorización de usuarios, proporcionando un mecanismo seguro de inicio de sesión.
* SES (Simple Email Service): Envía correos electrónicos transaccionales, como confirmaciones de pedido o restablecimientos de contraseña(en conjunto con congnito).


# Parte 5: Discusión y Evaluación
## Elección de Servicios

* EC2, Fargate, S3, VPC: Seleccionados por su capacidad de escalabilidad, seguridad y flexibilidad.
ALB, WAF, Route 53, CloudFront, Auto Scaling: Aseguran la distribución eficiente del tráfico, la protección contra ataques comunes y la capacidad para manejar picos de demanda.
* RDS Aurora Serverless, Secret Manager: Proveen almacenamiento de datos seguro y gestión de secretos que incluso ofrece rotación automatica.
* Lambda, CloudWatch, Cognito, SES: Añaden funcionalidad avanzada para notificaciones, monitoreo, autenticación de usuarios y comunicación por correo electrónico.
  
## Políticas de IAM
* Seguridad: Las políticas de IAM aseguran que cada usuario y servicio tenga solo los permisos necesarios, minimizando el riesgo de accesos no autorizados, manteniendo el menor privilegio y la menor superficie de exposición.

* Cumplimiento: Facilitan el cumplimiento de principios de seguridad como el principio de menor privilegio posible.
  
## Estrategia de Gestión de Recursos

* Escalabilidad: El autoescalado garantiza que la aplicación pueda manejar aumentos en la carga de trabajo automáticamente y cuenta con una redundancia, y replicación hacía una segunda zona de disponibilidad.
* Costo-Eficiencia: Los presupuestos de AWS ayudan a monitorear y controlar los costos, evitando sorpresas en la factura mensual.

# Diagrama de arquitectura:
![lab-11-aws-architecture](https://github.com/ventura-gorostieta/week-3-lab-11/assets/97199485/6415e6a1-fd27-4fee-82fc-c8c2bf25781d)



