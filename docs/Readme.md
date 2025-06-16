# 📝 Informe del Proyecto Final

## 🔍 1. Introducción

### Descripción del proyecto

Este proyecto consiste en el desarrollo y despliegue de una aplicación web de gestión de productos, estructurada en tres componentes principales: frontend (interfaz de usuario), backend (API RESTful) y base de datos (MySQL). El objetivo es demostrar la separación de responsabilidades, la comunicación entre servicios y el despliegue eficiente usando contenedores Docker, tanto en entornos locales como en la nube.

La aplicación permite realizar operaciones CRUD (crear, leer, actualizar y eliminar) sobre productos, facilitando la administración de inventario de manera sencilla y visual.

### Objetivos del proyecto

- **Desplegar una aplicación web modular** separando frontend, backend y base de datos en contenedores independientes.
- **Implementar comunicación entre servicios** usando redes de Docker y variables de entorno.
- **Facilitar el despliegue local y en la nube**, asegurando portabilidad y escalabilidad.
- **Aplicar buenas prácticas de infraestructura como código** mediante el uso de `docker-compose` y archivos de configuración.
- **Demostrar integración y funcionamiento completo** de la solución, incluyendo pruebas de conectividad y operación CRUD desde la interfaz web.

## 💻 2. Solución Local

### Arquitectura local con Docker Compose

La arquitectura local de la aplicación está compuesta por tres servicios principales, gestionados y orquestados mediante Docker Compose:

1. **Base de datos (MySQL):**

   - Utiliza la imagen oficial de MySQL 5.7.
   - Expone el puerto 3306 para conexiones locales.
   - Se inicializa con un usuario, contraseña y base de datos definidos mediante variables de entorno.
   - Monta un volumen persistente para almacenar los datos y un archivo de inicialización SQL.

   ```
    .
    ├── database
    │   ├── init.sql
    │   └── Readme.md
   ```

2. **Backend (API RESTful):**

   - Construido a partir del código fuente ubicado en la carpeta `./backend`.
   - Expone el puerto 5000 para recibir solicitudes HTTP.
   - Se conecta a la base de datos mediante variables de entorno que definen el host, usuario, contraseña y nombre de la base de datos.
   - Depende de que el servicio de base de datos esté saludable antes de iniciar.

   ```
    .
    ├── backend
    │   ├── config
    │   │   └── db.js
    │   ├── controllers
    │   │   └── productos.controller.js
    │   ├── routes
    │   │   └── productos.route.js
    │   ├── Dockerfile
    │   ├── package.json
    │   ├── Readme.md
    │   └── server.js
   ```

3. **Frontend (Interfaz de usuario):**
   - Construido a partir del código fuente ubicado en la carpeta `./frontend`.
   - Expone el puerto 3000 (mapeado al 80 del contenedor) para acceso web.
   - Depende del backend para funcionar correctamente.
   ```
   .
   ├── frontend
   │   ├── public
   │   │   └── index.html
   │   ├── src
   │   │   ├── api
   │   │   ├── components
   │   │   ├── App.jsx
   │   │   └── index.js
   │   ├── Dockerfile
   │   ├── nginx.conf
   │   ├── package.json
   │   └── Readme.md
   ```

# ☁️ 3. Solución en la Nube

Esta sección describe la implementación de la arquitectura desplegada en la nube, replicando la arquitectura local pero utilizando servicios gestionados de Azure. Se utilizó AKS (Azure Kubernetes Service) para orquestar los contenedores, Azure Database for MySQL para la base de datos y un LoadBalancer para exponer la API con alta disponibilidad.

---

## 🔑 1. Iniciar sesión en Azure

```bash
az login
```

Se inició sesión en una cuenta de Azure for Students para acceder a los recursos necesarios.

---

## 🧱 2. Crear un grupo de recursos

```bash
az group create --name rg-infra-final --location eastus
```

Este grupo actúa como un contenedor lógico para los recursos asociados al proyecto.

---

## 📦 3. Crear un Azure Container Registry (ACR)

```bash
az acr create --name acrinfrafinal --resource-group rg-infra-final --sku Basic
```

Este registro permite almacenar y administrar imágenes Docker de manera privada en Azure.

---

## ☸️ 4. Crear un clúster AKS

