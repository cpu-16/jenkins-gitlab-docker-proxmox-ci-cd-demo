# 🔄 Guía CI/CD Local: Jenkins + GitLab + Docker + Proxmox

<div align="center">

![Jenkins](https://img.shields.io/badge/Jenkins-CI/CD-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![GitLab](https://img.shields.io/badge/GitLab-Repository-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Container-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-VE-E57000?style=for-the-badge&logo=proxmox&logoColor=white)

**Pipeline CI/CD completo desde Windows hacia VMs Linux en Proxmox** 🚀

</div>

---

## 📋 Tabla de Contenidos

- [Demo CI/CD](#-demo-cicd)
- [Decisiones de Arquitectura](#-decisiones-de-arquitectura)
- [Topología del Laboratorio](#-topología-que-estamos-montando)
- [Paso 1 - Preparar las VMs](#-paso-1--preparar-las-vms-en-proxmox)
- [Paso 2 - GitLab y Repositorio](#-paso-2--crear-cuenta-gitlab-y-preparar-el-repo)
- [Paso 3 - Instalar Jenkins](#-paso-3--instalar-jenkins-en-jenkins-vm)
- [Paso 4 - Instalar Docker y Git](#-paso-4--instalar-docker-y-git-en-ambas-vms)
- [Paso 5 - Configurar Nodo Jenkins](#-paso-5--preparar-app-vm-como-nodo-jenkins)
- [Paso 6 - Ajustar Monitores](#-paso-6--ajustar-monitores-de-nodo-en-jenkins)
- [Paso 7 - VS Code + Git](#-paso-7--vs-code--git-en-tu-windows)
- [Paso 8 - Token GitLab](#-paso-8--token-de-acceso-en-gitlab-para-jenkins)
- [Paso 9 - Credenciales Jenkins](#-paso-9--credenciales-en-jenkins)
- [Paso 10 - Jenkinsfile](#-paso-10--ajustar-el-jenkinsfile)
- [Paso 11 - App NodeJS](#-paso-11--app-nodejs-con-ui-devops)
- [Paso 12 - Crear Pipeline](#-paso-12--crear-el-pipeline-en-jenkins-desde-scm)
- [Paso 13 - Webhooks](#-paso-13--acceder-a-la-app-y-automatizar-con-webhooks)
- [Resumen Final](#-resumen-de-lo-que-tienes-montado)

---

## 🎥 Demo CI/CD

![Demo CI/CD](images/demo2.gif)

En este video se muestra:

1. `git commit` → `git push` al repositorio GitLab.
2. Jenkins dispara el pipeline automáticamente.
3. Se construye la imagen Docker y se publica en Docker Hub.
4. La app Node.js se redepliega en `http://APP_VM:3000`.

![Diagrama de topología](docs/images/01-topologia.png)

---

## 🏗 Decisiones de Arquitectura

### ¿Dónde va Jenkins?

Jenkins se ejecuta en **Linux dentro de Proxmox**, no en Windows.

**VM `jenkins-vm`** (Debian/Ubuntu) con:
- Jenkins (controller)
- Git
- Docker (para build y push a Docker Hub)

> 💡 Jenkins requiere Java 17 o 21 y se distribuye con paquetes para Debian/Ubuntu.

### ¿Dónde corre la app?

**VM `app-vm`** (Debian/Ubuntu) con:
- Docker (para correr el contenedor de la app)
- Jenkins Agent (conexión por SSH desde Jenkins)

### ¿Necesito VS Code?

No es obligatorio, pero ayuda para:
- Editar el código del repo GitLab
- Hacer `git add` / `commit` / `push`
- Revisar historial y cambios

### Cuentas necesarias

- **GitLab** → repo con código, Dockerfile y Jenkinsfile
- **Docker Hub** → registro donde Jenkins publicará la imagen

![Pantallas de login GitLab y Docker Hub](docs/images/02-cuentas.png)

---

## 🗺 Topología que estamos montando

### Desde tu laptop Windows

- **Navegador**: acceder a Jenkins (`http://IP_JENKINS:8080`) y a la app (`http://IP_APP:3000`)
- **VS Code + Git**: editar y hacer pushes al repo de GitLab

### En Proxmox

**VM 1 – jenkins-vm**
- Jenkins (controller)
- Docker
- Git

**VM 2 – app-vm**
- Docker
- Jenkins Agent (SSH)

### Servicios externos

**GitLab.com**
- Repo `Gitlab_Jenkins_collab` (tu fork)

**Docker Hub**
- Repositorio `TU_USUARIO/cicd-demo-app`

![Panel de Proxmox con las VMs](docs/images/03-proxmox-vms.png)

---

## 🔧 Paso 1 – Preparar las VMs en Proxmox

### 1.1. Crear las VMs

**VM Jenkins (jenkins-vm)**
- 2–4 vCPU
- 4 GB RAM
- 15–50 GB disco
- IP fija, por ejemplo: `172.25.205.50`

**VM App (app-vm)**
- 2 vCPU
- 2–4 GB RAM
- 20–30 GB disco
- IP fija, por ejemplo: `172.25.205.51`

Comprueba conectividad:

```bash
ping 172.25.205.50
ping 172.25.205.51
```

### 1.2. IP estática y conflicto por clonación de VM

Como clonaste VMs, ambas podían arrancar con la misma MAC/IP y el DHCP les reasignaba direcciones (51, 53, etc.).

Lo que hiciste para estabilizarlo fue:

1. Cambiar la MAC de la interfaz (`ens18`) en la VM clonada
2. Configurar IP estática usando `systemd-networkd` / `netplan` (o quedarte con DHCP pero con MAC diferente)

> 💡 Netplan permite definir IP estática en archivos YAML tipo `/etc/netplan/90-default.yaml`, usando `renderer: networkd`.

![Salida de ip a mostrando IP fija](docs/images/04-ip-fija.png)

---

## 🦊 Paso 2 – Crear cuenta GitLab y preparar el repo

1. Entra a https://gitlab.com y crea tu cuenta
2. Abre el repo del instructor:
   ```
   https://gitlab.com/dgruploads/Gitlab_Jenkins_collab
   ```
3. Haz **Fork** a tu espacio
4. Verifica que tu repo tenga:
   - `Dockerfile`
   - `Jenkinsfile`
   - `app.js`
   - `package.json`

> A partir de aquí Jenkins usará tu fork, no el repo original.

![Repositorio en GitLab con archivos](docs/images/05-gitlab-repo.png)

---

## ☕ Paso 3 – Instalar Jenkins en jenkins-vm

### 3.1. Instalar Java (requisito)

En Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

> Las versiones recientes de Jenkins soportan Java 17 y 21 en Debian/Ubuntu.

![Salida de java -version](docs/images/06-java-version.png)

### 3.2. Instalar Jenkins desde el repo oficial

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
```

Arrancar y habilitar:

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Por defecto Jenkins escucha en el puerto **8080**.

En tu navegador:

```
http://172.25.205.50:8080
```

![Pantalla de desbloqueo de Jenkins](docs/images/07-jenkins-unlock.png)

### 3.3. Desbloquear Jenkins

En la VM:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- Copias el token → pantalla "Unlock Jenkins"
- Eliges **Install suggested plugins**
- Creas tu usuario admin (ej. `muriel-admin`)

### 3.4. Ajustar espacio en disco

Te quedaste sin espacio porque el disco de la VM era muy pequeño (~3 GB).

En Debian con partición simple ext4 hiciste:

1. **Ver discos:**

```bash
lsblk -o NAME,FSTYPE
```

2. **Ampliar la partición con growpart:**

```bash
sudo growpart /dev/sda 1
```

3. **Redimensionar el sistema de archivos:**

```bash
sudo resize2fs /dev/sda1
df -h
```

Con esto `/` pasó a ~15 GB (o el tamaño que configuraste en Proxmox).

![Salida de df -h con más espacio](docs/images/08-df-espacio.png)

### 3.5. Crear swap para que Jenkins no se queje

Tu VM no tenía swap (`Swap: 0B`).

Creaste un archivo de swap (1–2 GB es suficiente):

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Agregar a `/etc/fstab`:

```
/swapfile none swap sw 0 0
```

Comprobar:

```bash
free -h
```

![Salida de free -h con swap](docs/images/09-swap.png)

---

## 🐳 Paso 4 – Instalar Docker y Git en ambas VMs

### 4.1. Docker Engine desde el repo oficial

En cada VM (Jenkins y app):

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

> 💡 Estos pasos siguen la guía oficial de Docker Engine para Ubuntu, que usa el repositorio oficial y GPG key de Docker.

Probar:

```bash
sudo docker run hello-world
```

![Salida de docker run hello-world](docs/images/10-docker-hello.png)

### 4.2. Usar Docker sin sudo

Para no tener que usar `sudo` siempre:

```bash
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins   # si existe el usuario jenkins
```

Cerrar sesión y volver a entrar.

> Docker recomienda añadir el usuario al grupo `docker` si se desea ejecutarlo sin sudo, según muchas guías de administración.

### 4.3. Instalar Git

```bash
sudo apt install -y git
git --version
```

---

## 🔗 Paso 5 – Preparar app-vm como nodo Jenkins

### 5.1. Requisitos en app-vm

- Docker instalado y funcionando
- Git instalado
- Java para el agente Jenkins (si lo necesita):

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
```

### 5.2. Acceso SSH desde jenkins-vm

En `jenkins-vm`:

```bash
ssh-keygen -t ed25519 -C "jenkins-proxmox"
# Dejar ruta por defecto (~/.ssh/id_ed25519)

ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@172.25.205.51
ssh usuario@172.25.205.51
```

Debe entrar sin pedir contraseña.

### 5.3. Crear nodo en Jenkins

En Jenkins:

**Manage Jenkins → Nodes → New Node**

**Configuración:**

- **Nombre:** `app-server`
- **Tipo:** Permanent Agent
- **Ejecutores:** 2
- **Remote root directory:** `/home/usuario/jenkins-agent`
- **Labels:** `application` (o el label que usarás en el `agent { label '...' }` del Jenkinsfile)
- **Launch method:** Launch agents via SSH
  - **Host:** `172.25.205.51`
  - **Credentials:**
    - Kind: SSH Username with private key
    - User: usuario de app-vm
    - Private Key: "Enter directly" → pegas `~/.ssh/id_ed25519` de jenkins-vm

Guarda y revisa el log hasta ver el nodo conectado (icono con monitor y ✔️).

![Página de Nodes con master y app-server](docs/images/11-jenkins-nodes.png)

---

## 📊 Paso 6 – Ajustar monitores de nodo en Jenkins

Jenkins monitoriza cada nodo: disco, temp, swap, reloj, respuesta, etc. Si algo baja del umbral, lo marca en rojo o lo pone offline.

Para que no se queje por detalles menores:

Ve a: **Manage Jenkins → Nodes → Configure Monitors** (`/computer/configure`)

En cada monitor:

**Disk space:**
- Marca "Don't mark agents temporarily offline"
- Umbral: por ejemplo 512MiB o 1GiB

**Temporary space:**
- Igual: bajar umbral a 512MiB o 1GiB

**Swap space:**
- Ajustar umbral a algo coherente con tu swap (por ejemplo, 256–512 MiB)

Guarda cambios.

![Configuración de Node Monitors](docs/images/12-node-monitors.png)

---

## 💻 Paso 7 – VS Code + Git en tu Windows

1. Instala **Git for Windows**
2. Instala **VS Code**
3. Clona tu repo de GitLab:

```bash
git clone https://gitlab.com/TU_USUARIO/Gitlab_Jenkins_collab.git
cd Gitlab_Jenkins_collab
code .
```

Desde aquí podrás editar `app.js`, `Jenkinsfile`, etc., hacer commit y push.

![VS Code con el proyecto abierto](docs/images/13-vscode.png)

---

## 🔑 Paso 8 – Token de acceso en GitLab para Jenkins

En GitLab:

1. Avatar arriba derecha → **Preferences / Edit profile**
2. Menú izquierdo → **Access Tokens**
3. Crea un token:
   - **Name:** `jenkins-gitlab-collab`
   - **Expiration:** una fecha razonable
   - **Scopes:** al menos `read_repository` (y `write_repository` si quieres permitir pushes)

> 💡 Los Personal Access Tokens permiten acceso al repositorio vía HTTPS con scopes como `read_repository`.

![Creación de Personal Access Token](docs/images/14-gitlab-token.png)

---

## 🔐 Paso 9 – Credenciales en Jenkins

En Jenkins:

**Manage Jenkins → Credentials → System → Global credentials**

### 9.1. Credencial GitLab

- **Kind:** Username with password
- **Username:** tu usuario de GitLab
- **Password:** el Personal Access Token
- **ID:** `gitlab-tokens` (este ID se usa en el Jenkinsfile)

### 9.2. Credencial Docker Hub

- **Kind:** Username with password
- **Username:** tu usuario de Docker Hub (ej. `gar16`)
- **Password:** tu contraseña o Access Token
- **ID:** `dockerhub-credentials`

![Tabla de credenciales en Jenkins](docs/images/15-jenkins-credentials.png)

---

## 📜 Paso 10 – Ajustar el Jenkinsfile

En lugar de pegar el Jenkinsfile completo aquí (ya está en tu repo), solo documentamos los puntos clave:

### Agent / label

```groovy
agent { label 'application' }
```

Para que el pipeline se ejecute en el nodo `app-server`.

### Imagen Docker

```groovy
DOCKER_IMAGE = "cicd-demo-app:${env.BUILD_NUMBER}"
```

### Checkout del repo

Usa tu fork de GitLab, con:

```groovy
branch: 'main'
credentialsId: 'gitlab-tokens'
url: 'https://gitlab.com/cpu-161/Gitlab_Jenkins_collab.git'
```

### Login a Docker Hub

Usa `withCredentials` con `dockerhub-credentials`.

### Tag & push

La imagen se etiqueta como `TU_USUARIO/cicd-demo-app:${BUILD_NUMBER}`.

```bash
docker image push TU_USUARIO/cicd-demo-app:${BUILD_NUMBER}
```

### Deploy

Se elimina el contenedor viejo:

```bash
docker rm -f cicd-demo-container || true
```

Se levanta uno nuevo:

```bash
docker run -d -p 3000:3000 --name cicd-demo-container cicd-demo-app:${BUILD_NUMBER}
```

### Jenkinsfile

El Jenkinsfile completo se encuentra en el repositorio de GitLab:

- https://gitlab.com/cpu-161/Gitlab_Jenkins_collab/-/blob/main/Jenkinsfile

En él se definen los stages:
1. Checkout del código
2. Build de la imagen Docker
3. Login y push a Docker Hub
4. Parada del contenedor anterior
5. Puesta en marcha del nuevo contenedor

![Consola de Jenkins con build exitoso](docs/images/16-jenkins-build.png)

---

## 🎨 Paso 11 – App NodeJS con UI DevOps

Tu archivo `app.js`:

- Usa Express en el puerto 3000
- Expone `/` con una página HTML con diseño tipo panel DevOps:
  - Fondo oscuro tipo dashboard
  - Tarjeta central con:
    - Badge: "Pipeline Activo · Jenkins + GitLab + Docker"
    - Título "CI/CD DevOps Demo"
    - Texto explicando que cada cambio en GitLab dispara el pipeline
  - Cuatro pasos visuales:
    1. Commit & Push (GitLab)
    2. Jenkins (build)
    3. Docker Hub (push)
    4. Deploy (app-server)
  - Se muestra la versión desplegada `v${appVersion}` y el entorno `Proxmox · app-server`

### App Node.js – UI DevOps

El archivo `app.js` define una pequeña API con Express que devuelve una interfaz gráfica que representa el pipeline CI/CD (Commit → Jenkins → Docker Hub → Deploy).

**Código completo:**
- https://gitlab.com/cpu-161/Gitlab_Jenkins_collab/-/blob/main/app.js

![UI DevOps en el navegador](docs/images/17-app-ui.png)

---

## ⚙️ Paso 12 – Crear el Pipeline en Jenkins desde SCM

En Jenkins:

1. **New Item → Pipeline**
2. **Nombre:** `pipeline-demo`
3. En la configuración:
   - **Definition:** Pipeline script from SCM
   - **SCM:** Git
   - **Repository URL:**
     ```
     https://gitlab.com/cpu-161/Gitlab_Jenkins_collab.git
     ```
   - **Credentials:** `gitlab-tokens`
   - **Branches to build:** `*/main`
   - **Script Path:** `Jenkinsfile`

4. Guarda y ejecuta **Build Now**

Deberías ver en la consola:

- Checkout desde GitLab
- Build de imagen Docker
- Login a Docker Hub
- Push de la imagen
- Eliminación del contenedor viejo
- Arranque del nuevo contenedor

![Historial de builds en Jenkins](docs/images/18-jenkins-history.png)

---

## 🔔 Paso 13 – Acceder a la app y automatizar con Webhooks

### 13.1. Probar la app

En tu navegador:

```
http://172.25.205.51:3000
```

Deberías ver la UI DevOps con la versión actual desplegada.

### 13.2. Webhook GitLab → Jenkins

**En el job de Jenkins:**

1. **Configure → Build Triggers → Trigger builds remotely**
2. Define un token, por ejemplo: `gitlab-trigger-token`

**En GitLab:**

1. **Settings → Webhooks**
2. **URL:**

```
http://TU_USUARIO_JENKINS:TU_API_TOKEN@172.25.205.50:8080/job/pipeline-demo/build?token=gitlab-trigger-token
```

3. Marca **Push events**
4. Guarda y prueba con **Test → Push events**

Si todo está bien:

- El test de Webhook devuelve **200 OK**
- Jenkins dispara un nuevo build automáticamente cada vez que haces `git push`

🔄 Alternativa con Poll SCM (cron en Jenkins)

Como en este laboratorio nuestro Jenkins está dentro de la red local de Proxmox (IP privada 172.25.205.50) y no tenemos IP pública ni túnel HTTPS hacia Internet, GitLab no puede enviarle webhooks directamente. Para no perder la automatización, configuramos en el job de Jenkins la opción “Consultar repositorio (SCM)” con una expresión cron (por ejemplo H/5 * * * * o H/10 * * * *).
Con esto, Jenkins pregunta periódicamente a GitLab si hay cambios en la rama main y, solo cuando detecta nuevos commits, dispara el pipeline (build de la imagen Docker, push a Docker Hub y redeploy en app-server). Esta solución introduce un pequeño retraso (hasta 5–10 minutos según el intervalo configurado) y genera algo de tráfico extra, pero es una buena alternativa temporal cuando aún no se dispone de una URL pública/túnel para usar webhooks GitLab → Jenkins.

![Webhook en GitLab con delivery 200 OK](docs/images/19-gitlab-webhook.png)

---

## ✅ Resumen de lo que tienes montado

- ✅ **Infraestructura local** en Proxmox con dos VMs Linux (`jenkins-vm` y `app-vm`)

- ✅ **Jenkins bien instalado y dimensionado:**
  - Java correcto (21)
  - Más espacio en `/`
  - Swap creada
  - Monitores ajustados para no molestar por detalles

- ✅ **Networking estable:**
  - IPs fijas y sin conflictos al clonar VMs

- ✅ **Pipeline completo:**
  - GitLab → Jenkins → Docker → Docker Hub → app-vm

- ✅ **App NodeJS** con UI pensada para explicar el flujo DevOps

- ✅ **Disparos automáticos** vía Webhook de GitLab

---

## 🤝 Contribuir

¿Mejoras o sugerencias? ¡Pull requests bienvenidos!

1. Fork el proyecto
2. Crea tu rama: `git checkout -b feature/nueva-funcionalidad`
3. Commit: `git commit -m 'Añade nueva funcionalidad'`
4. Push: `git push origin feature/nueva-funcionalidad`
5. Abre un Pull Request

---

## 📄 Licencia

Este proyecto es libre de usar para propósitos educativos y de laboratorio.

---

## 🙏 Agradecimientos

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [GitLab Documentation](https://docs.gitlab.com/)
- [Docker Documentation](https://docs.docker.com/)
- [Proxmox VE](https://www.proxmox.com/)

---

<div align="center">

**⭐ Pipeline CI/CD completo desde cero: Jenkins + GitLab + Docker en Proxmox! ⭐**

Hecho con ❤️ para aprender DevOps y automatización

</div>
