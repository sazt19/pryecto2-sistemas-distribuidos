# Proyecto: BookStore Monolithic App – Despliegue con Docker, NGINX y MySQL

**Materia:** ST0263 – Teleinformática

**Estudiante:** *Sara Zuluaga* *szuluagat@eafit.edu.co*

**Estudiante:** *Valeria Cardona* *vcardonau@eafit.edu.co*

**Profesor:** *Edwin Nelson Montoya Munera*, *emontoya@eafit.edu.co*

---

# 1. Descripción general del proyecto

Este proyecto implementa el despliegue de una aplicación monolítica llamada **BookStore**.
El objetivo principal fue construir un ambiente distribuido utilizando **dos máquinas virtuales (EC2) en AWS**, separadas por roles:

* **APP VM** → corre la aplicación BookStore en Docker + NGINX
* **DB VM** → corre una base de datos MySQL en Docker

Toda la comunicación se realiza mediante conexiones privadas dentro de la misma VPC, usando subredes diferentes y reglas de seguridad específicas (SG + NACL).

El enfoque principal del proyecto es aprender a:

* Contenerizar aplicaciones monolíticas
* Configurar reverse proxies con NGINX
* Administrar conectividad segura entre servicios en AWS
* Configurar ambientes de despliegue reproducibles con Docker Compose
* Diagnóstico avanzado: uso de `tcpdump`, NACL, SG, rutas, puertos expuestos, etc.

---

## 1.1 Requerimientos (objetivos) desarrollados y completados

### **Objetivo 1: Despliegue de la aplicación monolítica**

* Contenedor Docker para la app BookStore correctamente construido.
* Contenedor MySQL funcional con sus credenciales y base inicial.
* NGINX funcionando como reverse proxy (80 → 5000).
* APP VM comunicándose con DB VM usando puertos privados (3307).
* Configuración correcta de Security Groups y NACLs en ambas subredes.
* Conectividad verificada mediante:

  * `/dev/tcp`
  * `mysql -h -P`
  * `tcpdump`
  * `ss -lntp`
* Ambiente reproducible mediante `docker compose`.

**Objetivo 1 = CUMPLIDO**

---

## 1.2 Objetivos que NO se alcanzaron

### **Objetivo 2: Despliegue escalable con AWS**

* **Auto Scaling Group (ASG)**
* **Launch Templates** para instancias de la aplicación
* **Application Load Balancer (ALB)** con listeners HTTP/HTTPS
* Integración del ALB con instancias en ASG
* Balanceo de carga y health checks

### **Objetivo 3: Migración a servicios administrados**

* Migración de MySQL local a **Amazon RDS**
* Configuración de Security Groups específicos para RDS
* Conexión de la app hacia RDS

### **Objetivo 4: Almacenamiento compartido**

* Implementación de **AWS EFS** para archivos estáticos
* Montaje de EFS en múltiples instancias del ASG

### **Otros elementos no implementados**

* Automatización de HTTPS con Certbot + NGINX
* CI/CD para desplegar versiones automáticamente
* Scripts de automatización adicionales (CloudInit / bash)
* Extender el monolito hacia una arquitectura multi-servicio
* Logs centralizados (CloudWatch)
* Métricas y monitoreo
---

# 2. Arquitectura – Diseño de alto nivel

### Componentes

**APP VM**

* Docker Engine
* Docker Compose
* Contenedor: BookStore monolítico (Flask/Python)
* NGINX reverse proxy (80 → 5000)

**DB VM**

* Docker Engine
* Docker Compose
* Contenedor: MySQL 8.0
* Puerto privado expuesto: 3307

### Conectividad Privada

* Ambas instancias dentro de **la misma VPC**, pero en **subredes distintas**

  * APP: 172.30.x.x
  * DB: 172.31.x.x
* Seguridad configurada mediante:

  * Security Groups
  * Network ACLs (NACL) sincronizadas en ambas subredes
  * Rutas internas correctas

---

# 3. Ambiente de desarrollo

### Tecnologías utilizadas

| Componente     | Versión                         |
| -------------- | ------------------------------- |
| Ubuntu Server  | 22.04                           |
| Docker Engine  | 24+                             |
| Docker Compose | v2                              |
| Python         | 3.10                            |
| Flask/Werkzeug | según requirements del profesor |
| MySQL          | 8.0                             |
| NGINX          | 1.18                            |

---

## Cómo compilar / ejecutar

### Base de datos (DB VM)

```bash
cd ~/bookstore-db
docker compose up -d
```

Compose:

```yaml
services:
  mysql:
    image: mysql:8.0
    container_name: bookstore-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: bookstore
      MYSQL_USER: bookstore
      MYSQL_PASSWORD: bookstorepass
    ports:
      - "3307:3306"
    restart: unless-stopped

volumes:
  dbdata:
```

---

### Aplicación (APP VM)

```bash
cd ~/bookstore
docker compose build
docker compose up -d
```

Compose:

```yaml
services:
  app:
    build:
      context: ./app/BookStore-monolith
    container_name: bookstore-app
    environment:
      DB_HOST: "172.31.42.206"
      DB_PORT: "3307"
      DB_NAME: "bookstore"
      DB_USER: "bookstore"
      DB_PASSWORD: "bookstorepass"
    ports:
      - "5000:5000"
    restart: unless-stopped
```

---

## Variables y parámetros clave

* `DB_HOST=172.31.42.206`
* `DB_PORT=3307`
* `DB_NAME=bookstore`
* `DB_USER=bookstore`
* `DB_PASSWORD=bookstorepass`
* NGINX escucha en: `80`
* App internamente escucha en: `5000`

---

## Estructura del proyecto (simplificado)

```
bookstore/
│── compose.yml
│── app/
│     └── BookStore-monolith/
│           ├── Dockerfile
│           ├── app.py
│           ├── requirements.txt
│           ├── templates/
│           └── static/
bookstore-db/
│── compose.yml
└── dbdata/ (volumen)
```

---

# 4. Ambiente de ejecución en producción

### Infraestructura

* **EC2 APP VM**

  * IP Pública: <tu IP>
  * IP Privada: 172.30.x.x
* **EC2 DB VM**

  * IP Privada: 172.31.42.206

### Cómo iniciar servicios

```bash
cd ~/bookstore
docker compose up -d

cd ~/bookstore-db
docker compose up -d
```

### Acceso a la aplicación

Abrir en navegador:

```
http://<IP_PUBLICA_APP>
```

---

# 5. Otra información relevante

* El puerto 3307 se utilizó porque el puerto 3306 estaba ocupado por MySQL local del sistema.
* Se empleó diagnóstico avanzado:

  * `tcpdump`
  * `/dev/tcp`
  * `ss -lntp`
  * revisiones de rutas/NACL/SG
* Todo el ambiente es reproducible con Docker Compose.

---

# 🔗 6. Referencias
*https://github.com/st0263eafit/st0263-252/blob/main/proyecto2/BookStore.zip*

---
