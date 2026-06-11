Excelente idea. Implementar **tiempo real** hará que cualquier cambio (inserción, modificación o eliminación) hecho por otro usuario se refleje automáticamente en la pantalla de todos los que estén viendo la aplicación. Es una de las características más potentes de Supabase.

## ¿Cómo funciona?

Supabase expone un sistema de **subscriptions** sobre los cambios en las tablas PostgreSQL. Tu aplicación se suscribe a eventos (`INSERT`, `UPDATE`, `DELETE`) en la tabla `registros`. Cuando algo cambia, se ejecuta un callback que vuelve a cargar la lista.

Además, puedes suscribirte a cambios en la tabla `situaciones` por si alguien agrega nuevas opciones desde el panel de Supabase (aunque es menos común).

---

## Código actualizado (con realtime)

Solo necesitas **añadir el bloque de suscripción** después de inicializar el cliente. El resto del código sigue igual.

Aquí tienes el script completo ya integrado con **realtime**. Reemplaza todo tu `<script>` por este:

```javascript
// ================= CONFIGURACIÓN SUPABASE =================
const SUPABASE_URL = 'TU_SUPABASE_URL';      // tu URL
const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY'; // tu anon key

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
        tbody.innerHTML = `<table><td colspan="5" class="mensaje-vacio">Error al cargar registros</td></tr>`;
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
        const { error } = await supabaseClient
            .from('registros')
            .insert([nuevoRegistro]);
        if (error) {
            alert('Error al guardar: ' + error.message);
            return;
        }
    }
    // No hace falta llamar a cargarRegistros() aquí, el realtime lo hará automáticamente
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
    // El realtime refrescará la tabla automáticamente
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

// ============= 7. SUSCRIPCIÓN A CAMBIOS EN TIEMPO REAL =============
function suscribirCambios() {
    // Escuchar todos los eventos (INSERT, UPDATE, DELETE) en la tabla 'registros'
    const channel = supabaseClient
        .channel('cambios-registros')
        .on(
            'postgres_changes',
            {
                event: '*',           // '*' significa todos los eventos
                schema: 'public',
                table: 'registros'
            },
            (payload) => {
                console.log('Cambio detectado:', payload);
                // Recargar la tabla para reflejar el cambio
                cargarRegistros();
                // Si el registro eliminado o modificado era el que estábamos editando,
                // limpiamos el formulario (evita inconsistencias)
                if (payload.eventType === 'DELETE' && currentEditId === payload.old.id) {
                    resetFormulario();
                }
                if (payload.eventType === 'UPDATE' && currentEditId === payload.old.id) {
                    // Opcional: recargar datos del formulario si estábamos editando ese mismo
                    // Pero es mejor resetear para evitar conflictos
                    resetFormulario();
                }
            }
        )
        .subscribe((status) => {
            console.log('Estado de suscripción realtime:', status);
        });
    
    // Opcional: también suscribirse a cambios en la tabla 'situaciones' (para que aparezcan nuevas opciones)
    supabaseClient
        .channel('cambios-situaciones')
        .on(
            'postgres_changes',
            { event: '*', schema: 'public', table: 'situaciones' },
            () => {
                console.log('Cambio en las situaciones, recargando desplegable');
                cargarSituaciones();
            }
        )
        .subscribe();
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
    
    // Activar realtime después de la carga inicial
    suscribirCambios();
}

init();
```

---

## ¿Qué cambios importantes se hicieron?

1. **Nueva función `suscribirCambios()`**  
   - Se crea un canal (`channel`) específico para la tabla `registros`.  
   - Se escucha cualquier evento (`*` = INSERT, UPDATE, DELETE).  
   - Cuando ocurre un cambio, se ejecuta `cargarRegistros()` automáticamente.  
   - Además, si el cambio fue un DELETE o UPDATE sobre el registro que el usuario está editando en ese momento, se resetea el formulario para evitar inconsistencias.

2. **Suscripción a cambios en `situaciones`** (opcional pero muy útil)  
   - Si alguien agrega una nueva situación climática directamente en la base de datos (por ejemplo, desde el panel de Supabase), el desplegable se actualizará sin recargar la página.

3. **Ya no necesitas llamar manualmente a `cargarRegistros()`** después de guardar o eliminar (aunque podrías dejarlo, no hace daño). El realtime se encarga de refrescar la vista.

---

## Probando la funcionalidad realtime

- Abre la misma página en **dos navegadores diferentes** (o en una ventana normal y otra incógnito).  
- En una, añade un registro. Verás que en la otra aparece **inmediatamente** (o con fracciones de segundo).  
- Elimina o modifica en una ventana; la otra se actualizará al instante.  
- Puedes abrir la página en tu móvil y en el ordenador simultáneamente; los cambios se sincronizan.

---

## Nota sobre los permisos en Supabase

Asegúrate de que la tabla `registros` tenga habilitada la **replicación** para poder suscribirte a cambios. Por defecto, en Supabase las tablas recién creadas **ya tienen la replicación activada** (realtime). Si por algún motivo no funcionara, ve a **Database → Replication** y activa la tabla `registros` en la sección "Replication".

---

## Ventajas de esta implementación

- **Sincronización automática** entre múltiples usuarios/dispositivos.  
- **Sin necesidad de recargar la página** para ver cambios ajenos.  
- **Mejor experiencia de usuario** (colaborativa en tiempo real).  
- **Mínimo impacto en el código** (solo añadir la suscripción).

Ahora tu estación meteorológica es una aplicación **realmente colaborativa** y en directo. ¡Disfrútala! 🚀
[Volver a la Sesión](prompts.md)
