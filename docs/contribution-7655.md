## 📦 Solución Técnica  
**Objetivo:** Evitar que Coolify inyecte **todas** las variables de entorno del archivo global `.env` en **cada** contenedor del proyecto Docker‑Compose. Cada servicio debe recibir **solo** las variables que le correspondan.

---

## 1️⃣ Visión General

| **Componente** | **Responsabilidad** |
|----------------|---------------------|
| **Coolify UI** | Permitir al usuario subir/definir *env‑files* por servicio. |
| **Backend (Laravel/PHP)** | Parsear los archivos de entorno y generar el `docker‑compose.yml` con la sección `env_file` (o `environment`) **específica** por contenedor. |
| **Docker‑Compose** | Utilizar `env_file` o `environment` para cada servicio. Docker ya garantiza que **solo** esas variables estén disponibles dentro del contenedor. |
| **Persistencia** | Guardar los env‑files en la base de datos/almacenamiento de Coolify con cifrado (opcional) y sincronizarlos con los repositorios del proyecto. |

---

## 2️⃣ Análisis del Problema Actual

1. **Coolify lee un único `.env`** (ubicado en la raíz del proyecto o en la raíz del servidor) y lo pasa como **variables globales** al comando `docker compose up`.  
2. Docker‑Compose interpreta esas variables como **variables de sustitución** (`${VAR}`) *y* las exporta a **todos** los servicios, aunque no estén declaradas en `environment` ni `env_file`.  
3. Resultado: **contaminación** de entornos, posible fuga de credenciales y comportamiento inesperado.

---

## 3️⃣ Estrategia de Solución

1. **Desacoplar el `.env` global** del proyecto Docker‑Compose.  
2. **Introducir env‑files per‑service** (`service1/.env`, `service2/.env`, …) y/o bloques `environment` explícitos.  
3. **Modificar el generador de Compose** en el backend de Coolify para:
   - Detectar los archivos de entorno asociados a cada servicio.  
   - Insertar la cláusula `env_file:` (o `environment:`) solo para ese servicio.  
   - **No** añadir la variable `COMPOSE_PROJECT_ENVIRONMENT` ni pasar el `.env` global a `docker compose`.  
4. **Actualizar la UI** para gestionar estos archivos de forma visual.  
5. **Opcional:** Soportar **Docker Secrets** o **HashiCorp Vault** para variables sensibles.

---

## 4️⃣ Cambios en la Arquitectura

```mermaid
flowchart LR
    subgraph UI
        A[Form: Upload env per service] --> B[API Call]
    end
    subgraph Backend
        B --> C[Persist env files (encrypted)]
        C --> D[ComposeGenerator]
        D -->|docker-compose.yml| E[Docker Engine]
    end
    subgraph Docker
        E --> F[Service A (env from A/.env)]
        E --> G[Service B (env from B/.env)]
    end
```

*Los bloques rojos indican la ruta de datos de variables de entorno específicas.*

---

## 5️⃣ Implementación Paso a Paso

### 5.1. Modelo de Datos

```php
// app/Models/ServiceEnv.php
class ServiceEnv extends Model
{
    protected $table = 'service_envs';
    protected $fillable = ['service_id', 'file_name', 'encrypted_content'];
    protected $casts = [
        'encrypted_content' => 'encrypted', // Laravel 9+ built‑in encryption
    ];
}
```

- Cada registro enlaza un **servicio** con su archivo `.env`.  
- Se guarda cifrado para evitar exposición accidental.

### 5.2. API y UI

#### 5.2.1. Rutas

```php
// routes/api.php
Route::post('projects/{project}/services/{service}/env', [EnvController::class, 'store']);
Route::delete('projects/{project}/services/{service}/env', [EnvController::class, 'destroy']);
```

#### 5.2.2. Controlador

```php
class EnvController extends Controller
{
    public function store(Project $project, Service $service, Request $request)
    {
        $request->validate(['env_file' => 'required|file']);
        $content = $request->file('env_file')->get();
        ServiceEnv::updateOrCreate(
            ['service_id' => $service->id],
            ['file_name' => $request->file('env_file')->getClientOriginalName(),
             'encrypted_content' => $content]
        );
        return response()->json(['message' => 'Env file saved']);
    }

    public function destroy(Project $project, Service $service)
    {
        ServiceEnv::where('service_id', $service->id)->delete();
        return response()->json(['message' => 'Env file removed']);
    }
}
```

#### 5.2.3. UI (Vue/React)

- Añadir pestaña **“Environment”** en la vista de configuración del servicio.  
- Botón **Upload** → POST a `/api/projects/:id/services/:sid/env`.  
- Lista de archivos con opción **Delete**.

### 5.3. Generador de Docker‑Compose

#### 5.3.1. Clase `ComposeBuilder`

