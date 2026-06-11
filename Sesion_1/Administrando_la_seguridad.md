Tiene toda la razón. Un sistema con validación previa es mucho más seguro para una aplicación privada.

Para implementarlo, vamos a usar un modelo de "invitación". El administrador será el único que puede crear usuarios, y ellos solo podrán acceder si han recibido una invitación por correo. Le explico cómo funciona y, al final, le dejo el código listo para usar.

### ✨ Modelo de Invitación: ¿Cómo Funciona?

Este flujo es el estándar para aplicaciones privadas y se basa en estos pasos:

1.  **Deshabilitar los registros públicos**: En la configuración de Supabase, se desactiva la opción "Allow new users to sign up". Así, cualquiera que intente registrarse desde la página web obtendrá un error.
2.  **Administrador invita**: Usted, como administrador, utiliza una función secreta para invitar a nuevos usuarios.
3.  **Usuario recibe correo**: La persona invitada recibe un email.
4.  **Usuario establece su contraseña**: Al hacer clic en el enlace del correo, se le guiará para que defina su propia contraseña y complete su registro.

---

### ⚙️ Preparación en Supabase

Antes de usar el código, debe realizar estos cambios en el panel de Supabase.

1.  **Deshabilitar el registro público:**
    *   Vaya a **Authentication** → **Providers**.
    *   En la sección "Email", busque y **desactive** la opción **"Allow new users to sign up"**.

2.  **Obtener la "Service Role Key":**
    *   Vaya a **Project Settings** → **API**.
    *   Copie la **`service_role key`**. Esta clave **es muy sensible y no debe filtrarse**. La usaremos exclusivamente para la función de administrador que va a crear.

---

### 🧑‍💻 Panel de Administración y Código Actualizado

He modificado el código para separar las dos experiencias. Necesitará crear dos archivos HTML.

#### 1. El Panel de Control del Administrador (`admin.html`)

