<script setup>
import { ref, computed, watch, onMounted, onUnmounted, nextTick } from 'vue'
import { Html5Qrcode, Html5QrcodeSupportedFormats } from 'html5-qrcode'

// ── LocalStorage Helpers ────────────────────────────────────────────────────
function ls(key, fb) {
  try { return JSON.parse(localStorage.getItem(key) || 'null') ?? fb } catch { return fb }
}
function lsSet(key, val) {
  try { localStorage.setItem(key, JSON.stringify(val)) } catch (_) {}
}

// ── State ───────────────────────────────────────────────────────────────────
// activeTab: 'register' | 'history'
const activeTab = ref('register')

// registerView on mobile: 'scan' | 'product' | 'manual' | 'cart'
const mobileView = ref('scan')

// Modals
const showPaymentModal = ref(false)
const showReceiptModal = ref(false)

// Data state
const error         = ref('')
const product       = ref(null)
const cart          = ref(ls('pos_cart', []))
const receiptRecord = ref(null)
const priceMemory   = ref(ls('pos_prices', {}))
const salesHistory  = ref(ls('pos_history', []))

// Manual inputs
const manualBarcode = ref('')
const manualForm    = ref({ name: '', brand: '', price: 0, barcode: '' })

// Camera (Shutter Mode)
const isCameraActive  = ref(false)
const videoRef        = ref(null)
const canvasRef       = ref(null)
const imageFileRef    = ref(null)
const capturedDataUrl = ref('')
const cameraReady     = ref(false)
const decoding        = ref(false)
const scanError       = ref('')
const isDragOver      = ref(false)

// Payment
const TAX_RATE     = 0.07 // 7% VAT
const payMethod    = ref('cash') // 'cash' | 'card' | 'promptpay'
const cashReceived = ref('')

let stream = null

// ── Watchers & Storage ──────────────────────────────────────────────────────
watch(cart,         c => lsSet('pos_cart',    c), { deep: true })
watch(priceMemory,  p => lsSet('pos_prices',  p), { deep: true })
watch(salesHistory, h => lsSet('pos_history', h), { deep: true })

// ── Computed Properties ─────────────────────────────────────────────────────
const cartCount = computed(() => cart.value.reduce((sum, item) => sum + item.qty, 0))
const cartSubtotal = computed(() => cart.value.reduce((sum, item) => sum + item.qty * item.price, 0))
const cartTax = computed(() => cartSubtotal.value * TAX_RATE)
const cartTotal = computed(() => cartSubtotal.value + cartTax.value)

const changeOwed = computed(() => {
  const paid = parseFloat(cashReceived.value) || 0
  return Math.max(0, paid - cartTotal.value)
})

const canCompletePayment = computed(() => {
  if (cart.value.length === 0) return false
  if (payMethod.value === 'cash') {
    const paid = parseFloat(cashReceived.value) || 0
    return paid >= cartTotal.value
  }
  return true
})

// ── Audio Feedback ──────────────────────────────────────────────────────────
function playBeep() {
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)()
    const osc = ctx.createOscillator()
    const gain = ctx.createGain()
    osc.connect(gain)
    gain.connect(ctx.destination)
    osc.type = 'sine'
    osc.frequency.setValueAtTime(1400, ctx.currentTime)
    gain.gain.setValueAtTime(0.2, ctx.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.0001, ctx.currentTime + 0.12)
    osc.start(ctx.currentTime)
    osc.stop(ctx.currentTime + 0.12)
  } catch (_) {}
}

// ── Barcode Formats ─────────────────────────────────────────────────────────
const FORMATS = [
  Html5QrcodeSupportedFormats.EAN_13,
  Html5QrcodeSupportedFormats.EAN_8,
  Html5QrcodeSupportedFormats.UPC_A,
  Html5QrcodeSupportedFormats.UPC_E,
  Html5QrcodeSupportedFormats.UPC_EAN_EXTENSION,
  Html5QrcodeSupportedFormats.CODE_128,
  Html5QrcodeSupportedFormats.CODE_39,
  Html5QrcodeSupportedFormats.CODE_93,
  Html5QrcodeSupportedFormats.ITF,
  Html5QrcodeSupportedFormats.QR_CODE,
  Html5QrcodeSupportedFormats.DATA_MATRIX,
]

// ── Camera Management ───────────────────────────────────────────────────────
async function openCamera() {
  error.value = ''
  scanError.value = ''
  capturedDataUrl.value = ''
  cameraReady.value = false
  isCameraActive.value = true

  await nextTick()
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: 'environment',
        width: { ideal: 1280 },
        height: { ideal: 720 },
      },
    })
    if (videoRef.value) {
      videoRef.value.srcObject = stream
      videoRef.value.onloadedmetadata = () => {
        cameraReady.value = true
      }
    }
  } catch (e) {
    error.value = `Camera access denied or unavailable: ${e.message}`
    isCameraActive.value = false
  }
}

function closeCamera() {
  stopStream()
  isCameraActive.value = false
  capturedDataUrl.value = ''
  scanError.value = ''
}

function stopStream() {
  if (stream) {
    stream.getTracks().forEach(t => t.stop())
    stream = null
  }
  cameraReady.value = false
}

// ── Shutter: Capture Frame & Decode Barcode ─────────────────────────────────
async function captureAndDecode() {
  if (!cameraReady.value || decoding.value || !videoRef.value || !canvasRef.value) return
  scanError.value = ''

  const video = videoRef.value
  const canvas = canvasRef.value
  canvas.width = video.videoWidth || 1280
  canvas.height = video.videoHeight || 720
  const ctx = canvas.getContext('2d')
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height)

  capturedDataUrl.value = canvas.toDataURL('image/jpeg', 0.92)
  decoding.value = true

  canvas.toBlob(async (blob) => {
    if (!blob) {
      decoding.value = false
      scanError.value = 'Capture failed. Please try again.'
      return
    }
    const file = new File([blob], 'capture.jpg', { type: 'image/jpeg' })
    try {
      const scanner = new Html5Qrcode('__img_scanner__', { formatsToSupport: FORMATS, verbose: false })
      const result = await scanner.scanFile(file, false)
      try { scanner.clear() } catch (_) {}
      playBeep()
      decoding.value = false
      closeCamera()
      fetchProduct(result.trim())
    } catch {
      decoding.value = false
      scanError.value = 'No barcode detected. Ensure good lighting and hold barcode flat, or try retaking.'
    }
  }, 'image/jpeg', 0.92)
}

function retake() {
  capturedDataUrl.value = ''
  scanError.value = ''
}

// ── Image Import / Drag & Drop ──────────────────────────────────────────────
function triggerImageUpload() {
  imageFileRef.value?.click()
}

async function handleFileSelect(event) {
  const file = event.target.files?.[0]
  if (!file) return
  event.target.value = ''
  processImageFile(file)
}

function handleDrop(e) {
  isDragOver.value = false
  const file = e.dataTransfer?.files?.[0]
  if (file && file.type.startsWith('image/')) {
    processImageFile(file)
  }
}

async function processImageFile(file) {
  capturedDataUrl.value = URL.createObjectURL(file)
  scanError.value = ''
  decoding.value = true
  isCameraActive.value = false

  try {
    const scanner = new Html5Qrcode('__img_scanner__', { formatsToSupport: FORMATS, verbose: false })
    const result = await scanner.scanFile(file, false)
    try { scanner.clear() } catch (_) {}
    playBeep()
    decoding.value = false
    capturedDataUrl.value = ''
    fetchProduct(result.trim())
  } catch {
    decoding.value = false
    scanError.value = 'No barcode detected in image. Try a higher-resolution photo.'
  }
}

