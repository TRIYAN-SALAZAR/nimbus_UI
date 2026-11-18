<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { apiFetch, getApiUrl } from './services/api'

const sections = [
  { id: 'ventas', label: 'Ventas', icon: 'Ve', group: 'operacion' },
  { id: 'tickets', label: 'Tickets', icon: 'T', group: 'operacion' },
  { id: 'compras', label: 'Compras', icon: 'Co', group: 'operacion' },
  { id: 'reportes', label: 'Reportes', icon: 'R', group: 'operacion' },
  { id: 'productos', label: 'Productos', icon: 'P', group: 'catalogo' },
  { id: 'clientes', label: 'Clientes', icon: 'C', group: 'catalogo' },
  { id: 'proveedores', label: 'Proveedores', icon: 'Pr', group: 'catalogo' },
  { id: 'insumos', label: 'Insumos', icon: 'I', group: 'produccion' },
  { id: 'stock', label: 'Stock', icon: 'S', group: 'produccion' },
  { id: 'recetas', label: 'Recetas', icon: 'R', group: 'produccion' },
]

const navGroups = [
  { id: 'operacion', label: 'Operación' },
  { id: 'catalogo', label: 'Catálogo' },
  { id: 'produccion', label: 'Producción' },
]

const activeSection = ref('ventas')
const loading = ref(false)
const message = ref('')
const error = ref('')
const apiOnline = ref(false)
const productos = ref([])
const variantes = ref([])
const extras = ref([])
const insumos = ref([])
const stock = ref([])
const proveedores = ref([])
const clientes = ref([])
const metodosPago = ref([])
const origenesPago = ref([])
const compras = ref([])
const purchaseView = ref('registradas')
const ventas = ref([])
const savedTickets = ref([])
const cashReceived = ref('')
const showPayment = ref(false)
const selectedVariantIndex = ref(0)
const recetaSeleccionada = ref({ tipo: 'variantes', id: '' })
const recetaDetalle = ref([])
const catalogForm = ref('producto')
const recipeEditOpen = ref(false)
const expandedRecipes = ref({})
const recipeCache = ref({})
const reportLoading = ref(false)
const reportError = ref('')
const report = ref(null)
const productSearch = ref('')
const variantSearch = ref('')
const ticketSearch = ref('')
const ticketDateFrom = ref('')
const ticketDateTo = ref('')
const catalogEditMode = ref(false)
const editingProductId = ref('')
const editingVariantId = ref('')
const editingExtraId = ref('')
const reportFilters = reactive({
  fecha_desde: new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString().slice(0, 10),
  fecha_hasta: new Date().toISOString().slice(0, 10),
})

const forms = reactive({
  producto: { nombre: '', categoria: '' },
  insumo: { nombre: '', unidad_medida: '', stock_minimo: '' },
  cliente: { nombre: '', contacto: '' },
  variante: { id_producto: '', tamano: '', precio_actual: '' },
  extra: { nombre: '', precio_actual: '' },
  proveedor: { nombre: '', contacto: '' },
  metodoPago: { nombre: '' },
  origenPago: { nombre: '', tipo: '' },
  receta: { id_insumo: '', cantidad_consumida: '', solo_para_llevar: false },
  compra: {
    fecha: new Date().toISOString().slice(0, 10),
    notas: '',
    id_origen: '',
    id_proveedor: '',
    detalles: [],
  },
  venta: {
    consumo_local: true,
    requiere_nota: false,
    folio_nota: '',
    id_metodo_pago: '',
    id_cliente: '',
    detalles: [],
  },
})

const activeLabel = computed(
  () => sections.find((section) => section.id === activeSection.value)?.label || 'Ventas',
)

function saleDetails(venta) {
  return venta.detalle_venta || venta.detalles || []
}

function saleProductName(detail) {
  const variante = detail.variante || detail.variantes || {}
  const producto = variante.producto || variante.productos || {}
  if (producto.nombre) return producto.nombre

  const catalogVariant = variantes.value.find(
    (item) => Number(item.id_variante) === Number(detail.id_variante),
  )
  const catalogProduct = catalogVariant?.productos?.nombre
    ? catalogVariant
    : productos.value.find(
      (item) => Number(item.id_producto) === Number(catalogVariant?.id_producto),
    )
  return catalogVariant?.productos?.nombre
    || catalogVariant?.producto?.nombre
    || catalogProduct?.nombre
    || 'Producto'
}

function saleVariantSize(detail) {
  const variante = detail.variante || detail.variantes || {}
  if (variante.tamano) return variante.tamano
  return variantes.value.find(
    (item) => Number(item.id_variante) === Number(detail.id_variante),
  )?.tamano || 'Sin tamaño'
}

function saleExtras(detail) {
  return detail.detalle_venta_extra || detail.extras || []
}

function saleExtraName(extra) {
  const item = extra.extra || extra.extras || {}
  return item.nombre || extras.value.find(
    (entry) => Number(entry.id_extra) === Number(extra.id_extra),
  )?.nombre || `Extra ${extra.id_extra}`
}

function ticketDate(value) {
  if (!value) return ''
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return String(value).slice(0, 10)
  return `${String(date.getDate()).padStart(2, '0')}-${String(date.getMonth() + 1).padStart(2, '0')}-${date.getFullYear()}`
}

function ticketClient(venta) {
  return venta.clientes?.nombre || venta.cliente?.nombre || 'Público general'
}

function ticketVariantLabel(detail) {
  const size = saleVariantSize(detail)
  const code = variantCode(size)
  const name = saleProductName(detail)
  return code && code !== '?' ? `${name} ${code}.` : name
}

function ticketProductAmount(detail) {
  const variante = detail.variante || detail.variantes || {}
  const catalogVariant = variantes.value.find(
    (item) => Number(item.id_variante) === Number(detail.id_variante),
  )
  return Number(variante.precio_actual || catalogVariant?.precio_actual || 0) * Number(detail.cantidad || 0)
}

function ticketExtraAmount(extra) {
  const item = extra.extra || extra.extras || {}
  const catalogExtra = extras.value.find(
    (entry) => Number(entry.id_extra) === Number(extra.id_extra),
  )
  return Number(item.precio_actual || catalogExtra?.precio_actual || 0) * Number(extra.cantidad || 0)
}

function variantProductName(variante) {
  return variante.productos?.nombre || variante.producto?.nombre || productos.value.find(
    (item) => Number(item.id_producto) === Number(variante.id_producto),
  )?.nombre || `Producto ${variante.id_producto}`
}

function variantCode(tamano) {
  const normalized = String(tamano || '').trim().toLocaleLowerCase('es')
  if (normalized === 'grande' || normalized === 'large' || normalized === 'g') return 'G'
  if (normalized === 'mediano' || normalized === 'medium' || normalized === 'm') return 'M'
  if (normalized === 'pequeño' || normalized === 'pequeno' || normalized === 'chico' ||
    normalized === 'sencillo' || normalized === 'small' || normalized === 's') return 'S'
  return String(tamano || '?').trim().charAt(0).toLocaleUpperCase('es') || '?'
}

const combinedCatalog = computed(() => productos.value.map((producto) => ({
  id_producto: producto.id_producto,
  nombre: producto.nombre,
  categoria: producto.categoria || 'Sin categoría',
  activo: producto.activo !== false,
  precio_actual: producto.precio_actual,
  precio: producto.precio,
  variantes: variantes.value.filter(
    (variante) => Number(variante.id_producto) === Number(producto.id_producto),
  ),
})))

function ingredientName(recipe) {
  const related = recipe.insumo || recipe.insumos
  if (related?.nombre) return related.nombre
  return insumos.value.find((item) => Number(item.id_insumo) === Number(recipe.id_insumo))?.nombre || 'Insumo no encontrado'
}

function addInsumoToPurchase(insumo) {
  const existingIndex = forms.compra.detalles.findIndex(
    (detalle) => Number(detalle.id_insumo) === Number(insumo.id_insumo),
  )

  if (existingIndex >= 0) {
    forms.compra.detalles[existingIndex].cantidad = Number(forms.compra.detalles[existingIndex].cantidad || 0) + 1
  } else {
    forms.compra.detalles.push({
      id_insumo: insumo.id_insumo,
      cantidad: 1,
      precio: '',
    })
  }
  clearFeedback()
}

function removePurchaseItem(index) {
  forms.compra.detalles.splice(index, 1)
}

function changePurchaseQuantity(index, amount) {
  const detalle = forms.compra.detalles[index]
  detalle.cantidad = Math.max(0.001, Number(detalle.cantidad || 0) + amount)
}

function purchaseTotal() {
  return forms.compra.detalles.reduce(
    (total, detalle) => total + Number(detalle.cantidad || 0) * Number(detalle.precio || 0),
    0,
  )
}

function savedTicketsStorageKey() {
  return 'nimbus_saved_tickets'
}

function restoreSavedTickets() {
  try {
    savedTickets.value = JSON.parse(localStorage.getItem(savedTicketsStorageKey()) || '[]')
  } catch {
    savedTickets.value = []
  }
}

function persistSavedTickets() {
  localStorage.setItem(savedTicketsStorageKey(), JSON.stringify(savedTickets.value))
}

function addVariantToTicket(variante) {
  const existingIndex = forms.venta.detalles.findIndex(
    (detalle) => Number(detalle.id_variante) === Number(variante.id_variante),
  )

  if (existingIndex >= 0) {
    forms.venta.detalles[existingIndex].cantidad += 1
  } else {
    forms.venta.detalles.push({
      id_variante: variante.id_variante,
      cantidad: 1,
      extras: [],
    })
  }

  const detailIndex = forms.venta.detalles.findIndex(
    (detalle) => Number(detalle.id_variante) === Number(variante.id_variante),
  )
  selectedVariantIndex.value = detailIndex
  clearFeedback()
}

function removeTicketDetail(index) {
  if (!window.confirm('¿Eliminar este producto del ticket actual?')) return
  forms.venta.detalles.splice(index, 1)
  if (selectedVariantIndex.value >= forms.venta.detalles.length) {
    selectedVariantIndex.value = Math.max(forms.venta.detalles.length - 1, 0)
  }
}

function changeTicketQuantity(index, amount) {
  const detalle = forms.venta.detalles[index]
  detalle.cantidad = Math.max(1, Number(detalle.cantidad) + amount)
}

