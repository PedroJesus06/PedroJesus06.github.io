# Dockerización de una Aplicación Web y CI/CD con GitHub Actions

El objetivo principal de este proyecto ha sido empaquetar una aplicación web estática (el conocido juego 2048) dentro de un contenedor **Docker** y automatizar todo su ciclo de vida. Para ello, hemos implementado un flujo de **Integración Continua (CI)** utilizando **GitHub Actions**, logrando que cada cambio en el código construya y publique automáticamente una nueva versión de la imagen en **Docker Hub**.

## Herramientas y Tecnologías Utilizadas

En esta práctica dejamos atrás los despliegues manuales para centrarnos en la automatización y la portabilidad. Estas son las piezas clave de nuestra arquitectura:

* **Contenedores y Servidor Web:**
    * ![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
    * ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
    * ![Nginx](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)

* **Control de Versiones y Automatización:**
    * ![Git](https://img.shields.io/badge/git-%23F05033.svg?style=for-the-badge&logo=git&logoColor=white)
    * ![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)

## La Receta del Contenedor: El `Dockerfile`

En lugar de instalar servicios a mano en una máquina, definimos la infraestructura como código. Partimos de una imagen base limpia, instalamos nuestro servidor web y clonamos el código de la aplicación.

Este es el código que construye nuestra imagen:

```dockerfile
# Capa base 
FROM ubuntu:24.04

# Actualizamos los paquetes e instalamos el servidor web
RUN apt-get update && apt-get upgrade -y     && apt-get install nginx -y     && apt-get clean

# Configuramos el directorio de trabajo donde Nginx sirve los archivos
WORKDIR /var/www/html

# Clonamos el repositorio de la aplicación y limpiamos para reducir el tamaño de la imagen
RUN apt-get install git -y     && git clone https://github.com/josejuansanchez/2048     && mv 2048/* .     && rm -rf 2048     && apt-get remove git -y     && apt-get clean

# Definimos el entrypoint para que Nginx se ejecute en primer plano
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

### ¿Qué hace cada bloque exactamente?

* **Base Sólida (`FROM ubuntu:24.04`):** Usamos la última versión LTS de Ubuntu para garantizar estabilidad y parches de seguridad recientes.
* **Servidor Web (`nginx`):** Instalamos Nginx directamente desde los repositorios oficiales.
* **Optimización del Espacio:** Encadenamos comandos con `&&` y ejecutamos `apt-get clean` y `apt-get remove git` después de clonar el código. Esto es vital para no inflar el tamaño de la imagen final con herramientas que solo necesitamos durante la construcción.
* **Proceso Principal (`ENTRYPOINT`):** Obligamos a Nginx a correr en *foreground* (`daemon off;`). Si no hacemos esto, el contenedor arrancaría e inmediatamente se detendría al pensar que no tiene ningún proceso activo.

## Automatización Total: El `main.yml` de GitHub Actions

Para no tener que construir y subir la imagen a mano desde nuestra terminal local, delegamos el trabajo en **GitHub Actions**. Hemos configurado un **workflow** que salta automáticamente al hacer push en la rama `main`.

```yaml
name: Publish image to Docker Hub

on:
  push:
    branches: [ "main" ]
    tags: [ 'v*.*.*' ]
  workflow_dispatch:

env:
  REGISTRY: docker.io
  IMAGE_NAME: 2048
  IMAGE_TAG: latest

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@f95db51fddba0c2d1ec667646a06c2ce06100226 # v3.0.0

      - name: Log into registry ${{ env.REGISTRY }}
        uses: docker/login-action@343f7c4344506bcbf9b4de18042ae17996df046d # v3.0.0
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Debug
        run: |
          echo "github.repository: ${{ github.repository }}"
          echo "env.REGISTRY: ${{ env.REGISTRY }}"
          echo "github.sha: ${{ github.sha }}"
          echo "env.IMAGE_NAME: ${{ env.IMAGE_NAME }}"

      - name: Build and push Docker image
        id: build-and-push
        uses: docker/build-push-action@0565240e2d4ab88bba5387d719585280857ece09 # v5.0.0
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ env.REGISTRY }}/${{ secrets.DOCKERHUB_USERNAME }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Puntos clave del despliegue automático:

* **Secretos de Repositorio:** El *workflow* se conecta a Docker Hub usando las variables `${{ secrets.DOCKERHUB_USERNAME }}` y `${{ secrets.DOCKERHUB_TOKEN }}`. De esta forma, las credenciales nunca quedan expuestas en el código fuente.
* **Construcción Eficiente:** Usamos `setup-buildx-action` y activamos el sistema de **caché de GitHub Actions** (`cache-to` y `cache-from`). Esto hace que las reconstrucciones sean muchísimo más rápidas al aprovechar capas previas.
* **Etiquetado Dinámico:** La imagen resultante se empaqueta y se publica en nuestro registro público de Docker Hub etiquetada como `latest`.

## Resumen de la Implementación y Validación

Verificamos que todos los engranajes de nuestro sistema CI/CD funcionan correctamente, validando paso a paso el ciclo de vida del contenedor.

### 1. Construcción de la Imagen en Local

El primer paso es comprobar que el `Dockerfile` está bien definido construyendo la imagen en nuestra máquina.

> *Lanzando la construcción (docker build)*

Ejecutamos la construcción y vemos cómo Docker descarga la imagen base de Ubuntu y ejecuta los pasos definidos.

![](images/docker%205.4%20cap%201.png)

### 2. Preparación y Etiquetado de la Imagen (Docker Tag)

Una vez construida la imagen en local, el siguiente paso lógico antes de enviarla a la nube es prepararla adecuadamente. Para ello, utilizamos el comando **`docker tag`**, que nos permite asignar un alias y enlazar nuestra imagen local con la nomenclatura exacta que exige **Docker Hub**.

> *Etiquetado de la imagen local para el registro*

Ejecutamos el comando especificando el nombre de nuestra imagen original (`2048`) y asignándole la ruta completa de destino (`usuario/repositorio:etiqueta`).

![](images/docker%205.4%20cap%202.png)

### 3. Autenticación y Publicación en Docker Hub

Una vez etiquetada la imagen, el siguiente movimiento es subirla al registro público. Este entorno requiere validar nuestra identidad antes de aceptar cualquier modificación en los repositorios. 

Todo el proceso ocurre en dos fases que ejecutamos de forma consecutiva en la misma terminal:

* **Autenticación (`docker login`):** Iniciamos sesión directamente contra el servidor. Introducimos las credenciales y, al recibir el mensaje de validación exitosa (**Login Succeeded**), el demonio guarda el acceso de forma segura. Es el paso de seguridad obligatorio para que no nos rechacen la conexión.
* **Subida de la imagen (`docker push`):** Lanzamos la subida apuntando a nuestro repositorio exacto (`pedrojesus06/2048:1:0`). 

> *Proceso completo de validación y subida al registro*

![](images/docker%205.4%20cap%203.png)

### 4. Ejecución del Contenedor (Docker Run)

Una vez que la imagen está lista, el paso definitivo para levantar la aplicación y comprobar que todo funciona es utilizar el comando **`docker run`**. Este comando es el encargado de coger nuestra imagen estática y convertirla en un contenedor vivo y operativo.

Para este despliegue, le pasamos una serie de parámetros clave:

* **Ejecución en Segundo Plano (`-d` o `--detach`):** Permite que el contenedor arranque y se quede corriendo de fondo, liberando nuestra terminal para poder seguir trabajando.
* **Mapeo de Puertos (`-p`):** Enlazamos un puerto de nuestra máquina anfitriona (host) con el puerto **80** interno del contenedor, que es exactamente donde nuestro servidor **Nginx** está esperando las peticiones.
* **Asignación de Nombre (`--name`):** Bautizamos al contenedor con un nombre descriptivo.

> *Despliegue del contenedor a partir de nuestra imagen*

Al lanzar el comando contra nuestra imagen etiquetada, el demonio de Docker crea este entorno aislado y nos devuelve un identificador único largo, confirmando que el servidor web ya está activo y sirviendo el juego.

![](images/docker%205.4%20cap%204.png)

### 5. Validación en la Plataforma Web (Docker Hub)

Una vez completada la subida desde nuestra consola, la prueba definitiva para confirmar que nuestra imagen está disponible a nivel global es revisar nuestro perfil en la interfaz web de **Docker Hub**. 

En esta vista del panel de control podemos comprobar varios puntos críticos:

* **Disponibilidad del Repositorio:** El repositorio público (`2048`) se ha creado y sincronizado correctamente bajo nuestro usuario, quedando visible en la plataforma.
* **Verificación de la Etiqueta:** Confirmamos que la etiqueta **latest** aparece indexada con su correspondiente tamaño de almacenamiento comprimido y el registro de la última actualización.
* **Portabilidad Absoluta:** Con esta verificación, aseguramos que la imagen ha dejado de existir únicamente en nuestro entorno local.

> *Repositorio web actualizado y disponible en la nube*

![](images/docker%205.4%20cap%205.png)

### 6. Automatización con GitHub Actions
Para asegurar la integración continua, se implementó un flujo de trabajo (workflow) en .github/workflows/publish.yml.
![](images/docker%205.4%20cap%206.png)

## 7. Activación del Flujo Automático: El Disparador del CI/CD

Para culminar esta práctica y poner a prueba toda la infraestructura que hemos definido como código, necesitamos un evento que inicie el proceso. Ese evento definitivo es la subida de nuestros cambios al repositorio remoto mediante el comando **`git push origin main`**.

Con esta única instrucción en la terminal, dejamos de operar en local y delegamos todo el trabajo pesado a la nube, activando el **workflow de GitHub Actions** que configuramos en los pasos anteriores.

* **Sincronización del Código:** El comando empaqueta y envía todos nuestros *commits* (incluyendo el código de la aplicación, el `Dockerfile` y el archivo `main.yml`) directamente a la rama principal (**main**) de nuestro repositorio en GitHub.
* **El Gatillo de la Automatización:** Nuestro archivo de configuración CI/CD está programado específicamente para "escuchar" cualquier evento de tipo *push* sobre esta rama. Al detectar la subida, GitHub interpreta esta acción como una orden directa de despliegue.
* **Ejecución Desatendida:** A partir de este instante, **GitHub Actions** toma el control absoluto. De forma totalmente transparente y sin intervención humana, la plataforma arranca un servidor efímero, construye la imagen de Docker usando nuestro `Dockerfile`, se autentica de forma segura y hace el *push* de la nueva versión a **Docker Hub**.

> *Ejecución del comando y subida de cambios al repositorio remoto*

En la salida de la consola podemos observar cómo Git enumera, comprime y transfiere los objetos hacia la URL remota de nuestro proyecto, confirmando que la rama local ha sincronizado perfectamente con la remota. Este es el **único paso manual** necesario para desencadenar toda la actualización del entorno de producción.

![](images/docker%205.4%20cap%207.png)