# Documentación: Sistema de Sondas de Ejecución (Unibotics)

## 1. Descripción General
El sistema de "sondas" (probes) es un mecanismo de telemetría diseñado para monitorizar la interacción de los estudiantes con la plataforma Unibotics. El objetivo principal es registrar eventos clave, como cuándo un estudiante ejecuta o detiene su código, para poder analizar posteriormente métricas de aprendizaje, esfuerzo y tiempo de dedicación.

Actualmente, se ha implementado la **Sonda de Ejecución**, que registra en base de datos cada vez que un estudiante interactúa con los controles de simulación (Play).

## 2. Arquitectura y Flujo de Datos
El sistema sigue un modelo Cliente-Servidor desacoplado (React -> Django REST -> PostgreSQL):

1. **Interacción (React):** El estudiante pulsa el botón "Play" en la interfaz.
2. **Emisión (React):** El frontend intercepta el clic y envía una petición HTTP `POST` asíncrona (JSON) al backend con el token CSRF, sin interrumpir el arranque del contenedor RADI.
3. **Recepción (Django):** El endpoint en Django recibe el JSON, verifica que el usuario tiene una sesión activa (autenticación) y extrae los datos del evento.
4. **Persistencia (PostgreSQL):** Django guarda el evento en la base de datos `academy_db` relacionándolo con el usuario que hizo la petición y generando un *timestamp* automático.

---

## 3. Archivos Modificados / Creados

Toda la lógica del backend para esta funcionalidad se ha centralizado en la aplicación **`apps/academy`**. Siguiendo las directrices del equipo, no se ha utilizado `common` (reservado para estructura general) ni el `views.py` de `react_frontend` (reservado solo para renderizado y contexto de plantillas HTML).

### Backend (Django)

#### A. `apps/academy/models.py`
Se ha creado el modelo de base de datos que representa la sonda.
*   **Modelo añadido:** `ExecutionProbe`
*   **Campos:**
    *   `user`: ForeignKey conectada al modelo de usuarios (Django Auth).
    *   `exercise`: CharField para guardar el identificador del ejercicio (ej. `follow_line`).
    *   `event`: CharField para registrar la acción (ej. `start_execution`, `stop_execution`).
    *   `timestamp`: DateTimeField con `auto_now_add=True` para registrar la hora exacta del servidor.

#### B. `apps/academy/views.py`
Se ha añadido el controlador que procesa la petición de React.
*   **Función añadida:** `register_execution_probe(request)`
*   **Lógica:** Recibe un POST con formato JSON. Verifica `request.user.is_authenticated`. Si es válido, crea una instancia de `ExecutionProbe` y devuelve un código HTTP 201 (Created).

#### C. `apps/academy/urls.py`
Se ha expuesto la vista anterior a través de una ruta de la API.
*   **Ruta añadida:** `path('register_execution_probe/', views.register_execution_probe, name='register_execution_probe')`

*(Opcional: Se recomienda registrar el modelo en `apps/academy/admin.py` añadiendo `admin.site.register(ExecutionProbe)` para facilitar la visualización desde el panel de administrador web).*

### Frontend (React)

#### A. `playpause.tsx` (Componente de Interfaz)
Ubicado en la interfaz de usuario de los ejercicios.
*   **Modificación:** Se ha añadido una función tipo `fetch` (ej. `sendExecutionProbe`) que se dispara justo en el manejador del evento `onClick` del botón Play.
*   **Carga útil (Payload):** Envía un objeto JSON con la forma `{"event": "start_execution", "exercise": "<project_id>"}`.

---

## 4. Gestión de Base de Datos y Comprobación

Para que los cambios en `models.py` tuvieran efecto en la base de datos PostgreSQL local (contenedor `academy_db`), se ejecutaron las migraciones del ORM de Django:

```bash
python manage.py makemigrations academy
python manage.py migrate
```

### Cómo ver los registros guardados (Troubleshooting)

Para comprobar que las sondas están llegando y guardándose correctamente mediante la terminal en el entorno local (WSL/Docker):

1. Es necesario tener instalado el cliente de PostgreSQL en la máquina host (WSL):
   `sudo apt install postgresql-client`
2. Navegar al directorio del proyecto y activar el entorno virtual.
3. Abrir la consola de base de datos de Django:
   `python manage.py dbshell`
4. Ejecutar la consulta SQL para ver los últimos registros:
   `SELECT * FROM academy_executionprobe ORDER BY timestamp DESC LIMIT 10;`
5. Para salir, escribir `\q`.