function addExtraToSelected(extra) {
  if (!forms.venta.detalles.length) {
    error.value = 'Selecciona primero un producto.'
    return
  }

  if (selectedVariantIndex.value >= forms.venta.detalles.length) {
    selectedVariantIndex.value = 0
  }

  const detalle = forms.venta.detalles[selectedVariantIndex.value]
  const existing = detalle.extras.find((item) => Number(item.id_extra) === Number(extra.id_extra))
  if (existing) {
    existing.cantidad += 1
  } else {
    detalle.extras.push({ id_extra: extra.id_extra, cantidad: 1 })
  }
  clearFeedback()
}

function removeExtraFromDetail(detailIndex, extraIndex) {
  forms.venta.detalles[detailIndex].extras.splice(extraIndex, 1)
}

function findVariant(idVariante) {
  return variantes.value.find((item) => Number(item.id_variante) === Number(idVariante))
}

function findExtra(idExtra) {
  return extras.value.find((item) => Number(item.id_extra) === Number(idExtra))
}

function ticketSubtotal() {
  return forms.venta.detalles.reduce((total, detalle) => {
    const variante = findVariant(detalle.id_variante)
    const productPrice = Number(variante?.precio_actual || 0) * Number(detalle.cantidad)
    const extrasPrice = detalle.extras.reduce((subtotal, extra) => {
      const extraData = findExtra(extra.id_extra)
      return subtotal + Number(extraData?.precio_actual || 0) * Number(extra.cantidad)
    }, 0)
    return total + productPrice + extrasPrice
  }, 0)
}

function ticketChange() {
  const received = Number(cashReceived.value)
  if (!cashReceived.value || received < ticketSubtotal()) return 0
  return received - ticketSubtotal()
}

function salePayload(form) {
  return {
    consumo_local: form.consumo_local,
    requiere_nota: form.requiere_nota,
    folio_nota: form.requiere_nota ? form.folio_nota : undefined,
    id_metodo_pago: Number(form.id_metodo_pago),
    id_cliente: form.id_cliente ? Number(form.id_cliente) : null,
    detalles: form.detalles.map((detalle) => ({
      id_variante: Number(detalle.id_variante),
      cantidad: Number(detalle.cantidad),
      extras: detalle.extras
        .filter((extra) => extra.id_extra)
        .map((extra) => ({ id_extra: Number(extra.id_extra), cantidad: Number(extra.cantidad) })),
    })),
  }
}

function clearFeedback() {
  message.value = ''
  error.value = ''
}

function formatCurrency(value) {
  return Number(value || 0).toLocaleString('es-MX', {
    style: 'currency',
    currency: 'MXN',
  })
}

const filteredVariants = computed(() => {
  const term = variantSearch.value.trim().toLocaleLowerCase('es')
  if (!term) return variantes.value
  return variantes.value.filter((variante) => {
    const name = variantProductName(variante).toLocaleLowerCase('es')
    const size = String(variante.tamano || '').toLocaleLowerCase('es')
    return name.includes(term) || size.includes(term)
  })
})

const filteredCombinedCatalog = computed(() => {
  const term = productSearch.value.trim().toLocaleLowerCase('es')
  if (!term) return combinedCatalog.value
  return combinedCatalog.value.filter((producto) =>
    producto.nombre.toLocaleLowerCase('es').includes(term) ||
    producto.categoria.toLocaleLowerCase('es').includes(term),
  )
})

const filteredSales = computed(() => {
  const term = ticketSearch.value.trim().toLocaleLowerCase('es')
  return ventas.value.filter((venta) => {
    const date = String(venta.fecha_hora || venta.fecha || venta.created_at || '')
    if (ticketDateFrom.value && date.slice(0, 10) < ticketDateFrom.value) return false
    if (ticketDateTo.value && date.slice(0, 10) > ticketDateTo.value) return false
    if (!term) return true
    const customer = venta.clientes?.nombre || venta.cliente?.nombre || ''
    const details = saleDetails(venta).map((detail) =>
      `${detail.cantidad} ${saleProductName(detail)} ${saleVariantSize(detail)} ${saleExtras(detail).map(saleExtraName).join(' ')}`,
    ).join(' ')
    return String(venta.id_venta).includes(term) ||
      customer.toLocaleLowerCase('es').includes(term) ||
      details.toLocaleLowerCase('es').includes(term)
  })
})

function reportEntries(value) {
  return Object.entries(value || {})
}

async function loadReports() {
  reportLoading.value = true
  reportError.value = ''
  const params = new URLSearchParams()
  if (reportFilters.fecha_desde) params.set('fecha_desde', reportFilters.fecha_desde)
  if (reportFilters.fecha_hasta) params.set('fecha_hasta', reportFilters.fecha_hasta)

  try {
    report.value = await apiFetch(`/api/reportes/resumen?${params.toString()}`)
  } catch (requestError) {
    reportError.value = requestError.message
  } finally {
    reportLoading.value = false
  }
}

