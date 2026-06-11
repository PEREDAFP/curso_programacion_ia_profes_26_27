El error se debe a que el script de Supabase ya define una variable global `supabase` (que contiene el objeto con el método `createClient`), y luego en tu código intentas redeclarar `const supabase` con el resultado de `window.supabase.createClient(...)`. En JavaScript, no se puede redeclarar una propiedad global no configurable.

**Solución:** Usa un nombre de variable diferente, por ejemplo `supabaseClient`, y llama a `window.supabase.createClient()` (o simplemente `supabase.createClient()` ya que `supabase` ya existe globalmente).

Aquí tienes el código **corregido**. Reemplaza la sección del script en tu `index2.html` desde la configuración de Supabase hasta la declaración de variables globales.

## Corrección específica

Cambia estas líneas:

```javascript
const SUPABASE_URL = 'TU_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY';

const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

Por:

```javascript
const SUPABASE_URL = 'TU_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY';

// Usar window.supabase (o la variable global supabase que ya existe) y asignar a otra constante
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

**Y luego reemplaza TODAS las referencias a `supabase` en el resto del código por `supabaseClient`** (por ejemplo, `await supabaseClient.from('registros').select(...)`).

---

## Código completo corregido (solo la parte del script, manteniendo el HTML igual)

Para que no tengas que buscar línea por línea, aquí te dejo el **script completo** ya corregido (copia y pega dentro de las etiquetas `<script>` de tu HTML, reemplazando todo el contenido anterior):

```javascript
// ================= CONFIGURACIÓN DE SUPABASE =================
const SUPABASE_URL = 'TU_SUPABASE_URL';      // Reemplázala por tu URL
const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY'; // Reemplázala por tu anon key

// CORRECCIÓN: usar un nombre diferente de variable (no redeclarar 'supabase')
const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// Variables globales
let currentEditId = null;
let situacionesMap = [];

// Elementos DOM
const localidadInput = document.getElementById('localidad');
const fechaInput = document.getElementById('fecha');
const situacionSelect = document.getElementById('situacion');
const temperaturaInput = document.getElementById('temperatura');
const btnSubmit = document.getElementById('btnSubmit');
const btnCancelar = document.getElementById('btnCancelar');
const tbody = document.getElementById('tbodyRegistros');
const contadorSpan = document.getElementById('contador');

// ============= 1. CARGAR OPCIONES DEL DESPLEGABLE =============
async function cargarSituaciones() {
    const { data, error } = await supabaseClient
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

// ============= 2. CARGAR REGISTROS =============
async function cargarRegistros() {
    const { data, error } = await supabaseClient
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
        const { error } = await supabaseClient
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
        const { error } = await supabaseClient
            .from('registros')
            .insert([nuevoRegistro]);
        
        if (error) {
            alert('Error al guardar: ' + error.message);
            return;
        }
    }
    
    await cargarRegistros();
    resetFormulario();
}

// ============= 4. ELIMINAR REGISTRO =============
async function eliminarRegistro(id) {
    const confirmar = confirm('¿Eliminar este registro permanentemente?');
    if (!confirmar) return;
    
    const { error } = await supabaseClient
        .from('registros')
        .delete()
        .eq('id', id);
    
    if (error) {
        alert('Error al eliminar: ' + error.message);
        return;
    }
    
    if (currentEditId === id) resetFormulario();
    await cargarRegistros();
}

// ============= 5. EDITAR =============
function editarRegistro(id, localidad, fecha, situacion, temperatura) {
    currentEditId = id;
    localidadInput.value = localidad;
    fechaInput.value = fecha;
    situacionSelect.value = situacion;
    temperaturaInput.value = temperatura;
    btnSubmit.textContent = '✏️ Actualizar registro';
    document.querySelector('.form-card').scrollIntoView({ behavior: 'smooth' });
}

// ============= 6. RESET =============
function resetFormulario() {
    currentEditId = null;
    localidadInput.value = '';
    fechaInput.value = new Date().toISOString().slice(0,10);
    if (situacionSelect.options.length > 1) situacionSelect.value = '';
    temperaturaInput.value = '';
    btnSubmit.textContent = '➕ Guardar registro';
}

// ============= INICIALIZACIÓN =============
async function init() {
    await cargarSituaciones();
    await cargarRegistros();
    if (!fechaInput.value) {
        fechaInput.value = new Date().toISOString().slice(0,10);
    }
    document.getElementById('weatherForm').addEventListener('submit', guardarRegistro);
    btnCancelar.addEventListener('click', resetFormulario);
}

init();
```

---

## ¿Por qué ocurría el error?

El script de Supabase (CDN) inyecta una variable global **no configurable** llamada `supabase` en `window`. Cuando tú intentas hacer `const supabase = ...`, JavaScript detecta que ya existe una propiedad `window.supabase` que no se puede reasignar (non-configurable) y lanza un error.

La solución es respetar el objeto global `supabase` y usarlo para crear el cliente, pero asignándolo a una variable **distinta**.

---

## Verifica también la configuración de Supabase

Asegúrate de que las **políticas RLS** estén creadas correctamente (como te indiqué en la respuesta anterior). Sin ellas, las operaciones fallarán silenciosamente o darán error 401.

Si después de corregir el error ves problemas de permisos, revisa la consola del navegador (F12) y confirma que las tablas `situaciones` y `registros` existen y que las políticas están activas.

---

Con este cambio, tu aplicación debería funcionar sin errores en GitHub Pages y en local. Si persiste algún problema, comparte el mensaje de error exacto.
[Volver a la Sesión](prompts.md)
