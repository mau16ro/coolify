# Solución Técnica Completa  
**Objetivo:** Evitar que Coolify inyecte *todas* las variables de entorno en cada contenedor de un proyecto Docker‑Compose. Cada servicio debe recibir **solo** las variables que le correspondan (definidas explícitamente en su `environment:` o `env_file:`).

---

## 1. Visión general del problema

| Paso actual (Coolify) | Comportamiento no deseado |
|-----------------------|---------------------------|
| 1️⃣ Se lee un único archivo `.env` del proyecto. | Todas las variables quedan disponibles para la interpolación del archivo `docker‑compose.yml`. |
| 2️⃣ Coolify genera el `docker‑compose.yml` y **añade** cada variable leída a la sección `environment:` de **todos** los servicios. | Cada contenedor recibe variables que no le pertenecen, aumentando la superficie de ataque y provocando colisiones. |
| 3️⃣ Docker ejecuta el stack con esas variables globales. | Resultado: *leak* de secretos y comportamiento inesperado. |

Docker Compose, por diseño, **solo** expone al contenedor las variables que aparecen en:

* `environment:` (lista o mapa)
* `env_file:` (uno o varios archivos)

El archivo `.env` a nivel de proyecto **solo** sirve para *interpolación* de valores dentro del propio `docker‑compose.yml`. Por tanto, el comportamiento de Coolify es un **bug**.

---

## 2. Principios de la solución

1. **Separación de ámbitos** – El `.env` global solo se usará para interpolar el `docker‑compose.yml`. No será propagado a los contenedores.
2. **Definición explícita** – Cada servicio declarará sus variables mediante:
   * `environment:` (clave‑valor) **o**
   * `env_file:` (ruta a un archivo `.env` propio del servicio).
3. **Compatibilidad retro‑activa** – Si un proyecto ya define variables a nivel de proyecto (como antes), Coolify mantendrá la opción de “propagar globalmente” mediante un *toggle*.
4. **Seguridad por defecto** – El toggle estará **desactivado** en instalaciones nuevas; los proyectos existentes pueden migrar manualmente.

---

## 3. Arquitectura de los cambios

```
src/
├─ Services/
│   ├─ ComposeGenerator.php      <-- Genera docker‑compose.yml
│   └─ EnvResolver.php          <-- Nueva capa para resolver variables por servicio
├─ Models/
│   └─ Service.php              <-- Añade campo `env_file` y `environment_map`
└─ UI/
    └─ ServiceForm.vue          <-- UI para definir variables por servicio
```

### 3.1. Nueva clase `EnvResolver`

```php
<?php
namespace App\Services;

class EnvResolver
{
    /**
     * @param  Service $service
     * @param  array   $projectEnv   // variables del .env global
     * @return array                 // [ 'environment' => [], 'env_file' => [] ]
     */
    public static function resolve(Service $service, array $projectEnv): array
    {
        // 1️⃣ Variables definidas en la UI del servicio (clave‑valor)
        $explicit = $service->environment_map ?? [];

        // 2️⃣ Archivo .env propio del servicio (ruta relativa al proyecto)
        $envFile = $service->env_file
            ? [ $service->env_file ]
            : [];

        // 3️⃣ Si el proyecto está marcado como "global env propagation"
        //    (campo booleano `propagate_all`), añadimos el resto del .env
        if ($service->propagate_all) {
            $missing = array_diff_key($projectEnv, $explicit);
            $explicit = array_merge($explicit, $missing);
        }

        return [
            'environment' => $explicit,
            'env_file'    => $envFile,
        ];
    }
}
```

### 3.2. Modificación de `ComposeGenerator`

```php
public function generate(array $services, array $projectEnv): string
{
    $compose = [
        'version' => '3.9',
        'services'=> []
    ];

    foreach ($services as $service) {
        $resolved = EnvResolver::resolve($service, $projectEnv);

        $serviceDef = [
            'image' => $service->image,
            // otros atributos (ports, volumes, etc.)
        ];

        if (!empty($resolved['environment'])) {
            $serviceDef['environment'] = $resolved['environment'];
        }

        if (!empty($resolved['env_file'])) {
            $serviceDef['env_file'] = $resolved['env_file'];
        }

        $compose['services'][$service->name] = $serviceDef;
    }

    return Yaml::dump($compose, 4, 2);
}
```

### 3.3. Cambios en la base de datos

| Tabla | Campo nuevo | Tipo | Comentario |
|-------|--------------|------|------------|
| `services` | `environment_map` | JSON | Mapa `{ "DB_HOST": "mysql", "DB_PASSWORD": "secret" }` |
| `services` | `env_file` | STRING (nullable) | Ruta relativa al directorio del proyecto (`services/api/.env`) |
| `services` | `propagate_all` | BOOLEAN (default `false`) | Mantiene compatibilidad con versiones anteriores |

Ejecutar migración:

```bash
php artisan make:migration add_env_fields_to_services_table --table=services
```

```php
public function up()
{
    Schema::table('services', function (Blueprint $table) {
        $table->json('environment_map')->nullable();
        $table->string('env_file')->nullable();
        $table->boolean('propagate_all')->default(false);
    });
}
```

### 3.4. UI/UX

