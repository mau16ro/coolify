# 📦 Mejora: **No compartir todas las variables de entorno entre contenedores en un proyecto Docker‑Compose**  

**Autor:** @mau16ro  
**Fecha:** 2026‑11‑05  
**Versión objetivo:** Coolify v2.7 (y posteriores)

---  

## 1️⃣ Resumen del problema  

Actualmente **Coolify** inyecta **todas** las variables definidas en el archivo `.env` (o en la sección “Environment variables” del proyecto) a **todos** los servicios que aparecen en el `docker‑compose.yml` generado.  

- **Efectos**:  
  - Fugas de secretos entre contenedores que no los necesitan.  
  - Sobrescritura accidental de variables con el mismo nombre pero diferentes valores para distintos servicios.  
  - Dificultad para cumplir con normas de seguridad (principio del menor privilegio).  

## 2️⃣ Objetivo  

Permitir que **cada contenedor reciba únicamente las variables de entorno que le corresponden** y que el resto permanezca oculto.  

- Mantener compatibilidad con la forma actual de definir variables a nivel de proyecto.  
- No romper la experiencia “UI → Raw Docker Compose”.  
- Evitar cambios disruptivos en la API pública de Coolify.  

## 3️⃣ Análisis de la causa raíz  

| Paso | Qué ocurre hoy | Por qué ocurre |
|------|----------------|----------------|
| 1️⃣  | Coolify lee el `.env` del proyecto y todas las variables definidas en la UI → `Project.env` | El loader de variables está en `src/services/compose/ComposeBuilder.php` y siempre agrega `$projectEnv` al bloque `environment` de **todos** los servicios. |
| 2️⃣  | Genera `docker‑compose.yml` con la sección `environment:` bajo cada servicio, copiando **todas** las variables | El método `addGlobalEnvVars()` se llama sin filtro. |
| 3️⃣  | Docker, al lanzar los contenedores, exporta esas variables en el entorno del proceso del contenedor | No hay mecanismo de “whitelisting” por servicio. |

En Docker‑Compose, **solo** se inyectan variables cuando:  

- Se especifican bajo `environment:` (lista o mapa).  
- Se usa `env_file:` (uno o varios).  

Si la generación del archivo YAML incluye todas las variables, Docker las pasa a todos los contenedores sin opción de exclusión.

## 4️⃣ Solución propuesta  

### 4.1 Principios de diseño  

1. **Scoping explícito** – Cada servicio debe declarar explícitamente las variables que necesita.  
2. **Compatibilidad ascendente** – Si un proyecto ya funciona con la inyección global, seguirán recibiendo todas las variables (modo “legacy”).  
3. **UI‑First** – La interfaz debe permitir asignar variables por servicio sin necesidad de editar manualmente el YAML.  
4. **Seguridad por defecto** – El modo “strict” será el predeterminado a partir de v2.7; el modo “legacy” deberá habilitarse explícitamente.  

### 4.2 Cambios de alto nivel  

| Área | Cambio | Detalle |
|------|--------|---------|
| **Modelo de datos** | Añadir tabla `service_environment_variables` (many‑to‑many entre `services` y `environment_variables`). | Permite asignar variables a nivel de servicio. |
| **API/Backend** | Extender el endpoint `POST /projects/{id}/compose` para aceptar `serviceEnv: { serviceId: [varKey] }`. | Mantiene la API actual y agrega la nueva opción. |
| **Composer Builder** | Refactorizar `ComposeBuilder::addGlobalEnvVars()` → `ComposeBuilder::addScopedEnvVars()`. | Sólo inserta variables que aparecen en `serviceEnv`. |
| **UI** | En la vista del servicio (modal “Edit Service”), añadir sección “Variables de entorno específicas”. | Permite al usuario seleccionar de la lista global o crear nuevas. |
| **Modo de operación** | Variable de proyecto `COMPOSE_ENV_MODE=strict|legacy`. | En `strict` solo variables asignadas aparecen; en `legacy` se conserva el comportamiento actual. |
| **Documentación** | Actualizar “Deploying → Environment variables”. | Añadir ejemplos y advertencias. |
| **Tests** | Unit/Integration tests para ambos modos y para la migración de proyectos existentes. | Garantizar que no haya regresiones. |

### 4.3 Detalle de la implementación  

#### 4.3.1 Base de datos  

```sql
-- Nueva tabla pivot
CREATE TABLE service_environment_variables (
    service_id BIGINT UNSIGNED NOT NULL,
    env_var_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (service_id, env_var_id),
    CONSTRAINT fk_service FOREIGN KEY (service_id) REFERENCES services(id) ON DELETE CASCADE,
    CONSTRAINT fk_env_var FOREIGN KEY (env_var_id) REFERENCES environment_variables(id) ON DELETE CASCADE
);
```

- **Modelo Eloquent** (`Service.php`):  
  ```php
  public function envVariables()
  {
      return $this->belongsToMany(EnvironmentVariable::class,
          'service_environment_variables',
          'service_id',
          'env_var_id');
  }
  ```

- **Migración**: incluir en `2026_11_05_000000_create_service_environment_variables_table.php`.

#### 4.3.2 Backend – Endpoint  