// ── Manual Barcode Search ───────────────────────────────────────────────────
function submitManualBarcode() {
  const code = manualBarcode.value.trim()
  if (!code) return
  manualBarcode.value = ''
  fetchProduct(code)
}

// ── Open Food Facts API Integration ─────────────────────────────────────────
const isFetchingProduct = ref(false)

async function fetchProduct(barcode) {
  error.value = ''
  product.value = null
  isFetchingProduct.value = true
  mobileView.value = 'product'

  try {
    const res = await fetch(`https://world.openfoodfacts.org/api/v2/product/${barcode}.json`)
    const data = await res.json()

    if (data.status === 1 && data.product) {
      const p = data.product
      const savedPrice = priceMemory.value[barcode] ?? 0

      product.value = {
        barcode,
        name: p.product_name || p.product_name_en || 'Unknown Product',
        brand: p.brands || '',
        quantity: p.quantity || '',
        categories: p.categories_tags
          ? p.categories_tags.slice(0, 3).map(c => c.replace(/^en:/, '')).join(', ')
          : '',
        image: p.image_front_url || p.image_url || '',
        nutriScore: p.nutriscore_grade || '',
        price: savedPrice,
      }
    } else {
      // Product not found in Open Food Facts -> Prompt manual entry
      const savedPrice = priceMemory.value[barcode] ?? 0
      manualForm.value = {
        name: '',
        brand: '',
        price: savedPrice,
        barcode,
      }
      mobileView.value = 'manual'
    }
  } catch (e) {
    error.value = `Open Food Facts API error: ${e.message}`
    mobileView.value = 'scan'
  } finally {
    isFetchingProduct.value = false
  }
}

// ── Add to Cart & Price Management ──────────────────────────────────────────
function addScannedProductToCart() {
  if (!product.value) return
  addItemToCart(product.value)
  product.value = null
  mobileView.value = 'scan'
}

function saveManualProduct() {
  if (!manualForm.value.name.trim()) return
  addItemToCart({
    barcode: manualForm.value.barcode,
    name: manualForm.value.name.trim(),
    brand: manualForm.value.brand.trim(),
    quantity: '',
    categories: '',
    image: '',
    nutriScore: '',
    price: parseFloat(manualForm.value.price) || 0,
  })
  manualForm.value = { name: '', brand: '', price: 0, barcode: '' }
  mobileView.value = 'scan'
}

function addItemToCart(item) {
  const price = parseFloat(item.price) || 0
  if (price > 0 && item.barcode) {
    priceMemory.value[item.barcode] = price
  }

  const existing = cart.value.find(i => i.barcode === item.barcode)
  if (existing) {
    existing.qty++
    existing.price = price
  } else {
    cart.value.push({ ...item, price, qty: 1 })
  }
}

function changeQty(barcode, delta) {
  const item = cart.value.find(i => i.barcode === barcode)
  if (!item) return
  item.qty += delta
  if (item.qty <= 0) {
    cart.value = cart.value.filter(i => i.barcode !== barcode)
  }
}

function removeItem(barcode) {
  cart.value = cart.value.filter(i => i.barcode !== barcode)
}

function clearCart() {
  cart.value = []
}

// ── Payment & Checkout ──────────────────────────────────────────────────────
function openCheckout() {
  if (cart.value.length === 0) return
  payMethod.value = 'cash'
  cashReceived.value = ''
  showPaymentModal.value = true
}

function setQuickCash(amount) {
  if (amount === 'exact') {
    cashReceived.value = Math.ceil(cartTotal.value).toString()
  } else {
    cashReceived.value = amount.toString()
  }
}

function finalizePayment() {
  if (!canCompletePayment.value) return

  const record = {
    id: 'TX-' + Date.now().toString().slice(-6),
    timestamp: new Date().toISOString(),
    formattedDate: new Date().toLocaleString(),
    method: payMethod.value,
    items: cart.value.map(i => ({ ...i })),
    subtotal: cartSubtotal.value,
    tax: cartTax.value,
    total: cartTotal.value,
    cash: payMethod.value === 'cash' ? parseFloat(cashReceived.value) || 0 : null,
    change: payMethod.value === 'cash' ? changeOwed.value : null,
  }

  salesHistory.value = [record, ...salesHistory.value].slice(0, 100)
  receiptRecord.value = record
  cart.value = []
  showPaymentModal.value = false
  showReceiptModal.value = true
}

function closeReceipt() {
  showReceiptModal.value = false
  receiptRecord.value = null
  mobileView.value = 'scan'
}

function printReceipt() {
  window.print()
}

// ── Keyboard Shortcuts (PC Ergonomics) ──────────────────────────────────────
function handleKeydown(e) {
  if (e.key === 'Escape') {
    if (showPaymentModal.value) showPaymentModal.value = false
    else if (showReceiptModal.value) closeReceipt()
    else if (isCameraActive.value) closeCamera()
    else if (product.value) product.value = null
  }
  if (e.key === ' ' && isCameraActive.value && cameraReady.value && !decoding.value && !capturedDataUrl.value) {
    e.preventDefault()
    captureAndDecode()
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  stopStream()
})
</script>