async function loadData() {
  loading.value = true
  clearFeedback()

  try {
    const [
      productData,
      variantData,
      extraData,
      supplyData,
      stockData,
      providerData,
      clientData,
      paymentData,
      originData,
      purchaseData,
      saleData,
    ] = await Promise.all([
      apiFetch('/api/productos'),
      apiFetch('/api/variantes'),
      apiFetch('/api/extras'),
      apiFetch('/api/insumos'),
      apiFetch('/api/stock'),
      apiFetch('/api/proveedores'),
      apiFetch('/api/clientes'),
      apiFetch('/api/metodos-pago'),
      apiFetch('/api/origenes-pago'),
      apiFetch('/api/compras'),
      apiFetch('/api/ventas'),
    ])

    productos.value = productData || []
    variantes.value = variantData || []
    extras.value = extraData || []
    insumos.value = supplyData || []
    stock.value = stockData || []
    proveedores.value = providerData || []
    clientes.value = clientData || []
    metodosPago.value = paymentData || []
    origenesPago.value = originData || []
    compras.value = purchaseData || []
    ventas.value = saleData || []
    apiOnline.value = true
  } catch (requestError) {
    apiOnline.value = false
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

async function submitForm(resource, form, successMessage) {
  loading.value = true
  clearFeedback()

  try {
    const payload = { ...form }
    if (resource === 'insumos') {
      payload.stock_minimo = Number(payload.stock_minimo)
    }

    await apiFetch(`/api/${resource}`, {
      method: 'POST',
      body: JSON.stringify(payload),
    })

    Object.keys(form).forEach((key) => {
      form[key] = ''
    })
    await loadData()
    message.value = successMessage
  } catch (requestError) {
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

async function createProducto() {
  await submitForm('productos', forms.producto, 'Producto creado correctamente.')
}

async function createInsumo() {
  await submitForm('insumos', forms.insumo, 'Insumo creado correctamente.')
}

async function createCliente() {
  await submitForm('clientes', forms.cliente, 'Cliente creado correctamente.')
}

async function createVariant() {
  const form = forms.variante
  await submitForm(
    'variantes',
    { ...form, id_producto: Number(form.id_producto), precio_actual: Number(form.precio_actual) },
    'Variante creada correctamente.',
  )
}

async function createExtra() {
  await submitForm('extras', { ...forms.extra, precio_actual: Number(forms.extra.precio_actual) }, 'Extra creado correctamente.')
}

async function updateCatalogItem(resource, id, payload, successMessage) {
  if (!id) {
    error.value = 'Selecciona el registro que deseas editar.'
    return
  }
  loading.value = true
  clearFeedback()
  try {
    await apiFetch(`/api/${resource}/${id}`, {
      method: 'PATCH',
      body: JSON.stringify(payload),
    })
    await loadData()
    catalogEditMode.value = false
    message.value = successMessage
  } catch (requestError) {
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

async function updateProducto() {
  await updateCatalogItem('productos', editingProductId.value, {
    nombre: forms.producto.nombre,
    categoria: forms.producto.categoria,
  }, 'Producto actualizado correctamente.')
}

async function updateVariant() {
  await updateCatalogItem('variantes', editingVariantId.value, {
    id_producto: Number(forms.variante.id_producto),
    tamano: forms.variante.tamano,
    precio_actual: Number(forms.variante.precio_actual),
  }, 'Variante actualizada correctamente.')
}

async function updateExtra() {
  await updateCatalogItem('extras', editingExtraId.value, {
    nombre: forms.extra.nombre,
    precio_actual: Number(forms.extra.precio_actual),
  }, 'Extra actualizado correctamente.')
}

function selectProductoEdit(producto) {
  editingProductId.value = String(producto.id_producto)
  forms.producto.nombre = producto.nombre
  forms.producto.categoria = producto.categoria === 'Sin categoría' ? '' : producto.categoria
  catalogEditMode.value = true
}

function selectVariantEdit(variante) {
  editingVariantId.value = String(variante.id_variante)
  forms.variante.id_producto = variante.id_producto
  forms.variante.tamano = variante.tamano
  forms.variante.precio_actual = variante.precio_actual
  catalogForm.value = 'variante'
  catalogEditMode.value = true
}

function selectExtraEdit(extra) {
  editingExtraId.value = String(extra.id_extra)
  forms.extra.nombre = extra.nombre
  forms.extra.precio_actual = extra.precio_actual
  catalogForm.value = 'extra'
  catalogEditMode.value = true
}

async function createProvider() {
  await submitForm('proveedores', forms.proveedor, 'Proveedor creado correctamente.')
}

async function loadRecipe() {
  clearFeedback()
  const { tipo, id } = recetaSeleccionada.value
  if (!id) {
    recetaDetalle.value = []
    return
  }

  try {
    recetaDetalle.value = (await apiFetch(`/api/recetas/${tipo}/${id}`)) || []
  } catch (requestError) {
    error.value = requestError.message
  }
}

async function toggleRecipe(tipo, id) {
  const key = `${tipo}-${id}`
  if (expandedRecipes.value[key]) {
    expandedRecipes.value[key] = false
    return
  }

  recetaSeleccionada.value = { tipo, id }
  try {
    const details = (await apiFetch(`/api/recetas/${tipo}/${id}`)) || []
    recipeCache.value[key] = details
    recetaDetalle.value = details
    expandedRecipes.value[key] = true
  } catch (requestError) {
    error.value = requestError.message
  }
}

function recipeDetails(tipo, id) {
  return recipeCache.value[`${tipo}-${id}`] || []
}

function openRecipeEditor(tipo, id) {
  recetaSeleccionada.value = { tipo, id }
  recipeEditOpen.value = true
  clearFeedback()
  const key = `${tipo}-${id}`
  const load = async () => {
    if (!recipeCache.value[key]) {
      await toggleRecipe(tipo, id)
    }
    const details = recipeCache.value[key] || []
    if (details.length) {
      forms.receta.id_insumo = details[0].id_insumo
      forms.receta.cantidad_consumida = details[0].cantidad_consumida
    } else {
      forms.receta.id_insumo = ''
      forms.receta.cantidad_consumida = ''
    }
  }
  load()
}

async function refreshRecipe(tipo, id) {
  try {
    recipeCache.value[`${tipo}-${id}`] = (await apiFetch(`/api/recetas/${tipo}/${id}`)) || []
  } catch (requestError) {
    error.value = requestError.message
  }
}

async function createRecipe() {
  const { tipo, id } = recetaSeleccionada.value
  if (!id) {
    error.value = 'Selecciona una variante o extra para cargar su receta.'
    return
  }

  const payload = {
    id_insumo: Number(forms.receta.id_insumo),
    cantidad_consumida: Number(forms.receta.cantidad_consumida),
  }
  if (tipo === 'variantes') payload.solo_para_llevar = forms.receta.solo_para_llevar

  loading.value = true
  clearFeedback()
  try {
    await apiFetch(`/api/recetas/${tipo}/${id}`, { method: 'POST', body: JSON.stringify(payload) })
    await loadRecipe()
    await refreshRecipe(tipo, id)
    expandedRecipes.value[`${tipo}-${id}`] = true
    message.value = 'Receta actualizada correctamente.'
  } catch (requestError) {
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

async function deleteRecipe(idInsumo, tipo = recetaSeleccionada.value.tipo, id = recetaSeleccionada.value.id) {
  loading.value = true
  clearFeedback()
  try {
    await apiFetch(`/api/recetas/${tipo}/${id}/insumos/${idInsumo}`, { method: 'DELETE' })
    await loadRecipe()
    await refreshRecipe(tipo, id)
    message.value = 'Insumo retirado de la receta.'
  } catch (requestError) {
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

async function createPurchase() {
  const form = forms.compra
  if (!form.detalles.length) {
    error.value = 'Agrega al menos un insumo a la compra.'
    return
  }
  if (!form.id_proveedor || !form.id_origen) {
    error.value = 'Selecciona proveedor y origen de pago.'
    return
  }
  const payload = {
    fecha: form.fecha,
    notas: form.notas || undefined,
    id_origen: Number(form.id_origen),
    id_proveedor: Number(form.id_proveedor),
    detalles: form.detalles.map((detalle) => ({
      id_insumo: Number(detalle.id_insumo),
      cantidad: Number(detalle.cantidad),
      precio: Number(detalle.precio),
    })),
  }

  await registerOperation('/api/compras', payload, 'Compra registrada correctamente.')
  form.notas = ''
  form.id_proveedor = ''
  form.id_origen = ''
  form.detalles = []
  purchaseView.value = 'registradas'
}

async function createSale() {
  const form = forms.venta
  if (!form.detalles.length) {
    error.value = 'Agrega al menos un producto al ticket.'
    return
  }
  if (!form.id_metodo_pago) {
    error.value = 'Selecciona un método de pago.'
    return
  }
  const paymentMethod = metodosPago.value.find(
    (item) => Number(item.id_metodo_pago) === Number(form.id_metodo_pago),
  )
  const isCash = /efectivo|cash/i.test(paymentMethod?.nombre || '')
  if (isCash && Number(cashReceived.value) < ticketSubtotal()) {
    error.value = 'El importe recibido no cubre el total del ticket.'
    return
  }

  const payload = salePayload(form)
  await registerOperation('/api/ventas', payload, 'Venta cobrada y registrada correctamente.')
  if (!error.value) {
    form.folio_nota = ''
    form.id_cliente = ''
    form.detalles = []
    cashReceived.value = ''
    showPayment.value = false
  }
}

async function registerOperation(endpoint, payload, successMessage) {
  loading.value = true
  clearFeedback()
  try {
    await apiFetch(endpoint, { method: 'POST', body: JSON.stringify(payload) })
    await loadData()
    message.value = successMessage
  } catch (requestError) {
    error.value = requestError.message
  } finally {
    loading.value = false
  }
}

function saveTicket() {
  if (!forms.venta.detalles.length) {
    error.value = 'Agrega al menos un producto antes de guardar el ticket.'
    return
  }

  const ticket = {
    id: `ticket-${Date.now()}`,
    created_at: new Date().toISOString(),
    venta: JSON.parse(JSON.stringify({
      consumo_local: forms.venta.consumo_local,
      requiere_nota: forms.venta.requiere_nota,
      folio_nota: forms.venta.folio_nota,
      id_metodo_pago: forms.venta.id_metodo_pago,
      id_cliente: forms.venta.id_cliente,
      detalles: forms.venta.detalles,
    })),
  }
  savedTickets.value.unshift(ticket)
  persistSavedTickets()
  forms.venta.detalles = []
  forms.venta.folio_nota = ''
  forms.venta.id_cliente = ''
  cashReceived.value = ''
  showPayment.value = false
  message.value = 'Ticket guardado en este navegador.'
}

function restoreSavedTicket(ticket) {
  Object.assign(forms.venta, JSON.parse(JSON.stringify(ticket.venta)))
  savedTickets.value = savedTickets.value.filter((item) => item.id !== ticket.id)
  persistSavedTickets()
  selectedVariantIndex.value = 0
  activeSection.value = 'ventas'
  clearFeedback()
  message.value = 'Ticket recuperado.'
}

function deleteSavedTicket(ticketId) {
  savedTickets.value = savedTickets.value.filter((item) => item.id !== ticketId)
  persistSavedTickets()
}

function openPayment() {
  if (!forms.venta.detalles.length) {
    error.value = 'Agrega al menos un producto al ticket.'
    return
  }
  if (!forms.venta.id_metodo_pago) {
    error.value = 'Selecciona un método de pago.'
    return
  }
  showPayment.value = true
  clearFeedback()
}

function reportUnsupportedOperation(operation) {
  error.value = `${operation} no está disponible: el backend actual no expone una operación de anulación/devolución ni una tabla para registrarla.`
}

function printTicket(venta) {
  if (!venta) return
  const lines = saleDetails(venta).flatMap((detail) => {
    const product = `${ticketVariantLabel(detail)} x${detail.cantidad}`
      .replace(' x', ' x')
    const productLine = `${product.padEnd(34, ' ')}${ticketProductAmount(detail).toFixed(2)}`
    const extras = saleExtras(detail).map((extra) =>
      `  -${saleExtraName(extra)} x${extra.cantidad}`.padEnd(34, ' ') +
      ticketExtraAmount(extra).toFixed(2),
    )
    return [productLine, ...extras]
  }).join('\n')
  const printWindow = window.open('', '_blank', 'width=420,height=600')
  if (!printWindow) {
    error.value = 'El navegador bloqueó la ventana de impresión.'
    return
  }
  printWindow.document.write(
    `<pre>NIMBUS\n#${venta.id_venta}\n${ticketDate(venta.fecha_hora || venta.fecha || venta.created_at)}\n${ticketClient(venta)}\n\n${lines}\n\nTOTAL${' '.repeat(29)}${Number(venta.total || 0).toFixed(2)}</pre>`,
  )
  printWindow.document.close()
  printWindow.focus()
  printWindow.print()
}

function selectSection(section) {
  activeSection.value = section
  clearFeedback()
  if (section === 'reportes') loadReports()
}

onMounted(() => {
  restoreSavedTickets()
  loadData()
})
</script>

<template>
  <div class="app-shell">
    <aside class="sidebar">
      <div class="brand">
        <div class="brand-mark">N</div>
        <div>
          <strong>Nimbus</strong>
          <span>Administración</span>
        </div>
      </div>

      <nav class="navigation" aria-label="Navegación principal">
        <div v-for="group in navGroups" :key="group.id" class="nav-group">
          <span class="nav-group-label">{{ group.label }}</span>
          <button
            v-for="section in sections.filter((item) => item.group === group.id)"
            :key="section.id"
            class="nav-item"
            :class="{ active: activeSection === section.id }"
            type="button"
            @click="selectSection(section.id)"
          >
            <span class="nav-icon">{{ section.icon }}</span>
            {{ section.label }}
          </button>
        </div>
      </nav>

      <div class="sidebar-footer">
        <span class="status-dot" :class="{ online: apiOnline }"></span>
        {{ apiOnline ? 'API conectada' : 'API sin conexión' }}
      </div>
    </aside>

    <main class="main-content">
      <header class="topbar">
        <div>
          <p class="eyebrow">Cafetería</p>
          <h1>{{ activeLabel }}</h1>
        </div>
        <button class="button secondary" type="button" :disabled="loading" @click="loadData">
          {{ loading ? 'Actualizando...' : 'Actualizar' }}
        </button>
      </header>

      <div v-if="message" class="feedback success" role="status">{{ message }}</div>
      <div v-if="error" class="feedback error" role="alert">
        {{ error }}
        <span class="error-hint">Verifica que el backend esté ejecutándose en {{ getApiUrl() }}.</span>
      </div>

      <section v-if="activeSection === 'reportes'" class="content-section">
        <div class="report-toolbar panel">
          <div>
            <p class="eyebrow">Análisis financiero</p>
            <h2>Reportes de ventas y compras</h2>
          </div>
          <div class="report-filters">
            <label>Desde<input v-model="reportFilters.fecha_desde" type="date" /></label>
            <label>Hasta<input v-model="reportFilters.fecha_hasta" type="date" /></label>
            <button class="button primary report-button" type="button" :disabled="reportLoading" @click="loadReports">
              {{ reportLoading ? 'Consultando...' : 'Consultar' }}
            </button>
          </div>
        </div>

        <div v-if="reportError" class="feedback error" role="alert">{{ reportError }}</div>
        <template v-if="report">
          <div class="report-stats">
            <article class="stat-card"><span>Ventas</span><strong>{{ formatCurrency(report.ventas?.total) }}</strong><small>{{ report.ventas?.cantidad || 0 }} operaciones</small></article>
            <article class="stat-card"><span>Compras</span><strong>{{ formatCurrency(report.compras?.total) }}</strong><small>{{ report.compras?.cantidad || 0 }} operaciones</small></article>
            <article class="stat-card"><span>Utilidad bruta</span><strong>{{ formatCurrency(report.utilidad_bruta) }}</strong><small>Ventas menos compras</small></article>
            <article class="stat-card"><span>Ticket promedio</span><strong>{{ formatCurrency(report.ventas?.ticket_promedio) }}</strong><small>Promedio por venta</small></article>
          </div>

          <div class="report-grid">
            <div class="panel">
              <div class="panel-heading"><div><p class="eyebrow">Ventas</p><h2>Desglose por método de pago</h2></div></div>
              <div v-if="reportEntries(report.ventas?.por_metodo_pago).length" class="report-list">
                <div v-for="[name, total] in reportEntries(report.ventas?.por_metodo_pago)" :key="name" class="report-row"><span>{{ name }}</span><strong>{{ formatCurrency(total) }}</strong></div>
              </div>
              <p v-else class="empty-state">No hay ventas en el periodo.</p>
            </div>
            <div class="panel">
              <div class="panel-heading"><div><p class="eyebrow">Ventas</p><h2>Desglose por producto</h2></div></div>
              <div v-if="reportEntries(report.ventas?.por_producto).length" class="report-list">
                <div v-for="[name, total] in reportEntries(report.ventas?.por_producto)" :key="name" class="report-row"><span>{{ name }}</span><strong>{{ formatCurrency(total) }}</strong></div>
              </div>
              <p v-else class="empty-state">No hay productos vendidos en el periodo.</p>
            </div>
            <div class="panel">
              <div class="panel-heading"><div><p class="eyebrow">Compras</p><h2>Desglose por proveedor</h2></div></div>
              <div v-if="reportEntries(report.compras?.por_proveedor).length" class="report-list">
                <div v-for="[name, total] in reportEntries(report.compras?.por_proveedor)" :key="name" class="report-row"><span>{{ name }}</span><strong>{{ formatCurrency(total) }}</strong></div>
              </div>
              <p v-else class="empty-state">No hay compras en el periodo.</p>
            </div>
            <div class="panel">
              <div class="panel-heading"><div><p class="eyebrow">Compras</p><h2>Desglose por insumo</h2></div></div>
              <div v-if="reportEntries(report.compras?.por_insumo).length" class="report-list">
                <div v-for="[name, total] in reportEntries(report.compras?.por_insumo)" :key="name" class="report-row"><span>{{ name }}</span><strong>{{ formatCurrency(total) }}</strong></div>
              </div>
              <p v-else class="empty-state">No hay insumos comprados en el periodo.</p>
            </div>
          </div>
        </template>
        <div v-else-if="reportLoading" class="panel empty-state">Cargando reporte...</div>
      </section>

      <section v-else-if="activeSection === 'productos'" class="content-section">
        <div class="catalog-layout">
          <div class="catalog-lists">
            <div class="panel">
              <div class="panel-heading">
                <div><p class="eyebrow">Productos y variantes</p><h2>Catálogo</h2></div>
                <span class="count">{{ combinedCatalog.length }}</span>
              </div>
              <label class="catalog-search">Buscar<input v-model="productSearch" type="search" placeholder="Nombre o categoría" /></label>
              <div v-if="filteredCombinedCatalog.length" class="table-wrap">
                <table>
                  <thead><tr><th>Producto</th><th>Categoría</th><th>Tamaño y precio</th><th>Estado</th></tr></thead>
                  <tbody>
                    <tr v-for="producto in filteredCombinedCatalog" :key="producto.id_producto">
                        <td><button class="catalog-name-button" type="button" @click="selectProductoEdit(producto)"><strong>{{ producto.nombre }}</strong></button></td>
                      <td>{{ producto.categoria }}</td>
                      <td>
                        <div v-if="producto.variantes.length" class="variant-price-grid">
                          <button v-for="variante in producto.variantes" :key="variante.id_variante" class="variant-price variant-edit-button" type="button" @click="selectVariantEdit(variante)">
                            <span class="variant-code">{{ variantCode(variante.tamano) }}</span>
                            <strong>{{ Number(variante.precio_actual).toFixed(2) }}</strong>
                          </button>
                        </div>
                        <strong v-else>{{ Number(producto.precio_actual || producto.precio || 0).toFixed(2) }}</strong>
                      </td>
                      <td><span class="tag" :class="{ inactive: !producto.activo }">{{ producto.activo ? 'Activo' : 'Inactivo' }}</span></td>
                    </tr>
                  </tbody>
                </table>
              </div>
              <p v-else class="empty-state">No hay productos para mostrar.</p>
            </div>

            <div class="panel">
              <div class="panel-heading">
                <div><p class="eyebrow">Complementos</p><h2>Extras</h2></div>
                <span class="count">{{ extras.length }}</span>
              </div>
              <div v-if="extras.length" class="extra-list">
                <div v-for="extra in extras" :key="extra.id_extra" class="extra-list-row">
                  <button class="catalog-name-button" type="button" @click="selectExtraEdit(extra)"><strong>{{ extra.nombre }}</strong></button>
                  <span>{{ Number(extra.precio_actual).toFixed(2) }}</span>
                </div>
              </div>
              <p v-else class="empty-state">No hay extras para mostrar.</p>
            </div>
          </div>

          <div class="catalog-forms panel form-panel">
            <div class="tab-bar catalog-form-tabs">
              <button class="tab-button" :class="{ active: catalogForm === 'producto' }" type="button" @click="catalogForm = 'producto'">Producto</button>
              <button class="tab-button" :class="{ active: catalogForm === 'variante' }" type="button" @click="catalogForm = 'variante'">Variante</button>
              <button class="tab-button" :class="{ active: catalogForm === 'extra' }" type="button" @click="catalogForm = 'extra'">Extra</button>
            </div>

            <form v-if="catalogForm === 'producto'" @submit.prevent="createProducto">
              <div class="details-heading"><div><p class="eyebrow">{{ catalogEditMode ? 'Editar producto' : 'Nuevo producto' }}</p><h2>{{ catalogEditMode ? 'Modificar producto' : 'Agregar producto' }}</h2></div><button v-if="catalogEditMode" class="link-button" type="button" @click="catalogEditMode = false; editingProductId = ''">Cancelar</button></div>
              <label>Nombre<input v-model.trim="forms.producto.nombre" required placeholder="Latte" /></label>
              <label>Categoría<input v-model.trim="forms.producto.categoria" placeholder="Café caliente" /></label>
              <button v-if="catalogEditMode" class="button primary" type="button" :disabled="loading" @click="updateProducto">Guardar cambios</button>
              <button v-else class="button primary" type="submit" :disabled="loading">Guardar producto</button>
            </form>
            <form v-else-if="catalogForm === 'variante'" @submit.prevent="createVariant">
              <div class="details-heading"><div><p class="eyebrow">{{ catalogEditMode ? 'Editar variante' : 'Nueva variante' }}</p><h2>{{ catalogEditMode ? 'Modificar variante' : 'Agregar variante' }}</h2></div><button v-if="catalogEditMode" class="link-button" type="button" @click="catalogEditMode = false; editingVariantId = ''">Cancelar</button></div>
              <label>Producto
                <select v-model="forms.variante.id_producto" required>
                  <option value="" disabled>Selecciona un producto</option>
                  <option v-for="producto in productos" :key="producto.id_producto" :value="producto.id_producto">{{ producto.nombre }}</option>
                </select>
              </label>
              <label>Tamaño<input v-model.trim="forms.variante.tamano" required placeholder="Mediano" /></label>
              <label>Precio actual<input v-model.number="forms.variante.precio_actual" min="0.01" required type="number" step="0.01" /></label>
              <button v-if="catalogEditMode" class="button primary" type="button" :disabled="loading" @click="updateVariant">Guardar cambios</button>
              <button v-else class="button primary" type="submit" :disabled="loading">Guardar variante</button>
            </form>
            <form v-else @submit.prevent="createExtra">
              <div class="details-heading"><div><p class="eyebrow">{{ catalogEditMode ? 'Editar extra' : 'Nuevo extra' }}</p><h2>{{ catalogEditMode ? 'Modificar extra' : 'Agregar extra' }}</h2></div><button v-if="catalogEditMode" class="link-button" type="button" @click="catalogEditMode = false; editingExtraId = ''">Cancelar</button></div>
              <label>Nombre<input v-model.trim="forms.extra.nombre" required placeholder="Leche de almendra" /></label>
              <label>Precio actual<input v-model.number="forms.extra.precio_actual" min="0" required type="number" step="0.01" /></label>
              <button v-if="catalogEditMode" class="button primary" type="button" :disabled="loading" @click="updateExtra">Guardar cambios</button>
              <button v-else class="button primary" type="submit" :disabled="loading">Guardar extra</button>
            </form>
          </div>
        </div>
      </section>

      <section v-else-if="activeSection === 'insumos'" class="content-section">
        <div class="section-grid">
          <div class="panel">
            <div class="panel-heading">
              <div><p class="eyebrow">Inventario</p><h2>Insumos activos</h2></div>
              <span class="count">{{ insumos.length }}</span>
            </div>
            <div v-if="insumos.length" class="table-wrap">
              <table>
                <thead><tr><th>Nombre</th><th>Unidad</th><th>Stock mínimo</th></tr></thead>
                <tbody>
                  <tr v-for="insumo in insumos" :key="insumo.id_insumo">
                    <td>{{ insumo.nombre }}</td><td>{{ insumo.unidad_medida }}</td><td>{{ insumo.stock_minimo }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <p v-else class="empty-state">No hay insumos para mostrar.</p>
          </div>

          <form class="panel form-panel" @submit.prevent="createInsumo">
            <p class="eyebrow">Nuevo registro</p>
            <h2>Agregar insumo</h2>
            <label>Nombre<input v-model.trim="forms.insumo.nombre" required placeholder="Café en grano" /></label>
            <label>Unidad de medida<input v-model.trim="forms.insumo.unidad_medida" required placeholder="kg" /></label>
            <label>Stock mínimo<input v-model.number="forms.insumo.stock_minimo" min="0" required type="number" step="0.001" /></label>
            <button class="button primary" type="submit" :disabled="loading">Guardar insumo</button>
          </form>
        </div>
      </section>

      <section v-else-if="activeSection === 'stock'" class="content-section">
        <div class="panel">
          <div class="panel-heading">
            <div><p class="eyebrow">Inventario</p><h2>Existencias actuales</h2></div>
            <span class="count">{{ stock.length }}</span>
          </div>
          <div v-if="stock.length" class="table-wrap">
            <table>
              <thead><tr><th>Insumo</th><th>Comprado</th><th>Consumido</th><th>Actual</th><th>Estado</th></tr></thead>
              <tbody>
                <tr v-for="item in stock" :key="item.id_insumo">
                  <td>{{ item.nombre }}</td>
                  <td>{{ item.total_comprado }}</td>
                  <td>{{ item.total_consumido }}</td>
                  <td><strong>{{ item.stock_actual }}</strong> {{ item.unidad_medida }}</td>
                  <td><span class="tag" :class="{ warning: item.necesita_reabastecimiento }">{{ item.necesita_reabastecimiento ? 'Reabastecer' : 'Correcto' }}</span></td>
                </tr>
              </tbody>
            </table>
          </div>
          <p v-else class="empty-state">No hay datos de stock para mostrar.</p>
        </div>
      </section>

      <section v-else-if="activeSection === 'clientes'" class="content-section">
        <div class="section-grid">
          <div class="panel">
            <div class="panel-heading">
              <div><p class="eyebrow">Clientes</p><h2>Registro de clientes</h2></div>
              <span class="count">{{ clientes.length }}</span>
            </div>
            <div v-if="clientes.length" class="table-wrap">
              <table>
                <thead><tr><th>Nombre</th><th>Contacto</th></tr></thead>
                <tbody><tr v-for="cliente in clientes" :key="cliente.id_cliente"><td>{{ cliente.nombre }}</td><td>{{ cliente.contacto || '—' }}</td></tr></tbody>
              </table>
            </div>
            <p v-else class="empty-state">No hay clientes para mostrar.</p>
          </div>

          <form class="panel form-panel" @submit.prevent="createCliente">
            <p class="eyebrow">Nuevo registro</p>
            <h2>Agregar cliente</h2>
            <label>Nombre<input v-model.trim="forms.cliente.nombre" required placeholder="María González" /></label>
            <label>Contacto<input v-model.trim="forms.cliente.contacto" placeholder="555-0101" /></label>
            <button class="button primary" type="submit" :disabled="loading">Guardar cliente</button>
          </form>
        </div>
      </section>

      <section v-else-if="activeSection === 'proveedores'" class="content-section">
        <div class="section-grid">
          <div class="panel">
            <div class="panel-heading">
              <div><p class="eyebrow">Compras</p><h2>Proveedores</h2></div><span class="count">{{ proveedores.length }}</span>
            </div>
            <div v-if="proveedores.length" class="table-wrap">
              <table><thead><tr><th>Nombre</th><th>Contacto</th></tr></thead>
                <tbody><tr v-for="proveedor in proveedores" :key="proveedor.id_proveedor"><td>{{ proveedor.nombre }}</td><td>{{ proveedor.contacto || '—' }}</td></tr></tbody>
              </table>
            </div>
            <p v-else class="empty-state">No hay proveedores para mostrar.</p>
          </div>
          <form class="panel form-panel" @submit.prevent="createProvider">
            <p class="eyebrow">Nuevo registro</p><h2>Agregar proveedor</h2>
            <label>Nombre<input v-model.trim="forms.proveedor.nombre" required placeholder="Café Sierra Madre" /></label>
            <label>Contacto<input v-model.trim="forms.proveedor.contacto" placeholder="ventas@proveedor.com" /></label>
            <button class="button primary" type="submit" :disabled="loading">Guardar proveedor</button>
          </form>
        </div>
      </section>

      <section v-else-if="activeSection === 'recetas'" class="content-section">
        <div class="recipe-header">
          <div>
            <p class="eyebrow">Configuración</p>
            <h2>Recetas</h2>
          </div>
          <button class="button secondary" type="button" @click="recipeEditOpen = true">Editar recetas</button>
        </div>

        <div v-if="recipeEditOpen" class="recipe-modal-backdrop" role="presentation" @click.self="recipeEditOpen = false">
          <div class="panel recipe-editor" role="dialog" aria-modal="true" aria-labelledby="recipe-editor-title">
            <div class="panel-heading">
              <div><p class="eyebrow">Editor</p><h2 id="recipe-editor-title">Agregar o retirar insumos</h2></div>
              <button class="remove-button" type="button" aria-label="Cerrar editor" @click="recipeEditOpen = false">×</button>
            </div>
            <form class="form-row recipe-editor-form" @submit.prevent="createRecipe">
            <label>Elemento
              <select v-model="recetaSeleccionada.id" required>
                <option value="" disabled>Selecciona</option>
                <template v-if="recetaSeleccionada.tipo === 'variantes'">
                  <option v-for="item in variantes" :key="item.id_variante" :value="item.id_variante">
                    {{ variantProductName(item) }} · {{ item.tamano || 'Sin tamaño' }}
                  </option>
                </template>
                <template v-else>
                  <option v-for="item in extras" :key="item.id_extra" :value="item.id_extra">{{ item.nombre }}</option>
                </template>
              </select>
            </label>
            <label>Tipo
              <select v-model="recetaSeleccionada.tipo" @change="recetaSeleccionada.id = ''">
                <option value="variantes">Variante</option>
                <option value="extras">Extra</option>
              </select>
            </label>
            <label>Insumo
              <select v-model="forms.receta.id_insumo" required>
                <option value="" disabled>Selecciona</option>
                <option v-for="insumo in insumos" :key="insumo.id_insumo" :value="insumo.id_insumo">{{ insumo.nombre }}</option>
              </select>
            </label>
            <label>Cantidad consumida<input v-model.number="forms.receta.cantidad_consumida" min="0.000001" required type="number" step="0.000001" /></label>
            <label v-if="recetaSeleccionada.tipo === 'variantes'" class="checkbox-label"><input v-model="forms.receta.solo_para_llevar" type="checkbox" /> Solo para llevar</label>
            <button class="button primary" type="submit" :disabled="loading">Agregar insumo</button>
          </form>
          </div>
        </div>

        <div class="recipe-columns">
          <div class="panel">
            <div class="panel-heading"><div><p class="eyebrow">Variantes</p><h2>Recetas de productos</h2></div><span class="count">{{ variantes.length }}</span></div>
            <div v-if="variantes.length" class="recipe-list">
              <div v-for="variante in variantes" :key="variante.id_variante" class="recipe-item">
                <div class="recipe-row" role="button" tabindex="0" @click="toggleRecipe('variantes', variante.id_variante)" @keydown.enter="toggleRecipe('variantes', variante.id_variante)">
                  <span class="recipe-triangle" :class="{ open: expandedRecipes[`variantes-${variante.id_variante}`] }">▸</span>
                  <strong>{{ variantProductName(variante) }}</strong>
                  <span class="variant-code">{{ variantCode(variante.tamano) }}</span>
                  <small>{{ variante.tamano || 'Sin tamaño' }}</small>
                  <button class="link-button recipe-edit" type="button" @click.stop="openRecipeEditor('variantes', variante.id_variante)">Editar</button>
                </div>
                <div v-if="expandedRecipes[`variantes-${variante.id_variante}`]" class="recipe-ingredients">
                  <table v-if="recipeDetails('variantes', variante.id_variante).length">
                    <thead><tr><th>ID</th><th>Insumo</th><th>Cantidad</th><th></th></tr></thead>
                    <tbody>
                      <tr v-for="receta in recipeDetails('variantes', variante.id_variante)" :key="receta.id_insumo">
                        <td>{{ receta.id_insumo }}</td>
                        <td>{{ ingredientName(receta) }}</td>
                        <td>{{ receta.cantidad_consumida }}</td>
                        <td v-if="recipeEditOpen"><button class="link-button" type="button" @click="deleteRecipe(receta.id_insumo, 'variantes', variante.id_variante)">Eliminar</button></td>
                        <td v-else></td>
                      </tr>
                    </tbody>
                  </table>
                  <p v-else class="empty-state">Sin insumos registrados.</p>
                </div>
              </div>
            </div>
            <p v-else class="empty-state">No hay variantes disponibles.</p>
          </div>

          <div class="panel">
            <div class="panel-heading"><div><p class="eyebrow">Extras</p><h2>Recetas de extras</h2></div><span class="count">{{ extras.length }}</span></div>
            <div v-if="extras.length" class="recipe-list">
              <div v-for="extra in extras" :key="extra.id_extra" class="recipe-item">
                <div class="recipe-row" role="button" tabindex="0" @click="toggleRecipe('extras', extra.id_extra)" @keydown.enter="toggleRecipe('extras', extra.id_extra)">
                  <span class="recipe-triangle" :class="{ open: expandedRecipes[`extras-${extra.id_extra}`] }">▸</span>
                  <strong>{{ extra.nombre }}</strong>
                  <small>{{ Number(extra.precio_actual).toFixed(2) }}</small>
                  <button class="link-button recipe-edit" type="button" @click.stop="openRecipeEditor('extras', extra.id_extra)">Editar</button>
                </div>
                <div v-if="expandedRecipes[`extras-${extra.id_extra}`]" class="recipe-ingredients">
                  <table v-if="recipeDetails('extras', extra.id_extra).length">
                    <thead><tr><th>ID</th><th>Insumo</th><th>Cantidad</th><th></th></tr></thead>
                    <tbody>
                      <tr v-for="receta in recipeDetails('extras', extra.id_extra)" :key="receta.id_insumo">
                        <td>{{ receta.id_insumo }}</td>
                        <td>{{ ingredientName(receta) }}</td>
                        <td>{{ receta.cantidad_consumida }}</td>
                        <td v-if="recipeEditOpen"><button class="link-button" type="button" @click="deleteRecipe(receta.id_insumo, 'extras', extra.id_extra)">Eliminar</button></td>
                        <td v-else></td>
                      </tr>
                    </tbody>
                  </table>
                  <p v-else class="empty-state">Sin insumos registrados.</p>
                </div>
              </div>
            </div>
            <p v-else class="empty-state">No hay extras disponibles.</p>
          </div>
        </div>
      </section>

      <section v-else-if="activeSection === 'compras'" class="content-section">
        <div class="tab-bar">
          <button class="tab-button" :class="{ active: purchaseView === 'registradas' }" type="button" @click="purchaseView = 'registradas'">Compras registradas</button>
          <button class="tab-button" :class="{ active: purchaseView === 'nueva' }" type="button" @click="purchaseView = 'nueva'">Nueva compra</button>
        </div>

        <div v-if="purchaseView === 'registradas'" class="panel">
          <div class="panel-heading"><div><p class="eyebrow">Operaciones</p><h2>Compras registradas</h2></div><span class="count">{{ compras.length }}</span></div>
          <div v-if="compras.length" class="table-wrap">
            <table><thead><tr><th>Fecha</th><th>Proveedor</th><th>Total</th></tr></thead>
              <tbody><tr v-for="compra in compras" :key="compra.id_compra"><td>{{ compra.fecha }}</td><td>{{ compra.proveedores?.nombre || compra.proveedor?.nombre || compra.id_proveedor }}</td><td>{{ compra.total }}</td></tr></tbody>
            </table>
          </div>
          <p v-else class="empty-state">No hay compras para mostrar.</p>
        </div>

        <div v-else class="pos-layout">
          <div class="panel pos-catalog">
            <div class="panel-heading">
              <div><p class="eyebrow">Inventario</p><h2>Selecciona insumos</h2></div>
              <span class="count">{{ insumos.length }}</span>
            </div>
            <div v-if="insumos.length" class="product-grid">
              <button v-for="insumo in insumos" :key="insumo.id_insumo" class="product-card" type="button" @click="addInsumoToPurchase(insumo)">
                <strong>{{ insumo.nombre }}</strong>
                <span>ID {{ insumo.id_insumo }} · {{ insumo.unidad_medida }}</span>
                <b>Agregar</b>
              </button>
            </div>
            <p v-else class="empty-state">No hay insumos disponibles.</p>
          </div>

          <form class="panel pos-ticket form-panel" @submit.prevent="createPurchase">
            <div class="panel-heading"><div><p class="eyebrow">Nueva operación</p><h2>Compra</h2></div><span class="count">{{ forms.compra.detalles.length }}</span></div>
            <label>Fecha<input v-model="forms.compra.fecha" required type="date" /></label>
            <label>Proveedor<select v-model="forms.compra.id_proveedor" required><option value="" disabled>Selecciona</option><option v-for="item in proveedores" :key="item.id_proveedor" :value="item.id_proveedor">{{ item.nombre }}</option></select></label>
            <label>Origen de pago<select v-model="forms.compra.id_origen" required><option value="" disabled>Selecciona</option><option v-for="item in origenesPago" :key="item.id_origen" :value="item.id_origen">{{ item.nombre }}</option></select></label>
            <div class="ticket-lines purchase-lines">
              <div v-if="!forms.compra.detalles.length" class="empty-state">Selecciona insumos del catálogo.</div>
              <div v-for="(detalle, index) in forms.compra.detalles" :key="`${detalle.id_insumo}-${index}`" class="ticket-line">
                <div class="ticket-line-main">
                  <button class="quantity-button" type="button" @click="changePurchaseQuantity(index, -1)">−</button>
                  <strong>{{ detalle.cantidad }}</strong>
                  <button class="quantity-button" type="button" @click="changePurchaseQuantity(index, 1)">+</button>
                  <div>
                    <b>{{ insumos.find((item) => Number(item.id_insumo) === Number(detalle.id_insumo))?.nombre || `Insumo ${detalle.id_insumo}` }}</b>
                    <span>ID {{ detalle.id_insumo }}</span>
                  </div>
                  <button class="remove-button" type="button" @click="removePurchaseItem(index)">×</button>
                </div>
                <label>Precio unitario<input v-model.number="detalle.precio" min="0" required type="number" step="0.01" placeholder="0.00" /></label>
              </div>
            </div>
            <label>Notas<input v-model.trim="forms.compra.notas" placeholder="Compra semanal" /></label>
            <div class="ticket-total"><span>Total estimado</span><strong>{{ purchaseTotal().toFixed(2) }}</strong></div>
            <button class="button primary" type="submit" :disabled="loading">Registrar compra</button>
          </form>
        </div>
      </section>

      <section v-else-if="activeSection === 'ventas'" class="content-section">
        <div class="pos-layout">
          <div class="panel pos-catalog">
            <div class="panel-heading">
              <div><p class="eyebrow">Catálogo</p><h2>Selecciona productos</h2></div>
              <span class="count">{{ variantes.length }}</span>
            </div>
            <label class="catalog-search">Buscar producto o tamaño<input v-model="variantSearch" type="search" placeholder="Latte, grande..." /></label>
            <div v-if="filteredVariants.length" class="product-grid">
              <button
                v-for="variante in filteredVariants"
                :key="variante.id_variante"
                class="product-card"
                type="button"
                @click="addVariantToTicket(variante)"
              >
                <strong>{{ variantProductName(variante) }}</strong>
                <span>{{ variante.tamano || 'Sin tamaño' }}</span>
                <b>{{ Number(variante.precio_actual).toFixed(2) }}</b>
              </button>
            </div>
            <p v-else class="empty-state">No hay variantes disponibles.</p>

            <div class="extras-section">
              <div class="details-heading"><h3>Extras</h3><span class="muted">Se agregan al producto seleccionado</span></div>
              <div v-if="extras.length" class="extra-grid">
                <button v-for="extra in extras" :key="extra.id_extra" class="extra-card" type="button" @click="addExtraToSelected(extra)">
                  <span>{{ extra.nombre }}</span>
                  <strong>+ {{ Number(extra.precio_actual).toFixed(2) }}</strong>
                </button>
              </div>
              <p v-else class="empty-state">No hay extras disponibles.</p>
            </div>
          </div>

          <aside class="panel pos-ticket">
            <div class="panel-heading">
              <div><p class="eyebrow">Ticket actual</p><h2>Venta</h2></div>
              <span class="count">{{ forms.venta.detalles.length }}</span>
            </div>

            <label>Método de pago
              <select v-model="forms.venta.id_metodo_pago" required>
                <option value="" disabled>Selecciona</option>
                <option v-for="item in metodosPago" :key="item.id_metodo_pago" :value="item.id_metodo_pago">{{ item.nombre }}</option>
              </select>
            </label>
            <label>Cliente (opcional)
              <select v-model="forms.venta.id_cliente">
                <option value="">Público general</option>
                <option v-for="item in clientes" :key="item.id_cliente" :value="item.id_cliente">{{ item.nombre }}</option>
              </select>
            </label>

            <div class="ticket-lines">
              <div v-if="!forms.venta.detalles.length" class="empty-state">Selecciona productos del catálogo.</div>
              <div
                v-for="(detalle, index) in forms.venta.detalles"
                :key="`${detalle.id_variante}-${index}`"
                class="ticket-line"
                :class="{ selected: selectedVariantIndex === index }"
                @click="selectedVariantIndex = index"
              >
                <div class="ticket-line-main">
                  <button class="quantity-button" type="button" @click.stop="changeTicketQuantity(index, -1)">−</button>
                  <strong>{{ detalle.cantidad }}</strong>
                  <button class="quantity-button" type="button" @click.stop="changeTicketQuantity(index, 1)">+</button>
                  <div>
                    <b>{{ variantProductName(findVariant(detalle.id_variante) || {}) }}</b>
                    <span>{{ findVariant(detalle.id_variante)?.tamano || 'Sin tamaño' }}</span>
                  </div>
                  <button class="remove-button" type="button" @click.stop="removeTicketDetail(index)">×</button>
                </div>
                <div v-for="(extra, extraIndex) in detalle.extras" :key="extraIndex" class="ticket-extra">
                  <span>+ {{ extra.cantidad }} × {{ findExtra(extra.id_extra)?.nombre || 'Extra' }}</span>
                  <button type="button" @click.stop="removeExtraFromDetail(index, extraIndex)">×</button>
                </div>
              </div>
            </div>

            <label class="checkbox-label"><input v-model="forms.venta.consumo_local" type="checkbox" /> Consumo local</label>
            <label class="checkbox-label"><input v-model="forms.venta.requiere_nota" type="checkbox" /> Requiere nota</label>
            <label v-if="forms.venta.requiere_nota">Folio de nota<input v-model.trim="forms.venta.folio_nota" required placeholder="A-000123" /></label>

            <div class="ticket-total"><span>Total</span><strong>{{ ticketSubtotal().toFixed(2) }}</strong></div>

            <div v-if="showPayment" class="payment-box">
              <label>Pago recibido<input v-model.number="cashReceived" min="0" type="number" step="0.01" placeholder="0.00" /></label>
              <div class="change-row"><span>Cambio</span><strong>{{ ticketChange().toFixed(2) }}</strong></div>
            </div>

            <div class="ticket-actions">
              <button class="button secondary" type="button" :disabled="loading" @click="saveTicket">Guardar ticket</button>
              <button v-if="!showPayment" class="button primary" type="button" :disabled="loading" @click="openPayment">Cobrar</button>
              <button v-else class="button primary" type="button" :disabled="loading" @click="createSale">Confirmar cobro</button>
            </div>

            <div v-if="savedTickets.length" class="saved-tickets">
              <div class="details-heading"><h3>Tickets guardados</h3><span class="count">{{ savedTickets.length }}</span></div>
              <div v-for="ticket in savedTickets" :key="ticket.id" class="saved-ticket">
                <div><strong>{{ ticket.venta.detalles.length }} producto(s)</strong><span>{{ new Date(ticket.created_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }) }}</span></div>
                <button class="link-button" type="button" @click="restoreSavedTicket(ticket)">Recuperar</button>
                <button class="remove-button" type="button" @click="deleteSavedTicket(ticket.id)">×</button>
              </div>
            </div>
          </aside>
        </div>
      </section>

      <section v-else-if="activeSection === 'tickets'" class="content-section">
        <div class="ticket-filters panel">
          <label>Buscar<input v-model="ticketSearch" type="search" placeholder="Folio, cliente o producto" /></label>
          <label>Desde<input v-model="ticketDateFrom" type="date" /></label>
          <label>Hasta<input v-model="ticketDateTo" type="date" /></label>
        </div>

        <div v-if="filteredSales.length" class="ticket-grid">
          <article v-for="venta in filteredSales" :key="venta.id_venta" class="receipt-card">
            <div class="receipt-header">
              <strong>NIMBUS</strong>
              <span>#{{ venta.id_venta }}</span>
            </div>
            <div class="receipt-meta">
              <span>{{ ticketDate(venta.fecha_hora || venta.fecha || venta.created_at) }}</span>
              <span>{{ ticketClient(venta) }}</span>
            </div>
            <div class="receipt-lines">
              <div v-for="(detalle, detailIndex) in saleDetails(venta)" :key="detailIndex" class="receipt-detail">
                <div class="receipt-line">
                  <span><strong>{{ ticketVariantLabel(detalle) }}</strong> x{{ detalle.cantidad }}</span>
                  <strong>{{ ticketProductAmount(detalle).toFixed(2) }}</strong>
                </div>
                <div v-for="(extra, extraIndex) in saleExtras(detalle)" :key="extraIndex" class="receipt-extra">
                  <span>-{{ saleExtraName(extra) }} x{{ extra.cantidad }}</span>
                  <span>{{ ticketExtraAmount(extra).toFixed(2) }}</span>
                </div>
              </div>
            </div>
            <div class="receipt-total">
              <span>Total</span>
              <strong>{{ Number(venta.total || 0).toFixed(2) }}</strong>
            </div>
            <div class="ticket-detail-actions">
              <button class="button secondary" type="button" @click="printTicket(venta)">Imprimir</button>
              <button class="button secondary" type="button" @click="reportUnsupportedOperation('Devolución')">Devolver</button>
            </div>
          </article>
        </div>
        <p v-else class="empty-state">No hay ventas que coincidan con el filtro.</p>
      </section>
    </main>
  </div>
</template>

<style>
:root {
  color: #263238;
  background: #f5f6f4;
  font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  font-synthesis: none;
}

* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; }
button, input { font: inherit; }
button { cursor: pointer; }

.app-shell { display: flex; min-height: 100vh; }
.sidebar { display: flex; flex-direction: column; width: 238px; padding: 28px 16px 18px; background: #fff; border-right: 1px solid #e6e9e5; }
.brand { display: flex; align-items: center; gap: 11px; padding: 0 10px 38px; }
.brand-mark { display: grid; width: 36px; height: 36px; place-items: center; border-radius: 10px; background: #304d42; color: #fff; font-weight: 700; }
.brand strong, .brand span { display: block; }
.brand strong { font-size: 15px; }
.brand span { margin-top: 3px; color: #87918c; font-size: 11px; }
.navigation { display: grid; gap: 5px; }
.nav-group { display: grid; gap: 4px; margin-bottom: 8px; }
.nav-group-label { padding: 8px 12px 3px; color: #a0a8a3; font-size: 9px; font-weight: 800; letter-spacing: .08em; text-transform: uppercase; }
.nav-item { display: flex; align-items: center; gap: 12px; padding: 11px 12px; border: 0; border-radius: 8px; background: transparent; color: #68736e; text-align: left; }
.nav-item:hover, .nav-item.active { background: #edf3ef; color: #304d42; }
.nav-icon { display: inline-grid; width: 20px; place-items: center; font-size: 14px; font-weight: 700; }
.sidebar-footer { display: flex; align-items: center; gap: 8px; margin-top: auto; padding: 14px 10px 0; border-top: 1px solid #edf0ed; color: #87918c; font-size: 12px; }
.status-dot { width: 7px; height: 7px; border-radius: 50%; background: #d78c75; }
.status-dot.online { background: #5e9d72; }
.main-content { width: min(1180px, 100%); margin: 0 auto; padding: 42px 46px; }
.topbar, .panel-heading { display: flex; align-items: flex-start; justify-content: space-between; gap: 20px; }
.topbar { align-items: center; margin-bottom: 32px; }
.eyebrow { margin: 0 0 7px; color: #829089; font-size: 11px; font-weight: 700; letter-spacing: .1em; text-transform: uppercase; }
h1, h2, p { margin-top: 0; }
h1 { margin-bottom: 0; font-size: 28px; letter-spacing: -.03em; }
h2 { margin-bottom: 9px; font-size: 19px; letter-spacing: -.02em; }
.content-section { display: grid; gap: 22px; }
.tab-bar { display: flex; gap: 6px; padding: 5px; border-radius: 9px; background: #e9eeea; }
.tab-button { padding: 10px 15px; border: 0; border-radius: 7px; background: transparent; color: #68736e; font-size: 13px; font-weight: 600; }
.tab-button.active { background: #fff; color: #304d42; box-shadow: 0 1px 3px #cfd8d1; }
.welcome-card { display: flex; align-items: center; justify-content: space-between; min-height: 180px; padding: 30px 34px; border-radius: 14px; background: #304d42; color: #fff; }
.welcome-card .eyebrow { color: #bdcfc5; }
.welcome-card h2 { margin-bottom: 10px; font-size: 26px; }
.welcome-card p:last-child { margin-bottom: 0; color: #d9e3dd; }
.welcome-icon { font-size: 54px; opacity: .8; }
.stats-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
.stat-card, .panel { padding: 22px; border: 1px solid #e5e9e5; border-radius: 12px; background: #fff; }
.stat-card span { display: block; color: #7d8983; font-size: 12px; }
.stat-card strong { display: block; margin-top: 13px; font-size: 27px; color: #304d42; }
.stat-card small { display: block; margin-top: 7px; color: #89938e; font-size: 11px; }
.report-toolbar { display: flex; align-items: flex-end; justify-content: space-between; gap: 22px; }
.report-toolbar h2 { margin-bottom: 0; }
.report-filters { display: flex; align-items: flex-end; gap: 10px; }
.report-button { margin: 0; white-space: nowrap; }
.report-stats { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
.report-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 22px; }
.report-list { display: grid; }
.report-row { display: flex; justify-content: space-between; gap: 14px; padding: 11px 0; border-top: 1px solid #edf0ed; color: #53635b; font-size: 13px; }
.report-row strong { color: #304d42; }
.panel-heading { align-items: center; margin-bottom: 22px; }
.panel-heading .eyebrow { margin-bottom: 5px; }
.panel-heading h2 { margin-bottom: 0; }
.count, .connection-badge { display: inline-flex; align-items: center; justify-content: center; min-width: 28px; padding: 5px 9px; border-radius: 20px; background: #edf3ef; color: #4f7461; font-size: 12px; font-weight: 700; }
.connection-badge { background: #f3eee9; color: #9c7658; }
.connection-badge.connected { background: #e9f4ec; color: #548362; }
.muted { margin: 6px 0; color: #78827d; font-size: 13px; }
code { padding: 2px 5px; border-radius: 4px; background: #f2f4f2; color: #53635b; font-size: 12px; }
.section-grid { display: grid; grid-template-columns: minmax(0, 1.65fr) minmax(260px, .75fr); align-items: start; gap: 22px; }
.form-panel { display: grid; gap: 14px; }
.form-panel h2 { margin-bottom: 5px; }
.details-heading, .detail-card-heading { display: flex; align-items: center; justify-content: space-between; gap: 10px; }
.details-heading { margin-top: 5px; }
.details-heading.compact { margin-top: 4px; }
.details-heading h3 { margin: 0; color: #53635b; font-size: 12px; }
.detail-card { display: grid; gap: 11px; padding: 14px; border: 1px solid #e5e9e5; border-radius: 8px; background: #fafcfb; }
.detail-card-heading { color: #53635b; font-size: 12px; }
.extra-row { display: grid; grid-template-columns: minmax(0, 1fr) 78px 18px; align-items: center; gap: 7px; }
.add-button { color: #4f7461; }
label { display: grid; gap: 6px; color: #56635c; font-size: 12px; font-weight: 600; }
input { width: 100%; padding: 10px 11px; border: 1px solid #dfe5e0; border-radius: 7px; outline: none; color: #263238; background: #fff; }
input:focus { border-color: #719681; box-shadow: 0 0 0 3px #edf3ef; }
select { width: 100%; padding: 10px 11px; border: 1px solid #dfe5e0; border-radius: 7px; outline: none; color: #263238; background: #fff; }
select:focus { border-color: #719681; box-shadow: 0 0 0 3px #edf3ef; }
.button { padding: 10px 14px; border: 1px solid transparent; border-radius: 7px; font-size: 13px; font-weight: 600; }
.button:disabled { cursor: wait; opacity: .6; }
.button.primary { margin-top: 5px; background: #304d42; color: #fff; }
.button.primary:hover:not(:disabled) { background: #223b32; }
.button.secondary { border-color: #dbe3dd; background: #fff; color: #4f6257; }
.button.secondary:hover:not(:disabled) { background: #f0f4f1; }
.table-wrap { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
th { padding: 0 12px 11px; color: #89938e; font-size: 11px; font-weight: 700; text-align: left; text-transform: uppercase; }
td { padding: 13px 12px; border-top: 1px solid #edf0ed; }
td:first-child, th:first-child { padding-left: 0; }
td:last-child, th:last-child { padding-right: 0; }
.sale-details { display: grid; gap: 7px; min-width: 240px; }
.sale-details > div { display: grid; gap: 2px; }
.sale-details span, .sale-details small { color: #78827d; font-size: 11px; }
.sale-details small { line-height: 1.4; }
.pos-layout { display: grid; grid-template-columns: minmax(0, 1fr) 330px; align-items: start; gap: 22px; }
.pos-catalog { min-height: 620px; }
.pos-ticket { position: sticky; top: 24px; display: grid; gap: 12px; }
.product-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 10px; }
.product-card { display: grid; gap: 5px; min-height: 112px; padding: 16px; border: 1px solid #dde4df; border-radius: 10px; background: #fafcfb; color: #263238; text-align: left; }
.product-card:hover { border-color: #719681; background: #f1f6f3; }
.product-card strong { font-size: 14px; }
.product-card span { color: #78827d; font-size: 12px; }
.product-card b { margin-top: auto; color: #304d42; font-size: 15px; }
.extras-section { margin-top: 24px; padding-top: 18px; border-top: 1px solid #edf0ed; }
.extra-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 8px; margin-top: 12px; }
.extra-card { display: flex; justify-content: space-between; gap: 8px; min-height: 48px; padding: 10px 12px; border: 1px solid #e5e9e5; border-radius: 8px; background: #fff; color: #53635b; font-size: 12px; text-align: left; }
.extra-card:hover { border-color: #719681; }
.ticket-lines { display: grid; max-height: 270px; gap: 7px; overflow-y: auto; }
.ticket-line { padding: 9px; border: 1px solid #edf0ed; border-radius: 8px; background: #fafcfb; cursor: pointer; }
.ticket-line.selected { border-color: #719681; background: #f2f6f3; }
.ticket-line-main { display: grid; grid-template-columns: 24px 18px 24px minmax(0, 1fr) 20px; align-items: center; gap: 4px; }
.ticket-line-main > div { display: grid; min-width: 0; gap: 2px; }
.ticket-line-main b { overflow: hidden; font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }
.ticket-line-main span { color: #78827d; font-size: 11px; }
.quantity-button, .remove-button { display: grid; width: 24px; height: 24px; padding: 0; place-items: center; border: 0; border-radius: 50%; background: #edf3ef; color: #53635b; }
.remove-button { margin-left: auto; background: transparent; color: #a2584c; }
.ticket-extra { display: flex; justify-content: space-between; align-items: center; margin: 5px 0 0 70px; color: #78827d; font-size: 10px; }
.ticket-extra button { border: 0; background: transparent; color: #a2584c; }
.ticket-total { display: flex; justify-content: space-between; align-items: center; padding: 13px 0 5px; border-top: 1px solid #edf0ed; }
.ticket-total strong { color: #304d42; font-size: 22px; }
.payment-box { display: grid; gap: 7px; padding: 12px; border-radius: 8px; background: #f4f6f4; }
.change-row { display: flex; justify-content: space-between; color: #53635b; font-size: 13px; }
.change-row strong { color: #304d42; }
.ticket-actions { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
.ticket-actions .button { margin: 0; }
.ticket-filters { display: grid; grid-template-columns: minmax(200px, 1fr) 150px 150px; gap: 10px; margin-bottom: 16px; }
.ticket-filters label { align-self: end; }
.ticket-detail-actions { display: flex; justify-content: flex-end; gap: 8px; margin-top: 18px; }
.ticket-detail-actions .button { margin: 0; }
.saved-tickets { display: grid; gap: 6px; padding-top: 10px; border-top: 1px solid #edf0ed; }
.saved-ticket { display: grid; grid-template-columns: minmax(0, 1fr) auto 20px; align-items: center; gap: 7px; }
.saved-ticket > div { display: grid; font-size: 11px; }
.saved-ticket span { color: #89938e; }
.tag { display: inline-block; padding: 4px 8px; border-radius: 12px; background: #eaf3ec; color: #4e835e; font-size: 11px; font-weight: 600; }
.tag.inactive { background: #edf0ed; color: #6f7973; }
.variant-format { display: flex; align-items: center; gap: 8px; }
.variant-code { display: inline-grid; width: 30px; height: 30px; place-items: center; border: 1px solid #cfd7d2; border-radius: 7px; background: #edf1ee; color: #405149; font-size: 12px; font-weight: 800; }
.catalog-layout { display: grid; grid-template-columns: minmax(0, 1fr) 320px; align-items: start; gap: 22px; }
.catalog-lists { display: grid; gap: 22px; }
.catalog-forms { display: grid; align-content: start; gap: 14px; }
.catalog-search { display: grid; gap: 6px; margin: 0 0 12px; color: #56635c; font-size: 12px; font-weight: 600; }
.catalog-name-button { padding: 0; border: 0; background: transparent; color: inherit; text-align: left; }
.variant-edit-button { padding: 0; border: 0; background: transparent; cursor: pointer; }
.catalog-forms form { display: grid; gap: 14px; }
.catalog-form-tabs { margin-bottom: 4px; }
.extra-list { display: grid; }
.extra-list-row { display: flex; align-items: center; justify-content: space-between; padding: 11px 0; border-top: 1px solid #edf0ed; font-size: 13px; }
.extra-list-row span { color: #53635b; font-weight: 700; }
.recipe-header { display: flex; align-items: center; justify-content: space-between; }
.recipe-columns { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); align-items: start; gap: 22px; }
.recipe-list { display: grid; }
.recipe-item { border-top: 1px solid #edf0ed; }
.recipe-row { display: grid; grid-template-columns: 18px minmax(0, 1fr) 32px minmax(70px, auto) auto; align-items: center; gap: 8px; width: 100%; min-height: 54px; padding: 8px 0; border: 0; background: transparent; color: #263238; text-align: left; cursor: pointer; }
.recipe-row strong { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.recipe-row small { color: #78827d; }
.recipe-triangle { color: #78827d; transition: transform .15s ease; }
.recipe-triangle.open { transform: rotate(90deg); }
.recipe-edit { justify-self: end; }
.recipe-ingredients { padding: 5px 0 12px 26px; }
.recipe-ingredients table { margin-top: 4px; }
.recipe-editor { display: grid; gap: 10px; }
.recipe-editor-form { grid-template-columns: repeat(2, minmax(0, 1fr)); align-items: end; }
.recipe-modal-backdrop { position: fixed; inset: 0; z-index: 100; display: grid; place-items: center; padding: 24px; background: rgba(38, 50, 56, .42); }
.recipe-editor { width: min(680px, 100%); max-height: 82vh; overflow-y: auto; box-shadow: 0 18px 50px rgba(38, 50, 56, .22); }
.ticket-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 18px; }
.receipt-card { padding: 22px 20px 16px; border: 1px solid #dfe5e0; border-radius: 10px; background: #fff; box-shadow: 0 5px 18px rgba(64, 81, 73, .06); }
.receipt-header { display: flex; justify-content: space-between; align-items: center; padding-bottom: 12px; border-bottom: 1px dashed #cfd7d2; font-size: 13px; }
.receipt-header strong { letter-spacing: .12em; }
.receipt-header span { color: #89938e; font-size: 11px; }
.receipt-meta { display: flex; justify-content: space-between; gap: 10px; padding: 11px 0; color: #53635b; font-size: 11px; }
.receipt-lines { min-height: 90px; padding: 8px 0; border-top: 1px solid #edf0ed; border-bottom: 1px dashed #cfd7d2; }
.receipt-detail { margin-bottom: 8px; }
.receipt-line, .receipt-extra { display: flex; justify-content: space-between; gap: 12px; font-size: 12px; }
.receipt-line span { min-width: 0; }
.receipt-extra { padding: 3px 0 0 14px; color: #78827d; font-size: 11px; }
.receipt-total { display: flex; justify-content: space-between; padding: 12px 0 7px; }
.receipt-total strong { font-size: 17px; }
.variant-price-grid { display: flex; align-items: center; gap: 8px; }
.variant-price { display: grid; justify-items: center; gap: 5px; }
.variant-price strong { color: #53635b; font-size: 12px; }
.na-value { color: #89938e; font-size: 12px; }
.tag.warning { background: #fbf0e5; color: #ad7448; }
.empty-state { margin: 0; padding: 25px 0 5px; color: #89938e; font-size: 13px; text-align: center; }
.two-panels { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 22px; }
.form-row { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 12px; }
.simple-list { display: grid; gap: 9px; margin: 0 0 18px; padding: 0; list-style: none; color: #53635b; font-size: 13px; }
.simple-list li { display: flex; justify-content: space-between; padding-bottom: 9px; border-bottom: 1px solid #edf0ed; }
.inline-form { display: flex; gap: 8px; }
.inline-form input { min-width: 0; }
.link-button { padding: 0; border: 0; background: transparent; color: #a2584c; font-size: 12px; }
.checkbox-label { display: flex; grid-template-columns: 16px auto; align-items: center; gap: 8px; }
.checkbox-label input { width: 15px; height: 15px; }
.feedback { margin: -12px 0 22px; padding: 11px 14px; border-radius: 8px; font-size: 13px; }
.feedback.success { background: #eaf3ec; color: #487457; }
.feedback.error { background: #fbecea; color: #a2584c; }
.error-hint { display: block; margin-top: 4px; font-size: 12px; opacity: .85; }

@media (max-width: 800px) {
  .sidebar { width: 70px; padding: 20px 10px; }
  .brand { justify-content: center; padding: 0 0 28px; }
  .brand > div:last-child, .nav-item:not(.active)::after, .nav-item { font-size: 0; }
  .nav-item { justify-content: center; padding: 12px; }
  .nav-icon { font-size: 14px; }
  .sidebar-footer { justify-content: center; padding: 14px 0 0; font-size: 0; }
  .main-content { padding: 28px 20px; }
  .stats-grid { grid-template-columns: repeat(2, 1fr); }
  .report-stats { grid-template-columns: repeat(2, 1fr); }
  .report-grid { grid-template-columns: 1fr; }
  .report-toolbar { align-items: stretch; flex-direction: column; }
  .report-filters { flex-wrap: wrap; }
  .section-grid { grid-template-columns: 1fr; }
  .two-panels { grid-template-columns: 1fr; }
  .pos-layout { grid-template-columns: 1fr; }
  .pos-ticket { position: static; }
  .catalog-layout { grid-template-columns: 1fr; }
  .catalog-forms { grid-template-columns: 1fr; }
  .recipe-columns { grid-template-columns: 1fr; }
  .recipe-editor-form { grid-template-columns: 1fr; }
  .ticket-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .ticket-filters { grid-template-columns: 1fr 1fr; }
  .product-grid, .extra-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}

@media (max-width: 500px) {
  .topbar { align-items: flex-start; flex-direction: column; }
  .welcome-card { padding: 24px; }
  .welcome-icon { display: none; }
  .stats-grid { gap: 9px; }
  .report-stats { gap: 9px; }
  .stat-card { padding: 16px; }
  .form-row { grid-template-columns: 1fr; }
  .inline-form { flex-wrap: wrap; }
  .extra-row { grid-template-columns: minmax(0, 1fr) 60px 18px; }
  .product-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .extra-grid { grid-template-columns: 1fr; }
  .catalog-forms { grid-template-columns: 1fr; }
  .ticket-filters { grid-template-columns: 1fr; }
  .ticket-detail-actions { flex-direction: column; }
  .ticket-grid { grid-template-columns: 1fr; }
  .recipe-editor { position: static; }
}
</style>
