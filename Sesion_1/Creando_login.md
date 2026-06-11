Perfecto. Ahora implementaremos un **sistema de autenticación** para que solo usuarios registrados puedan acceder a la aplicación. Usaremos el servicio integrado de Supabase (email/contraseña) y modificaremos las políticas de seguridad para que **solo usuarios autenticados** puedan leer y escribir datos.

## Arquitectura de la solución

1. **Pantalla de login/registro** (dos modos: iniciar sesión o crear cuenta).
2. **Pantalla principal** con el formulario y listado (solo visible tras autenticarse).
3. **Políticas RLS** actualizadas para exigir `auth.role() = 'authenticated'`.
4. **El realtime seguirá funcionando** porque el cliente ya estará autenticado.

---

## Paso 1: Configurar Autenticación en Supabase

1. En tu panel de Supabase, ve a **Authentication** → **Providers**.
2. Asegúrate de que **Email** esté habilitado (debería estarlo por defecto).
3. Opcional: desactiva "Confirm email" para pruebas (en **Settings** → **Auth Settings** → "Enable email confirmations" → desmarca). En producción sería recomendable mantenerlo.
4. Anota la **URL** y **anon key** que ya usas (son las mismas).

---

## Paso 2: Actualizar políticas RLS (solo usuarios autenticados)

Ejecuta este SQL en el **SQL Editor** de Supabase:

```sql
-- Eliminar políticas antiguas (públicas) de la tabla registros
DROP POLICY IF EXISTS "Lectura pública registros" ON registros;
DROP POLICY IF EXISTS "Inserción pública registros" ON registros;
DROP POLICY IF EXISTS "Actualización pública registros" ON registros;
DROP POLICY IF EXISTS "Eliminación pública registros" ON registros;

-- Crear nuevas políticas solo para usuarios autenticados
CREATE POLICY "Lectura autenticada registros" ON registros
    FOR SELECT USING (auth.role() = 'authenticated');

CREATE POLICY "Inserción autenticada registros" ON registros
    FOR INSERT WITH CHECK (auth.role() = 'authenticated');

CREATE POLICY "Actualización autenticada registros" ON registros
    FOR UPDATE USING (auth.role() = 'authenticated');

CREATE POLICY "Eliminación autenticada registros" ON registros
    FOR DELETE USING (auth.role() = 'authenticated');

-- Mismo cambio para la tabla situaciones (si quieres que solo usuarios logueados puedan leer las opciones)
DROP POLICY IF EXISTS "Lectura pública situaciones" ON situaciones;
CREATE POLICY "Lectura autenticada situaciones" ON situaciones
    FOR SELECT USING (auth.role() = 'authenticated');
```

---

## Paso 3: Código HTML + JavaScript completo con autenticación

Aquí tienes la aplicación completa. Reemplaza todo el contenido de tu `index.html` (o el archivo que estés usando) con el siguiente código:

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Estación Meteorológica - Acceso Restringido</title>
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
        /* Pantalla de login */
        .auth-container {
            max-width: 450px;
            margin: 2rem auto;
            background: white;
            border-radius: 2rem;
            padding: 2rem;
            box-shadow: 0 20px 30px -10px rgba(0,0,0,0.1);
        }
        .auth-tabs {
            display: flex;
            gap: 1rem;
            margin-bottom: 1.5rem;
            border-bottom: 2px solid #e2e8f0;
        }
        .auth-tab {
            padding: 0.5rem 1rem;
            background: none;
            border: none;
            font-size: 1.1rem;
            font-weight: 600;
            cursor: pointer;
            color: #64748b;
            transition: 0.2s;
        }
        .auth-tab.active {
            color: #2563eb;
            border-bottom: 2px solid #2563eb;
            margin-bottom: -2px;
        }
        .auth-form {
            display: flex;
            flex-direction: column;
            gap: 1rem;
        }
        .auth-form input {
            padding: 0.8rem;
            border-radius: 1rem;
            border: 1.5px solid #cbd5e1;
            font-size: 1rem;
        }
        .auth-form button {
            background: #2563eb;
            color: white;
            padding: 0.8rem;
            border-radius: 2rem;
            font-weight: bold;
            margin-top: 0.5rem;
        }
        .error-message {
            color: #dc2626;
            font-size: 0.9rem;
            text-align: center;
            margin-top: 0.5rem;
        }
        /* Aplicación principal (inicialmente oculta) */
        .app-container {
            display: none;
        }
        .user-info {
            display: flex;
            justify-content: flex-end;
            align-items: center;
            gap: 1rem;
            margin-bottom: 1rem;
            padding-bottom: 0.5rem;
            border-bottom: 1px solid #cbd5e1;
        }
        .logout-btn {
            background: #ef4444;
            color: white;
            padding: 0.4rem 1rem;
            border-radius: 2rem;
            font-size: 0.8rem;
            cursor: pointer;
        }
        /* Estilos del formulario y tabla (igual que antes) */
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
        .btn-secondary {
            background: #e2e8f0;
            color: #1e293b;
        }
        .btn-edit {
            background: #eab308;
            color: #1e293b;
        }
        .btn-delete {
            background: #dc2626;
            color: white;
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
        }
        footer {
            text-align: center;
            margin-top: 1.5rem;
            font-size: 0.75rem;
            color: #1e3a5f;
        }
    </style>
