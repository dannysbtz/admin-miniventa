<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const apiBase = (import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000').trim()
const adminKey = ref('')
const connected = ref(false)
const loading = ref(false)
const users = ref([])
const deviceRequests = ref([])
const search = ref('')
const activeView = ref('customers')
const errorMessage = ref('')
const successMessage = ref('')
const editingUser = ref(null)
const accessStatus = ref('inactive')
const accessEndsAt = ref('')
const accessPaidAt = ref('')
const showCreateUser = ref(false)
const createUsername = ref('')
const createName = ref('')
const createPassword = ref('')
const createStatus = ref('active')
const createEndsAt = ref(toLocalDateTime(Date.now() + 30 * 24 * 60 * 60 * 1000))
const createPaidAt = ref(toLocalDateTime(Date.now()))
const clock = ref(Date.now())
let clockTimer

const normalizedApiBase = () => apiBase.replace(/\/+$/, '').replace(/\/api$/, '')
const visibleUsers = computed(() => {
  const query = search.value.trim().toLocaleLowerCase()
  if (!query) return users.value
  return users.value.filter((user) => `${user.name} ${user.username}`.toLocaleLowerCase().includes(query))
})
const activeCount = computed(() => users.value.filter((user) => accessState(user) === 'active').length)
const blockedCount = computed(() => users.value.filter((user) => accessState(user) !== 'active').length)

function accessState(user) {
  if (user.subscription.status !== 'active') return 'inactive'
  return new Date(user.subscription.endsAt).getTime() > clock.value ? 'active' : 'expired'
}

function formatDate(value) {
  if (!value) return 'Sin registro'
  return new Intl.DateTimeFormat('es-MX', { dateStyle: 'medium' }).format(new Date(value))
}

function toLocalDateTime(value) {
  const date = new Date(value)
  const local = new Date(date.getTime() - date.getTimezoneOffset() * 60000)
  return local.toISOString().slice(0, 16)
}

async function apiRequest(path, options = {}) {
  const headers = { 'X-Admin-Key': adminKey.value, ...options.headers }
  if (options.body) headers['Content-Type'] = 'application/json'
  let response
  try {
    response = await fetch(`${normalizedApiBase()}${path}`, { ...options, headers })
  } catch {
    throw new Error('No se pudo conectar al API configurado. Verifica VITE_API_BASE_URL y que el servidor esté disponible.')
  }
  const data = await response.json().catch(() => ({}))
  if (!response.ok) {
    const messages = {
      invalid_admin_key: 'La clave administrativa no es válida.',
      admin_not_configured: 'El API no tiene ADMIN_API_KEY configurada.',
      user_not_found: 'No se encontró el cliente.',
      username_exists: 'Ese nombre de usuario ya está registrado.',
      request_not_found: 'La solicitud ya no está pendiente.',
      invalid_request: 'Revisa los datos ingresados.',
    }
    throw new Error(messages[data.error] || `El API respondió con error (${response.status}).`)
  }
  return data
}

async function loadData() {
  const [userData, requestData] = await Promise.all([
    apiRequest('/api/admin/users'),
    apiRequest('/api/admin/device-requests'),
  ])
  users.value = userData.users
  deviceRequests.value = requestData.requests
}

async function connect() {
  errorMessage.value = ''
  successMessage.value = ''
  if (!adminKey.value) {
    errorMessage.value = 'Ingresa la clave administrativa.'
    return
  }
  loading.value = true
  try {
    const health = await fetch(`${normalizedApiBase()}/api/health`)
    if (!health.ok) throw new Error('No se pudo verificar el API en esa dirección.')
    await loadData()
    connected.value = true
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

async function refreshData() {
  loading.value = true
  errorMessage.value = ''
  try {
    await loadData()
    successMessage.value = 'Información actualizada.'
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

function disconnect() {
  connected.value = false
  adminKey.value = ''
  users.value = []
  deviceRequests.value = []
  successMessage.value = ''
  errorMessage.value = ''
}

function openAccessEditor(user) {
  editingUser.value = user
  accessStatus.value = accessState(user) === 'active' ? 'active' : 'inactive'
  accessEndsAt.value = toLocalDateTime(user.subscription.endsAt)
  accessPaidAt.value = user.subscription.paidAt ? toLocalDateTime(user.subscription.paidAt) : ''
  errorMessage.value = ''
}

async function saveAccess() {
  if (!editingUser.value || !accessEndsAt.value) return
  loading.value = true
  errorMessage.value = ''
  try {
    const endsAt = new Date(accessEndsAt.value)
    const result = await apiRequest(`/api/admin/users/${editingUser.value.id}/access`, {
      method: 'PATCH',
      body: JSON.stringify({
        status: accessStatus.value,
        endsAt: endsAt.toISOString(),
        paidAt: accessPaidAt.value ? new Date(accessPaidAt.value).toISOString() : null,
      }),
    })
    users.value = users.value.map((user) => user.id === result.user.id ? result.user : user)
    editingUser.value = null
    successMessage.value = `Acceso de ${result.user.name} actualizado. Sus sesiones anteriores quedaron invalidadas.`
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

function openCreateUser() {
  errorMessage.value = ''
  createUsername.value = ''
  createName.value = ''
  createPassword.value = ''
  createStatus.value = 'active'
  createEndsAt.value = toLocalDateTime(Date.now() + 30 * 24 * 60 * 60 * 1000)
  createPaidAt.value = toLocalDateTime(Date.now())
  showCreateUser.value = true
}

async function createUser() {
  loading.value = true
  errorMessage.value = ''
  try {
    const result = await apiRequest('/api/admin/users', {
      method: 'POST',
      body: JSON.stringify({
        username: createUsername.value,
        name: createName.value,
        password: createPassword.value,
        status: createStatus.value,
        endsAt: new Date(createEndsAt.value).toISOString(),
        paidAt: createPaidAt.value ? new Date(createPaidAt.value).toISOString() : null,
      }),
    })
    users.value = [result.user, ...users.value]
    showCreateUser.value = false
    successMessage.value = `Cliente ${result.user.name} registrado.`
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

async function revokeSessions(user) {
  if (!window.confirm(`¿Cerrar todas las sesiones de ${user.name}?`)) return
  loading.value = true
  errorMessage.value = ''
  try {
    const result = await apiRequest(`/api/admin/users/${user.id}/revoke-sessions`, { method: 'POST' })
    users.value = users.value.map((entry) => entry.id === result.user.id ? result.user : entry)
    successMessage.value = `Sesiones de ${user.name} cerradas.`
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

async function handleDeviceRequest(request, action) {
  const verb = action === 'approve' ? 'aprobar' : 'rechazar'
  if (action === 'reject' && !window.confirm(`¿Rechazar el cambio de dispositivo de ${request.username}?`)) return
  loading.value = true
  errorMessage.value = ''
  try {
    await apiRequest(`/api/admin/device-requests/${request.requestId}/${action}`, { method: 'POST' })
    deviceRequests.value = deviceRequests.value.filter((entry) => entry.requestId !== request.requestId)
    await loadData()
    successMessage.value = `Solicitud de ${request.username} ${verb === 'aprobar' ? 'aprobada' : 'rechazada'}.`
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  clockTimer = window.setInterval(() => { clock.value = Date.now() }, 60_000)
})

onUnmounted(() => window.clearInterval(clockTimer))
</script>

<template>
  <main v-if="!connected" class="connection-screen">
    <div class="connection-topline"><span class="brand-mark">m<span>.</span></span><span>MINIVENTA / CONTROL</span></div>
    <section class="connection-content">
      <p class="eyebrow">CONSOLA DE OPERACIONES</p>
      <h1>Tu negocio,<br><em>bajo control.</em></h1>
      <p class="connection-copy">Administra clientes y licencias.</p>
      <form class="connection-form" @submit.prevent="connect">
        <label for="admin-key">Clave administrativa</label>
        <input id="admin-key" v-model="adminKey" type="password" autocomplete="current-password" placeholder="ADMIN_API_KEY" required>
        <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
        <button class="button button-primary connect-button" type="submit" :disabled="loading">
          {{ loading ? 'Conectando…' : 'Conectar al API' }} <span aria-hidden="true">↗</span>
        </button>
      </form>
      <p class="connection-footnote"><span class="status-dot"></span> La clave solo permanece en esta sesión del navegador.</p>
    </section>
    <div class="connection-index">01 <span>CLIENTES</span><i></i> 02 <span>ACCESO</span><i></i> 03 <span>DISPOSITIVOS</span></div>
  </main>

  <div v-else class="admin-shell">
    <aside class="sidebar">
      <a class="sidebar-brand" href="#" @click.prevent="activeView = 'customers'">
        <span class="brand-mark">m<span>.</span></span><span>MINIVENTA<small>ADMINISTRACIÓN</small></span>
      </a>
      <div class="sidebar-section-label">OPERACIÓN</div>
      <nav class="side-nav" aria-label="Secciones">
        <button :class="{ selected: activeView === 'customers' }" @click="activeView = 'customers'">
          <span class="nav-symbol">◉</span> Clientes <span class="nav-count">{{ users.length }}</span>
        </button>
        <button :class="{ selected: activeView === 'devices' }" @click="activeView = 'devices'">
          <span class="nav-symbol">⌁</span> Dispositivos <span v-if="deviceRequests.length" class="nav-count nav-count-alert">{{ deviceRequests.length }}</span>
        </button>
      </nav>
      <div class="sidebar-bottom">
        <div class="api-indicator"><span class="status-dot"></span><span>API CONECTADA<small>{{ normalizedApiBase() }}</small></span></div>
        <button class="disconnect-button" @click="disconnect">Desconectar <span aria-hidden="true">↗</span></button>
      </div>
    </aside>

    <main class="workspace">
      <header class="topbar">
        <div><p class="eyebrow">MINIVENTA / ADMIN</p><p class="topbar-caption">Gestión de clientes y accesos</p></div>
        <button class="button button-light refresh-button" :disabled="loading" @click="refreshData"><span aria-hidden="true">↻</span> Actualizar</button>
      </header>

      <div class="page-content">
        <div v-if="errorMessage || successMessage" class="notice" :class="errorMessage ? 'notice-error' : 'notice-success'" role="status">
          <span>{{ errorMessage || successMessage }}</span>
          <button aria-label="Cerrar aviso" @click="errorMessage = ''; successMessage = ''">×</button>
        </div>

        <template v-if="activeView === 'customers'">
          <section class="page-heading">
            <div><p class="eyebrow">CUENTAS Y LICENCIAS</p><h1>Clientes</h1><p class="page-description">Registra pagos, renueva licencias y bloquea accesos.</p></div>
            <div class="heading-actions"><span class="record-count">{{ users.length }} <small>REGISTROS</small></span><button class="button button-primary" :disabled="loading" @click="openCreateUser">+ Registrar cliente</button></div>
          </section>
          <section class="metrics" aria-label="Resumen de cuentas">
            <article class="metric metric-total"><span class="metric-label">CLIENTES</span><strong>{{ users.length.toString().padStart(2, '0') }}</strong><span class="metric-note">Cuentas registradas</span></article>
            <article class="metric metric-active"><span class="metric-label">ACCESO VIGENTE</span><strong>{{ activeCount.toString().padStart(2, '0') }}</strong><span class="metric-note">Activos y al corriente</span></article>
            <article class="metric metric-blocked"><span class="metric-label">SIN ACCESO</span><strong>{{ blockedCount.toString().padStart(2, '0') }}</strong><span class="metric-note">Bloqueados o vencidos</span></article>
            <article class="metric metric-pending"><span class="metric-label">CAMBIOS DE EQUIPO</span><strong>{{ deviceRequests.length.toString().padStart(2, '0') }}</strong><span class="metric-note">Solicitudes por revisar</span></article>
          </section>
          <section class="table-section">
            <div class="table-toolbar"><div><h2>Directorio</h2><p>Hasta 500 cuentas, ordenadas por actividad reciente.</p></div><label class="search-field"><span aria-hidden="true">⌕</span><input v-model="search" type="search" placeholder="Buscar cliente o usuario" aria-label="Buscar cliente o usuario"></label></div>
            <div v-if="!visibleUsers.length" class="empty-state"><span class="empty-symbol">—</span><h3>{{ users.length ? 'Sin coincidencias' : 'Aún no hay clientes' }}</h3><p>{{ users.length ? 'Prueba con otro nombre o usuario.' : 'Las cuentas creadas en el API aparecerán aquí. El registro público de clientes no está habilitado.' }}</p></div>
            <div v-else class="table-scroll">
              <table class="customer-table">
                <thead><tr><th>CLIENTE</th><th>ESTADO</th><th>ÚLTIMO PAGO</th><th>VENCE</th><th>DISPOSITIVO</th><th>SESIÓN</th><th></th></tr></thead>
                <tbody>
                  <tr v-for="user in visibleUsers" :key="user.id">
                    <td><div class="customer-name">{{ user.name }}</div><div class="customer-username">@{{ user.username }}</div></td>
                    <td><span class="state-pill" :class="`state-${accessState(user)}`"><i></i>{{ accessState(user) === 'active' ? 'Activo' : accessState(user) === 'expired' ? 'Vencido' : 'Bloqueado' }}</span></td>
                    <td class="date-cell">{{ formatDate(user.subscription.paidAt) }}</td>
                    <td class="date-cell">{{ formatDate(user.subscription.endsAt) }}</td>
                    <td><span class="device-state"><i></i>{{ user.devices.some((device) => device.status === 'active') ? 'Vinculado' : 'Sin equipo' }}</span></td>
                    <td class="session-cell">v{{ user.sessionVersion }}</td>
                    <td><div class="row-actions"><button class="row-button" @click="openAccessEditor(user)">Gestionar</button><button class="row-button row-button-muted" @click="revokeSessions(user)">Cerrar sesiones</button></div></td>
                  </tr>
                </tbody>
              </table>
            </div>
          </section>
        </template>

        <template v-else>
          <section class="page-heading">
            <div><p class="eyebrow">VALIDACIÓN DE IDENTIDAD</p><h1>Dispositivos</h1><p class="page-description">Aprueba un equipo nuevo para reemplazar el dispositivo actual del cliente.</p></div>
            <span class="record-count">{{ deviceRequests.length.toString().padStart(2, '0') }} <small>PENDIENTES</small></span>
          </section>
          <section v-if="deviceRequests.length" class="request-list">
            <article v-for="request in deviceRequests" :key="request.requestId" class="request-row">
              <div class="request-index">{{ request.username.slice(0, 1).toUpperCase() }}</div>
              <div class="request-details"><div class="customer-name">@{{ request.username }} <span class="pending-label">ESPERANDO APROBACIÓN</span></div><div class="request-meta">Solicitado {{ formatDate(request.createdAt) }} <span>·</span> Huella <code>{{ request.fingerprint.slice(0, 12) }}…</code></div></div>
              <div class="request-actions"><button class="button button-light" :disabled="loading" @click="handleDeviceRequest(request, 'reject')">Rechazar</button><button class="button button-primary" :disabled="loading" @click="handleDeviceRequest(request, 'approve')">Aprobar equipo</button></div>
            </article>
          </section>
          <section v-else class="empty-state empty-devices"><span class="empty-symbol">⌁</span><h3>No hay solicitudes pendientes</h3><p>Los cambios de equipo que requieran aprobación aparecerán en esta bandeja.</p></section>
        </template>
        <footer class="page-footer"><span>MINIVENTA / CONTROL DE ACCESO</span><span>Los cambios de acceso invalidan las sesiones anteriores.</span></footer>
      </div>
    </main>

    <div v-if="editingUser || showCreateUser" class="modal-backdrop" @click.self="editingUser = null; showCreateUser = false">
      <section class="access-modal" role="dialog" aria-modal="true" aria-labelledby="access-title">
        <div class="modal-topline"><span class="eyebrow">{{ showCreateUser ? 'NUEVA CUENTA' : 'GESTIÓN DE LICENCIA' }}</span><button class="modal-close" aria-label="Cerrar" @click="editingUser = null; showCreateUser = false">×</button></div>
        <h2 id="access-title">{{ showCreateUser ? 'Registrar cliente' : editingUser.name }}</h2><p v-if="editingUser" class="modal-username">@{{ editingUser.username }}</p>
        <form v-if="showCreateUser" @submit.prevent="createUser">
          <label for="create-name">Nombre del cliente</label>
          <input id="create-name" v-model="createName" autocomplete="name" maxlength="100" required>
          <label for="create-username">Usuario para iniciar sesión</label>
          <input id="create-username" v-model="createUsername" autocomplete="username" maxlength="254" required>
          <label for="create-password">Contraseña inicial</label>
          <input id="create-password" v-model="createPassword" type="password" autocomplete="new-password" minlength="12" maxlength="72" required>
          <label for="create-status">Estado de la licencia</label>
          <select id="create-status" v-model="createStatus"><option value="active">Activar ahora</option><option value="inactive">Registrar sin acceso</option></select>
          <label for="create-paid-at">Fecha de pago</label>
          <input id="create-paid-at" v-model="createPaidAt" type="datetime-local">
          <label for="create-ends-at">Vencimiento</label>
          <input id="create-ends-at" v-model="createEndsAt" type="datetime-local" required>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="showCreateUser = false">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">{{ loading ? 'Registrando…' : 'Registrar cliente' }}</button></div>
        </form>
        <form v-else @submit.prevent="saveAccess">
          <label for="access-status">Estado del acceso</label>
          <select id="access-status" v-model="accessStatus"><option value="active">Dar acceso</option><option value="inactive">Bloquear acceso</option></select>
          <label for="access-paid-at">Último pago</label>
          <input id="access-paid-at" v-model="accessPaidAt" type="datetime-local">
          <label for="access-expiry">Vence el</label>
          <input id="access-expiry" v-model="accessEndsAt" type="datetime-local" required>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <p class="modal-note">El acceso se bloquea al llegar el vencimiento. Guardar invalida las sesiones online; las autorizaciones offline pueden durar hasta 7 días.</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="editingUser = null">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">{{ loading ? 'Guardando…' : 'Guardar cambios' }}</button></div>
        </form>
      </section>
    </div>
  </div>
</template>