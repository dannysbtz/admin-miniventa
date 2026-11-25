<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const apiBase = (import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000').trim()
const adminKey = ref('')
const connected = ref(false)
const loading = ref(false)
const users = ref([])
const deviceRequests = ref([])
const payments = ref([])
const products = ref([])
const productEditor = ref(null)
const productNumber = ref('')
const productName = ref('')
const productPrice = ref('')
const incomeTotalCents = ref(0)
const paymentCount = ref(0)
const search = ref('')
const activeView = ref('customers')
const errorMessage = ref('')
const successMessage = ref('')
const editingUser = ref(null)
const blockAccess = ref(false)
const editPrice = ref('')
const showCreateUser = ref(false)
const createUsername = ref('')
const createName = ref('')
const createPassword = ref('')
const createPrice = ref('')
const payingUser = ref(null)
const paymentRequestId = ref('')
const paymentAmount = ref('')
const paymentPaidAt = ref(toLocalDateTime(Date.now()))
const paymentEndsAt = ref('')
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

function formatMoney(amountCents) {
  return new Intl.NumberFormat('es-MX', { style: 'currency', currency: 'MXN' }).format((amountCents || 0) / 100)
}

function toCents(amount) {
  const parsed = Number(amount)
  if (!Number.isFinite(parsed) || parsed < 0) return null
  const cents = Math.round(parsed * 100)
  return Number.isSafeInteger(cents) ? cents : null
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
      payment_request_conflict: 'Ya existe una confirmación con otros datos. Actualiza la información antes de registrar otro pago.',
      renewal_not_extended: 'El nuevo vencimiento debe ser posterior al vencimiento actual.',
      product_number_exists: 'Ya existe un producto con ese número.',
      product_not_found: 'El producto ya no existe en el catálogo.',
    }
    throw new Error(messages[data.error] || `El API respondió con error (${response.status}).`)
  }
  return data
}

async function loadData() {
  const [userData, requestData, paymentData, productData] = await Promise.all([
    apiRequest('/api/admin/users'),
    apiRequest('/api/admin/device-requests'),
    apiRequest('/api/admin/payments'),
    apiRequest('/api/admin/products'),
  ])
  users.value = userData.users
  deviceRequests.value = requestData.requests
  payments.value = paymentData.payments
  products.value = productData.products
  incomeTotalCents.value = paymentData.totalCents
  paymentCount.value = paymentData.count
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
  payments.value = []
  products.value = []
  productEditor.value = null
  incomeTotalCents.value = 0
  paymentCount.value = 0
  successMessage.value = ''
  errorMessage.value = ''
}

function openProductEditor(product = null) {
  errorMessage.value = ''
  successMessage.value = ''
  productEditor.value = product || { id: null }
  productNumber.value = product?.number || ''
  productName.value = product?.name || ''
  productPrice.value = product ? (product.priceCents / 100).toFixed(2) : ''
}

async function saveProduct() {
  const priceCents = toCents(productPrice.value)
  if (priceCents === null || !productNumber.value.trim() || !productName.value.trim()) {
    errorMessage.value = 'Revisa el número, nombre y precio del producto.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  try {
    const id = productEditor.value.id
    const result = await apiRequest(id ? `/api/admin/products/${id}` : '/api/admin/products', {
      method: id ? 'PUT' : 'POST',
      body: JSON.stringify({ number: productNumber.value, name: productName.value, priceCents }),
    })
    products.value = [...products.value.filter((product) => product.id !== id), result.product].sort((first, second) => first.number.localeCompare(second.number))
    productEditor.value = null
    successMessage.value = 'Producto guardado en el catálogo.'
  } catch (error) { errorMessage.value = error.message }
  finally { loading.value = false }
}

async function deleteProduct(product) {
  if (!window.confirm(`¿Eliminar ${product.name} del catálogo de la nube? Los productos locales no se eliminarán.`)) return
  loading.value = true
  errorMessage.value = ''
  try {
    await apiRequest(`/api/admin/products/${product.id}`, { method: 'DELETE' })
    products.value = products.value.filter((entry) => entry.id !== product.id)
    successMessage.value = 'Producto eliminado del catálogo.'
  } catch (error) { errorMessage.value = error.message }
  finally { loading.value = false }
}

function openAccessEditor(user) {
  editingUser.value = user
  blockAccess.value = false
  editPrice.value = (user.subscription.priceCents / 100).toFixed(2)
  errorMessage.value = ''
}