</head>
<body>
<div class="container">
    <!-- Pantalla de autenticación -->
    <div id="authPanel" class="auth-container">
        <div class="auth-tabs">
            <button id="loginTab" class="auth-tab active">Iniciar sesión</button>
            <button id="registerTab" class="auth-tab">Registrarse</button>
        </div>
        <form id="authForm" class="auth-form">
            <input type="email" id="authEmail" placeholder="Correo electrónico" required>
            <input type="password" id="authPassword" placeholder="Contraseña" required>
            <button type="submit" id="authSubmitBtn">Iniciar sesión</button>
            <div id="authError" class="error-message"></div>
        </form>
    </div>

    <!-- Aplicación principal (solo visible tras login) -->
    <div id="appPanel" class="app-container">
        <div class="user-info">
            <span id="userEmail"></span>
            <button id="logoutBtn" class="logout-btn">Cerrar sesión</button>
        </div>
        <h1>🌦️ Clima Global · Acceso Autorizado</h1>
        <div class="subtitulo">Datos compartidos en tiempo real con control de acceso</div>

        <div class="form-card">
            <form id="weatherForm">
                <div class="form-grid">
                    <div class="campo"><label>📍 Localidad</label><input type="text" id="localidad" required></div>
                    <div class="campo"><label>📅 Fecha</label><input type="date" id="fecha" required></div>
                    <div class="campo"><label>🌤️ Situación</label><select id="situacion" required><option>Cargando...</option></select></div>
                    <div class="campo"><label>🌡️ Temperatura (°C)</label><input type="number" step="any" id="temperatura" required></div>
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
                <thead><tr><th>Localidad</th><th>Fecha</th><th>Situación</th><th>Temperatura</th><th>Acciones</th></tr></thead>
                <tbody id="tbodyRegistros"><tr><td colspan="5" class="mensaje-vacio">Conectando...</td></tr></tbody>
            </table>
        </div>
        <footer>🔐 Solo usuarios autorizados · Datos en PostgreSQL (Supabase)</footer>
    </div>
</div>