```php
// Routes/api.php
Route::post('projects/{project}/compose', [ComposeController::class, 'store'])
     ->middleware('auth');

// Controller
public function store(Request $request, Project $project)
{
    $data = $request->validate([
        'globalEnv' => 'array',
        'serviceEnv' => 'array', // [{service_id: [env_key,...]}, …]
        'mode' => 'in:strict,legacy',
    ]);

    // Persist global env (unchanged)
    $project->environmentVariables()->sync(
        collect($data['globalEnv'])->pluck('key')->toArray()
    );

    // Persist per‑service mapping
    foreach ($data['serviceEnv'] as $serviceId => $keys) {
        $service = $project->services()->findOrFail($serviceId);
        $envIds = EnvironmentVariable::whereIn('key', $keys)->pluck('id');
        $service->envVariables()->sync($envIds);
    }

    $project->settings->compose_env_mode = $data['mode'] ?? 'strict';
    $project->settings->save();

    // Regenerar compose file
    (new ComposeBuilder($project))->generate();

    return response()->json(['status' => 'ok']);
}
```

#### 4.3.3 Generador de Compose (`ComposeBuilder.php`)

```php
class ComposeBuilder
{
    protected Project $project;
    protected array $yaml = [];

    public function __construct(Project $project)
    {
        $this->project = $project;
    }

    public function generate(): string
    {
        $this->yaml['version'] = '3.9';
        $this->yaml['services'] = [];

        foreach ($this->project->services as $service) {
            $svc = $this->baseServiceDefinition($service);
            $svc['environment'] = $this->environmentForService($service);
            $this->yaml['services'][$service->slug] = $svc;
        }

        $output = Yaml::dump($this->yaml, 4, 2);
        Storage::disk('compose')->put("{$this->project->uuid}.yml", $output);
        return $output;
    }

    private function environmentForService(Service $service): array
    {
        // Modo legacy → todas las variables globales
        if ($this->project->settings->compose_env_mode === 'legacy') {
            return $this->project->environmentVariables()
                ->pluck('value', 'key')
                ->toArray();
        }

        // Modo strict → solo variables asignadas al servicio
        $env = [];

        // 1️⃣ Variables globales explícitamente asignadas al servicio
        $globalKeys = $service->envVariables()
            ->where('scope', 'global')
            ->pluck('key')
            ->toArray();

        foreach ($globalKeys as $key) {
            $env[$key] = $this->project->environmentVariables()
                ->where('key', $key)
                ->value('value');
        }

        // 2️⃣ Variables definidas directamente en el servicio (custom)
        foreach ($service->customEnv as $custom) { // relation customEnv -> ServiceEnvVariable (key/value)
            $env[$custom->key] = $custom->value;
        }

        return $env;
    }

    // … método baseServiceDefinition() sin cambios
}
```

- **Nota**: Se mantiene la generación de `env_file:` si el usuario lo define manualmente; el builder no sobrescribe esas secciones.

#### 4.3.4 UI – Cambios en el panel de edición de servicio  

1. **Nuevo tab “Entorno”** dentro del modal “Edit Service”.  
2. **Lista de variables globales** con checkboxes → al marcar, se añade a la tabla pivot.  
3. **Botón “Crear variable específica”** que abre un pequeño formulario `key/value` (se guarda como `customEnv`).  
4. **Indicador de modo** en la página del proyecto (toggle “Legacy env injection”).  

Código (Vue 3 + Tailwind):

```vue
<template>
  <div class="space-y-4">
    <h3 class="text-lg font-medium">Variables de entorno específicas</h3>

    <div v-for="env in globalVars" :key="env.key" class="flex items-center">
      <input type="checkbox"
             :id="`svc-${service.id}-env-${env.key}`"
             v-model="selectedGlobal"
             :value="env.key"
             class="mr-2"/>
      <label :for="`svc-${service.id}-env-${env.key}`">{{ env.key }}</label>
    </div>

    <button @click="showCreate = true"
            class="btn btn-primary mt-2">
      + Crear variable específica
    </button>

    <!-- Modal creación -->
    <Modal v-if="showCreate" @close="showCreate = false">
      <template #title>Crear variable para {{ service.name }}</template>
      <form @submit.prevent="addCustom">
        <input v-model="newKey" placeholder="KEY" class="input" required/>
        <input v-model="newValue" placeholder="VALUE" class="input" required/>
        <button type="submit" class="btn btn-success mt-2">Guardar</button>
      </form>
    </Modal>
  </div>
</template>

<script>
export default {
  props: ['service', 'globalVars'],
  data() {
    return {
      selectedGlobal: [],          // <-- será enviado al backend
      showCreate: false,
      newKey: '',
      newValue: '',
    };
  },
  watch: {
    selectedGlobal(val) {
      this.$emit('update-service-env', this.service.id, val);
    },
  },
  methods: {
    async addCustom() {
      await axios.post(`/api/services/${this.service.id}/custom-env`, {
        key: this.newKey,
        value: this.newValue,
      });
      this.$emit('refresh');
      this.showCreate = false;
      this.newKey = '';
      this.newValue = '';
    },
  },
};
</script>
```

#### 4.3.5 Modo de migración (legacy → strict)

- **Detección automática**: Si un proyecto tiene **más de 1 servicio** y **más de 0 variables** sin asignación explícita, el instalador muestra un banner:  
  > “Este proyecto está en modo *legacy*. Para mejorar la seguridad, cambie a *strict* y asigne variables a cada servicio.”  
- **Herramienta CLI** `coolify env:convert --project <id>` que:  
  1. Copia todas las variables globales a cada servicio (asignación implícita).  
  2. Cambia `compose_env_mode` a `strict`.  
  3. Genera un commit de auditoría.  

#### 4.3.6 Tests  

| Tipo | Caso de prueba | Expectativa |
|------|----------------|-------------|
| Unit | `ComposeBuilder::environmentForService()` en modo `strict` devuelve solo variables asignadas. | Array con keys esperadas. |
| Unit | En modo `legacy` devuelve todas las variables del proyecto. | Igual que comportamiento actual. |
| Integration | Crear proyecto → 2 servicios → asignar var `DB_PASSWORD` solo al **backend