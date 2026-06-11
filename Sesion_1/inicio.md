He creado una aplicación web completa que permite gestionar registros meteorológicos usando SQLite en el navegador, con todas las operaciones CRUD y almacenamiento persistente local.
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>Estación Meteorológica - Gestor de Clima con SQLite</title>
    <!-- SQL.js (SQLite para navegador) -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.3/sql-wasm.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: system-ui, 'Segoe UI', 'Roboto', 'Helvetica Neue', sans-serif;
        }

        body {
            background: linear-gradient(145deg, #e0eafc 0%, #cfdef3 100%);
            min-height: 100vh;
            padding: 2rem 1rem;
        }

        .container {
            max-width: 1400px;
            margin: 0 auto;
            background: rgba(255, 255, 255, 0.85);
            backdrop-filter: blur(2px);
            border-radius: 2.5rem;
            box-shadow: 0 20px 35px -12px rgba(0, 0, 0, 0.2);
            padding: 2rem 1.8rem;
            transition: all 0.2s ease;
        }

        h1 {
            font-size: 1.9rem;
            font-weight: 700;
            background: linear-gradient(135deg, #1f3b4c, #2c5a6e);
            background-clip: text;
            -webkit-background-clip: text;
            color: transparent;
            margin-bottom: 0.3rem;
            display: inline-block;
        }

        .subtitulo {
            color: #2c3e50;
            margin-bottom: 1.8rem;
            border-left: 4px solid #3498db;
            padding-left: 0.8rem;
            font-weight: 500;
        }

        /* Formulario */
        .form-card {
            background: white;
            border-radius: 1.8rem;
            padding: 1.6rem 1.8rem;
            margin-bottom: 2.5rem;
            box-shadow: 0 8px 18px rgba(0, 0, 0, 0.05);
            border: 1px solid rgba(52, 152, 219, 0.2);
            transition: all 0.2s;
        }

        .form-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
            gap: 1.2rem;
            align-items: end;
        }

        .campo {
            display: flex;
            flex-direction: column;
            gap: 0.4rem;
        }

        .campo label {
            font-weight: 600;
            color: #1e4663;
            font-size: 0.85rem;
            letter-spacing: 0.3px;
        }

        .campo input, .campo select {
            padding: 0.7rem 0.9rem;
            border-radius: 1rem;
            border: 1.5px solid #cbd5e1;
            background: #fefefe;
            font-size: 0.9rem;
            transition: 0.2s;
            outline: none;
        }

        .campo input:focus, .campo select:focus {
            border-color: #3498db;
            box-shadow: 0 0 0 3px rgba(52,152,219,0.2);
        }

        .botones-form {
            display: flex;
            gap: 0.8rem;
            align-items: center;
            margin-top: 1.6rem;
            flex-wrap: wrap;
        }

        button {
            padding: 0.7rem 1.5rem;
            border-radius: 2rem;
            font-weight: 600;
            border: none;
            cursor: pointer;
            transition: transform 0.1s, background 0.2s;
            font-size: 0.85rem;
        }

        button:active {
            transform: scale(0.97);
        }

        .btn-primary {
            background: #2c6e9e;
            color: white;
            box-shadow: 0 2px 6px rgba(0,0,0,0.1);
        }
        .btn-primary:hover {
            background: #1f557d;
        }

        .btn-secondary {
            background: #e2e8f0;
            color: #1f3a4b;
        }
        .btn-secondary:hover {
            background: #cbd5e1;
        }

        .btn-edit {
            background: #f1c40f;
            color: #2c3e2f;
        }
        .btn-edit:hover {
            background: #e0b50f;
        }
        .btn-delete {
            background: #e67e22;
            color: white;
        }
        .btn-delete:hover {
            background: #c0392b;
        }

        /* Tabla / listado */
        .listado-section {
            background: white;
            border-radius: 1.8rem;
            padding: 1.2rem 0.5rem 1.5rem 0.5rem;
            box-shadow: 0 8px 20px rgba(0, 0, 0, 0.05);
        }

        .listado-header {
            display: flex;
            justify-content: space-between;
            align-items: baseline;
            flex-wrap: wrap;
            padding: 0 1rem 1rem 1rem;
            border-bottom: 2px solid #eef2f8;
        }
        .badge-count {
            background: #2c6e9e20;
            padding: 0.3rem 0.8rem;
            border-radius: 40px;
            font-size: 0.8rem;
            font-weight: 600;
            color: #1f557d;
        }

        .tabla-responsive {
            overflow-x: auto;
            margin-top: 1rem;
            padding: 0 0.5rem;
        }

        table {
            width: 100%;
            border-collapse: collapse;
            font-size: 0.9rem;
        }

        th, td {
            text-align: left;
            padding: 1rem 0.8rem;
            border-bottom: 1px solid #e2edf7;
        }

        th {
            background-color: #f8fafc;
            color: #1e4663;
            font-weight: 700;
        }

        td {
            color: #1e2f3e;
            vertical-align: middle;
        }

        .acciones {
            display: flex;
            gap: 0.6rem;
            flex-wrap: wrap;
        }

        .acciones button {
            padding: 0.3rem 1rem;
            font-size: 0.75rem;
            border-radius: 30px;
        }

        .mensaje-vacio {
            text-align: center;
            padding: 2.5rem;
            color: #6c7a89;
            font-style: italic;
        }

        footer {
            margin-top: 2rem;
            text-align: center;
            font-size: 0.75rem;
            color: #3a6279;
        }

        @media (max-width: 680px) {
            .container {
                padding: 1rem;
            }
            th, td {
                padding: 0.7rem 0.4rem;
            }
            .botones-form button {
                width: 100%;
            }
        }
    </style>
</head>
<body>

<div class="container">
    <h1>🌦️ Sistema Meteorológico</h1>
    <div class="subtitulo">Registro climático con SQLite persistente (IndexedDB/localStorage)</div>

    <!-- Formulario superior (nuevo registro + modificación) -->
    <div class="form-card">
        <form id="weatherForm">
            <div class="form-grid">
                <div class="campo">
                    <label>📍 Localidad *</label>
                    <input type="text" id="localidad" placeholder="Ej: Madrid, Barcelona" required autocomplete="off">
                </div>
                <div class="campo">
                    <label>📅 Fecha *</label>
                    <input type="date" id="fecha" required>
                </div>
                <div class="campo">
                    <label>🌤️ Situación climatológica *</label>
                    <select id="situacion" required>
                        <option value="" disabled selected>-- Seleccionar --</option>
                        <option value="Soleado">☀️ Soleado</option>
                        <option value="Parcialmente nublado">⛅ Parcialmente nublado</option>
                        <option value="Nublado">☁️ Nublado</option>
                        <option value="Lluvioso">🌧️ Lluvioso</option>
                        <option value="Tormenta">⛈️ Tormenta</option>
                        <option value="Nevado">❄️ Nevado</option>
                        <option value="Brumoso">🌫️ Brumoso</option>
                    </select>
                </div>
                <div class="campo">
                    <label>🌡️ Temperatura (°C) *</label>
                    <input type="number" step="any" id="temperatura" placeholder="Ej: 22.5" required>
                </div>
            </div>
            <div class="botones-form">
                <button type="submit" id="btnSubmit" class="btn-primary">➕ Guardar registro</button>
                <button type="button" id="btnCancelar" class="btn-secondary">✖️ Cancelar edición</button>
            </div>
        </form>
    </div>

    <!-- Listado inferior con registros + opciones modificar/eliminar -->
    <div class="listado-section">
        <div class="listado-header">
            <h3>📋 Registros climatológicos</h3>
            <span class="badge-count" id="contadorRegistros">0 registros</span>
        </div>
        <div class="tabla-responsive">
            <table id="tablaRegistros">
                <thead>
                    <tr><th>Localidad</th><th>Fecha</th><th>Situación</th><th>Temperatura</th><th>Acciones</th></tr>
                </thead>
                <tbody id="tbodyRegistros">
                    <tr><td colspan="5" class="mensaje-vacio">Cargando datos meteorológicos...</td></tr>
                </tbody>
            </table>
        </div>
    </div>
    <footer>🗄️ Base de datos SQLite embebida (SQL.js) · Los cambios se guardan automáticamente en el navegador</footer>
</div>

<script>
    // ------------------ VARIABLES GLOBALES ------------------
    let db = null;              // Instancia de la base de datos SQLite
    let currentEditId = null;  // ID del registro que se está editando (null = nuevo)
    let SQL = null;            // Referencia a la librería SQL.js
    
    // Elementos del DOM
    const localidadInput = document.getElementById('localidad');
    const fechaInput = document.getElementById('fecha');
    const situacionSelect = document.getElementById('situacion');
    const temperaturaInput = document.getElementById('temperatura');
    const btnSubmit = document.getElementById('btnSubmit');
    const btnCancelar = document.getElementById('btnCancelar');
    const tbody = document.getElementById('tbodyRegistros');
    const contadorSpan = document.getElementById('contadorRegistros');

    // ------------------ FUNCIONES AUXILIARES DE BASE DE DATOS (SQLITE) ------------------
    // Inicializar o cargar DB desde localStorage (persistencia real)
    function getStoredDatabase() {
        const stored = localStorage.getItem('weatherDB_sqlite');
        if (stored && stored.length > 0) {
            try {
                const binaryArray = Uint8Array.from(atob(stored), c => c.charCodeAt(0));
                return new SQL.Database(binaryArray);
            } catch(e) {
                console.warn("Error al restaurar DB, se crea nueva", e);
                return new SQL.Database();
            }
        }
        return new SQL.Database();
    }

    // Guardar estado actual de la base de datos en localStorage (exportar binario)
    function persistDatabase() {
        if (!db) return;
        const exportData = db.export();  // Uint8Array
        let binaryString = '';
        for (let i = 0; i < exportData.length; i++) {
            binaryString += String.fromCharCode(exportData[i]);
        }
        const base64 = btoa(binaryString);
        localStorage.setItem('weatherDB_sqlite', base64);
    }

    // Crear tabla si no existe
    function createTableIfNeeded() {
        const sqlCreate = `
            CREATE TABLE IF NOT EXISTS weather_records (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                localidad TEXT NOT NULL,
                fecha TEXT NOT NULL,
                situacion TEXT NOT NULL,
                temperatura REAL NOT NULL
            );
        `;
        db.run(sqlCreate);
    }

    // Insertar registro nuevo
    function insertRecord(localidad, fecha, situacion, temperatura) {
        const stmt = db.prepare("INSERT INTO weather_records (localidad, fecha, situacion, temperatura) VALUES (?, ?, ?, ?)");
        stmt.run([localidad, fecha, situacion, temperatura]);
        stmt.free();
        persistDatabase();
    }

    // Actualizar registro existente
    function updateRecord(id, localidad, fecha, situacion, temperatura) {
        const stmt = db.prepare("UPDATE weather_records SET localidad = ?, fecha = ?, situacion = ?, temperatura = ? WHERE id = ?");
        stmt.run([localidad, fecha, situacion, temperatura, id]);
        stmt.free();
        persistDatabase();
    }

    // Eliminar registro por ID
    function deleteRecordById(id) {
        const stmt = db.prepare("DELETE FROM weather_records WHERE id = ?");
        stmt.run([id]);
        stmt.free();
        persistDatabase();
    }

    // Obtener todos los registros (ordenados por fecha descendente + id)
    function getAllRecords() {
        const result = db.exec("SELECT id, localidad, fecha, situacion, temperatura FROM weather_records ORDER BY fecha DESC, id DESC");
        if (result.length === 0) return [];
        const { columns, values } = result[0];
        const records = values.map(row => {
            let obj = {};
            columns.forEach((col, idx) => { obj[col] = row[idx]; });
            return obj;
        });
        return records;
    }
    
    // Contar registros
    function countRecords() {
        const result = db.exec("SELECT COUNT(*) as total FROM weather_records");
        if (result.length === 0) return 0;
        return result[0].values[0][0];
    }

    // Sembrar datos de ejemplo si la base está vacía (primera vez o después de limpiar)
    function seedInitialDataIfEmpty() {
        const total = countRecords();
        if (total === 0) {
            const hoy = new Date().toISOString().slice(0,10);
            const ayer = new Date(Date.now() - 86400000).toISOString().slice(0,10);
            const ejemplo1 = ["Madrid", hoy, "Soleado", 24.5];
            const ejemplo2 = ["Barcelona", ayer, "Nublado", 19.2];
            const ejemplo3 = ["Bilbao", hoy, "Lluvioso", 14.8];
            const stmt = db.prepare("INSERT INTO weather_records (localidad, fecha, situacion, temperatura) VALUES (?, ?, ?, ?)");
            stmt.run(ejemplo1);
            stmt.run(ejemplo2);
            stmt.run(ejemplo3);
            stmt.free();
            persistDatabase();
        }
    }
    
    // ------------------ RENDERIZAR TABLA (LISTADO) ------------------
    function renderTable() {
        const records = getAllRecords();
        const total = records.length;
        contadorSpan.innerText = `${total} registro${total !== 1 ? 's' : ''}`;
        
        if (total === 0) {
            tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">🌱 No hay registros aún. ¡Agrega tu primer dato climático!</td></tr>`;
            return;
        }
        
        tbody.innerHTML = '';
        records.forEach(record => {
            const row = document.createElement('tr');
            // localidad
            const tdLocalidad = document.createElement('td'); tdLocalidad.textContent = record.localidad;
            // fecha
            const tdFecha = document.createElement('td'); tdFecha.textContent = record.fecha;
            // situación
            const tdSituacion = document.createElement('td'); tdSituacion.textContent = record.situacion;
            // temperatura
            const tdTemp = document.createElement('td'); tdTemp.textContent = `${record.temperatura} °C`;
            // acciones
            const tdAcciones = document.createElement('td');
            tdAcciones.className = 'acciones';
            
            const btnEditar = document.createElement('button');
            btnEditar.textContent = '✏️ Editar';
            btnEditar.classList.add('btn-edit');
            btnEditar.addEventListener('click', (e) => {
                e.preventDefault();
                cargarDatosEnFormularioParaEditar(record.id, record.localidad, record.fecha, record.situacion, record.temperatura);
            });
            
            const btnEliminar = document.createElement('button');
            btnEliminar.textContent = '🗑️ Eliminar';
            btnEliminar.classList.add('btn-delete');
            btnEliminar.addEventListener('click', (e) => {
                e.preventDefault();
                if (confirm(`¿Eliminar el registro de ${record.localidad} el día ${record.fecha}?`)) {
                    deleteRecordById(record.id);
                    // Si estabamos editando ese mismo registro, resetear el formulario
                    if (currentEditId === record.id) {
                        resetFormularioYEdicion();
                    }
                    renderTable();      // refrescar vista
                }
            });
            
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
    
    // Cargar datos en el formulario para edición (modo modificar)
    function cargarDatosEnFormularioParaEditar(id, localidad, fecha, situacion, temperatura) {
        currentEditId = id;
        localidadInput.value = localidad;
        fechaInput.value = fecha;
        situacionSelect.value = situacion;
        temperaturaInput.value = temperatura;
        // Cambiar el botón de submit a modo "Actualizar"
        btnSubmit.textContent = '✏️ Actualizar registro';
        btnSubmit.classList.add('btn-primary');
        // Cambiar estilo visual (pequeño detalle)
        btnCancelar.style.display = 'inline-flex';
        // Scroll suave al formulario
        document.querySelector('.form-card').scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
    
    // Resetear formulario y salir del modo edición
    function resetFormularioYEdicion() {
        currentEditId = null;
        localidadInput.value = '';
        fechaInput.value = '';
        situacionSelect.value = '';
        temperaturaInput.value = '';
        btnSubmit.textContent = '➕ Guardar registro';
        btnCancelar.style.display = 'inline-flex';  // siempre visible pero no hace nada si no hay edición
        // opcional: limpiar clases extra
    }
    
    // Validar y obtener datos del formulario
    function obtenerDatosFormulario() {
        const localidad = localidadInput.value.trim();
        const fecha = fechaInput.value;
        const situacion = situacionSelect.value;
        const temperaturaRaw = temperaturaInput.value.trim();
        
        if (!localidad) {
            alert("❌ La localidad es obligatoria.");
            return null;
        }
        if (!fecha) {
            alert("❌ Selecciona una fecha válida.");
            return null;
        }
        if (!situacion) {
            alert("❌ Elige una situación climatológica.");
            return null;
        }
        if (temperaturaRaw === "") {
            alert("❌ Ingresa la temperatura.");
            return null;
        }
        const temperatura = parseFloat(temperaturaRaw);
        if (isNaN(temperatura)) {
            alert("❌ La temperatura debe ser un número válido (ej: 22.5).");
            return null;
        }
        return { localidad, fecha, situacion, temperatura };
    }
    
    // Acción principal: guardar (insert o update según currentEditId)
    function guardarRegistro(event) {
        event.preventDefault();
        if (!db) {
            alert("Base de datos no inicializada aún. Espera un momento.");
            return;
        }
        
        const datos = obtenerDatosFormulario();
        if (!datos) return;
        
        const { localidad, fecha, situacion, temperatura } = datos;
        
        if (currentEditId !== null) {
            // Modo edición: actualizar registro existente
            updateRecord(currentEditId, localidad, fecha, situacion, temperatura);
            resetFormularioYEdicion();
        } else {
            // Modo nuevo registro
            insertRecord(localidad, fecha, situacion, temperatura);
            resetFormularioYEdicion();
        }
        // Refrescar el listado
        renderTable();
    }
    
    // Cancelar edición (reset)
    function cancelarEdicion() {
        if (currentEditId !== null) {
            resetFormularioYEdicion();
            // Opcional mensaje
        } else {
            // si no hay edición activa, solo limpiar campos
            localidadInput.value = '';
            fechaInput.value = '';
            situacionSelect.value = '';
            temperaturaInput.value = '';
        }
        currentEditId = null;
        btnSubmit.textContent = '➕ Guardar registro';
    }
    
    // ------------------ INICIALIZAR SQLITE + CARGAR PERSISTENCIA ------------------
    async function inicializarBaseDeDatos() {
        // Cargar SQL.js (wasm)
        return new Promise((resolve, reject) => {
            if (window.initSqlJs) {
                window.initSqlJs({
                    locateFile: filename => `https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.10.3/${filename}`
                }).then(sqlModule => {
                    SQL = sqlModule;
                    // Recuperar o crear DB
                    db = getStoredDatabase();
                    createTableIfNeeded();
                    seedInitialDataIfEmpty();   // solo si vacía mete datos demo
                    persistDatabase();           // asegurar sync inicial
                    resolve();
                }).catch(reject);
            } else {
                reject(new Error("initSqlJs no está disponible"));
            }
        });
    }
    
    // ------------------ EVENT LISTENERS Y ARRANQUE ------------------
    document.addEventListener('DOMContentLoaded', async () => {
        try {
            await inicializarBaseDeDatos();
            // Render inicial
            renderTable();
            // Asignar eventos del formulario
            document.getElementById('weatherForm').addEventListener('submit', guardarRegistro);
            btnCancelar.addEventListener('click', cancelarEdicion);
            // Si el usuario pulsa cancelar en cualquier momento, resetea bien
            btnCancelar.style.display = 'inline-flex';
            
            // Ajustar fecha por defecto para comodidad: poner hoy si el campo está vacío (opcional)
            if (!fechaInput.value) {
                const today = new Date().toISOString().slice(0,10);
                fechaInput.value = today;
            }
        } catch (error) {
            console.error("Error crítico inicializando SQLite:", error);
            tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">⚠️ Error al cargar la base de datos. Recarga la página o contacta soporte.</td></tr>`;
            alert("No se pudo inicializar SQLite. Verifica la conexión a internet o recarga.");
        }
    });
    
    // Extra: asegurar que antes de cerrar, la DB se persista (ya se hace en cada transacción)
    window.addEventListener('beforeunload', () => {
        if (db) persistDatabase();
    });
</script>
</body>
</html>
```
[Volver a la Sesión](prompts.md)