* **Formulario de Servicio** – Añadir dos secciones:
  1. **Variables de entorno** – tabla editable (key/value) que rellena `environment_map`.
  2. **Archivo .env del servicio** – selector de archivo dentro del proyecto (crea `services/<service>/`.env` si no existe).

* **Opciones avanzadas** – Checkbox “Propagar todas las variables del proyecto a este contenedor” (para migraciones).

* **Validación** – Evitar nombres duplicados dentro de `environment_map`. Avisar si la ruta `env_file` no existe.

---

## 4. Flujo de trabajo recomendado para los usuarios

1. **Crear archivos `.env` por servicio** (ej. `services/web/.env`, `services/db/.env`).
2. **En la UI de Coolify**:
   * Seleccionar el servicio → “Variables de entorno”.
   * Si el archivo existe, pulsar “Importar desde .env” (carga automática a la tabla).
   * Guardar.
3. **Desactivar “Propagar globalmente”** (por defecto está desactivado).
4. **Deploy** – Coolify genera un `docker‑compose.yml` que contiene, por servicio, solo:
   ```yaml
   services:
     web:
       env_file:
         - ./services/web/.env
   ```
   o bien la sección `environment:` con las variables definidas en la tabla.

---

## 5. Compatibilidad con proyectos existentes

1. **Detección automática** – En la primera ejecución después del despliegue del parche, Coolify revisa si existen variables definidas a nivel de proyecto (`project.env`). Si `propagate_all` está activado en **algún** servicio, se conserva el comportamiento anterior.
2. **Migración asistida**:
   * Botón “Dividir variables por servicio” → analiza el `docker‑compose.yml` actual y propone crear archivos `.env` por servicio basándose en las dependencias (`links`, `depends_on`).
   * El usuario confirma y Coolify rellena automáticamente `environment_map` y crea los archivos `.env`.

---

## 6. Pruebas

### 6.1. Unit Tests (PHPUnit)

```php
public function test_env_resolver_respects_explicit_variables()
{
    $service = Service::factory()->make([
        'environment_map' => ['FOO' => 'bar'],
        'env_file' => null,
        'propagate_all' => false,
    ]);
    $projectEnv = ['FOO' => 'global', 'BAZ' => 'global2'];

    $resolved = EnvResolver::resolve($service, $projectEnv);

    $this->assertEquals(['FOO' => 'bar'], $resolved['environment']);
    $this->assertEmpty($resolved['env_file']);
}

public function test_env_resolver_propagates_when_flag_set()
{
    $service = Service::factory()->make([
        'environment_map' => ['FOO' => 'service'],
        'propagate_all' => true,
    ]);
    $projectEnv = ['FOO' => 'global', 'BAZ' => 'global2'];

    $resolved = EnvResolver::resolve($service, $projectEnv);

    $this->assertEquals(['FOO' => 'service', 'BAZ' => 'global2'], $resolved['environment']);
}
```

### 6.2. Integration Tests (Docker‑Compose)

1. **Escenario A** – Servicio *web* con `env_file: ./services/web/.env` que contiene `APP_ENV=prod`.  
   *Ejecutar:* `docker compose up -d web` → `docker exec web env | grep APP_ENV` debe devolver `APP_ENV=prod` **únicamente** en *web*.

2. **Escenario B** – Servicio *worker* sin archivo `.env` y sin variables explícitas.  
   *Ejecutar:* `docker exec worker env | grep APP_ENV` → **no** debe aparecer.

3. **Escenario C** – `propagate_all = true` en *api*.  
   *Resultado esperado:* `api` recibe todas las variables del `.env` global, los demás no.

Los tests deben ejecutarse en CI (GitHub Actions) bajo un job que levante Docker y verifique los `docker exec`.

---

## 7. Seguridad y buenas prácticas

| Aspecto | Acción |
|---------|--------|
| **Principio de menor privilegio** | Solo exponer variables necesarias a cada contenedor. |
| **Auditoría** | Añadir log (`info`) en `ComposeGenerator` indicando cuántas variables se inyectan por servicio. |
| **Escaneo de secretos** | Integrar con TruffleHog / GitGuardian para detectar variables sensibles que se hayan dejado en el `.env` global. |
| **Documentación** | Actualizar la guía de usuario de Coolify: *“Cómo manejar variables de entorno por servicio”.* |

---

## 8. Roadmap de despliegue

| Fase | Acción | Responsable | Tiempo estimado |
|------|--------|--------------|-----------------|
| **1. Desarrollo** | Implementar `EnvResolver`, migraciones, UI, tests | Backend & Frontend devs | 2 semanas |
| **2. QA** | Ejecutar tests unit/integración, validar casos de uso reales | QA team | 1 semana |
| **3. Beta** | Habilitar la opción en entorno de staging, notificar a usuarios | Product Manager | 1 semana |
| **4. Producción** | Deploy a producción, habilitar por defecto `propagate_all = false` | DevOps | 1 día |
| **5. Follow‑up** | Monitorear incidencias, crear tutorial de migración | Docs & Support | 2 semanas |

---

## 9. Resumen de los cambios clave

| Archivo / Clase | Cambio principal |
|------------------|------------------|
| `src/Services/EnvResolver.php` | Nueva lógica de resolución de variables por servicio. |
| `src/Services/ComposeGenerator.php` | Usa `EnvResolver` y elimina la inyección global. |
| Migración DB | Campos `environment_map`, `env_file`, `propagate_all`. |
| UI (`ServiceForm.vue`) | Formulario para definir variables y asignar archivo `.env`. |
| Tests (`tests/Unit/EnvResolverTest.php`) | Cobertura de los nuevos casos. |
| Docs (`docs/environment.md`) | Guía de uso y migración. |

Con estos cambios Coolify dejará de compartir **todas** las variables de entorno entre contenedores y ofrecerá a los usuarios un control granular, alineado con la práctica recomendada de Docker Compose y con una postura de seguridad mucho más robusta. 🚀