```bash
az aks create \
 --resource-group rg-infra-final \
 --name aks-infra-final \
 --node-count 1 \
 --node-vm-size Standard_B2ms \
 --enable-managed-identity \
 --enable-addons monitoring \
 --generate-ssh-keys \
 --attach-acr acrinfrafinal
```

Se crea un clúster Kubernetes con un nodo, monitoreo habilitado y conectado al ACR.

### Obtener credenciales para kubectl:

```bash
az aks get-credentials \
 --resource-group rg-infra-final \
 --name aks-infra-final \
 --overwrite-existing
```

### Verificar los nodos:

```bash
kubectl get nodes
```

Debe aparecer el nodo con estado **Ready**.

---

## ✅ 5. Login en ACR desde Docker

```bash
az acr login --name acrinfrafinal
```

---

## 💪 6. Construir y subir las imágenes

Desde la raíz del proyecto:

### Backend:

```bash
docker build -t acrinfrafinal.azurecr.io/backend:v1 ./backend
docker push acrinfrafinal.azurecr.io/backend:v1
```

### Frontend:

```bash
docker build -t acrinfrafinal.azurecr.io/frontend:v2 ./frontend
docker push acrinfrafinal.azurecr.io/frontend:v2
```

### Base de datos:

```bash
docker build -t acrinfrafinal.azurecr.io/mysql-custom:v1 ./database
docker push acrinfrafinal.azurecr.io/mysql-custom:v1
```

### Verificar imágenes:

```bash
docker images
```

---

## 📂 7. Estructura de manifiestos YAML

Ubicar los siguientes archivos en la carpeta `k8s/`:

```
k8s/
├── backend-deploy.yaml
├── backend-svc.yaml
├── frontend-deploy.yaml
├── frontend-svc.yaml
├── mysql-deploy.yaml
└── mysql-svc.yaml
```

---

## ✏️ 8. Contenido de los manifiestos YAML

### `backend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      imagePullSecrets:
        - name: acr-auth
      containers:
        - name: backend
          image: acrinfrafinal.azurecr.io/backend:v1
          ports:
            - containerPort: 5000
          env:
            - name: MYSQL_HOST
              value: mysql-final.mysql.database.azure.com
            - name: MYSQL_USER
              value: crud_admin@mysql-final
            - name: MYSQL_PASSWORD
              value: <TU_PASSWORD>
            - name: MYSQL_DATABASE
              value: crud_db
```

### `backend-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

### `frontend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      imagePullSecrets:
        - name: acr-auth
      containers:
        - name: frontend
          image: acrinfrafinal.azurecr.io/frontend:v2
          ports:
            - containerPort: 80
```

### `frontend-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### `mysql-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      imagePullSecrets:
        - name: acr-auth
      containers:
        - name: mysql
          image: acrinfrafinal.azurecr.io/mysql-custom:v1
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: rootpassword
            - name: MYSQL_DATABASE
              value: crud_db
            - name: MYSQL_USER
              value: crud_user
            - name: MYSQL_PASSWORD
              value: crud_password
          ports:
            - containerPort: 3306
```

### `mysql-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - port: 3306
      targetPort: 3306
```

---

## 🔐 9. Crear el secreto de autenticación ACR

```bash
kubectl create secret docker-registry acr-auth \
  --docker-server=acrinfrafinal.azurecr.io \
  --docker-username=aks-token \
  --docker-password=<TOKEN_PASSWORD> \
  --docker-email=<correo de azure>
```

**Nota:** Esto es opcional sino agregaste el parámetro `--attach-acr` al crear el cluster o si la creaciòn del token no se hizo automatica, en cuyo caso debemos crear el token manualmente y ejecutar el comando anterior.

---

## ⚘️ 10. Aplicar manifiestos

```bash
kubectl apply -f k8s/
```

Eliminar los pods antiguos para que usen las imágenes actualizadas:

```bash
kubectl delete pod -l app=backend
kubectl delete pod -l app=frontend
kubectl delete pod -l app=db
```

---

## 📊 11. Verificación final

```bash
kubectl get pods
kubectl get svc
```

---

Este procedimiento completa el despliegue de la solución en la nube utilizando los servicios gestionados de Azure.

## 📊 4. Análisis de calidad de código

## 🗒️ 5. Análisis y Conclusiones

```

```
