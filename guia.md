# Guía Definitiva: CI/CD con GitHub Actions, Docker y Kubernetes

Esta guía contiene los pasos exactos para dockerizar una aplicación (Spring Boot), desplegarla manualmente en un clúster local de Kubernetes y automatizar todo el proceso mediante GitHub Actions utilizando un *Self-Hosted Runner*.

---

## Fase 1: Preparación del Entorno y Credenciales

Antes de escribir código, necesitamos preparar las "llaves" de los servicios.

1. **Subir el código base:** Asegúrate de que tu proyecto (ej. Spring Boot con el `pom.xml` y la carpeta `src`) está en la rama `main` de tu repositorio de GitHub.
2. **Generar Token de Docker Hub:**
   * Entra en tu cuenta de Docker Hub.
   * Ve a **Account settings > Security > New Access Token**.
   * Crea el token, cópialo y guárdalo temporalmente (no lo pierdas).

---

## Fase 2: Dockerización y Primera Subida Manual

Kubernetes necesitará descargar la imagen desde internet en la Fase 3, por lo que primero debemos compilarla y subirla a mano.

### 1. Crear el `Dockerfile`
En la raíz de tu proyecto, crea un archivo llamado exactamente `Dockerfile` y pega esto (adapta las versiones si es necesario):

```dockerfile
# Etapa 1: Compilar la aplicación con Maven
FROM maven:3.9.6-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Etapa 2: Crear la imagen final superligera
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Para subirlo a Docker Hub
docker login
docker build -t chewi9/springuma:latest .
docker push chewi9/springuma:latest


(Kubernetes)
Dentro de carpeta k8s
Deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: ips
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
      - name: backend-app
        image: chewi9/springuma:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080

Service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-app-service
  namespace: ips
spec:
  type: NodePort
  selector:
    app: mi-app
  ports:
    - protocol: TCP
      port: 8080
      nodePort: 30007

kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

con kubectl get pods -n ips se comprueba si todo va bien


GitHub Actions necesita permisos para subir la imagen a Docker Hub de forma automática.
Ve a tu repositorio en GitHub > Settings > Secrets and variables > Actions.

Haz clic en New repository secret y crea dos secretos exactos:
DOCKERHUB_USERNAME: tu nombre de usuario de Docker (ej. chewi9).
DOCKERHUB_TOKEN: el token que creaste en la Fase 1.

Se crea carpeta .github/workflows

ci-cd.yml
name: CI/CD Hospital

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout del código
        uses: actions/checkout@v4

      - name: Login a Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Construir y Subir
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: chewi9/springuma:latest

  deploy:
    needs: build
    runs-on: self-hosted
    steps:
      - name: Actualizar Kubernetes local
        run: kubectl rollout restart deployment backend-app -n ips


Para activar todo el workflow
git add .
git commit -m "feat: config workflow automatizado"
git push origin main




ERRORES
- Username and password required	GitHub no encuentra los secretos. 
Comprueba si los guardaste en Variables en lugar de Secrets, o si hay una errata en el nombre (DOCKERHUB_USERNAME).

- El circulito amarillo no aparece tras hacer push	
1. Comprueba si la carpeta .github/workflows existe.
2. Comprueba si en tu código pone branches: ["main"] pero tu repo en GitHub usa la rama master.

- Cuándo usar ubuntu-latest vs self-hosted	
Usa ubuntu-latest para tareas en la nube (compilar código, subir a DockerHub). Usa self-hosted cuando necesites tocar algo de tu propio ordenador (como un clúster local de Kubernetes).

- Error al ejecutar config.cmd del Runner	
Cierra la terminal y ábrela haciendo clic derecho -> Ejecutar como Administrador.

- Tokens cruzados	
Recuerda: GitHub Runner usa un token temporal automático que te da la web al instalarlo. El archivo .yml necesita el token creado en Docker Hub. No los mezcles.