<script>
    // ========== CONFIGURACIÓN SUPABASE ==========
    const SUPABASE_URL = 'TU_SUPABASE_URL';          // Reemplázala
    const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY'; // Reemplázala

    const supabaseClient = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

    // Elementos UI
    const authPanel = document.getElementById('authPanel');
    const appPanel = document.getElementById('appPanel');
    const authForm = document.getElementById('authForm');
    const authEmail = document.getElementById('authEmail');
    const authPassword = document.getElementById('authPassword');
    const authSubmitBtn = document.getElementById('authSubmitBtn');
    const authError = document.getElementById('authError');
    const loginTab = document.getElementById('loginTab');
    const registerTab = document.getElementById('registerTab');
    const userEmailSpan = document.getElementById('userEmail');
    const logoutBtn = document.getElementById('logoutBtn');

    // Variables de la app
    let currentEditId = null;
    let currentMode = 'login'; // 'login' o 'register'

    // Elementos de la app principal (se inicializarán después del login)
    let localidadInput, fechaInput, situacionSelect, temperaturaInput, btnSubmit, btnCancelar, tbody, contadorSpan;

    // ========== FUNCIONES DE AUTENTICACIÓN ==========
    function setAuthMode(mode) {
        currentMode = mode;
        if (mode === 'login') {
            authSubmitBtn.textContent = 'Iniciar sesión';
            loginTab.classList.add('active');
            registerTab.classList.remove('active');
        } else {
            authSubmitBtn.textContent = 'Registrarse';
            registerTab.classList.add('active');
            loginTab.classList.remove('active');
        }
        authError.textContent = '';
    }

    async function handleAuthSubmit(e) {
        e.preventDefault();
        const email = authEmail.value.trim();
        const password = authPassword.value.trim();
        if (!email || !password) {
            authError.textContent = 'Por favor completa ambos campos.';
            return;
        }
        authError.textContent = '';
        
        try {
            let result;
            if (currentMode === 'login') {
                result = await supabaseClient.auth.signInWithPassword({ email, password });
            } else {
                result = await supabaseClient.auth.signUp({ email, password });
                if (result.error) throw result.error;
                // Si registro exitoso, automáticamente iniciar sesión? Supabase no inicia sesión tras registro hasta confirmar email.
                // Para simplificar, mostramos mensaje y cambiamos a login.
                if (!result.error && result.data.user && !result.data.session) {
                    authError.textContent = 'Registro exitoso. Ahora inicia sesión.';
                    setAuthMode('login');
                    authForm.reset();
                    return;
                }
            }
            if (result.error) throw result.error;
            
            // Si llegamos aquí, hay sesión activa (login o registro con confirmación automática)
            if (result.data.session) {
                // Mostrar aplicación
                await onAuthSuccess(result.data.session.user);
            }
        } catch (error) {
            authError.textContent = error.message;
        }
    }

    async function onAuthSuccess(user) {
        // Guardar la sesión ya está en supabaseClient
        userEmailSpan.textContent = user.email;
        authPanel.style.display = 'none';
        appPanel.style.display = 'block';
        // Inicializar la aplicación (cargar datos, suscripciones)
        initApp();
    }

    async function checkExistingSession() {
        const { data: { session } } = await supabaseClient.auth.getSession();
        if (session) {
            await onAuthSuccess(session.user);
        } else {
            // Mostrar panel de auth
            authPanel.style.display = 'block';
            appPanel.style.display = 'none';
        }
    }

    async function logout() {
        await supabaseClient.auth.signOut();
        // Resetear UI
        authPanel.style.display = 'block';
        appPanel.style.display = 'none';
        authForm.reset();
        setAuthMode('login');
        // Limpiar cualquier suscripción activa (opcional)
        if (window.realtimeChannel) {
            supabaseClient.removeChannel(window.realtimeChannel);
        }
    }

    // ========== FUNCIONES DE LA APLICACIÓN PRINCIPAL (CRUD + REALTIME) ==========
    async function cargarSituaciones() {
        const { data, error } = await supabaseClient
            .from('situaciones')
            .select('id, nombre')
            .order('id');
        if (error) {
            console.error('Error cargando situaciones:', error);
            situacionSelect.innerHTML = '<option>Error</option>';
            return;
        }
        situacionSelect.innerHTML = '<option value="" disabled selected>-- Seleccionar --</option>';
        data.forEach(sit => {
            const option = document.createElement('option');
            option.value = sit.nombre;
            option.textContent = sit.nombre;
            situacionSelect.appendChild(option);
        });
    }

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
            tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">🌱 No hay datos. Agrega tu primer registro.</td></tr>`;
            return;
        }
        tbody.innerHTML = '';
        data.forEach(reg => {
            const row = document.createElement('tr');
            row.innerHTML = `
                <td>${escapeHtml(reg.localidad)}</td>
                <td>${reg.fecha}</td>
                <td>${escapeHtml(reg.situacion)}</td>
                <td>${reg.temperatura} °C</td>
                <td class="acciones">
                    <button class="btn-edit" data-id="${reg.id}" data-localidad="${escapeHtml(reg.localidad)}" data-fecha="${reg.fecha}" data-situacion="${escapeHtml(reg.situacion)}" data-temperatura="${reg.temperatura}">✏️ Editar</button>
                    <button class="btn-delete" data-id="${reg.id}">🗑️ Eliminar</button>
                </td>
            `;
            tbody.appendChild(row);
        });
        // Añadir event listeners a botones dinámicos
        document.querySelectorAll('.btn-edit').forEach(btn => {
            btn.addEventListener('click', (e) => {
                const id = parseInt(btn.dataset.id);
                const localidad = btn.dataset.localidad;
                const fecha = btn.dataset.fecha;
                const situacion = btn.dataset.situacion;
                const temperatura = parseFloat(btn.dataset.temperatura);
                editarRegistro(id, localidad, fecha, situacion, temperatura);
            });
        });
        document.querySelectorAll('.btn-delete').forEach(btn => {
            btn.addEventListener('click', (e) => eliminarRegistro(parseInt(btn.dataset.id)));
        });
    }

    function escapeHtml(str) {
        if (!str) return '';
        return str.replace(/[&<>]/g, function(m) {
            if (m === '&') return '&amp;';
            if (m === '<') return '&lt;';
            if (m === '>') return '&gt;';
            return m;
        });
    }

    async function guardarRegistro(event) {
        event.preventDefault();
        const localidad = localidadInput.value.trim();
        const fecha = fechaInput.value;
        const situacion = situacionSelect.value;
        const temperatura = parseFloat(temperaturaInput.value);
        if (!localidad || !fecha || !situacion || isNaN(temperatura)) {
            alert('Completa todos los campos correctamente.');
            return;
        }
        const nuevoRegistro = { localidad, fecha, situacion, temperatura };
        if (currentEditId !== null) {
            const { error } = await supabaseClient.from('registros').update(nuevoRegistro).eq('id', currentEditId);
            if (error) alert('Error al actualizar: ' + error.message);
            resetFormulario();
        } else {
            const { error } = await supabaseClient.from('registros').insert([nuevoRegistro]);
            if (error) alert('Error al guardar: ' + error.message);
        }
        // El realtime refrescará la tabla
        resetFormulario();
    }

    async function eliminarRegistro(id) {
        if (!confirm('¿Eliminar este registro?')) return;
        const { error } = await supabaseClient.from('registros').delete().eq('id', id);
        if (error) alert('Error al eliminar: ' + error.message);
        if (currentEditId === id) resetFormulario();
    }

    function editarRegistro(id, localidad, fecha, situacion, temperatura) {
        currentEditId = id;
        localidadInput.value = localidad;
        fechaInput.value = fecha;
        situacionSelect.value = situacion;
        temperaturaInput.value = temperatura;
        btnSubmit.textContent = '✏️ Actualizar registro';
        document.querySelector('.form-card').scrollIntoView({ behavior: 'smooth' });
    }

    function resetFormulario() {
        currentEditId = null;
        localidadInput.value = '';
        fechaInput.value = new Date().toISOString().slice(0,10);
        if (situacionSelect.options.length > 1) situacionSelect.value = '';
        temperaturaInput.value = '';
        btnSubmit.textContent = '➕ Guardar registro';
    }

    function suscribirCambios() {
        // Guardar referencia del canal para poder limpiar después si es necesario
        window.realtimeChannel = supabaseClient
            .channel('cambios-registros')
            .on('postgres_changes', { event: '*', schema: 'public', table: 'registros' }, (payload) => {
                console.log('Cambio en tiempo real:', payload);
                cargarRegistros();
                if (payload.eventType === 'DELETE' && currentEditId === payload.old.id) resetFormulario();
                if (payload.eventType === 'UPDATE' && currentEditId === payload.old.id) resetFormulario();
            })
            .subscribe((status) => {
                console.log('Suscripción realtime (registros):', status);
            });
    }

    async function initApp() {
        // Obtener referencias a elementos DOM de la app principal
        localidadInput = document.getElementById('localidad');
        fechaInput = document.getElementById('fecha');
        situacionSelect = document.getElementById('situacion');
        temperaturaInput = document.getElementById('temperatura');
        btnSubmit = document.getElementById('btnSubmit');
        btnCancelar = document.getElementById('btnCancelar');
        tbody = document.getElementById('tbodyRegistros');
        contadorSpan = document.getElementById('contador');
        
        // Cargar datos iniciales
        await cargarSituaciones();
        await cargarRegistros();
        if (!fechaInput.value) fechaInput.value = new Date().toISOString().slice(0,10);
        
        // Eventos formulario
        document.getElementById('weatherForm').addEventListener('submit', guardarRegistro);
        btnCancelar.addEventListener('click', resetFormulario);
        
        // Activar realtime
        suscribirCambios();
    }

    // ========== EVENTOS Y ARRANQUE ==========
    loginTab.addEventListener('click', () => setAuthMode('login'));
    registerTab.addEventListener('click', () => setAuthMode('register'));
    authForm.addEventListener('submit', handleAuthSubmit);
    logoutBtn.addEventListener('click', logout);

    // Iniciar: comprobar si ya hay sesión activa
    checkExistingSession();
