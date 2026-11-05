# Solución técnica completa  
**Objetivo:** Evitar que Coolify inyecte **todas** las variables de entorno definidas en el proyecto Docker Compose a **todos** los contenedores, permitiendo que cada servicio reciba **solo** las variables que le correspondan.

---

## 📋 Índice

1. [Entendiendo el problema](#entendiendo-el-problema)  
2. [Principios de Docker‑Compose sobre variables de entorno](#principios-de-docker-compose-sobre-variables-de-entorno)  
3. [Arquitectura actual de Coolify (punto de inyección)](#arquitectura-actual-de-coolify-punto-de-inyección)  
4. [Diseño de la solución](#diseño-de-la-solución)  
   - 4.1. Separación de `.env` global vs. `env_file` por‑servicio  
   - 4.2. Extensión de la UI de Coolify  
   - 4.3. Cambios en el backend (generación de `docker-compose.yml`)  
   - 4.4. Compatibilidad con Docker secrets (opcional)  
5. [Implementación paso a paso]  
   - 5.1. Refactor del modelo de datos  
   - 5.2. Modificaciones en la API/Controlador  
   - 5.3. Plantilla de `docker-compose.yml`  
   - 5.4. UI/UX (Vue/React)  
   - 5.5. Scripts de despliegue (Docker CLI)  
   - 5.6. Tests unitarios & integración  
6. [Ejemplo completo (antes / después)](#ejemplo-completo)  
7. [Consideraciones de seguridad y buenas prácticas]  
8. [Despliegue y validación]  
9. [FAQ & troubleshooting]  

---

## 1️⃣ Entendiendo el problema

- **Comportamiento actual:** Coolify lee un único archivo `.env` ubicado en la raíz del proyecto y lo pasa a `docker compose up`. Docker Compose, por defecto, **expone todas esas variables a cada servicio**, a menos que se sobreescriban.
- **Consecuencia:**  
  - Variables de base de datos, claves secretas o cualquier configuración específica de un micro‑servicio terminan disponibles en los demás contenedores.  
  - Riesgo de fuga de credenciales y colisiones de nombres.  
  - Dificultad para depurar problemas de configuración.

---

## 2️⃣ Principios de Docker‑Compose sobre variables de entorno

| Fuente | Prioridad (alta → baja) |
|--------|------------------------|
| `environment:` (definido en el service) | 1 |
| `env_file:` (uno o varios) | 2 |
| Archivo `.env` del **directorio del proyecto** | 3 |
| Variables del host (cuando se usa `docker compose run`) | 4 |

> **Importante:** Docker‑Compose **no** permite limitar el alcance del archivo `.env` a un servicio concreto. Por eso, la solución pasa por **no usar `.env`** para variables de servicio y, en su lugar, utilizar `env_file:` o `environment:`.

---

## 3️⃣ Arquitectura actual de Coolify (punto de inyección)

```text
┌───────────────┐
│ UI (Vue)      │  <-- usuario escribe variables en "Env Vars"
└─────┬─────────┘
      │
      ▼
┌───────────────┐
│ API (Laravel)│  <-- guarda variables en tabla `environment_variables`
└─────┬─────────┘
      │
      ▼
┌───────────────────────┐
│ Composer Service       │   // Genera docker‑compose.yml
│   - merge .env global   │   // inserta todas en la sección `environment:` de cada service
└─────────────┬─────────┘
              ▼
        docker compose up
```

El **cuello de botella** está en la generación del `docker-compose.yml`: Coolify **añade** todas las variables a cada servicio sin distinguir a cuál pertenecen.

---

## 4️⃣ Diseño de la solución

### 4.1. Separación de `.env` global vs. `env_file` por‑servicio  

| Tipo de variable | Uso recomendado | Implementación |
|------------------|-----------------|----------------|
| **Globales** (p.ej. `APP_ENV`, `HOST_URL`) | Necesarias para **todos** los contenedores | Mantener un `.env` raíz (Coolify lo seguirá leyendo) |
| **Servicio‑específicas** | Sólo para un contenedor | Crear **archivos `serviceX.env`** y declararlos en `env_file:` dentro del service |
| **Secrets** | Información sensible que no debe quedar en archivos | Usar Docker Secrets o `environment:` con valores encriptados (opcional) |

> Con esta estrategia, el único archivo que Coolify seguirá usando como “global” será el `.env` raíz; el resto se gestiona por servicio.

### 4.2. Extensión de la UI de Coolify  

- Añadir una pestaña **“Variables por servicio”**.  
- Permitir al usuario:
  1. **Crear / editar** un archivo `.env` asociado a un servicio (p.ej. `web.env`, `db.env`).  
  2. **Asignar** variables a un servicio mediante formulario (se guardarán en la tabla `service_env_vars`).  
  3. **Marcar** variables como **Secret** (para usar Docker Secrets).  

### 4.3. Cambios en el backend (generación de `docker-compose.yml`)  

1. **Modelo**  
   - Nueva tabla `service_env_files` (`service_id`, `file_path`).  
   - Nueva tabla `service_env_vars` (`service_id`, `key`, `value`, `is_secret`).  

2. **Lógica de renderizado**  
   - Al construir cada service en el archivo `docker-compose.yml`:  
     ```yaml
     services:
       web:
         image: my/web:latest
         env_file:
           - ./env/global.env               # <-- .env global (si existe)
           - ./env/web.env                  # <-- archivo específico del service
         environment:
           - DB_PASSWORD=${DB_PASSWORD}    # <-- solo si es secret (Docker secret)
     ```
   - **Solo** incluir en `env_file:` los archivos que pertenezcan al service.  

3. **Inyección de secrets** (opcional)  
   - Si `is_secret = true`, crear un secret en Docker y referenciarlo:  
     ```yaml
     secrets:
       db_password:
         external: true
     services:
       db:
         secrets:
           - source: db_password
             target: DB_PASSWORD
     ```

### 4.4. Compatibilidad con Docker secrets (opcional)

- Si la plataforma ya soporta Docker secrets, reutilizar esa infraestructura.  
- Caso contrario, agregar un pequeño wrapper que genere `docker secret create` antes de `docker compose up`.

---

## 5️⃣ Implementación paso a paso  

> **Requisitos:** Acceso al repositorio de Coolify (Laravel + Vue), conocimientos de Docker‑Compose, Composer, y Node.

### 5.1. Refactor del modelo de datos

```php
// database/migrations/2024_10_01_000001_create_service_env_files_table.php
Schema::create('service_env_files', function (Blueprint $table) {
    $table->id();
    $table->foreignId('service_id')->constrained()->onDelete('cascade');
    $table->string('file_name');      // p.ej. web.env
    $table->timestamps();
});

// database/migrations/2024_10_01_000002_create_service_env_vars_table.php
Schema::create('service_env_vars', function (Blueprint $table) {
    $table->id();
    $table->foreignId('service_id')->constrained()->onDelete('cascade');
    $table->string('key');
    $table->text('value')->nullable();
    $table->boolean('is_secret')->default(false);
    $table->timestamps();
});
```

- Ejecutar `php artisan migrate`.

### 5.2. Modificaciones en la API/Controlador

```php
// app/Http/Controllers/ServiceEnvController.php
public function store(Request $request, Service $service)
{
    $data = $request->validate([
        'key'        => 'required|string',
        'value'      => 'nullable|string',
        'is_secret'  => 'boolean',
    ]);

    $service->envVars()->create($data);
    return response()->json(['status' => 'created']);
}

// similar para env files (upload / create)
```

- Añadir relaciones en el modelo `Service`:

```php
class Service extends Model
{
    public function envFiles()
    {
        return $this->hasMany(ServiceEnvFile::class);
    }

    public function envVars()
    {
        return $this->hasMany(ServiceEnvVar::class);
    }
}
```

### 5.3. Plantilla de `docker-compose.yml`

En el **builder** que genera el YAML (normalmente un `ComposeBuilder.php`), cambiar:

```php
foreach ($project->services as $service) {
    $yaml['services'][$service->name] = [
        'image' => $service->image,
        // 1️⃣ Añadir env_file por servicio
        'env_file' => array_merge(
            $globalEnvFile ? [$globalEnvFile] : [],
            $service->envFiles->pluck('file_name')->map(fn($f) => "./env/{$f}")->toArray()
        ),
        // 2️⃣ Añadir variables no secret en environment
        'environment' => $service->envVars
            ->where('is_secret', false)
            ->pluck('value', 'key')
            ->toArray(),
        // 3️⃣ Opcional: secrets
        'secrets' => $service->envVars
            ->where('is_secret', true)
            ->map(fn($v) => ['source' => $v->key, 'target' => $v->key])
            ->values()
            ->toArray(),
    ];
}
```

- **Persistencia de archivos**: Cada `ServiceEnvFile` se guarda físicamente bajo `/var/www/project/.coolify/env/{file_name}` (o el directorio que use Coolify).  

- **Creación de secrets** (si se habilita):

```php
foreach ($project->services as $service) {
    foreach ($service->envVars->where('is_secret', true) as $secret) {
        // docker secret create --dry-run para obtener la definición en compose
        $yaml['secrets'][$secret->key] = [
            'external' => true,
        ];
    }
}
```

### 5.4. UI/UX (Vue/React)

1. **Nuevo componente** `ServiceEnvEditor.vue`  
   - Listado de variables con botón “Add”.  
   - Checkbox “Secret”.  
   - Botón “Export to .env file”.  

2. **Nuevo componente** `ServiceEnvFileUploader.vue`  
   - Drag‑&‑drop o selector de archivo.  
   - Muestra el nombre del archivo asociado al service.  

3. **Integración**  
   - En la página de edición del *service*, añadir tabs: **General | Variables | Archivos .env**.  

4. **API Calls**  
   - `POST /api/services/{id}/env-vars` → `ServiceEnvController@store`.  
   - `POST /api/services/{id}/env-files` → `ServiceEnvFileController@store`.  

### 5.5. Scripts de despliegue (Docker CLI)

Coolify ejecuta algo similar a:

```bash
docker compose -f /tmp/compose.yml up -d
```

Después de generar el nuevo `compose.yml`, **no** es necesario pasar `--env-file`. Docker leerá automáticamente los archivos listados