<template>
  <div class="pos-app">

    <!-- ════════════════ TOP NAVIGATION BAR ════════════════ -->
    <header class="app-nav">
      <div class="nav-container">
        <div class="brand">
          <div class="brand-logo">🛒</div>
          <div class="brand-text">
            <h1 class="brand-title">BanMai POS</h1>
            <span class="brand-tag">Terminal & Barcode Scanner</span>
          </div>
        </div>

        <!-- Desktop Navigation Tabs -->
        <nav class="nav-tabs">
          <button
            class="tab-btn"
            :class="{ active: activeTab === 'register' }"
            @click="activeTab = 'register'"
          >
            <span class="tab-icon">⚡</span> Register
          </button>
          <button
            class="tab-btn"
            :class="{ active: activeTab === 'history' }"
            @click="activeTab = 'history'"
          >
            <span class="tab-icon">📜</span> Sales History ({{ salesHistory.length }})
          </button>
        </nav>

        <!-- Cart Quick Button (Mobile trigger or Quick info) -->
        <div class="nav-actions">
          <button
            class="mobile-cart-btn"
            :class="{ 'has-items': cartCount > 0 }"
            @click="mobileView = mobileView === 'cart' ? 'scan' : 'cart'"
          >
            <span class="btn-icon">🛍</span>
            <span class="cart-label">Cart</span>
            <span class="badge">{{ cartCount }}</span>
          </button>
        </div>
      </div>
    </header>

    <!-- ════════════════ MAIN CONTENT AREA ════════════════ -->
    <main class="app-main">

      <!-- ─────── TAB 1: REGISTER (RESPONSIVE SPLIT ON PC) ─────── -->
      <div v-if="activeTab === 'register'" class="register-layout">

        <!-- ── LEFT PANE: SCANNER & PRODUCT CONTROLS ── -->
        <section
          class="pane operation-pane"
          :class="{ 'mobile-hidden': mobileView === 'cart' }"
          @dragover.prevent="isDragOver = true"
          @dragleave.prevent="isDragOver = false"
          @drop.prevent="handleDrop"
        >
          <!-- Hidden file input for image import -->
          <input
            ref="imageFileRef"
            type="file"
            accept="image/*"
            style="display: none"
            @change="handleFileSelect"
          />

          <!-- Quick Action Buttons -->
          <div class="action-bar">
            <button
              class="primary-btn pulse-glow"
              :class="{ active: isCameraActive }"
              @click="isCameraActive ? closeCamera() : openCamera()"
            >
              <span class="btn-icon">📷</span>
              {{ isCameraActive ? 'Close Camera' : 'Camera Shutter' }}
            </button>

            <button class="outline-btn" @click="triggerImageUpload">
              <span class="btn-icon">🖼</span>
              Import Image
            </button>

            <div class="drag-hint hidden-mobile">
              <span>or drag & drop barcode image here</span>
            </div>
          </div>

          <!-- Manual Barcode Search Input -->
          <div class="barcode-search-box">
            <div class="search-input-wrapper">
              <span class="search-icon">🔍</span>
              <input
                v-model="manualBarcode"
                type="text"
                inputmode="numeric"
                class="search-input"
                placeholder="Scan or enter barcode (e.g. 051111407592)"
                @keyup.enter="submitManualBarcode"
              />
              <button
                class="search-submit-btn"
                :disabled="!manualBarcode.trim()"
                @click="submitManualBarcode"
              >
                Search
              </button>
            </div>
          </div>

          <!-- Alert / Error message -->
          <div v-if="error" class="alert-banner error">
            <span>⚠ {{ error }}</span>
            <button class="close-alert" @click="error = ''">✕</button>
          </div>

          <!-- Dropzone overlay when dragging -->
          <div v-if="isDragOver" class="drop-overlay">
            <div class="drop-card">
              <span class="drop-icon">📥</span>
              <p>Drop barcode image to scan</p>
            </div>
          </div>

          <!-- ── LIVE CAMERA VIEW WITH SHUTTER ── -->
          <div v-if="isCameraActive" class="camera-card">
            <div class="camera-header">
              <span class="status-indicator live">● LIVE CAMERA</span>
              <button class="close-btn" @click="closeCamera">✕</button>
            </div>

            <!-- Viewfinder Mode -->
            <div v-if="!capturedDataUrl" class="viewfinder-container">
              <video ref="videoRef" class="video-feed" autoplay playsinline muted />
              <div class="scanner-guide">
                <div class="corner corner-tl"></div>
                <div class="corner corner-tr"></div>
                <div class="corner corner-bl"></div>
                <div class="corner corner-br"></div>
                <div class="aim-line"></div>
              </div>
              <p class="guide-text">Position barcode in frame and press Shutter</p>

              <!-- Camera Controls -->
              <div class="camera-footer">
                <button
                  class="shutter-trigger"
                  :disabled="!cameraReady || decoding"
                  title="Capture snapshot (Spacebar)"
                  @click="captureAndDecode"
                >
                  <div class="shutter-ring">
                    <div class="shutter-dot"></div>
                  </div>
                </button>
              </div>
            </div>

            <!-- Captured Snapshot Mode -->
            <div v-else class="snapshot-container">
              <div class="snapshot-view">
                <img :src="capturedDataUrl" class="snapshot-image" alt="Captured Frame" />
                <div v-if="decoding" class="decoding-scrim">
                  <div class="spinner"></div>
                  <p>Analyzing barcode from snapshot...</p>
                </div>
              </div>

              <div v-if="scanError" class="scan-error-msg">
                <span>⚠ {{ scanError }}</span>
              </div>

              <div v-if="!decoding" class="snapshot-actions">
                <button class="primary-btn" @click="retake">📷 Retake</button>
                <button class="outline-btn" @click="triggerImageUpload">🖼 Use File</button>
                <button class="secondary-btn" @click="closeCamera">Cancel</button>
              </div>
            </div>
          </div>

          <!-- ── LOADING SPINNER (API Call) ── -->
          <div v-if="isFetchingProduct" class="loading-card">
            <div class="spinner large"></div>
            <h3>Connecting to Open Food Facts...</h3>
            <p class="muted">Retrieving nutritional metadata and product image</p>
          </div>

          <!-- ── PRODUCT DETAIL CARD (Scanned Result) ── -->
          <div v-if="product && !isFetchingProduct" class="product-preview-card">
            <div class="card-badge-header">
              <span class="found-badge">✓ Product Found in Open Food Facts</span>
              <span class="barcode-chip mono">{{ product.barcode }}</span>
            </div>

            <div class="product-body">
              <div class="product-media">
                <img
                  v-if="product.image"
                  :src="product.image"
                  :alt="product.name"
                  class="product-image"
                />
                <div v-else class="product-image-fallback">📦</div>

                <div
                  v-if="product.nutriScore"
                  class="nutri-tag"
                  :class="`nutri-${product.nutriScore.toLowerCase()}`"
                >
                  Nutri-Score <strong>{{ product.nutriScore.toUpperCase() }}</strong>
                </div>
              </div>

              <div class="product-details">
                <h2 class="product-title">{{ product.name }}</h2>
                <div class="product-meta-row">
                  <span v-if="product.brand" class="meta-item"><strong>Brand:</strong> {{ product.brand }}</span>
                  <span v-if="product.quantity" class="meta-item"><strong>Net:</strong> {{ product.quantity }}</span>
                  <span v-if="product.categories" class="meta-item"><strong>Category:</strong> {{ product.categories }}</span>
                </div>

                <!-- Price Input with quick adjust chips -->
                <div class="price-edit-box">
                  <label class="input-label">Price per unit (฿)</label>
                  <div class="price-input-row">
                    <span class="currency-prefix">฿</span>
                    <input
                      v-model.number="product.price"
                      type="number"
                      min="0"
                      step="0.5"
                      class="price-input-field"
                      placeholder="0.00"
                    />
                  </div>
                  <div class="quick-price-chips">
                    <button class="chip" @click="product.price = (product.price || 0) + 10">+10</button>
                    <button class="chip" @click="product.price = (product.price || 0) + 20">+20</button>
                    <button class="chip" @click="product.price = (product.price || 0) + 50">+50</button>
                    <button class="chip" @click="product.price = (product.price || 0) + 100">+100</button>
                    <button class="chip reset" @click="product.price = 0">Reset</button>
                  </div>
                </div>

                <!-- Action Button -->
                <div class="product-actions">
                  <button class="primary-btn btn-full" @click="addScannedProductToCart">
                    ＋ Add to Cart
                  </button>
                  <button class="secondary-btn" @click="product = null">Dismiss</button>
                </div>
              </div>
            </div>
          </div>

          <!-- ── MANUAL PRODUCT FORM (Barcode not in OFF) ── -->
          <div v-if="mobileView === 'manual' && !product && !isFetchingProduct" class="manual-product-card">
            <div class="manual-header">
              <span class="notice-icon">📋</span>
              <div>
                <h3>Add New Product</h3>
                <p class="muted">
                  Barcode <span class="mono">{{ manualForm.barcode }}</span> wasn't in Open Food Facts. Save it here:
                </p>
              </div>
            </div>

            <div class="form-grid">
              <div class="form-field">
                <label>Product Name *</label>
                <input
                  v-model="manualForm.name"
                  type="text"
                  class="field-input"
                  placeholder="e.g. Potato Chips"
                />
              </div>

              <div class="form-field">
                <label>Brand / Supplier</label>
                <input
                  v-model="manualForm.brand"
                  type="text"
                  class="field-input"
                  placeholder="e.g. Snack Co."
                />
              </div>

              <div class="form-field">
                <label>Price (฿) *</label>
                <input
                  v-model.number="manualForm.price"
                  type="number"
                  min="0"
                  step="0.5"
                  class="field-input"
                  placeholder="0.00"
                />
              </div>
            </div>

            <div class="form-actions">
              <button
                class="primary-btn"
                :disabled="!manualForm.name.trim()"
                @click="saveManualProduct"
              >
                Save & Add to Cart
              </button>
              <button class="secondary-btn" @click="mobileView = 'scan'">Cancel</button>
            </div>
          </div>

          <!-- Empty State / Helpful Quick Guide (when idle) -->
          <div v-if="!isCameraActive && !product && !isFetchingProduct && mobileView !== 'manual'" class="scanner-hero">
            <div class="hero-icon-cluster">
              <span class="cluster-icon">📷</span>
              <span class="cluster-icon">🖼</span>
              <span class="cluster-icon">⌨</span>
            </div>
            <h3>Ready to Scan</h3>
            <p class="hero-desc">
              Use the camera shutter, import a barcode screenshot or photo, or enter a barcode manually to fetch product details from Open Food Facts.
            </p>
            <div class="feature-pills">
              <span class="pill">EAN-13 / EAN-8</span>
              <span class="pill">UPC-A / UPC-E</span>
              <span class="pill">Code 128</span>
              <span class="pill">QR Code</span>
              <span class="pill">PromptPay Ready</span>
            </div>
          </div>
        </section>

        <!-- ── RIGHT PANE: DESKTOP TICKET & CART (ALWAYS VISIBLE ON PC) ── -->
        <aside
          class="pane cart-pane"
          :class="{ 'mobile-active': mobileView === 'cart' }"
        >
          <div class="cart-card">
            <!-- Cart Header -->
            <div class="cart-card-header">
              <div class="cart-title-row">
                <div class="cart-heading">
                  <span class="pane-icon">🧾</span>
                  <h2>Current Order</h2>
                </div>
                <span class="item-count-badge">{{ cartCount }} items</span>
              </div>
              <div class="cart-header-actions">
                <button
                  v-if="cart.length > 0"
                  class="text-btn danger"
                  @click="clearCart"
                >
                  Clear All
                </button>
                <button
                  class="mobile-close-cart-btn"
                  @click="mobileView = 'scan'"
                >
                  ✕ Close
                </button>
              </div>
            </div>

            <!-- Cart Items List -->
            <div class="cart-items-wrapper">
              <div v-if="cart.length === 0" class="empty-cart-state">
                <span class="empty-icon">🛒</span>
                <p class="empty-title">Cart is currently empty</p>
                <p class="muted">Scanned products will automatically show up on this ticket.</p>
              </div>

              <ul v-else class="cart-list">
                <li
                  v-for="item in cart"
                  :key="item.barcode"
                  class="cart-item-row"
                >
                  <div class="item-img-box">
                    <img
                      v-if="item.image"
                      :src="item.image"
                      :alt="item.name"
                      class="item-thumb"
                    />
                    <span v-else class="thumb-fallback">📦</span>
                  </div>

                  <div class="item-main">
                    <h4 class="item-name" :title="item.name">{{ item.name }}</h4>
                    <span class="item-barcode mono">{{ item.barcode }}</span>
                    <span class="item-unit-price">฿{{ item.price.toFixed(2) }} ea</span>
                  </div>

                  <!-- Quantity Controls -->
                  <div class="qty-stepper">
                    <button class="step-btn" @click="changeQty(item.barcode, -1)">−</button>
                    <span class="qty-display">{{ item.qty }}</span>
                    <button class="step-btn" @click="changeQty(item.barcode, +1)">＋</button>
                  </div>

                  <div class="item-subtotal">
                    ฿{{ (item.qty * item.price).toFixed(2) }}
                  </div>

                  <button
                    class="remove-item-btn"
                    title="Remove item"
                    @click="removeItem(item.barcode)"
                  >
                    🗑
                  </button>
                </li>
              </ul>
            </div>

            <!-- Cart Financial Summary -->
            <div class="cart-summary-footer">
              <div class="summary-line">
                <span class="summary-label">Subtotal</span>
                <span class="summary-value">฿{{ cartSubtotal.toFixed(2) }}</span>
              </div>
              <div class="summary-line">
                <span class="summary-label">VAT (7% Thai Tax)</span>
                <span class="summary-value">฿{{ cartTax.toFixed(2) }}</span>
              </div>
              <div class="summary-divider"></div>
              <div class="summary-line grand-total">
                <span class="summary-label">Grand Total</span>
                <span class="summary-value highlight">฿{{ cartTotal.toFixed(2) }}</span>
              </div>

              <!-- Checkout Trigger Button -->
              <button
                class="checkout-submit-btn"
                :disabled="cart.length === 0"
                @click="openCheckout"
              >
                <span>Proceed to Payment</span>
                <span class="checkout-arrow">→</span>
              </button>
            </div>
          </div>
        </aside>
      </div>

      <!-- ─────── TAB 2: SALES HISTORY ─────── -->
      <div v-if="activeTab === 'history'" class="history-layout">
        <div class="history-card">
          <div class="history-header">
            <div>
              <h2>Sales & Transaction History</h2>
              <p class="muted">Recorded sales stored locally in your browser</p>
            </div>
            <button
              v-if="salesHistory.length > 0"
              class="outline-btn danger"
              @click="salesHistory = []"
            >
              Clear History
            </button>
          </div>

          <div v-if="salesHistory.length === 0" class="empty-history">
            <span class="history-empty-icon">📜</span>
            <p>No transaction history recorded yet.</p>
            <button class="primary-btn mt-4" @click="activeTab = 'register'">
              Go to Register
            </button>
          </div>

          <div v-else class="history-table-wrapper">
            <table class="history-table">
              <thead>
                <tr>
                  <th>Receipt #</th>
                  <th>Date & Time</th>
                  <th>Method</th>
                  <th>Items</th>
                  <th>Total</th>
                  <th>Action</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="sale in salesHistory" :key="sale.id">
                  <td class="mono font-bold">{{ sale.id }}</td>
                  <td>{{ sale.formattedDate }}</td>
                  <td>
                    <span class="method-tag" :class="sale.method">
                      {{ sale.method === 'cash' ? '💵 Cash' : sale.method === 'card' ? '💳 Card' : '📲 PromptPay' }}
                    </span>
                  </td>
                  <td>{{ sale.items.reduce((s, i) => s + i.qty, 0) }} items</td>
                  <td class="font-bold highlight">฿{{ sale.total.toFixed(2) }}</td>
                  <td>
                    <button
                      class="view-receipt-btn"
                      @click="receiptRecord = sale; showReceiptModal = true"
                    >
                      View Receipt
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </main>

    <!-- ════════════════ PAYMENT MODAL ════════════════ -->
    <div v-if="showPaymentModal" class="modal-backdrop" @click.self="showPaymentModal = false">
      <div class="modal-card payment-modal">
        <div class="modal-header">
          <h3>Payment Checkout</h3>
          <button class="modal-close-btn" @click="showPaymentModal = false">✕</button>
        </div>

        <div class="payment-body">
          <!-- Total Callout -->
          <div class="pay-total-callout">
            <span class="callout-label">Amount Due:</span>
            <span class="callout-value">฿{{ cartTotal.toFixed(2) }}</span>
          </div>

          <!-- Payment Method Selector -->
          <div class="payment-methods-grid">
            <button
              class="method-btn"
              :class="{ selected: payMethod === 'cash' }"
              @click="payMethod = 'cash'"
            >
              <span class="method-icon">💵</span>
              <span class="method-name">Cash</span>
            </button>
            <button
              class="method-btn"
              :class="{ selected: payMethod === 'promptpay' }"
              @click="payMethod = 'promptpay'"
            >
              <span class="method-icon">📲</span>
              <span class="method-name">PromptPay QR</span>
            </button>
            <button
              class="method-btn"
              :class="{ selected: payMethod === 'card' }"
              @click="payMethod = 'card'"
            >
              <span class="method-icon">💳</span>
              <span class="method-name">Card</span>
            </button>
          </div>

          <!-- Cash Section -->
          <div v-if="payMethod === 'cash'" class="cash-section">
            <label class="input-label">Cash Received (฿)</label>
            <div class="cash-input-row">
              <span class="currency-prefix">฿</span>
              <input
                v-model="cashReceived"
                type="number"
                min="0"
                step="1"
                class="field-input large"
                placeholder="0.00"
                autofocus
              />
            </div>

            <!-- Quick Cash Denominations -->
            <div class="quick-cash-row">
              <button class="cash-chip" @click="setQuickCash('exact')">Exact (฿{{ Math.ceil(cartTotal) }})</button>
              <button class="cash-chip" @click="setQuickCash(50)">฿50</button>
              <button class="cash-chip" @click="setQuickCash(100)">฿100</button>
              <button class="cash-chip" @click="setQuickCash(500)">฿500</button>
              <button class="cash-chip" @click="setQuickCash(1000)">฿1,000</button>
            </div>

            <!-- Change Calculation -->
            <div class="change-callout" :class="{ ready: changeOwed >= 0 && (parseFloat(cashReceived) || 0) >= cartTotal }">
              <span class="change-label">Change to Return:</span>
              <span class="change-amount">฿{{ changeOwed.toFixed(2) }}</span>
            </div>
          </div>

          <!-- PromptPay QR Section -->
          <div v-if="payMethod === 'promptpay'" class="promptpay-section">
            <div class="qr-mock-card">
              <div class="qr-badge">PROMPTPAY</div>
              <div class="qr-code-placeholder">
                <img
                  :src="`https://api.qrserver.com/v1/create-qr-code/?size=180x180&data=PROMPTPAY-MOCK-${cartTotal.toFixed(2)}`"
                  alt="PromptPay QR Code"
                  class="qr-image"
                />
              </div>
              <p class="qr-hint">Scan with any Thai Banking App</p>
              <span class="qr-amount">฿{{ cartTotal.toFixed(2) }}</span>
            </div>
          </div>

          <!-- Card Section -->
          <div v-if="payMethod === 'card'" class="card-section">
            <div class="card-terminal-mock">
              <span class="terminal-icon">💳</span>
              <h4>Tap, Insert, or Swipe Card</h4>
              <p class="muted">Ready on POS terminal for ฿{{ cartTotal.toFixed(2) }}</p>
            </div>
          </div>
        </div>

        <div class="modal-footer">
          <button class="secondary-btn" @click="showPaymentModal = false">Cancel</button>
          <button
            class="primary-btn btn-lg"
            :disabled="!canCompletePayment"
            @click="finalizePayment"
          >
            Complete Sale ✓
          </button>
        </div>
      </div>
    </div>

    <!-- ════════════════ RECEIPT MODAL ════════════════ -->
    <div v-if="showReceiptModal && receiptRecord" class="modal-backdrop" @click.self="closeReceipt">
      <div class="modal-card receipt-modal">
        <div class="modal-header no-print">
          <h3>Transaction Receipt</h3>
          <button class="modal-close-btn" @click="closeReceipt">✕</button>
        </div>

        <!-- Printable Paper Receipt -->
        <div class="receipt-paper" id="printable-receipt">
          <div class="receipt-brand">
            <h2>BANMAI POS</h2>
            <p>Automated Store & Barcode Checkout</p>
            <p class="receipt-id mono">Receipt #: {{ receiptRecord.id }}</p>
            <p class="receipt-date">{{ receiptRecord.formattedDate }}</p>
          </div>

          <div class="receipt-separator">================================</div>

          <div class="receipt-items-table">
            <div
              v-for="item in receiptRecord.items"
              :key="item.barcode"
              class="receipt-line-item"
            >
              <div class="receipt-line-left">
                <span class="receipt-item-title">{{ item.name }}</span>
                <span class="receipt-item-calc">{{ item.qty }} x ฿{{ item.price.toFixed(2) }}</span>
              </div>
              <span class="receipt-line-right">
                ฿{{ (item.qty * item.price).toFixed(2) }}
              </span>
            </div>
          </div>

          <div class="receipt-separator">--------------------------------</div>

          <div class="receipt-totals">
            <div class="receipt-row">
              <span>Subtotal:</span>
              <span>฿{{ receiptRecord.subtotal.toFixed(2) }}</span>
            </div>
            <div class="receipt-row">
              <span>VAT 7%:</span>
              <span>฿{{ receiptRecord.tax.toFixed(2) }}</span>
            </div>
            <div class="receipt-row grand-line">
              <span>TOTAL:</span>
              <span>฿{{ receiptRecord.total.toFixed(2) }}</span>
            </div>

            <div class="receipt-row payment-info">
              <span>Method:</span>
              <span class="capitalize">{{ receiptRecord.method }}</span>
            </div>
            <div v-if="receiptRecord.cash" class="receipt-row">
              <span>Cash Tendered:</span>
              <span>฿{{ receiptRecord.cash.toFixed(2) }}</span>
            </div>
            <div v-if="receiptRecord.change !== null" class="receipt-row">
              <span>Change:</span>
              <span>฿{{ receiptRecord.change.toFixed(2) }}</span>
            </div>
          </div>

          <div class="receipt-separator">================================</div>

          <div class="receipt-footer">
            <p>Thank you for shopping with us!</p>
            <p>Open Food Facts Database Verified</p>
          </div>
        </div>

        <div class="modal-footer no-print">
          <button class="outline-btn" @click="printReceipt">🖨 Print Receipt</button>
          <button class="primary-btn" @click="closeReceipt">New Sale ＋</button>
        </div>
      </div>
    </div>

    <!-- Hidden canvas for frame capture & decoding -->
    <canvas ref="canvasRef" style="display: none;"></canvas>
    <!-- Hidden DOM container for Html5Qrcode file scan -->
    <div id="__img_scanner__" style="display: none;"></div>

  </div>
