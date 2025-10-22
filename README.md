# Estrategia de Integración: Historia Clínica Distribuida con PostgreSQL + Citus en CentOS

## 📋 Resumen del Proyecto
Sistema de historia clínica distribuida usando PostgreSQL con extensión Citus para fragmentación automática de datos.

**Arquitectura:**
- 1 Coordinador (puerto 5432)
- 2 Workers
- Tablas distribuidas por `documento_id`
- Tabla `profesional_salud` replicada

---

## 🔧 Fase 1: Preparación del Entorno CentOS

### Paso 1: Instalar Docker y Docker Compose

```bash
# Actualizar el sistema
sudo yum update -y

# Instalar dependencias necesarias
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# Agregar repositorio oficial de Docker
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Instalar Docker
sudo yum install -y docker-ce docker-ce-cli containerd.io

# Iniciar y habilitar Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verificar instalación
sudo docker --version

# Agregar tu usuario al grupo docker (opcional, para no usar sudo)
sudo usermod -aG docker $USER
# Nota: Necesitarás cerrar sesión y volver a entrar para que esto tome efecto
```

### Paso 2: Instalar Docker Compose

```bash
# Descargar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Dar permisos de ejecución
sudo chmod +x /usr/local/bin/docker-compose

# Verificar instalación
docker-compose --version
```

### Paso 3: Instalar cliente PostgreSQL (psql)

```bash
# Instalar repositorio de PostgreSQL
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-$(rpm -E %{rhel})-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Instalar cliente PostgreSQL
sudo yum install -y postgresql15

# Verificar instalación
psql --version
```

---

## 📦 Fase 2: Despliegue del Proyecto

### Paso 1: Clonar el Repositorio

```bash
# Ir al directorio home
cd ~

# Clonar el repositorio
git clone https://github.com/jaiderreyes/historia_clinica_distribuida_citus.git

# Entrar al directorio
cd historia_clinica_distribuida_citus
```

### Paso 2: Verificar Archivos

```bash
# Listar archivos para confirmar que están todos
ls -la

# Deberías ver:
# - docker-compose-citus.yml
# - schema_citus.sql
# - insert_datos.sql
# - README_CITUS.md
```

### Paso 3: Configurar Firewall (si está activo)

```bash
# Verificar si firewalld está corriendo
sudo systemctl status firewalld

# Si está activo, abrir puerto 5432
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

### Paso 4: Levantar los Servicios

```bash
# Iniciar los contenedores
docker-compose -f docker-compose-citus.yml up -d

# Verificar que los contenedores estén corriendo
docker ps

# Deberías ver 3 contenedores:
# - citus_coordinator
# - citus_worker_1
# - citus_worker_2
```

### Paso 5: Esperar a que los Servicios Estén Listos

```bash
# Esperar 30 segundos para que PostgreSQL inicie completamente
sleep 30

# Verificar logs del coordinador
docker logs citus_coordinator
```

---

## 🗄️ Fase 3: Configuración de la Base de Datos

### Paso 1: Crear la Extensión Citus

```bash
docker exec -it citus_coordinator psql -U admin -d historia_clinica -c "CREATE EXTENSION IF NOT EXISTS citus;"
```

### Paso 2: Ejecutar el Schema

```bash
# Copiar el archivo al contenedor (si no está montado)
docker cp schema_citus.sql citus_coordinator:/tmp/

# Ejecutar el schema
docker exec -i citus_coordinator psql -U admin -d historia_clinica < schema_citus.sql
```

### Paso 3: Insertar Datos de Prueba

```bash
# Copiar el archivo al contenedor (si no está montado)
docker cp insert_datos.sql citus_coordinator:/tmp/

# Insertar datos
docker exec -i citus_coordinator psql -U admin -d historia_clinica < insert_datos.sql
```

---

## ✅ Fase 4: Validación y Pruebas

### Verificar la Distribución de Datos

```bash
# Conectarse al coordinador
docker exec -it citus_coordinator psql -U admin -d historia_clinica

# Dentro de psql, ejecutar:
```

```sql
-- Ver nodos del cluster
SELECT * FROM citus_get_active_worker_nodes();

-- Consultar usuarios
SELECT * FROM usuario LIMIT 5;

-- Verificar distribución de shards
SELECT * FROM citus_shards;

-- Probar una consulta distribuida
SELECT u.nombre, u.apellido, p.nombre as ciudad
FROM usuario u
JOIN paciente p ON u.documento_id = p.documento_id
WHERE u.documento_id > 0;

-- Salir
\q
```

### Verificar Logs

```bash
# Ver logs del coordinador
docker logs citus_coordinator

# Ver logs de un worker
docker logs citus_worker_1
docker logs citus_worker_2
```

---

## 🛠️ Comandos Útiles

### Gestión de Contenedores

```bash
# Detener todos los servicios
docker-compose -f docker-compose-citus.yml down

# Reiniciar servicios
docker-compose -f docker-compose-citus.yml restart

# Ver estado de contenedores
docker-compose -f docker-compose-citus.yml ps

# Ver logs en tiempo real
docker-compose -f docker-compose-citus.yml logs -f
```

### Acceso Directo a la Base de Datos

```bash
# Desde el host (si psql está instalado)
psql -h localhost -p 5432 -U admin -d historia_clinica

# Desde el contenedor
docker exec -it citus_coordinator psql -U admin -d historia_clinica
```

### Backup y Restore

```bash
# Hacer backup
docker exec -t citus_coordinator pg_dump -U admin historia_clinica > backup.sql

# Restaurar
docker exec -i citus_coordinator psql -U admin -d historia_clinica < backup.sql
```

---

## 🚨 Solución de Problemas Comunes

### Problema: Docker no inicia
```bash
# Verificar status
sudo systemctl status docker

# Reiniciar Docker
sudo systemctl restart docker
```

### Problema: Puerto 5432 ya en uso
```bash
# Verificar qué está usando el puerto
sudo netstat -tulpn | grep 5432

# Detener el servicio PostgreSQL local si existe
sudo systemctl stop postgresql
```

### Problema: Contenedores no se pueden conectar
```bash
# Revisar red de Docker
docker network ls
docker network inspect historia_clinica_distribuida_citus_default

# Recrear la red
docker-compose -f docker-compose-citus.yml down
docker-compose -f docker-compose-citus.yml up -d
```

### Problema: Permisos denegados
```bash
# Asegurarse de tener permisos
sudo chown -R $USER:$USER .

# O ejecutar con sudo
sudo docker-compose -f docker-compose-citus.yml up -d
```

---

## 📊 Próximos Pasos

1. **Crear una aplicación cliente** que se conecte a la base de datos
2. **Implementar endpoints REST** para CRUD de historias clínicas
3. **Configurar backups automáticos** con cron
4. **Monitorear performance** de las consultas distribuidas
5. **Escalar horizontalmente** agregando más workers si es necesario

---

## 📚 Recursos Adicionales

- [Documentación oficial de Citus](https://docs.citusdata.com/)
- [PostgreSQL en CentOS](https://www.postgresql.org/download/linux/redhat/)
- [Docker en CentOS](https://docs.docker.com/engine/install/centos/)

---

**¡Listo!** Tu sistema de historia clínica distribuida debería estar funcionando. Si encuentras algún problema, revisa los logs con `docker logs` o usa la sección de solución de problemas.