Esta página es solo para usted. Le permite enviar invitaciones por correo electrónico.

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Panel de Administración - Invitar Usuarios</title>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        /* Estilos básicos para el panel de admin */
        body { font-family: system-ui, sans-serif; background: #f0f4f8; padding: 2rem; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
        .container { max-width: 500px; background: white; padding: 2rem; border-radius: 1rem; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.1); }
        h1 { color: #1e3a5f; }
        input, button { width: 100%; padding: 0.75rem; margin-top: 0.5rem; margin-bottom: 1rem; border-radius: 0.5rem; border: 1px solid #ccc; }
        button { background-color: #2563eb; color: white; font-weight: bold; border: none; cursor: pointer; }
        button:active { transform: scale(0.98); }
        .message { margin-top: 1rem; padding: 0.75rem; border-radius: 0.5rem; text-align: center; }
        .success { background-color: #dcfce7; color: #166534; }
        .error { background-color: #fee2e2; color: #991b1b; }
        .warning { background-color: #fef9c3; color: #854d0e; }
    </style>
</head>
<body>
<div class="container">
    <h1>👑 Invitar Nuevo Usuario</h1>
    <p>Envíe una invitación por correo para que el usuario pueda registrarse.</p>
    <input type="email" id="inviteEmail" placeholder="correo@ejemplo.com" required>
    <button id="sendInviteBtn">Enviar Invitación</button>
    <div id="messageArea"></div>
    <hr style="margin: 2rem 0;">
    <p style="font-size: 0.8rem; color: gray;">⚠️ Recuerda: Los registros públicos están desactivados. Solo usuarios invitados pueden crear una cuenta.</p>
</div>

<script>
    // ========== CONFIGURACIÓN DE SUPABASE ==========
    const SUPABASE_URL = 'TU_SUPABASE_URL';
    // ¡ATENCIÓN! Usa la SERVICE_ROLE_KEY aquí, NUNCA en la aplicación principal.
    const SUPABASE_SERVICE_ROLE_KEY = 'TU_SERVICE_ROLE_KEY';
    
    const supabaseAdmin = window.supabase.createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);
    
    const sendInviteBtn = document.getElementById('sendInviteBtn');
    const emailInput = document.getElementById('inviteEmail');
    const messageArea = document.getElementById('messageArea');
    
    function showMessage(text, type) {
        messageArea.innerHTML = `<div class="message ${type}">${text}</div>`;
        setTimeout(() => { if(messageArea.innerHTML) messageArea.innerHTML = ''; }, 5000);
    }
    
    sendInviteBtn.addEventListener('click', async () => {
        const email = emailInput.value.trim();
        if (!email) {
            showMessage('Por favor, introduce un correo electrónico.', 'error');
            return;
        }
        
        sendInviteBtn.disabled = true;
        sendInviteBtn.textContent = 'Enviando...';
        messageArea.innerHTML = '';
        
        try {
            // Llamada a la API de administración para invitar a un usuario
            const { data, error } = await supabaseAdmin.auth.admin.inviteUserByEmail(email);
            
            if (error) throw error;
            
            showMessage(`✅ ¡Invitación enviada a ${email}! El usuario recibirá un correo para crear su cuenta.`, 'success');
            emailInput.value = '';
        } catch (error) {
            console.error(error);
            showMessage(`❌ Error al enviar la invitación: ${error.message}`, 'error');
        } finally {
            sendInviteBtn.disabled = false;
            sendInviteBtn.textContent = 'Enviar Invitación';
        }
    });
</script>
</body>
</html>
```

#### 2. La Aplicación Principal (`index.html`)

Es muy similar a la versión anterior, pero ahora el flujo de "Registro" (`Sign Up`) está adaptado para usuarios invitados. Al hacer clic en el enlace del correo de invitación, se les redirigirá a esta página para que establezcan su contraseña.

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Estación Meteorológica - Acceso Privado</title>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
    <style>
        /* ... (mantén los mismos estilos que en la versión anterior) ... */
        * { margin: 0; padding: 0; box-sizing: border-box; font-family: system-ui, 'Segoe UI', 'Roboto', sans-serif; }
        body { background: linear-gradient(135deg, #e0f2fe 0%, #bae6fd 100%); min-height: 100vh; padding: 2rem 1rem; }
        .container { max-width: 1300px; margin: 0 auto; background: rgba(255, 255, 255, 0.9); backdrop-filter: blur(2px); border-radius: 2rem; box-shadow: 0 20px 35px -12px rgba(0, 0, 0, 0.2); padding: 2rem; }
        h1 { font-size: 2rem; background: linear-gradient(135deg, #0f3b5c, #1e6f5c); background-clip: text; -webkit-background-clip: text; color: transparent; }
        .auth-container { max-width: 450px; margin: 2rem auto; background: white; border-radius: 2rem; padding: 2rem; box-shadow: 0 20px 30px -10px rgba(0,0,0,0.1); }
        .auth-tabs { display: flex; gap: 1rem; margin-bottom: 1.5rem; border-bottom: 2px solid #e2e8f0; }
        .auth-tab { padding: 0.5rem 1rem; background: none; border: none; font-size: 1.1rem; font-weight: 600; cursor: pointer; color: #64748b; transition: 0.2s; }
        .auth-tab.active { color: #2563eb; border-bottom: 2px solid #2563eb; margin-bottom: -2px; }
        .auth-form { display: flex; flex-direction: column; gap: 1rem; }
        .auth-form input { padding: 0.8rem; border-radius: 1rem; border: 1.5px solid #cbd5e1; font-size: 1rem; }
        .auth-form button { background: #2563eb; color: white; padding: 0.8rem; border-radius: 2rem; font-weight: bold; margin-top: 0.5rem; }
        .error-message { color: #dc2626; font-size: 0.9rem; text-align: center; margin-top: 0.5rem; }
        .app-container { display: none; }
        .user-info { display: flex; justify-content: flex-end; align-items: center; gap: 1rem; margin-bottom: 1rem; padding-bottom: 0.5rem; border-bottom: 1px solid #cbd5e1; }
        .logout-btn { background: #ef4444; color: white; padding: 0.4rem 1rem; border-radius: 2rem; font-size: 0.8rem; cursor: pointer; }
        .form-card { background: white; border-radius: 1.5rem; padding: 1.5rem; margin-bottom: 2rem; box-shadow: 0 4px 10px rgba(0,0,0,0.05); }
        .form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 1rem; }
        .campo { display: flex; flex-direction: column; gap: 0.3rem; }
        .campo label { font-weight: 600; color: #1e3a5f; font-size: 0.85rem; }
        .campo input, .campo select { padding: 0.7rem; border-radius: 1rem; border: 1.5px solid #cbd5e1; font-size: 0.9rem; }
        .botones-form { display: flex; gap: 1rem; margin-top: 1.5rem; flex-wrap: wrap; }
        button { padding: 0.6rem 1.3rem; border-radius: 2rem; font-weight: 600; border: none; cursor: pointer; transition: transform 0.1s, background 0.2s; }
        button:active { transform: scale(0.97); }
        .btn-primary { background: #2563eb; color: white; }
        .btn-secondary { background: #e2e8f0; color: #1e293b; }
        .btn-edit { background: #eab308; color: #1e293b; }
        .btn-delete { background: #dc2626; color: white; }
        .listado-section { background: white; border-radius: 1.5rem; padding: 1rem; overflow-x: auto; }
        table { width: 100%; border-collapse: collapse; }
        th, td { text-align: left; padding: 0.8rem; border-bottom: 1px solid #e2e8f0; }
        .acciones { display: flex; gap: 0.5rem; flex-wrap: wrap; }
        .mensaje-vacio { text-align: center; padding: 2rem; color: #64748b; }
        .badge { background: #dbeafe; border-radius: 30px; padding: 0.2rem 0.8rem; font-size: 0.8rem; }
        footer { text-align: center; margin-top: 1.5rem; font-size: 0.75rem; color: #1e3a5f; }
        @media (max-width: 640px) { .container { padding: 1rem; } th, td { padding: 0.5rem; font-size: 0.8rem; } }
    </style>
</head>
<body>
<div class="container">
    <!-- Pantalla de autenticación (Login y "Registro" para invitados) -->
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
        <div class="subtitulo">Datos compartidos en tiempo real con invitación previa</div>

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
        <footer>🔐 Acceso restringido · Solo usuarios invitados</footer>
    </div>
</div>

<script>
    // ========== CONFIGURACIÓN DE SUPABASE ==========
    const SUPABASE_URL = 'TU_SUPABASE_URL';
    const SUPABASE_ANON_KEY = 'TU_SUPABASE_ANON_KEY'; 

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

    let currentEditId = null;
    let currentMode = 'login';

    // Elementos de la app
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
                if (result.error) throw result.error;
            } else {
                // Registro para usuarios invitados: esto usará la invitación pendiente
                result = await supabaseClient.auth.signUp({ 
                    email, 
                    password,
                    options: {
                        emailRedirectTo: window.location.origin + window.location.pathname
                    }
                });
                if (result.error) throw result.error;
                if (result.data.user && !result.data.session) {
                    authError.textContent = '✅ Registro exitoso! Ahora inicia sesión con tu contraseña.';
                    setAuthMode('login');
                    authForm.reset();
                    return;
                }
            }
            if (result.error) throw result.error;
            if (result.data.session) {
                await onAuthSuccess(result.data.session.user);
            }
        } catch (error) {
            authError.textContent = error.message;
        }
    }

    async function onAuthSuccess(user) {
        userEmailSpan.textContent = user.email;
        authPanel.style.display = 'none';
        appPanel.style.display = 'block';
        initApp();
    }

    async function checkExistingSession() {
        const { data: { session } } = await supabaseClient.auth.getSession();
        if (session) {
            await onAuthSuccess(session.user);
        } else {
            authPanel.style.display = 'block';
            appPanel.style.display = 'none';
        }
    }

    async function logout() {
        await supabaseClient.auth.signOut();
        authPanel.style.display = 'block';
        appPanel.style.display = 'none';
        authForm.reset();
        setAuthMode('login');
        if (window.realtimeChannel) supabaseClient.removeChannel(window.realtimeChannel);
    }

    // ========== FUNCIONES DE LA APLICACIÓN PRINCIPAL ==========
    // (las mismas que en la versión anterior: cargarSituaciones, cargarRegistros, guardarRegistro, etc...)
    async function cargarSituaciones() {
        const { data, error } = await supabaseClient.from('situaciones').select('id, nombre').order('id');
        if (error) { console.error(error); return; }
        situacionSelect.innerHTML = '<option value="" disabled selected>-- Seleccionar --</option>';
        data.forEach(sit => {
            const option = document.createElement('option');
            option.value = sit.nombre;
            option.textContent = sit.nombre;
            situacionSelect.appendChild(option);
        });
    }

    async function cargarRegistros() {
        const { data, error } = await supabaseClient.from('registros').select('*').order('fecha', { ascending: false }).order('id', { ascending: false });
        if (error) { console.error(error); tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">Error al cargar</td></tr>`; return; }
        contadorSpan.innerText = `${data.length} registro${data.length !== 1 ? 's' : ''}`;
        if (data.length === 0) { tbody.innerHTML = `<tr><td colspan="5" class="mensaje-vacio">🌱 No hay datos.</td></tr>`; return; }
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
        document.querySelectorAll('.btn-edit').forEach(btn => btn.addEventListener('click', (e) => {
            editarRegistro(parseInt(btn.dataset.id), btn.dataset.localidad, btn.dataset.fecha, btn.dataset.situacion, parseFloat(btn.dataset.temperatura));
        }));
        document.querySelectorAll('.btn-delete').forEach(btn => btn.addEventListener('click', (e) => eliminarRegistro(parseInt(btn.dataset.id))));
    }

    function escapeHtml(str) { if (!str) return ''; return str.replace(/[&<>]/g, function(m) { if (m === '&') return '&amp;'; if (m === '<') return '&lt;'; if (m === '>') return '&gt;'; return m; }); }

    async function guardarRegistro(event) {
        event.preventDefault();
        const localidad = localidadInput.value.trim();
        const fecha = fechaInput.value;
        const situacion = situacionSelect.value;
        const temperatura = parseFloat(temperaturaInput.value);
        if (!localidad || !fecha || !situacion || isNaN(temperatura)) { alert('Completa todos los campos.'); return; }
        const nuevoRegistro = { localidad, fecha, situacion, temperatura };
        if (currentEditId !== null) {
            await supabaseClient.from('registros').update(nuevoRegistro).eq('id', currentEditId);
            resetFormulario();
        } else {
            await supabaseClient.from('registros').insert([nuevoRegistro]);
        }
        resetFormulario();
    }

    async function eliminarRegistro(id) {
        if (!confirm('¿Eliminar este registro?')) return;
        await supabaseClient.from('registros').delete().eq('id', id);
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
        window.realtimeChannel = supabaseClient.channel('cambios-registros').on('postgres_changes', { event: '*', schema: 'public', table: 'registros' }, () => cargarRegistros()).subscribe();
    }

    async function initApp() {
        localidadInput = document.getElementById('localidad');
        fechaInput = document.getElementById('fecha');
        situacionSelect = document.getElementById('situacion');
        temperaturaInput = document.getElementById('temperatura');
        btnSubmit = document.getElementById('btnSubmit');
        btnCancelar = document.getElementById('btnCancelar');
        tbody = document.getElementById('tbodyRegistros');
        contadorSpan = document.getElementById('contador');
        await cargarSituaciones();
        await cargarRegistros();
        if (!fechaInput.value) fechaInput.value = new Date().toISOString().slice(0,10);
        document.getElementById('weatherForm').addEventListener('submit', guardarRegistro);
        btnCancelar.addEventListener('click', resetFormulario);
        suscribirCambios();
    }

    // Event Listeners
    loginTab.addEventListener('click', () => setAuthMode('login'));
    registerTab.addEventListener('click', () => setAuthMode('register'));
    authForm.addEventListener('submit', handleAuthSubmit);
    logoutBtn.addEventListener('click', logout);

    checkExistingSession();
</script>
</body>
</html>
```

---

### ✅ Resumen de la Implementación

| Componente | Función |
| :--- | :--- |
| **`admin.html`** | Panel privado para que usted invite usuarios. Utiliza la `service_role_key` para crear invitaciones. |
| **`index.html`** | La aplicación pública. Los usuarios solo pueden iniciar sesión o, si han sido invitados, completar su registro. |

Espero que esta solución se ajuste perfectamente a lo que necesita. Si le surge cualquier duda durante la configuración, no dude en consultarme.
[Volver a la Sesión](prompts.md)