</template>

<style scoped>
/* ─────────────────────────── COLOR SYSTEM & BASE ─────────────────────────── */
:root {
  --bg-main: #0b1120;
  --bg-card: #151f32;
  --bg-card-hover: #1e293b;
  --bg-input: #0f172a;
  --accent-cyan: #38bdf8;
  --accent-blue: #6366f1;
  --text-main: #f8fafc;
  --text-muted: #94a3b8;
  --border: #334155;
  --success: #10b981;
  --danger: #ef4444;
}

.pos-app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  width: 100%;
  background-color: #0b1120;
  color: #f8fafc;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  overflow-x: hidden;
}

/* ─────────────────────────── TOP NAVIGATION ─────────────────────────── */
.app-nav {
  position: sticky;
  top: 0;
  z-index: 50;
  width: 100%;
  background: rgba(15, 23, 42, 0.85);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid #334155;
}

.nav-container {
  max-width: 1600px;
  margin: 0 auto;
  padding: 0.75rem 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.brand {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.brand-logo {
  font-size: 1.75rem;
}

.brand-title {
  font-size: 1.25rem;
  font-weight: 800;
  letter-spacing: -0.025em;
  background: linear-gradient(135deg, #38bdf8 0%, #818cf8 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.brand-tag {
  font-size: 0.75rem;
  color: #94a3b8;
  display: block;
}

.nav-tabs {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: #1e293b;
  padding: 0.25rem;
  border-radius: 12px;
}

.tab-btn {
  background: transparent;
  border: none;
  color: #94a3b8;
  padding: 0.45rem 1rem;
  font-size: 0.875rem;
  font-weight: 600;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.15s ease;
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.tab-btn.active {
  background: #38bdf8;
  color: #0f172a;
}

.nav-actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.mobile-cart-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.5rem 0.9rem;
  border-radius: 12px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
}

.mobile-cart-btn.has-items {
  border-color: #38bdf8;
  background: rgba(56, 189, 248, 0.15);
}

.badge {
  background: #38bdf8;
  color: #0f172a;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 800;
  padding: 0.1rem 0.5rem;
}

/* ─────────────────────────── MAIN LAYOUT ─────────────────────────── */
.app-main {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 1.25rem;
}

/* ── PC DUAL-PANE SPLIT ── */
.register-layout {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1.5rem;
  align-items: start;
}

@media (min-width: 1024px) {
  .register-layout {
    grid-template-columns: 1.25fr 0.95fr;
  }
  .mobile-close-cart-btn {
    display: none !important;
  }
}

.pane {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  width: 100%;
}

/* ─────────────────────────── OPERATION PANE CONTROLS ─────────────────────────── */
.action-bar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.drag-hint {
  font-size: 0.8rem;
  color: #64748b;
  margin-left: auto;
}

/* ── Barcode Search Box ── */
.barcode-search-box {
  width: 100%;
}

.search-input-wrapper {
  display: flex;
  align-items: center;
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 14px;
  padding: 0.35rem 0.5rem 0.35rem 1rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.search-input-wrapper:focus-within {
  border-color: #38bdf8;
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.2);
}

.search-icon {
  font-size: 1.1rem;
  color: #94a3b8;
  margin-right: 0.5rem;
}

.search-input {
  flex: 1;
  background: transparent;
  border: none;
  outline: none;
  color: #f8fafc;
  font-size: 1rem;
  font-family: monospace;
}

.search-submit-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.5rem 1.1rem;
  border-radius: 10px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
}

.search-submit-btn:hover:not(:disabled) {
  background: #38bdf8;
  color: #0f172a;
}

.search-submit-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* ─────────────────────────── CAMERA VIEW WITH SHUTTER ─────────────────────────── */
.camera-card {
  background: #000;
  border: 1px solid #334155;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.5);
  display: flex;
  flex-direction: column;
}

.camera-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1.25rem;
  background: #151f32;
  border-bottom: 1px solid #1e293b;
}

.status-indicator.live {
  font-size: 0.75rem;
  font-weight: 800;
  color: #ef4444;
  letter-spacing: 0.05em;
  animation: pulse 1.5s infinite;
}

.viewfinder-container {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 10;
  max-height: 480px;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.video-feed {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.scanner-guide {
  position: absolute;
  width: 70%;
  max-width: 320px;
  height: 55%;
  pointer-events: none;
}

.corner {
  position: absolute;
  width: 28px;
  height: 28px;
  border: 3px solid #38bdf8;
}

.corner-tl { top: 0; left: 0; border-right: none; border-bottom: none; border-radius: 6px 0 0 0; }
.corner-tr { top: 0; right: 0; border-left: none; border-bottom: none; border-radius: 0 6px 0 0; }
.corner-bl { bottom: 0; left: 0; border-right: none; border-top: none; border-radius: 0 0 0 6px; }
.corner-br { bottom: 0; right: 0; border-left: none; border-top: none; border-radius: 0 0 6px 0; }

.aim-line {
  position: absolute;
  top: 50%;
  left: 5%;
  right: 5%;
  height: 2px;
  background: rgba(56, 189, 248, 0.85);
  box-shadow: 0 0 8px #38bdf8;
}

.guide-text {
  position: absolute;
  bottom: 85px;
  width: 100%;
  text-align: center;
  font-size: 0.85rem;
  color: #f8fafc;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.8);
  pointer-events: none;
}

.camera-footer {
  position: absolute;
  bottom: 12px;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ── Shutter Button ── */
.shutter-trigger {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  transition: transform 0.15s ease;
}

.shutter-trigger:hover:not(:disabled) {
  transform: scale(1.05);
}

.shutter-trigger:active:not(:disabled) {
  transform: scale(0.92);
}

.shutter-ring {
  width: 68px;
  height: 68px;
  border-radius: 9999px;
  border: 4px solid #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 0 20px rgba(56, 189, 248, 0.5);
}

.shutter-dot {
  width: 50px;
  height: 50px;
  border-radius: 9999px;
  background: linear-gradient(135deg, #38bdf8, #818cf8);
}

/* ── Snapshot Preview Screen ── */
.snapshot-container {
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
  background: #151f32;
}

.snapshot-view {
  position: relative;
  width: 100%;
  max-width: 480px;
  border-radius: 14px;
  overflow: hidden;
  background: #000;
}

.snapshot-image {
  width: 100%;
  display: block;
  object-fit: contain;
}

.decoding-scrim {
  position: absolute;
  inset: 0;
  background: rgba(11, 17, 32, 0.85);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  color: #f8fafc;
}

.scan-error-msg {
  color: #ef4444;
  font-size: 0.9rem;
  text-align: center;
  max-width: 420px;
}

.snapshot-actions {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
  justify-content: center;
}

/* ─────────────────────────── PRODUCT PREVIEW CARD ─────────────────────────── */
.product-preview-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
  animation: slideDown 0.25s ease-out;
}

.card-badge-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1.25rem;
  background: #1e293b;
  border-bottom: 1px solid #334155;
}

.found-badge {
  font-size: 0.8rem;
  font-weight: 700;
  color: #10b981;
}

.barcode-chip {
  background: #0f172a;
  padding: 0.2rem 0.6rem;
  border-radius: 6px;
  font-size: 0.8rem;
  color: #38bdf8;
}

.product-body {
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

@media (min-width: 640px) {
  .product-body {
    flex-direction: row;
  }
}

.product-media {
  flex-shrink: 0;
  width: 100%;
  max-width: 180px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
}

.product-image {
  width: 100%;
  height: 160px;
  object-fit: contain;
  background: #ffffff;
  border-radius: 12px;
  padding: 0.5rem;
}

.product-image-fallback {
  width: 100%;
  height: 160px;
  background: #1e293b;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 4rem;
}

.nutri-tag {
  font-size: 0.75rem;
  padding: 0.25rem 0.65rem;
  border-radius: 9999px;
  color: #ffffff;
  font-weight: 600;
}
.nutri-a { background: #038141; }
.nutri-b { background: #85bb2f; }
.nutri-c { background: #fecb02; color: #000; }
.nutri-d { background: #ee8100; }
.nutri-e { background: #e63e11; }

.product-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.product-title {
  font-size: 1.35rem;
  font-weight: 700;
  color: #f8fafc;
  line-height: 1.3;
}

.product-meta-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  font-size: 0.85rem;
  color: #94a3b8;
}

.price-edit-box {
  margin-top: 0.5rem;
  padding: 0.9rem;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 12px;
}

.price-input-row {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  margin: 0.4rem 0;
}

.currency-prefix {
  font-size: 1.3rem;
  font-weight: 800;
  color: #38bdf8;
}

.price-input-field {
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 8px;
  color: #f8fafc;
  font-size: 1.3rem;
  font-weight: 700;
  padding: 0.35rem 0.75rem;
  width: 140px;
  outline: none;
}

.quick-price-chips {
  display: flex;
  gap: 0.4rem;
  flex-wrap: wrap;
  margin-top: 0.5rem;
}

.chip {
  background: #1e293b;
  border: 1px solid #334155;
  color: #cbd5e1;
  padding: 0.25rem 0.6rem;
  font-size: 0.75rem;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
}

.chip:hover {
  background: #334155;
  color: #38bdf8;
}

.chip.reset {
  color: #ef4444;
}

.product-actions {
  display: flex;
  gap: 0.75rem;
  margin-top: 0.5rem;
}

/* ─────────────────────────── MANUAL PRODUCT CARD ─────────────────────────── */
.manual-product-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.manual-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.notice-icon {
  font-size: 2rem;
}

.form-grid {
  display: flex;
  flex-direction: column;
  gap: 0.9rem;
}

.form-field {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.form-field label {
  font-size: 0.85rem;
  color: #94a3b8;
}

.field-input {
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 10px;
  color: #f8fafc;
  padding: 0.65rem 0.9rem;
  font-size: 0.95rem;
  outline: none;
}

.field-input:focus {
  border-color: #38bdf8;
}

.form-actions {
  display: flex;
  gap: 0.75rem;
}

/* ─────────────────────────── SCANNER HERO (IDLE) ─────────────────────────── */
.scanner-hero {
  background: #151f32;
  border: 1px dashed #334155;
  border-radius: 20px;
  padding: 2.5rem 1.5rem;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.hero-icon-cluster {
  display: flex;
  gap: 0.75rem;
}

.cluster-icon {
  font-size: 2.2rem;
  background: #1e293b;
  width: 54px;
  height: 54px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.hero-desc {
  color: #94a3b8;
  font-size: 0.95rem;
  max-width: 480px;
  line-height: 1.5;
}

.feature-pills {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  justify-content: center;
}

.pill {
  background: #1e293b;
  color: #38bdf8;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  border: 1px solid #334155;
}

/* ─────────────────────────── CART PANE & SUMMARY ─────────────────────────── */
.cart-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.4);
  position: sticky;
  top: 5rem;
  max-height: calc(100vh - 6rem);
}

.cart-card-header {
  padding: 1.1rem 1.25rem;
  background: #1e293b;
  border-bottom: 1px solid #334155;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.cart-title-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.cart-heading {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.cart-heading h2 {
  font-size: 1.15rem;
  font-weight: 700;
}

.item-count-badge {
  background: #0f172a;
  border: 1px solid #334155;
  color: #38bdf8;
  font-size: 0.75rem;
  font-weight: 700;
  padding: 0.2rem 0.6rem;
  border-radius: 9999px;
}

.cart-header-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.cart-items-wrapper {
  flex: 1;
  overflow-y: auto;
  padding: 1rem 1.25rem;
  min-height: 220px;
  max-height: 380px;
}

.empty-cart-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  padding: 2.5rem 1rem;
  text-align: center;
  color: #64748b;
  gap: 0.5rem;
}

.empty-icon {
  font-size: 3rem;
  opacity: 0.6;
}

.empty-title {
  font-size: 1.05rem;
  font-weight: 600;
  color: #94a3b8;
}

.cart-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.cart-item-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  background: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 14px;
  padding: 0.75rem 0.9rem;
}

.item-img-box {
  width: 44px;
  height: 44px;
  flex-shrink: 0;
  border-radius: 8px;
  overflow: hidden;
  background: #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
}

.item-thumb {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.thumb-fallback {
  font-size: 1.4rem;
}

.item-main {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 0.9rem;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: #f8fafc;
}

.item-barcode {
  font-size: 0.72rem;
  color: #64748b;
  display: block;
}

.item-unit-price {
  font-size: 0.78rem;
  color: #94a3b8;
}

.qty-stepper {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  background: #1e293b;
  border-radius: 8px;
  padding: 0.15rem;
}

.step-btn {
  background: #334155;
  border: none;
  color: #f8fafc;
  width: 24px;
  height: 24px;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.qty-display {
  min-width: 20px;
  text-align: center;
  font-size: 0.85rem;
  font-weight: 700;
}

.item-subtotal {
  font-size: 0.95rem;
  font-weight: 700;
  color: #38bdf8;
  min-width: 60px;
  text-align: right;
}

.remove-item-btn {
  background: none;
  border: none;
  cursor: pointer;
  opacity: 0.45;
  transition: opacity 0.15s;
  padding: 0.2rem;
}

.remove-item-btn:hover {
  opacity: 1;
}

/* ── Cart Financial Summary ── */
.cart-summary-footer {
  padding: 1.25rem;
  background: #111827;
  border-top: 1px solid #334155;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.summary-line {
  display: flex;
  justify-content: space-between;
  font-size: 0.9rem;
  color: #94a3b8;
}

.summary-divider {
  height: 1px;
  background: #334155;
  margin: 0.2rem 0;
}

.summary-line.grand-total {
  font-size: 1.2rem;
  font-weight: 800;
  color: #f8fafc;
}

.highlight {
  color: #38bdf8;
}

.checkout-submit-btn {
  background: linear-gradient(135deg, #38bdf8 0%, #6366f1 100%);
  border: none;
  color: #0f172a;
  padding: 0.9rem 1.25rem;
  border-radius: 12px;
  font-size: 1.05rem;
  font-weight: 700;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 0.5rem;
  transition: all 0.2s ease;
  box-shadow: 0 4px 14px rgba(56, 189, 248, 0.35);
}

.checkout-submit-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  box-shadow: none;
}

.checkout-submit-btn:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 6px 20px rgba(56, 189, 248, 0.5);
}

/* ─────────────────────────── MODALS & DIALOGS ─────────────────────────── */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(11, 17, 32, 0.85);
  backdrop-filter: blur(8px);
  z-index: 100;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

.modal-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  width: 100%;
  max-width: 520px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.6);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  max-height: 90vh;
}

.modal-header {
  padding: 1rem 1.25rem;
  background: #1e293b;
  border-bottom: 1px solid #334155;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.modal-close-btn {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 1.2rem;
  cursor: pointer;
}

.modal-footer {
  padding: 1rem 1.25rem;
  background: #111827;
  border-top: 1px solid #334155;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

/* ── Payment Modal Content ── */
.payment-body {
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  overflow-y: auto;
}

.pay-total-callout {
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 14px;
  padding: 1rem 1.25rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.callout-value {
  font-size: 1.6rem;
  font-weight: 800;
  color: #38bdf8;
}

.payment-methods-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.75rem;
}

.method-btn {
  background: #1e293b;
  border: 2px solid transparent;
  color: #f8fafc;
  padding: 0.9rem 0.5rem;
  border-radius: 12px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4rem;
  transition: all 0.15s ease;
}

.method-btn.selected {
  border-color: #38bdf8;
  background: rgba(56, 189, 248, 0.15);
}

.method-icon {
  font-size: 1.6rem;
}

.method-name {
  font-size: 0.8rem;
  font-weight: 600;
}

.cash-section {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.cash-input-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.field-input.large {
  font-size: 1.4rem;
  font-weight: 700;
}

.quick-cash-row {
  display: flex;
  gap: 0.4rem;
  flex-wrap: wrap;
}

.cash-chip {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.35rem 0.75rem;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 600;
  cursor: pointer;
}

.cash-chip:hover {
  background: #38bdf8;
  color: #0f172a;
}

.change-callout {
  margin-top: 0.5rem;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 12px;
  padding: 0.9rem 1.1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.change-amount {
  font-size: 1.35rem;
  font-weight: 800;
  color: #10b981;
}

.qr-mock-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 16px;
  padding: 1.5rem;
  gap: 0.75rem;
}

.qr-badge {
  background: #03496b;
  color: #ffffff;
  padding: 0.2rem 0.8rem;
  border-radius: 6px;
  font-size: 0.8rem;
  font-weight: 800;
  letter-spacing: 0.05em;
}

.qr-code-placeholder {
  background: #ffffff;
  padding: 0.75rem;
  border-radius: 12px;
}

.qr-image {
  width: 160px;
  height: 160px;
  display: block;
}

.qr-hint {
  font-size: 0.85rem;
  color: #94a3b8;
}

.qr-amount {
  font-size: 1.4rem;
  font-weight: 800;
  color: #38bdf8;
}

.card-terminal-mock {
  text-align: center;
  padding: 2.5rem 1.5rem;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.terminal-icon {
  font-size: 3rem;
}

/* ─────────────────────────── PAPER RECEIPT ─────────────────────────── */
.receipt-paper {
  background: #ffffff;
  color: #111827;
  padding: 1.5rem 1.25rem;
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.875rem;
  line-height: 1.4;
  overflow-y: auto;
  max-height: 60vh;
}

.receipt-brand {
  text-align: center;
}

.receipt-brand h2 {
  font-size: 1.3rem;
  font-weight: 900;
  letter-spacing: 0.05em;
}

.receipt-separator {
  text-align: center;
  margin: 0.5rem 0;
  overflow: hidden;
  letter-spacing: -0.05em;
  color: #64748b;
}

.receipt-line-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.4rem;
}

.receipt-line-left {
  display: flex;
  flex-direction: column;
}

.receipt-item-title {
  font-weight: bold;
}

.receipt-item-calc {
  font-size: 0.75rem;
  color: #475569;
}

.receipt-totals {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.receipt-row {
  display: flex;
  justify-content: space-between;
}

.receipt-row.grand-line {
  font-size: 1.1rem;
  font-weight: 900;
  margin: 0.35rem 0;
  border-top: 1px dashed #111;
  border-bottom: 1px dashed #111;
  padding: 0.25rem 0;
}

.receipt-footer {
  text-align: center;
  font-size: 0.75rem;
  color: #64748b;
}

/* ─────────────────────────── SALES HISTORY ─────────────────────────── */
.history-layout {
  width: 100%;
}

.history-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 1.5rem;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.25rem;
}

.history-table-wrapper {
  overflow-x: auto;
}

.history-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
  font-size: 0.9rem;
}

.history-table th {
  background: #1e293b;
  padding: 0.75rem 1rem;
  color: #94a3b8;
  font-weight: 600;
}

.history-table td {
  padding: 0.75rem 1rem;
  border-bottom: 1px solid #1e293b;
}

.method-tag {
  font-size: 0.75rem;
  padding: 0.2rem 0.6rem;
  border-radius: 6px;
  background: #1e293b;
  font-weight: 600;
}

.view-receipt-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #38bdf8;
  padding: 0.35rem 0.75rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
}

.view-receipt-btn:hover {
  background: #38bdf8;
  color: #0f172a;
}

/* ─────────────────────────── COMMON UTILITIES & BUTTONS ─────────────────────────── */
.primary-btn {
  background: linear-gradient(135deg, #38bdf8 0%, #6366f1 100%);
  border: none;
  color: #0f172a;
  padding: 0.65rem 1.4rem;
  border-radius: 12px;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.15s ease;
}

.primary-btn:hover:not(:disabled) {
  opacity: 0.95;
  transform: translateY(-1px);
}

.primary-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.secondary-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.65rem 1.25rem;
  border-radius: 12px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
}

.outline-btn {
  background: transparent;
  border: 2px solid #38bdf8;
  color: #38bdf8;
  padding: 0.6rem 1.25rem;
  border-radius: 12px;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  gap: 0.4rem;
}

.outline-btn:hover {
  background: rgba(56, 189, 248, 0.1);
}

.outline-btn.danger {
  border-color: #ef4444;
  color: #ef4444;
}

.btn-full {
  width: 100%;
  justify-content: center;
}

.text-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 0.8rem;
  color: #94a3b8;
}

.text-btn.danger {
  color: #ef4444;
}

.close-btn {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 1.2rem;
  cursor: pointer;
}

.mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}

.muted {
  color: #94a3b8;
  font-size: 0.85rem;
}

.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid rgba(255, 255, 255, 0.2);
  border-top-color: #38bdf8;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.spinner.large {
  width: 48px;
  height: 48px;
  border-width: 4px;
}

.loading-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 3rem 1.5rem;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.alert-banner {
  padding: 0.75rem 1rem;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.alert-banner.error {
  background: rgba(239, 68, 68, 0.15);
  border: 1px solid #ef4444;
  color: #fca5a5;
}

.close-alert {
  background: none;
  border: none;
  color: #fca5a5;
  cursor: pointer;
}

.drop-overlay {
  position: absolute;
  inset: 0;
  background: rgba(11, 17, 32, 0.9);
  border: 2px dashed #38bdf8;
  border-radius: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 40;
}

.drop-card {
  text-align: center;
  color: #38bdf8;
  font-weight: 700;
}

.drop-icon {
  font-size: 3rem;
  display: block;
  margin-bottom: 0.5rem;
}

/* ─────────────────────────── RESPONSIVE BREAKPOINTS (PHONE & PC) ─────────────────────────── */
@media (max-width: 1023px) {
  /* On Mobile (< 1024px) */
  .hidden-mobile {
    display: none !important;
  }

  .operation-pane.mobile-hidden {
    display: none !important;
  }

  .cart-pane:not(.mobile-active) {
    display: none !important;
  }

  .cart-card {
    position: static;
    max-height: none;
  }

  .cart-items-wrapper {
    max-height: 50vh;
  }
}

@media (min-width: 1024px) {
  /* On PC (>= 1024px) */
  .mobile-cart-btn {
    display: none !important;
  }
}

/* ─────────────────────────── PRINT STYLES ─────────────────────────── */
@media print {
  body * {
    visibility: hidden;
  }
  #printable-receipt, #printable-receipt * {
    visibility: visible;
  }
  #printable-receipt {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    margin: 0;
    padding: 0;
  }
  .no-print {
    display: none !important;
  }
}

/* ─────────────────────────── KEYFRAMES ─────────────────────────── */
@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

@keyframes slideDown {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>