```php
class ComposeBuilder
{
    protected $project;
    protected $services; // Collection of Service models

    public function __construct(Project $project)
    {
        $this->project = $project;
        $this->services = $project->services;
    }

    public function render(): string
    {
        $compose = [
            'version' => '3.9',
            'services' => [],
        ];

        foreach ($this->services as $service) {
            $serviceDef = $this->baseServiceDefinition($service);
            $envFilePath = $this->writeEnvFileToTmp($service);
            if ($envFilePath) {
                $serviceDef['env_file'] = [$envFilePath];
            }
            $compose['services'][$service->docker_name] = $serviceDef;
        }

        return Yaml::dump($compose, 4, 2);
    }

    protected function baseServiceDefinition(Service $service): array
    {
        return [
            'image' => $service->image,
            'restart' => $service->restart_policy,
            // ... otros atributos
        ];
    }

    /** 
     * 1️⃣ Desencripta, escribe a /tmp y devuelve la ruta absoluta.
     * 2️⃣ Si no hay env, devuelve null.
     */
    protected function writeEnvFileToTmp(Service $service): ?string
    {
        $env = ServiceEnv::where('service_id', $service->id)->first();
        if (!$env) {
            return null;
        }

        $tmpPath = sys_get_temp_dir() . '/' . Str::uuid() . '_' . $env->file_name;
        file_put_contents($tmpPath, $env->encrypted_content);
        return $tmpPath;
    }
}
```

- **Clave:** Cada servicio tiene su propio `env_file`. Docker solo lo inyecta al contenedor correspondiente.  
- No se usa `--env-file` global al lanzar `docker compose up`.

### 5.4. Orquestación del Deploy

```bash
# En vez de:
docker compose --env-file .env up -d

# Se usa:
docker compose -f /tmp/generated-compose.yml up -d
```

- El backend escribe el `docker-compose.yml` temporalmente y lo pasa a Docker.  
- No se pasa ninguna opción `--env-file` global.

### 5.5. Compatibilidad con Variables de Sustitución

- Si un servicio necesita **referenciar** variables de otro servicio (p.ej., `DATABASE_URL=${DB_HOST}`), esas variables deben **también** declararse dentro del mismo `env_file` o mediante la sección `environment:`.  
- Docker resuelve las variables **solo** dentro del scope del archivo del servicio, evitando “spillover”.

### 5.6. Soporte para **Docker Secrets** (opcional)

1. Añadir campo `is_secret` en `service_envs`.  
2. En `ComposeBuilder`, generar:

```yaml
secrets:
  my_secret:
    external: true

services:
  app:
    secrets:
      - source: my_secret
        target: MY_SECRET
        uid: '0'
        gid: '0'
        mode: 0440
```

- Coolify podrá crear los secretos con `docker secret create` antes del `docker compose up`.

---

## 6️⃣ Plan de Migración

| Paso | Acción | Impacto |
|------|--------|---------|
| **1** | Exportar el `.env` actual a archivos per‑service (script o UI). | Separa variables. |
| **2** | Desactivar la opción “Inject global env” en la configuración del proyecto (añadir flag `use_global_env = false`). | Previene regresión. |
| **3** | Ejecutar `docker compose down && docker compose up -d` con el nuevo `docker‑compose.yml`. | Reinicia contenedores con entornos limpios. |
| **4** | Verificar logs y health‑checks. | Asegura que todo funciona. |
| **5** | (Opcional) Eliminar el antiguo `.env` del servidor. | Reduce superficie de ataque. |

> **Nota:** Para proyectos legacy que dependan del comportamiento actual, se mantendrá un **modo de compatibilidad** (`global_env_mode = true`). Este modo será *deprecated* y será removido en la versión **v2.0**.

---

## 7️⃣ Pruebas

### 7.1. Unit Tests (PHPUnit)

```php
public function test_service_receives_its_own_env_file()
{
    $project = Project::factory()->create();
    $serviceA = Service::factory()->create(['project_id' => $project->id, 'docker_name' => 'svc-a']);
    $serviceB = Service::factory()->create(['project_id' => $project->id, 'docker_name' => 'svc-b']);

    ServiceEnv::create([
        'service_id' => $serviceA->id,
        'file_name' => '.env',
        'encrypted_content' => encrypt("FOO=foo\nSHARED=shared"),
    ]);
    ServiceEnv::create([
        'service_id' => $serviceB->id,
        'file_name' => '.env',
        'encrypted_content' => encrypt("BAR=bar\nSHARED=shared"),
    ]);

    $builder = new ComposeBuilder($project);
    $yaml = $builder->render();
    $doc = Yaml::parse($yaml);

    $this->assertArrayHasKey('env_file', $doc['services']['svc-a']);
    $this->assertStringContainsString('FOO=foo', file_get_contents($doc['services']['svc-a']['env_file'][0]));
    $this->assertStringNotContainsString('BAR=bar', file_get_contents($doc['services']['svc-a']['env_file'][0]));
}
```

### 7.2. Integration Tests (Docker)

1. **Escenario**: 2 servicios, cada uno con su propio `.env`.  
2. **Pasos**:  
   - `docker compose up -d` con el compose generado.  
   - `docker exec svc-a env | grep FOO` → debe existir.  
   - `docker exec svc-a env | grep BAR` → **no** debe existir.  
   - Repetir para `svc-b`.  
3. **Resultado esperado**: aislamiento completo.

### 7.3. Tests de UI

- Simular carga de archivo `.env` para un servicio.  
- Verificar que en la vista de “Raw Docker compose” aparezca la clave `env_file:` bajo el servicio correcto.  
- Probar eliminación del archivo y confirmar que desaparece del compose.

---

## 8️⃣ Documentación

1. **Guía de usuario** → *“Cómo definir variables de entorno por servicio”* (añadir captura de pantalla del uploader).  
2. **FAQ** → *“¿Por qué mis variables desaparecen?”* (explic