async function saveAccess() {
  if (!editingUser.value) return
  const priceCents = toCents(editPrice.value)
  if (priceCents === null) {
    errorMessage.value = 'Ingresa un precio válido en pesos.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  try {
    const result = await apiRequest(`/api/admin/users/${editingUser.value.id}/access`, {
      method: 'PATCH',
      body: JSON.stringify({
        ...(blockAccess.value ? { status: 'inactive' } : {}),
        priceCents,
      }),
    })
    users.value = users.value.map((user) => user.id === result.user.id ? result.user : user)
    editingUser.value = null
    successMessage.value = blockAccess.value ? `Acceso de ${result.user.name} bloqueado.` : `Precio de ${result.user.name} actualizado.`
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
  createPrice.value = ''
  showCreateUser.value = true
}

async function createUser() {
  const priceCents = toCents(createPrice.value)
  if (priceCents === null) {
    errorMessage.value = 'Ingresa un precio válido en pesos.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  try {
    const result = await apiRequest('/api/admin/users', {
      method: 'POST',
      body: JSON.stringify({
        username: createUsername.value,
        name: createName.value,
        password: createPassword.value,
        priceCents,
      }),
    })
    users.value = [result.user, ...users.value]
    showCreateUser.value = false
    successMessage.value = `Cliente ${result.user.name} registrado sin acceso hasta confirmar el primer pago.`
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

function openPaymentForm(user) {
  payingUser.value = user
  paymentRequestId.value = crypto.randomUUID()
  paymentAmount.value = (user.subscription.priceCents / 100).toFixed(2)
  paymentPaidAt.value = toLocalDateTime(Date.now())
  paymentEndsAt.value = ''
  errorMessage.value = ''
}

async function confirmPayment() {
  if (!payingUser.value || !paymentPaidAt.value || !paymentEndsAt.value) return
  const amountCents = toCents(paymentAmount.value)
  if (amountCents === null || amountCents < 1) {
    errorMessage.value = 'El monto recibido debe ser mayor a cero.'
    return
  }
  loading.value = true
  errorMessage.value = ''
  try {
    const result = await apiRequest(`/api/admin/users/${payingUser.value.id}/confirm-payment`, {
      method: 'POST',
      body: JSON.stringify({
        requestId: paymentRequestId.value,
        amountCents,
        paidAt: new Date(paymentPaidAt.value).toISOString(),
        endsAt: new Date(paymentEndsAt.value).toISOString(),
      }),
    })
    users.value = users.value.map((user) => user.id === result.user.id ? result.user : user)
    payingUser.value = null
    successMessage.value = `Pago de ${formatMoney(amountCents)} confirmado; licencia de ${result.user.name} renovada.`
    try {
      await loadData()
    } catch {
      errorMessage.value = 'El pago se confirmó, pero no se pudo actualizar el resumen. Usa Actualizar antes de continuar.'
    }
  } catch (error) {
    errorMessage.value = error.message
  } finally {
    loading.value = false
  }
}

async function deleteUser(user) {
  if (!window.confirm(`¿Eliminar permanentemente la cuenta de ${user.name}? El historial de pagos se conservará en ingresos.`)) return
  if (window.prompt(`Escribe ${user.username} para confirmar la eliminación.`) !== user.username) return
  loading.value = true
  errorMessage.value = ''
  try {
    await apiRequest(`/api/admin/users/${user.id}`, { method: 'DELETE' })
    users.value = users.value.filter((entry) => entry.id !== user.id)
    successMessage.value = `Cuenta de ${user.name} eliminada. Su historial financiero se conservó.`
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
        <button :class="{ selected: activeView === 'products' }" @click="activeView = 'products'">
          <span class="nav-symbol">▦</span> Productos <span class="nav-count">{{ products.length }}</span>
        </button>
        <button :class="{ selected: activeView === 'finance' }" @click="activeView = 'finance'">
          <span class="nav-symbol">$</span> Ingresos <span class="nav-count">{{ paymentCount }}</span>
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
            <article class="metric metric-pending"><span class="metric-label">INGRESOS COBRADOS</span><strong class="metric-currency">{{ formatMoney(incomeTotalCents) }}</strong><span class="metric-note">{{ paymentCount }} pagos confirmados</span></article>
          </section>
          <section class="table-section">
            <div class="table-toolbar"><div><h2>Directorio</h2><p>Hasta 500 cuentas, ordenadas por actividad reciente.</p></div><label class="search-field"><span aria-hidden="true">⌕</span><input v-model="search" type="search" placeholder="Buscar cliente o usuario" aria-label="Buscar cliente o usuario"></label></div>
            <div v-if="!visibleUsers.length" class="empty-state"><span class="empty-symbol">—</span><h3>{{ users.length ? 'Sin coincidencias' : 'Aún no hay clientes' }}</h3><p>{{ users.length ? 'Prueba con otro nombre o usuario.' : 'Las cuentas creadas en el API aparecerán aquí. El registro público de clientes no está habilitado.' }}</p></div>
            <div v-else class="table-scroll">
              <table class="customer-table">
                <thead><tr><th>CLIENTE</th><th>ESTADO</th><th>PRECIO</th><th>ÚLTIMO PAGO</th><th>VENCE</th><th>DISPOSITIVO</th><th>SESIÓN</th><th></th></tr></thead>
                <tbody>
                  <tr v-for="user in visibleUsers" :key="user.id">
                    <td><div class="customer-name">{{ user.name }}</div><div class="customer-username">@{{ user.username }}</div></td>
                    <td><span class="state-pill" :class="`state-${accessState(user)}`"><i></i>{{ accessState(user) === 'active' ? 'Activo' : accessState(user) === 'expired' ? 'Vencido' : 'Bloqueado' }}</span></td>
                    <td class="date-cell">{{ formatMoney(user.subscription.priceCents) }}</td>
                    <td class="date-cell">{{ formatDate(user.subscription.paidAt) }}</td>
                    <td class="date-cell">{{ formatDate(user.subscription.endsAt) }}</td>
                    <td><span class="device-state"><i></i>{{ user.devices.some((device) => device.status === 'active') ? 'Vinculado' : 'Sin equipo' }}</span></td>
                    <td class="session-cell">v{{ user.sessionVersion }}</td>
                    <td><div class="row-actions"><button class="row-button" :disabled="loading" @click="openPaymentForm(user)">Pago / renovar</button><button class="row-button" :disabled="loading" @click="openAccessEditor(user)">Gestionar</button><button class="row-button row-button-muted" :disabled="loading" @click="revokeSessions(user)">Cerrar sesiones</button><button class="row-button row-button-danger" :disabled="loading" @click="deleteUser(user)">Eliminar</button></div></td>
                  </tr>
                </tbody>
              </table>
            </div>
          </section>
        </template>

        <template v-else-if="activeView === 'devices'">
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
        <template v-else-if="activeView === 'products'">
          <section class="page-heading">
            <div><p class="eyebrow">CATÁLOGO EN LA NUBE</p><h1>Productos</h1></div>
            <div class="heading-actions"><span class="record-count">{{ products.length }} <small>PRODUCTOS</small></span><button class="button button-primary" :disabled="loading" @click="openProductEditor()">+ Registrar producto</button></div>
          </section>
          <section class="table-section">
            <div class="table-toolbar"><div><h2>Catálogo</h2></div></div>
            <div v-if="!products.length" class="empty-state"><h3>Aún no hay productos</h3></div>
            <div v-else class="table-scroll"><table class="customer-table">
              <thead><tr><th>NÚMERO</th><th>NOMBRE</th><th>PRECIO</th><th></th></tr></thead>
              <tbody><tr v-for="product in products" :key="product.id">
                <td>{{ product.number }}</td><td class="customer-name">{{ product.name }}</td><td>{{ formatMoney(product.priceCents) }}</td>
                <td><div class="row-actions"><button class="row-button" :disabled="loading" @click="openProductEditor(product)">Editar</button><button class="row-button row-button-danger" :disabled="loading" @click="deleteProduct(product)">Eliminar</button></div></td>
              </tr></tbody>
            </table></div>
          </section>
        </template>
        <template v-else>
          <section class="page-heading">
            <div><p class="eyebrow">CONTROL DE COBROS</p><h1>Ingresos</h1><p class="page-description">Pagos confirmados manualmente; los montos no incluyen costos ni comisiones.</p></div>
            <span class="record-count">{{ paymentCount.toString().padStart(2, '0') }} <small>PAGOS</small></span>
          </section>
          <section class="finance-summary" aria-label="Resumen de ingresos">
            <div><span class="metric-label">INGRESOS COBRADOS · HISTÓRICO</span><strong>{{ formatMoney(incomeTotalCents) }}</strong></div>
            <span>El total considera todos los pagos registrados.</span>
          </section>
          <section class="table-section finance-table-section">
            <div class="table-toolbar"><div><h2>Movimientos recientes</h2><p>Se muestran hasta 500 pagos; el resumen incluye el historial completo.</p></div></div>
            <div v-if="!payments.length" class="empty-state"><span class="empty-symbol">$</span><h3>Aún no hay pagos confirmados</h3><p>Al confirmar un pago desde la ficha de un cliente, aparecerá en este historial.</p></div>
            <div v-else class="table-scroll">
              <table class="customer-table finance-table">
                <thead><tr><th>CLIENTE</th><th>MONTO</th><th>FECHA DE PAGO</th><th>REGISTRADO</th><th>VENCIMIENTO</th></tr></thead>
                <tbody>
                  <tr v-for="payment in payments" :key="payment._id">
                    <td><div class="customer-name">{{ payment.name }}</div><div class="customer-username">@{{ payment.username }}</div></td>
                    <td class="payment-amount">{{ formatMoney(payment.amountCents) }}</td>
                    <td class="date-cell">{{ formatDate(payment.paidAt) }}</td>
                    <td class="date-cell">{{ formatDate(payment.recordedAt) }}</td>
                    <td class="date-cell">{{ formatDate(payment.endsAt) }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </section>
        </template>
        <footer class="page-footer"><span>MINIVENTA / CONTROL DE ACCESO</span><span>Los cambios de acceso invalidan las sesiones anteriores.</span></footer>
      </div>
    </main>

    <div v-if="productEditor" class="modal-backdrop" @click.self="productEditor = null">
      <section class="access-modal" role="dialog" aria-modal="true" aria-labelledby="product-title">
        <div class="modal-topline"><span class="eyebrow">CATÁLOGO</span><button class="modal-close" aria-label="Cerrar" @click="productEditor = null">×</button></div>
        <h2 id="product-title">{{ productEditor.id ? 'Editar producto' : 'Registrar producto' }}</h2>
        <form @submit.prevent="saveProduct">
          <label for="product-number">Número</label><input id="product-number" v-model="productNumber" maxlength="40" required>
          <label for="product-name">Nombre</label><input id="product-name" v-model="productName" maxlength="150" required>
          <label for="product-price">Precio (MXN)</label><input id="product-price" v-model="productPrice" type="number" min="0" step="0.01" inputmode="decimal" required>
          <p v-if="productEditor.id" class="modal-note">Si cambias el número, el número anterior permanecerá en los dispositivos que ya lo descargaron.</p>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="productEditor = null">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">Guardar</button></div>
        </form>
      </section>
    </div>

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
          <label for="create-price">Precio por suscripción (MXN)</label>
          <input id="create-price" v-model="createPrice" type="number" min="0" step="0.01" inputmode="decimal" required>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <p class="modal-note">La cuenta queda bloqueada. El acceso se activa al confirmar el primer pago.</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="showCreateUser = false">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">{{ loading ? 'Registrando…' : 'Registrar cliente' }}</button></div>
        </form>
        <form v-else @submit.prevent="saveAccess">
          <label for="edit-price">Precio por suscripción (MXN)</label>
          <input id="edit-price" v-model="editPrice" type="number" min="0" step="0.01" inputmode="decimal" required>
          <label class="check-label" for="block-access"><input id="block-access" v-model="blockAccess" type="checkbox"> Bloquear acceso y cerrar sesiones</label>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <p class="modal-note">No se puede activar desde aquí. Para dar acceso, registra el pago y define el vencimiento en “Pago / renovar”.</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="editingUser = null">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">{{ loading ? 'Guardando…' : 'Guardar precio' }}</button></div>
        </form>
      </section>
    </div>

    <div v-if="payingUser" class="modal-backdrop" @click.self="payingUser = null">
      <section class="access-modal" role="dialog" aria-modal="true" aria-labelledby="payment-title">
        <div class="modal-topline"><span class="eyebrow">CONFIRMACIÓN MANUAL</span><button class="modal-close" aria-label="Cerrar" @click="payingUser = null">×</button></div>
        <h2 id="payment-title">Confirmar pago</h2><p class="modal-username">{{ payingUser.name }} · @{{ payingUser.username }}</p>
        <form @submit.prevent="confirmPayment">
          <label for="payment-amount">Monto recibido (MXN)</label>
          <input id="payment-amount" v-model="paymentAmount" type="number" min="0.01" step="0.01" inputmode="decimal" required>
          <label for="payment-paid-at">Fecha en que pagó</label>
          <input id="payment-paid-at" v-model="paymentPaidAt" type="datetime-local" required>
          <label for="payment-ends-at">Nuevo vencimiento</label>
          <input id="payment-ends-at" v-model="paymentEndsAt" type="datetime-local" required>
          <p v-if="errorMessage" class="form-error" role="alert">{{ errorMessage }}</p>
          <p class="modal-note">Al confirmar se registrará el ingreso y se activará la licencia hasta el vencimiento elegido. No hay cobros ni renovaciones automáticas.</p>
          <div class="modal-actions"><button class="button button-light" type="button" @click="payingUser = null">Cancelar</button><button class="button button-primary" type="submit" :disabled="loading">{{ loading ? 'Confirmando…' : 'Confirmar y renovar' }}</button></div>
        </form>
      </section>
    </div>
  </div>
</template>