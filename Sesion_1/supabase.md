Excelente decisión. **Supabase** te da una base de datos PostgreSQL real, con API automática, tiempo real, y un plan gratuito muy generoso (500 MB, suficiente para proyectos pequeños/medianos). Además, podrás alojar el frontend en **GitHub Pages** mientras Supabase corre en la nube, logrando persistencia global entre dispositivos.

A continuación te explico **paso a paso** cómo configurar Supabase y el código completo para tu aplicación.

---

## 1. Crear proyecto en Supabase

1. Ve a [supabase.com](https://supabase.com) y regístrate (puedes usar GitHub).
2. Crea un nuevo proyecto (elige nombre, contraseña de base de datos, región cercana).
3. Espera a que se cree (un par de minutos).
4. En el panel lateral, ve a **SQL Editor** para ejecutar las siguientes consultas.

---

## 2. Crear las tablas necesarias

Ejecuta este script SQL (lo puedes pegar en el editor SQL de Supabase y ejecutar):

```sql
-- Tabla de opciones para el desplegable (situación climatológica)
CREATE TABLE situaciones (
    id SERIAL PRIMARY KEY,
    nombre TEXT NOT NULL UNIQUE
);

-- Tabla principal de registros meteorológicos
CREATE TABLE registros (
    id SERIAL PRIMARY KEY,
    localidad TEXT NOT NULL,
    fecha DATE NOT NULL,
    situacion TEXT NOT NULL,
    temperatura DECIMAL(5,2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Insertar opciones iniciales para el desplegable
INSERT INTO situaciones (nombre) VALUES
('Soleado'),
('Parcialmente nublado'),
('Nublado'),
('Lluvioso'),
('Tormenta'),
('Nevado'),
('Brumoso');
```

También puedes crear índices para mejorar rendimiento (opcional):

```sql
CREATE INDEX idx_registros_fecha ON registros(fecha DESC);
```

---

## 3. Configurar políticas de seguridad (Row Level Security)

Queremos que cualquier persona (sin autenticación) pueda **leer y escribir** registros porque es una aplicación pública simple. Para eso:

- Ve a **Authentication** → **Settings** y desactiva “Enable Email Confirmations” si no necesitas registro (opcional).
- Ve a **Database** → **Policies** y crea políticas para ambas tablas.

O más fácil: ejecuta este SQL para permitir todas las operaciones anónimas:

```sql
-- Habilitar RLS (ya viene activado por defecto en tablas nuevas)
ALTER TABLE situaciones ENABLE ROW LEVEL SECURITY;
ALTER TABLE registros ENABLE ROW LEVEL SECURITY;

-- Políticas para situaciones (solo lectura)
CREATE POLICY "Lectura pública situaciones" ON situaciones
    FOR SELECT USING (true);

-- Políticas para registros (lectura, inserción, actualización, eliminación)
CREATE POLICY "Lectura pública registros" ON registros
    FOR SELECT USING (true);

CREATE POLICY "Inserción pública registros" ON registros
    FOR INSERT WITH CHECK (true);

CREATE POLICY "Actualización pública registros" ON registros
    FOR UPDATE USING (true) WITH CHECK (true);

CREATE POLICY "Eliminación pública registros" ON registros
    FOR DELETE USING (true);
```

> ⚠️ En un entorno real considerarías limitar permisos, pero para este demo es funcional.

---

## 4. Obtener las claves de conexión

En el panel de Supabase:

- Ve a **Project Settings** → **API**.
- Copia la **URL** del proyecto (algo como `https://xxxxxxxxxxxx.supabase.co`).
- Copia la **anon public key** (clave anónima, no la `service_role`).

Estas las usarás en tu código JavaScript.

---

## 5. Crear el frontend (HTML + JavaScript) para GitHub Pages

Crea un archivo `index.html` con el siguiente código completo. Reemplaza `TU_SUPABASE_URL` y `TU_SUPABASE_ANON_KEY` con los valores reales.

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Estación Meteorológica - Supabase</title>
    <!-- Supabase JS client -->
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: system-ui, 'Segoe UI', 'Roboto', sans-serif;
        }
        body {
            background: linear-gradient(135deg, #e0f2fe 0%, #bae6fd 100%);
            min-height: 100vh;
            padding: 2rem 1rem;
        }
        .container {
            max-width: 1300px;
            margin: 0 auto;
            background: rgba(255, 255, 255, 0.9);
            backdrop-filter: blur(2px);
            border-radius: 2rem;
            box-shadow: 0 20px 35px -12px rgba(0, 0, 0, 0.2);
            padding: 2rem;
        }
        h1 {
            font-size: 2rem;
            background: linear-gradient(135deg, #0f3b5c, #1e6f5c);
            background-clip: text;
            -webkit-background-clip: text;
            color: transparent;
        }
        .subtitulo {
            color: #2c3e50;
            border-left: 4px solid #3b82f6;
            padding-left: 1rem;
            margin: 0.5rem 0 1.5rem 0;
            font-weight: 500;
        }
        /* Formulario */
        .form-card {
            background: white;
            border-radius: 1.5rem;
            padding: 1.5rem;
            margin-bottom: 2rem;
            box-shadow: 0 4px 10px rgba(0,0,0,0.05);
        }
        .form-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
            gap: 1rem;
        }
        .campo {
            display: flex;
            flex-direction: column;
            gap: 0.3rem;
        }
        .campo label {
            font-weight: 600;
            color: #1e3a5f;
            font-size: 0.85rem;
        }
        .campo input, .campo select {
            padding: 0.7rem;
            border-radius: 1rem;
            border: 1.5px solid #cbd5e1;
            font-size: 0.9rem;
            outline: none;
            transition: 0.2s;
        }
        .campo input:focus, .campo select:focus {
            border-color: #3b82f6;
            box-shadow: 0 0 0 3px rgba(59,130,246,0.2);
        }
        .botones-form {
            display: flex;
            gap: 1rem;
            margin-top: 1.5rem;
            flex-wrap: wrap;
        }
        button {
            padding: 0.6rem 1.3rem;
            border-radius: 2rem;
            font-weight: 600;
            border: none;
            cursor: pointer;
            transition: transform 0.1s, background 0.2s;
        }
        button:active {
            transform: scale(0.97);
        }
        .btn-primary {
            background: #2563eb;
            color: white;
        }
        .btn-primary:hover {
            background: #1d4ed8;
        }
        .btn-secondary {
            background: #e2e8f0;
            color: #1e293b;
        }
        .btn-secondary:hover {
            background: #cbd5e1;
        }
        .btn-edit {
            background: #eab308;
            color: #1e293b;
        }
        .btn-edit:hover {
            background: #ca8a04;
            color: white;
        }
        .btn-delete {
            background: #dc2626;
            color: white;
        }
        .btn-delete:hover {
            background: #b91c1c;
        }
        .listado-section {
            background: white;
            border-radius: 1.5rem;
            padding: 1rem;
            overflow-x: auto;
        }
        table {
            width: 100%;
            border-collapse: collapse;
        }
        th, td {
            text-align: left;
            padding: 0.8rem;
            border-bottom: 1px solid #e2e8f0;
        }
        th {
            background-color: #f8fafc;
            color: #0f3b5c;
        }
        .acciones {
            display: flex;
            gap: 0.5rem;
            flex-wrap: wrap;
        }
        .mensaje-vacio {
            text-align: center;
            padding: 2rem;
            color: #64748b;
        }
        .badge {
            background: #dbeafe;
            border-radius: 30px;
            padding: 0.2rem 0.8rem;
            font-size: 0.8rem;
            font-weight: 600;
        }
        footer {
            text-align: center;
            margin-top: 1.5rem;
            font-size: 0.75rem;
            color: #1e3a5f;
        }
        @media (max-width: 640px) {
            .container { padding: 1rem; }
            th, td { padding: 0.5rem; font-size: 0.8rem; }
        }
    </style>
</head>
<body>
<div class="container">
    <h1>🌦️ Clima Global · Supabase</h1>
    <div class="subtitulo">Datos compartidos en tiempo real · PostgreSQL</div>

    <div class="form-card">
        <form id="weatherForm">
            <div class="form-grid">
                <div class="campo">
                    <label>📍 Localidad</label>
                    <input type="text" id="localidad" placeholder="Ej: Madrid" required>
                </div>
                <div class="campo">
                    <label>📅 Fecha</label>
                    <input type="date" id="fecha" required>
                </div>
                <div class="campo">
                    <label>🌤️ Situación</label>
                    <select id="situacion" required>
                        <option value="">Cargando opciones...</option>
                    </select>
                </div>
                <div class="campo">
                    <label>🌡️ Temperatura (°C)</label>
                    <input type="number" step="any" id="temperatura" placeholder="22.5" required>
                </div>
            </div>
            <div class="botones-form">
                <button type="submit" id="btnSubmit" class="btn-primary">➕ Guardar registro</button>
                <button type="button" id="btnCancelar" class="btn-secondary">✖️ Cancelar edición</button>
            </div>
        </form>
    </div>

    <div class="listado-section">
        <div style="display: flex; justify-content: space-between; margin-bottom: 1rem;">
            <h3>📋 Registros meteorológicos</h3>
            <span class="badge" id="contador">0 registros</span>
        </div>
        <table id="tablaRegistros">
            <thead>
                <tr><th>Localidad</th><th>Fecha</th><th>Situación</th><th>Temperatura</th><th>Acciones</th></tr>
            </thead>
            <tbody id="tbodyRegistros">
                <tr><td colspan="5" class="mensaje-vacio">Cargando datos desde Supabase...</td></tr>
            </tbody>
        </table>
    </div>
    <footer>🔐 Datos almacenados en PostgreSQL (Supabase) · Accesibles desde cualquier dispositivo</footer>
</div>

<script>
    // ================= CONFIGURACIÓN DE SUPABASE =================
    const SUPABASE_URL = 'TU_SUPABASE_URL';      // Reemplázala
    const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY'; // Reemplázala
    
    const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
    
    // Variables globales
    let currentEditId = null;
    let situacionesMap = [];  // Guardar { id, nombre }
    
    // Elementos DOM
    const localidadInput = document.getElementById('localidad');
    const fechaInput = document.getElementById('fecha');
    const situacionSelect = document.getElementById('situacion');
    const temperaturaInput = document.getElementById('temperatura');
    const btnSubmit = document.getElementById('btnSubmit');
    const btnCancelar = document.getElementById('btnCancelar');
    const tbody = document.getElementById('tbodyRegistros');
    const contadorSpan = document.getElementById('contador');
    
    // ============= 1. CARGAR OPCIONES DEL DESPLEGABLE (desde tabla situaciones) =============
    async function cargarSituaciones() {
        const { data, error } = await supabase
            .from('situaciones')
            .select('id, nombre')
            .order('id');
        
        if (error) {
            console.error('Error cargando situaciones:', error);
            situacionSelect.innerHTML = '<option>Error al cargar opciones</option>';
            return;
        }
        
        situacionesMap = data;
        situacionSelect.innerHTML = '<option value="" disabled selected>-- Seleccionar --</option>';
        data.forEach(sit => {
            const option = document.createElement('option');
            option.value = sit.nombre;
            option.textContent = sit.nombre;
            situacionSelect.appendChild(option);
        });
    }
    
    // ============= 2. CARGAR REGISTROS (CRUD principal) =============
    async function cargarRegistros() {
        const { data, error } = await supabase
            .from('registros')
            .select('*')
            .order('fecha', { ascending: false })
            .order('id', { ascending: false });
        
        if (error) {
            console.error(error);
            tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">Error al cargar registros</td></tr>`;
            contadorSpan.innerText = '0 registros';
            return;
        }
        
        contadorSpan.innerText = `${data.length} registro${data.length !== 1 ? 's' : ''}`;
        
        if (data.length === 0) {
            tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">🌱 Aún no hay datos. Agrega tu primer registro.</td></tr>`;
            return;
        }
        
        tbody.innerHTML = '';
        data.forEach(reg => {
            const row = document.createElement('tr');
            
            const tdLocalidad = document.createElement('td'); tdLocalidad.textContent = reg.localidad;
            const tdFecha = document.createElement('td'); tdFecha.textContent = reg.fecha;
            const tdSituacion = document.createElement('td'); tdSituacion.textContent = reg.situacion;
            const tdTemp = document.createElement('td'); tdTemp.textContent = `${reg.temperatura} °C`;
            
            const tdAcciones = document.createElement('td');
            tdAcciones.className = 'acciones';
            
            const btnEditar = document.createElement('button');
            btnEditar.textContent = '✏️ Editar';
            btnEditar.classList.add('btn-edit');
            btnEditar.onclick = () => editarRegistro(reg.id, reg.localidad, reg.fecha, reg.situacion, reg.temperatura);
            
            const btnEliminar = document.createElement('button');
            btnEliminar.textContent = '🗑️ Eliminar';
            btnEliminar.classList.add('btn-delete');
            btnEliminar.onclick = () => eliminarRegistro(reg.id);
            
            tdAcciones.appendChild(btnEditar);
            tdAcciones.appendChild(btnEliminar);
            
            row.appendChild(tdLocalidad);
            row.appendChild(tdFecha);
            row.appendChild(tdSituacion);
            row.appendChild(tdTemp);
            row.appendChild(tdAcciones);
            tbody.appendChild(row);
        });
    }
    
    // ============= 3. GUARDAR (INSERT / UPDATE) =============
    async function guardarRegistro(event) {
        event.preventDefault();
        
        const localidad = localidadInput.value.trim();
        const fecha = fechaInput.value;
        const situacion = situacionSelect.value;
        const temperaturaRaw = temperaturaInput.value.trim();
        
        if (!localidad || !fecha || !situacion || !temperaturaRaw) {
            alert('Por favor completa todos los campos.');
            return;
        }
        const temperatura = parseFloat(temperaturaRaw);
        if (isNaN(temperatura)) {
            alert('La temperatura debe ser un número válido.');
            return;
        }
        
        const nuevoRegistro = { localidad, fecha, situacion, temperatura };
        
        if (currentEditId !== null) {
            // UPDATE
            const { error } = await supabase
                .from('registros')
                .update(nuevoRegistro)
                .eq('id', currentEditId);
            
            if (error) {
                alert('Error al actualizar: ' + error.message);
                return;
            }
            resetFormulario();
        } else {
            // INSERT
            const { error } = await supabase
                .from('registros')
                .insert([nuevoRegistro]);
            
            if (error) {
                alert('Error al guardar: ' + error.message);
                return;
            }
        }
        
        // Refrescar lista y resetear
        await cargarRegistros();
        resetFormulario();
    }
    
    // ============= 4. ELIMINAR REGISTRO =============
    async function eliminarRegistro(id) {
        const confirmar = confirm('¿Eliminar este registro permanentemente?');
        if (!confirmar) return;
        
        const { error } = await supabase
            .from('registros')
            .delete()
            .eq('id', id);
        
        if (error) {
            alert('Error al eliminar: ' + error.message);
            return;
        }
        
        // Si estábamos editando ese mismo, cancelar edición
        if (currentEditId === id) resetFormulario();
        await cargarRegistros();
    }
    
    // ============= 5. EDITAR: cargar datos al formulario =============
    function editarRegistro(id, localidad, fecha, situacion, temperatura) {
        currentEditId = id;
        localidadInput.value = localidad;
        fechaInput.value = fecha;
        situacionSelect.value = situacion;
        temperaturaInput.value = temperatura;
        btnSubmit.textContent = '✏️ Actualizar registro';
        // scroll suave al formulario
        document.querySelector('.form-card').scrollIntoView({ behavior: 'smooth' });
    }
    
    // ============= 6. RESET FORMULARIO Y SALIR DE MODO EDICIÓN =============
    function resetFormulario() {
        currentEditId = null;
        localidadInput.value = '';
        fechaInput.value = new Date().toISOString().slice(0,10);  // fecha actual por defecto
        if (situacionSelect.options.length > 1) situacionSelect.value = '';
        temperaturaInput.value = '';
        btnSubmit.textContent = '➕ Guardar registro';
    }
    
    // ============= INICIALIZACIÓN =============
    async function init() {
        await cargarSituaciones();
        await cargarRegistros();
        
        // Setear fecha por defecto si está vacía
        if (!fechaInput.value) {
            fechaInput.value = new Date().toISOString().slice(0,10);
        }
        
        // Eventos del formulario
        document.getElementById('weatherForm').addEventListener('submit', guardarRegistro);
        btnCancelar.addEventListener('click', resetFormulario);
    }
    
    init();
</script>
</body>
</html>
```

---

## 6. Subir a GitHub Pages

1. Crea un repositorio en GitHub (ej. `clima-supabase`).
2. Guarda el archivo anterior como `index.html`.
3. Haz commit y push al repositorio.
4. Ve a **Settings → Pages** → Source = `main` (o `master`) y carpeta `/ (root)`.
5. En unos minutos estará disponible en `https://TU-USUARIO.github.io/clima-supabase`.

---

## 7. Verificar funcionamiento multi-dispositivo

- Abre la página desde tu móvil y ordenador.
- Verás los mismos registros en ambos.
- Los desplegables se cargan desde la tabla `situaciones` (si añades un nuevo clima directamente en Supabase, aparecerá automáticamente al recargar la página).

---

## 💡 Posibles mejoras futuras (opcionales)

- **Autenticación**: Si quieres que cada usuario vea solo sus datos, añade una columna `user_id` y políticas RLS basadas en `auth.uid()`.
- **Tiempo real**: Suscribirte a cambios con `supabase.channel()` para que la tabla se actualice en vivo sin recargar.
- **Paginación**: Para muchos registros, usa `.range()` en las consultas.

---

## Resumen

✅ Base de datos **PostgreSQL real** en la nube.  
✅ Datos compartidos entre todos los dispositivos que visiten tu GitHub Pages.  
✅ Los desplegables se obtienen desde la base de datos (tabla `situaciones`).  
✅ CRUD completo: crear, leer, actualizar, eliminar.  
✅ **Cero backend** que mantener; Supabase genera la API automática.

Ahora solo necesitas crear el proyecto en Supabase, copiar las claves en el código, y subir el HTML a GitHub. ¡Tu aplicación climática global estará funcionando en minutos!
[Volver a la Sesión](prompts.md)
