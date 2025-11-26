
# Servidor Web & Proxy Reverso - Ingeniería de Software III

Este repositorio contiene la configuración del servidor web principal para la materia **Ingeniería de Software III**. Actúa como punto de entrada (Gateway) y Proxy Reverso para los distintos proyectos desarrollados durante la cursada.

El objetivo es tener un único punto de acceso (`http://lab-sys.untdf.edu.ar/`) que enrute el tráfico hacia los contenedores frontend específicos de cada grupo o trabajo práctico, sin exponer múltiples puertos en el servidor host.

## 📋 Arquitectura

El servicio se basa en **Nginx** corriendo en un contenedor Docker.
* **Red:** Utiliza una red externa llamada `is3_red` para comunicarse con los contenedores de los proyectos.
* **Enrutamiento:** Redirige el tráfico basado en la URL (path-based routing) hacia el contenedor correspondiente.
* **Landing Page:** Sirve una página estática en la raíz (`/`) que actúa como índice.

## 🚀 Requisitos Previos

Antes de levantar este contenedor, asegúrate de que el servidor cumpla con lo siguiente:

1.  **Docker** y **Docker Compose** instalados.
2.  **Red Docker creada:**
    El archivo `compose.yaml` espera una red externa. Debes crearla antes de iniciar el servicio:
    ```bash
    docker network create is3_red
    ```

## 🛠️ Instalación y Despliegue

1.  **Clonar el repositorio:**
    ```bash
    git clone [https://github.com/is3-untdf/servidor-web](https://github.com/is3-untdf/servidor-web)
    cd servidor-web
    ```

2.  **Verificar estructura:**
    Asegúrate de tener la carpeta `landing` con un `index.html` básico, ya que se monta como volumen.
    ```text
    .
    ├── compose.yaml
    ├── nginx.conf
    └── landing/
        └── index.html
    ```

3.  **Iniciar el servicio:**
    ```bash
    docker compose up -d
    ```

## ⚙️ Configuración de Nginx (Detalles Técnicos)

La configuración (`nginx.conf`) utiliza una estrategia de **resolución dinámica de DNS** para evitar caídas del servicio principal.

### ¿Por qué está configurado así?
Normalmente, si Nginx inicia y un `upstream` (el contenedor de un proyecto) no existe, Nginx falla y se detiene. Para evitar esto en un entorno educativo donde los proyectos se prenden y apagan constantemente, usamos:

```nginx
resolver 127.0.0.11 valid=10s;
...
set $gpe_front "http://gpe_front:80";
proxy_pass $gpe_front;
````

  * **Resolver:** Fuerza a Nginx a usar el DNS interno de Docker (127.0.0.11).
  * **Variables:** Al usar una variable en `proxy_pass` (ej. `set $gpe_front...`), Nginx resuelve la IP en el momento de la petición (runtime) y no al inicio del servicio. Esto permite que el Proxy siga funcionando aunque los proyectos de los alumnos estén apagados o se reinicien.

## ➕ Cómo agregar un nuevo proyecto

Para agregar un nuevo proyecto (ej. `nuevo_proyecto`) al proxy:

1.  **En el proyecto del alumno:**

      * Asegurar que su contenedor frontend esté conectado a la red `is3_red`.
      * Asignarle un nombre de contenedor estable (ej. `nuevo_front`).

2.  **En este repositorio (nginx.conf):**
    Agregar un nuevo bloque `location`:

    ```nginx
    location = /nuevo_proyecto {
        return 301 /nuevo_proyecto/;
    }

    location /nuevo_proyecto/ {
        set $nuevo_front "http://nuevo_front:80"; # Nombre del contenedor
        proxy_pass $nuevo_front;
        
        # Headers para manejo correcto de IPs y protocolos
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
    }
    ```

3.  **Recargar configuración:**
    No es necesario bajar el contenedor, solo recargar Nginx para que tome los cambios:

    ```bash
    docker compose exec nginx_reverse_proxy nginx -s reload
    ```

## 📂 Proyectos Activos

| Ruta | Contenedor Destino | Descripción |
| :--- | :--- | :--- |
| `/` | (Local) | Landing Page / Índice |
| `/gpe2024` | `gpe_front` | Proyecto GPE 2024 |
| `/mapa2025` | `mapa_front` | Proyecto Mapa 2025 |

-----

### ⚠️ Solución de Problemas

  * **Error "Gateway Timeout" (504):** Verifica que el contenedor destino (ej. `gpe_front`) esté encendido y correctamente conectado a la red `is3_red`.
  * **Error "Host not found" (Logs):** Si el contenedor destino no existe, Nginx devolverá un error 502 (Bad Gateway) al intentar acceder a esa URL específica, pero el servidor principal **no se caerá** gracias a la configuración dinámica del DNS explicada anteriormente.