</script>
</body>
</html>
```

---

## Paso 4: Explicación de los cambios

### Autenticación
- Se añadió un panel de inicio de sesión/registro con pestañas.
- Usamos `supabaseClient.auth.signInWithPassword` y `.signUp`.
- Tras login exitoso, se oculta el panel de autenticación y se muestra la aplicación principal.
- El botón "Cerrar sesión" llama a `supabaseClient.auth.signOut()` y vuelve a mostrar el login.

### Seguridad en la base de datos
- Las nuevas políticas RLS permiten solo **usuarios autenticados** (cualquier usuario con cuenta) para todas las operaciones sobre `registros` y `situaciones`. 
- Si quieres que **cada usuario vea solo sus propios registros**, necesitarías añadir una columna `user_id` y usar `auth.uid() = user_id`. Avísame si quieres esa variante.

### Realtime
- La suscripción a cambios sigue funcionando, pero ahora con el cliente autenticado. Todo cambio hecho por cualquier usuario autenticado se reflejará en tiempo real en todas las pantallas activas.

### Experiencia de usuario
- Si un usuario no autenticado intenta acceder directamente a la URL, verá solo el formulario de login.
- Una vez autenticado, puede usar toda la funcionalidad CRUD y ver los datos compartidos con otros usuarios autorizados.

---

## Paso 5: Probar la aplicación

1. Sustituye `TU_SUPABASE_URL` y `TU_SUPABASE_ANON_KEY` en el código.
2. Abre la página en tu navegador.
3. Regístrate con un email y contraseña (por ejemplo, `prueba@ejemplo.com` / `12345678`).  
   *Nota: si tienes confirmación de email activada, deberás verificar el enlace antes de poder iniciar sesión. Para pruebas rápidas, desactívala en Authentication → Settings.*
4. Una vez registrado e iniciada sesión, verás la aplicación completa.
5. Abre la misma página en otro navegador (o modo incógnito) y regístrate con otro usuario. Ambos verán y modificarán los mismos datos.

---

## Posibles extensiones (si las necesitas)

- **Recuperación de contraseña**: se puede añadir fácilmente con `supabaseClient.auth.resetPasswordForEmail(email)`.
- **Datos por usuario**: añadir columna `user_id` y filtrar en consultas y políticas RLS.
- **Proveedores sociales** (Google, GitHub): habilitarlos en Supabase y añadir botones.

Ahora tu aplicación es **privada y segura**, solo accesible para usuarios con cuenta autorizada. ¡Disfruta!
[Volver a la Sesión](prompts.md)