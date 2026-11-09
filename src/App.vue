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

// ── Navigation & Views ──────────────────────────────────────────────────────
// currentTab: 'register' | 'cart' | 'history'
const currentTab = ref('register')

// Modals
const showPaymentModal = ref(false)
const showReceiptModal = ref(false)

// Data State
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
const torchOn         = ref(false)
const torchAvailable  = ref(false)

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
  torchOn.value = false
  torchAvailable.value = false

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
        checkTorchSupport()
      }
    }
  } catch (e) {
    error.value = `Camera access denied or unavailable: ${e.message}`
    isCameraActive.value = false
  }
}

function checkTorchSupport() {
  try {
    const track = stream?.getVideoTracks()[0]
    const capabilities = track?.getCapabilities?.() || {}
    torchAvailable.value = !!capabilities.torch
  } catch (_) {
    torchAvailable.value = false
  }
}

async function toggleTorch() {
  try {
    const track = stream?.getVideoTracks()[0]
    if (track && torchAvailable.value) {
      torchOn.value = !torchOn.value
      await track.applyConstraints({
        advanced: [{ torch: torchOn.value }],
      })
    }
  } catch (_) {}
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
  torchOn.value = false
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
  currentTab.value = 'register'

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
    }
  } catch (e) {
    error.value = `Open Food Facts API error: ${e.message}`
  } finally {
    isFetchingProduct.value = false
  }
}

// ── Add to Cart & Price Management ──────────────────────────────────────────
function addScannedProductToCart() {
  if (!product.value) return
  addItemToCart(product.value)
  product.value = null
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
  currentTab.value = 'register'
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

        <!-- Desktop Tabs (Visible on PC >= 1024px) -->
        <nav class="desktop-nav-tabs">
          <button
            class="tab-btn"
            :class="{ active: currentTab === 'register' }"
            @click="currentTab = 'register'"
          >
            <span class="tab-icon">⚡</span> Register
          </button>
          <button
            class="tab-btn"
            :class="{ active: currentTab === 'history' }"
            @click="currentTab = 'history'"
          >
            <span class="tab-icon">📜</span> Sales History ({{ salesHistory.length }})
          </button>
        </nav>

        <!-- Mobile Header Quick Cart Trigger -->
        <div class="mobile-header-cart">
          <button
            class="quick-cart-pill"
            :class="{ 'has-items': cartCount > 0, active: currentTab === 'cart' }"
            @click="currentTab = currentTab === 'cart' ? 'register' : 'cart'"
          >
            <span class="pill-icon">🛍</span>
            <span class="pill-count">{{ cartCount }}</span>
            <span v-if="cartTotal > 0" class="pill-total">฿{{ cartTotal.toFixed(0) }}</span>
          </button>
        </div>
      </div>
    </header>

    <!-- ════════════════ MAIN CONTENT AREA ════════════════ -->
    <main class="app-main">

      <!-- ─────── TAB 1: REGISTER / SCANNER ─────── -->
      <div
        v-show="currentTab === 'register' || currentTab === 'cart'"
        class="register-layout"
        :class="{ 'mobile-show-cart': currentTab === 'cart' }"
      >

        <!-- ── LEFT COLUMN: SCANNER & OPERATING CONTROLS ── -->
        <section
          class="pane operation-pane"
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

          <!-- Mobile Touch Action Grid -->
          <div class="action-grid">
            <button
              class="touch-btn primary"
              :class="{ active: isCameraActive }"
              @click="isCameraActive ? closeCamera() : openCamera()"
            >
              <span class="touch-btn-icon">📷</span>
              <span class="touch-btn-text">{{ isCameraActive ? 'Close Camera' : 'Camera Shutter' }}</span>
            </button>

            <button class="touch-btn outline" @click="triggerImageUpload">
              <span class="touch-btn-icon">🖼</span>
              <span class="touch-btn-text">Import Image</span>
            </button>
          </div>

          <!-- Manual Barcode Search Box -->
          <div class="barcode-search-box">
            <div class="search-input-wrapper">
              <span class="search-icon">🔍</span>
              <input
                v-model="manualBarcode"
                type="text"
                inputmode="numeric"
                class="search-input"
                placeholder="Scan or enter barcode"
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
            <span class="alert-text">⚠ {{ error }}</span>
            <button class="close-alert" @click="error = ''">✕</button>
          </div>

          <!-- Dropzone overlay when dragging on desktop -->
          <div v-if="isDragOver" class="drop-overlay">
            <div class="drop-card">
              <span class="drop-icon">📥</span>
              <p>Drop barcode image to scan</p>
            </div>
          </div>

          <!-- ── LIVE CAMERA VIEW WITH SHUTTER ── -->
          <div v-if="isCameraActive" class="camera-card">
            <!-- Camera Top Bar -->
            <div class="camera-header">
              <div class="camera-status">
                <span class="status-dot"></span>
                <span class="status-label">CAMERA ACTIVE</span>
              </div>
              <div class="camera-header-actions">
                <button
                  v-if="torchAvailable"
                  class="cam-tool-btn"
                  :class="{ on: torchOn }"
                  title="Toggle Flash"
                  @click="toggleTorch"
                >
                  ⚡ {{ torchOn ? 'Flash ON' : 'Flash' }}
                </button>
                <button class="cam-close-btn" @click="closeCamera">✕</button>
              </div>
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
              <p class="guide-text">Align barcode in frame, then tap Shutter</p>

              <!-- Camera Bottom Controls (Shutter Button) -->
              <div class="camera-controls-bar">
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
                  <p>Analyzing barcode...</p>
                </div>
              </div>

              <div v-if="scanError" class="scan-error-msg">
                <span>⚠ {{ scanError }}</span>
              </div>

              <div v-if="!decoding" class="snapshot-actions">
                <button class="touch-btn primary" @click="retake">📷 Retake</button>
                <button class="touch-btn outline" @click="triggerImageUpload">🖼 Use Image</button>
                <button class="touch-btn secondary" @click="closeCamera">Cancel</button>
              </div>
            </div>
          </div>

          <!-- ── LOADING SPINNER (API Call) ── -->
          <div v-if="isFetchingProduct" class="loading-card">
            <div class="spinner large"></div>
            <h3>Looking up product...</h3>
            <p class="muted">Fetching details from Open Food Facts</p>
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

                <!-- Action Buttons -->
                <div class="product-actions">
                  <button class="touch-btn primary flex-1" @click="addScannedProductToCart">
                    ＋ Add to Cart
                  </button>
                  <button class="touch-btn secondary" @click="product = null">Dismiss</button>
                </div>
              </div>
            </div>
          </div>

          <!-- ── MANUAL PRODUCT FORM (Barcode not in OFF) ── -->
          <div v-if="manualForm.barcode && !product && !isFetchingProduct" class="manual-product-card">
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
                class="touch-btn primary flex-1"
                :disabled="!manualForm.name.trim()"
                @click="saveManualProduct"
              >
                Save & Add to Cart
              </button>
              <button class="touch-btn secondary" @click="manualForm.barcode = ''">Cancel</button>
            </div>
          </div>

          <!-- Empty State / Helpful Quick Guide (when idle) -->
          <div v-if="!isCameraActive && !product && !isFetchingProduct && !manualForm.barcode" class="scanner-hero">
            <div class="hero-icon-cluster">
              <span class="cluster-icon">📷</span>
              <span class="cluster-icon">🖼</span>
              <span class="cluster-icon">⚡</span>
            </div>
            <h3>Ready to Scan</h3>
            <p class="hero-desc">
              Tap <strong>Camera Shutter</strong> to scan with your phone, import a barcode image from photos, or enter a number directly.
            </p>
            <div class="feature-pills">
              <span class="pill">UPC-A / UPC-E</span>
              <span class="pill">EAN-13 / EAN-8</span>
              <span class="pill">Code 128</span>
              <span class="pill">QR Code</span>
            </div>
          </div>
        </section>

        <!-- ── RIGHT COLUMN: DESKTOP TICKET & MOBILE CART VIEW ── -->
        <aside class="pane cart-pane">
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
                  class="mobile-back-to-scan-btn"
                  @click="currentTab = 'register'"
                >
                  ‹ Back to Scan
                </button>
              </div>
            </div>

            <!-- Cart Items List -->
            <div class="cart-items-wrapper">
              <div v-if="cart.length === 0" class="empty-cart-state">
                <span class="empty-icon">🛒</span>
                <p class="empty-title">Your cart is empty</p>
                <p class="muted">Scanned products will show up here.</p>
                <button class="touch-btn primary mobile-only-btn mt-3" @click="currentTab = 'register'; openCamera()">
                  📷 Open Camera Scanner
                </button>
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
                    <div class="item-sub-info">
                      <span class="item-unit-price">฿{{ item.price.toFixed(2) }}</span>
                      <span class="item-barcode mono">{{ item.barcode }}</span>
                    </div>
                  </div>

                  <!-- Quantity Stepper -->
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
      <div v-show="currentTab === 'history'" class="history-layout">
        <div class="history-card">
          <div class="history-header">
            <div>
              <h2>Sales & History</h2>
              <p class="muted">Recorded transactions in this browser</p>
            </div>
            <button
              v-if="salesHistory.length > 0"
              class="touch-btn outline danger-btn"
              @click="salesHistory = []"
            >
              Clear
            </button>
          </div>

          <div v-if="salesHistory.length === 0" class="empty-history">
            <span class="history-empty-icon">📜</span>
            <p>No transaction history recorded yet.</p>
            <button class="touch-btn primary mt-3" @click="currentTab = 'register'">
              Back to Register
            </button>
          </div>

          <div v-else class="history-list-mobile">
            <!-- Mobile Friendly History Cards -->
            <div
              v-for="sale in salesHistory"
              :key="sale.id"
              class="history-item-card"
              @click="receiptRecord = sale; showReceiptModal = true"
            >
              <div class="history-card-top">
                <span class="mono font-bold">{{ sale.id }}</span>
                <span class="method-tag" :class="sale.method">
                  {{ sale.method === 'cash' ? '💵 Cash' : sale.method === 'card' ? '💳 Card' : '📲 PromptPay' }}
                </span>
              </div>
              <div class="history-card-mid">
                <span class="muted text-sm">{{ sale.formattedDate }}</span>
                <span class="item-count-text">{{ sale.items.reduce((s, i) => s + i.qty, 0) }} items</span>
              </div>
              <div class="history-card-bot">
                <span class="text-sm">Total</span>
                <span class="history-total-amt">฿{{ sale.total.toFixed(2) }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>

    <!-- ════════════════ MOBILE BOTTOM NAVIGATION BAR ════════════════ -->
    <nav class="mobile-bottom-nav">
      <button
        class="bottom-nav-item"
        :class="{ active: currentTab === 'register' }"
        @click="currentTab = 'register'"
      >
        <span class="bottom-icon">⚡</span>
        <span class="bottom-label">Register</span>
      </button>

      <button
        class="bottom-nav-item"
        :class="{ active: currentTab === 'cart' }"
        @click="currentTab = 'cart'"
      >
        <div class="icon-badge-wrapper">
          <span class="bottom-icon">🛍</span>
          <span v-if="cartCount > 0" class="bottom-badge">{{ cartCount }}</span>
        </div>
        <span class="bottom-label">Cart</span>
      </button>

      <button
        class="bottom-nav-item"
        :class="{ active: currentTab === 'history' }"
        @click="currentTab = 'history'"
      >
        <span class="bottom-icon">📜</span>
        <span class="bottom-label">History</span>
      </button>
    </nav>

    <!-- ════════════════ PAYMENT MODAL (BOTTOM SHEET ON MOBILE) ════════════════ -->
    <div v-if="showPaymentModal" class="modal-backdrop" @click.self="showPaymentModal = false">
      <div class="modal-card payment-modal">
        <div class="modal-sheet-handle mobile-only"></div>

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
              <span class="method-name">PromptPay</span>
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
              <p class="qr-hint">Scan with any Mobile Banking App</p>
              <span class="qr-amount">฿{{ cartTotal.toFixed(2) }}</span>
            </div>
          </div>

          <!-- Card Section -->
          <div v-if="payMethod === 'card'" class="card-section">
            <div class="card-terminal-mock">
              <span class="terminal-icon">💳</span>
              <h4>Tap or Insert Card</h4>
              <p class="muted">Amount: ฿{{ cartTotal.toFixed(2) }}</p>
            </div>
          </div>
        </div>

        <div class="modal-footer">
          <button class="touch-btn secondary" @click="showPaymentModal = false">Cancel</button>
          <button
            class="touch-btn primary flex-1"
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
        <div class="modal-sheet-handle mobile-only"></div>

        <div class="modal-header no-print">
          <h3>Receipt</h3>
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
            <p>Verified with Open Food Facts</p>
          </div>
        </div>

        <div class="modal-footer no-print">
          <button class="touch-btn outline flex-1" @click="printReceipt">🖨 Print</button>
          <button class="touch-btn primary flex-1" @click="closeReceipt">New Sale ＋</button>
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
.pos-app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  width: 100%;
  background-color: #0b1120;
  color: #f8fafc;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  padding-bottom: calc(64px + env(safe-area-inset-bottom, 16px));
}

@media (min-width: 1024px) {
  .pos-app {
    padding-bottom: 0;
  }
}

/* ─────────────────────────── TOP NAVIGATION ─────────────────────────── */
.app-nav {
  position: sticky;
  top: 0;
  z-index: 40;
  width: 100%;
  background: rgba(15, 23, 42, 0.92);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid #334155;
  padding-top: env(safe-area-inset-top, 0px);
}

.nav-container {
  max-width: 1600px;
  margin: 0 auto;
  padding: 0.65rem 1rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
}

.brand {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  min-width: 0;
}

.brand-logo {
  font-size: 1.6rem;
  flex-shrink: 0;
}

.brand-text {
  min-width: 0;
}

.brand-title {
  font-size: 1.15rem;
  font-weight: 800;
  letter-spacing: -0.025em;
  background: linear-gradient(135deg, #38bdf8 0%, #818cf8 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.brand-tag {
  font-size: 0.7rem;
  color: #94a3b8;
  display: none;
}

@media (min-width: 640px) {
  .brand-tag {
    display: block;
  }
}

/* Desktop navigation tabs in top header */
.desktop-nav-tabs {
  display: none;
  align-items: center;
  gap: 0.5rem;
  background: #1e293b;
  padding: 0.25rem;
  border-radius: 12px;
}

@media (min-width: 1024px) {
  .desktop-nav-tabs {
    display: flex;
  }
  .mobile-header-cart {
    display: none;
  }
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

/* Mobile Quick Cart Pill in Header */
.quick-cart-pill {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.4rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.85rem;
  font-weight: 700;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.4rem;
  transition: all 0.2s ease;
}

.quick-cart-pill.has-items {
  border-color: #38bdf8;
  background: rgba(56, 189, 248, 0.15);
  color: #38bdf8;
}

.quick-cart-pill.active {
  background: #38bdf8;
  color: #0f172a;
}

.pill-count {
  background: #38bdf8;
  color: #0f172a;
  border-radius: 9999px;
  font-size: 0.72rem;
  font-weight: 800;
  padding: 0.05rem 0.4rem;
}

.quick-cart-pill.active .pill-count {
  background: #0f172a;
  color: #38bdf8;
}

.pill-total {
  font-size: 0.82rem;
  font-weight: 800;
}

/* ─────────────────────────── MAIN CONTENT ─────────────────────────── */
.app-main {
  flex: 1;
  max-width: 1600px;
  width: 100%;
  margin: 0 auto;
  padding: 0.75rem;
}

@media (min-width: 640px) {
  .app-main {
    padding: 1.25rem;
  }
}

/* ── LAYOUT SWITCHING (PHONE VS PC) ── */
.register-layout {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;
}

@media (max-width: 1023px) {
  /* On mobile: show scanner by default, hide cart */
  .cart-pane {
    display: none;
  }

  /* When currentTab === 'cart', hide scanner and show cart */
  .register-layout.mobile-show-cart .operation-pane {
    display: none;
  }
  .register-layout.mobile-show-cart .cart-pane {
    display: flex;
  }
}

@media (min-width: 1024px) {
  /* On PC: side-by-side dual-pane */
  .register-layout {
    grid-template-columns: 1.2fr 0.9fr;
    gap: 1.5rem;
    align-items: start;
  }
  .cart-pane {
    display: flex !important;
  }
  .operation-pane {
    display: flex !important;
  }
  .mobile-back-to-scan-btn {
    display: none !important;
  }
}

.pane {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
}

/* ─────────────────────────── ACTION BUTTONS (MOBILE-FIRST) ─────────────────────────── */
.action-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
  width: 100%;
}

.touch-btn {
  border: none;
  border-radius: 14px;
  padding: 0.75rem 1rem;
  font-size: 0.95rem;
  font-weight: 700;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  transition: all 0.15s ease;
  min-height: 48px; /* Touch ergonomics */
  text-align: center;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
}

.touch-btn:active {
  transform: scale(0.97);
}

.touch-btn.primary {
  background: linear-gradient(135deg, #38bdf8 0%, #6366f1 100%);
  color: #0f172a;
  box-shadow: 0 4px 12px rgba(56, 189, 248, 0.3);
}

.touch-btn.primary.active {
  background: #ef4444;
  color: #ffffff;
  box-shadow: 0 4px 12px rgba(239, 68, 68, 0.35);
}

.touch-btn.outline {
  background: #151f32;
  border: 1.5px solid #38bdf8;
  color: #38bdf8;
}

.touch-btn.secondary {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
}

.touch-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
  box-shadow: none;
}

.flex-1 {
  flex: 1;
}

/* ─────────────────────────── BARCODE SEARCH INPUT ─────────────────────────── */
.barcode-search-box {
  width: 100%;
}

.search-input-wrapper {
  display: flex;
  align-items: center;
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 14px;
  padding: 0.25rem 0.35rem 0.25rem 0.85rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.search-input-wrapper:focus-within {
  border-color: #38bdf8;
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.2);
}

.search-icon {
  font-size: 1.1rem;
  color: #94a3b8;
  margin-right: 0.4rem;
}

.search-input {
  flex: 1;
  background: transparent;
  border: none;
  outline: none;
  color: #f8fafc;
  font-size: 16px; /* 16px prevents iOS Safari auto-zoom on focus! */
  font-family: monospace;
  min-width: 0;
}

.search-submit-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.6rem 1rem;
  border-radius: 10px;
  font-weight: 700;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.search-submit-btn:hover:not(:disabled) {
  background: #38bdf8;
  color: #0f172a;
}

.search-submit-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* ─────────────────────────── MOBILE OPTIMIZED CAMERA ─────────────────────────── */
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
  padding: 0.75rem 1rem;
  background: #151f32;
  border-bottom: 1px solid #1e293b;
}

.camera-status {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 9999px;
  background: #ef4444;
  animation: pulse 1.5s infinite;
}

.status-label {
  font-size: 0.75rem;
  font-weight: 800;
  color: #ef4444;
  letter-spacing: 0.05em;
}

.camera-header-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.cam-tool-btn {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.3rem 0.65rem;
  border-radius: 8px;
  font-size: 0.75rem;
  font-weight: 700;
  cursor: pointer;
}

.cam-tool-btn.on {
  background: #f59e0b;
  border-color: #f59e0b;
  color: #0f172a;
}

.cam-close-btn {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 1.25rem;
  cursor: pointer;
  padding: 0 0.25rem;
}

.viewfinder-container {
  position: relative;
  width: 100%;
  aspect-ratio: 4 / 3;
  max-height: 60vh;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

@media (min-width: 640px) {
  .viewfinder-container {
    aspect-ratio: 16 / 10;
  }
}

.video-feed {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.scanner-guide {
  position: absolute;
  width: 75%;
  max-width: 300px;
  height: 55%;
  pointer-events: none;
}

.corner {
  position: absolute;
  width: 24px;
  height: 24px;
  border: 3px solid #38bdf8;
}

.corner-tl { top: 0; left: 0; border-right: none; border-bottom: none; border-radius: 6px 0 0 0; }
.corner-tr { top: 0; right: 0; border-left: none; border-bottom: none; border-radius: 0 6px 0 0; }
.corner-bl { bottom: 0; left: 0; border-right: none; border-top: none; border-radius: 0 0 0 6px; }
.corner-br { bottom: 0; right: 0; border-left: none; border-top: none; border-radius: 0 0 6px 0; }

.aim-line {
  position: absolute;
  top: 50%;
  left: 4%;
  right: 4%;
  height: 2px;
  background: rgba(56, 189, 248, 0.9);
  box-shadow: 0 0 10px #38bdf8;
}

.guide-text {
  position: absolute;
  bottom: 84px;
  width: 100%;
  text-align: center;
  font-size: 0.8rem;
  color: #f8fafc;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.9);
  pointer-events: none;
  padding: 0 1rem;
}

.camera-controls-bar {
  position: absolute;
  bottom: 12px;
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Big Mobile Touch Shutter Button */
.shutter-trigger {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  transition: transform 0.15s ease;
  -webkit-tap-highlight-color: transparent;
}

.shutter-trigger:active:not(:disabled) {
  transform: scale(0.9);
}

.shutter-ring {
  width: 72px;
  height: 72px;
  border-radius: 9999px;
  border: 4px solid #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 0 25px rgba(56, 189, 248, 0.55);
}

.shutter-dot {
  width: 54px;
  height: 54px;
  border-radius: 9999px;
  background: linear-gradient(135deg, #38bdf8, #818cf8);
}

/* ── Snapshot Preview Screen ── */
.snapshot-container {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
  background: #151f32;
}

.snapshot-view {
  position: relative;
  width: 100%;
  max-width: 440px;
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
  font-size: 0.85rem;
  text-align: center;
  max-width: 400px;
}

.snapshot-actions {
  display: flex;
  gap: 0.5rem;
  width: 100%;
  max-width: 440px;
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
  padding: 0.75rem 1rem;
  background: #1e293b;
  border-bottom: 1px solid #334155;
}

.found-badge {
  font-size: 0.78rem;
  font-weight: 700;
  color: #10b981;
}

.barcode-chip {
  background: #0f172a;
  padding: 0.2rem 0.5rem;
  border-radius: 6px;
  font-size: 0.75rem;
  color: #38bdf8;
}

.product-body {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

@media (min-width: 640px) {
  .product-body {
    flex-direction: row;
  }
}

.product-media {
  flex-shrink: 0;
  width: 100%;
  max-width: 160px;
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.product-image {
  width: 100%;
  height: 140px;
  object-fit: contain;
  background: #ffffff;
  border-radius: 12px;
  padding: 0.5rem;
}

.product-image-fallback {
  width: 100%;
  height: 140px;
  background: #1e293b;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 3.5rem;
}

.nutri-tag {
  font-size: 0.75rem;
  padding: 0.2rem 0.6rem;
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
  gap: 0.6rem;
}

.product-title {
  font-size: 1.15rem;
  font-weight: 700;
  color: #f8fafc;
  line-height: 1.3;
}

.product-meta-row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem 0.75rem;
  font-size: 0.8rem;
  color: #94a3b8;
}

.price-edit-box {
  margin-top: 0.35rem;
  padding: 0.75rem;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 12px;
}

.input-label {
  font-size: 0.78rem;
  color: #94a3b8;
}

.price-input-row {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  margin: 0.3rem 0;
}

.currency-prefix {
  font-size: 1.2rem;
  font-weight: 800;
  color: #38bdf8;
}

.price-input-field {
  background: #1e293b;
  border: 1px solid #334155;
  border-radius: 8px;
  color: #f8fafc;
  font-size: 1.25rem;
  font-weight: 700;
  padding: 0.35rem 0.65rem;
  width: 130px;
  outline: none;
}

.quick-price-chips {
  display: flex;
  gap: 0.35rem;
  flex-wrap: wrap;
  margin-top: 0.4rem;
}

.chip {
  background: #1e293b;
  border: 1px solid #334155;
  color: #cbd5e1;
  padding: 0.25rem 0.55rem;
  font-size: 0.72rem;
  font-weight: 600;
  border-radius: 6px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}

.chip:active {
  background: #38bdf8;
  color: #0f172a;
}

.chip.reset {
  color: #ef4444;
}

.product-actions {
  display: flex;
  gap: 0.6rem;
  margin-top: 0.4rem;
}

/* ─────────────────────────── MANUAL PRODUCT CARD ─────────────────────────── */
.manual-product-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.manual-header {
  display: flex;
  align-items: center;
  gap: 0.65rem;
}

.notice-icon {
  font-size: 1.8rem;
}

.form-grid {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.form-field {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.form-field label {
  font-size: 0.8rem;
  color: #94a3b8;
}

.field-input {
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 10px;
  color: #f8fafc;
  padding: 0.65rem 0.85rem;
  font-size: 16px; /* Prevents auto-zoom on mobile */
  outline: none;
}

.field-input:focus {
  border-color: #38bdf8;
}

.form-actions {
  display: flex;
  gap: 0.65rem;
}

/* ─────────────────────────── SCANNER HERO (IDLE) ─────────────────────────── */
.scanner-hero {
  background: #151f32;
  border: 1px dashed #334155;
  border-radius: 20px;
  padding: 2rem 1.25rem;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
}

.hero-icon-cluster {
  display: flex;
  gap: 0.5rem;
}

.cluster-icon {
  font-size: 1.8rem;
  background: #1e293b;
  width: 48px;
  height: 48px;
  border-radius: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.hero-desc {
  color: #94a3b8;
  font-size: 0.9rem;
  max-width: 440px;
  line-height: 1.45;
}

.feature-pills {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
  justify-content: center;
}

.pill {
  background: #1e293b;
  color: #38bdf8;
  font-size: 0.72rem;
  font-weight: 600;
  padding: 0.2rem 0.65rem;
  border-radius: 9999px;
  border: 1px solid #334155;
}

/* ─────────────────────────── CART CARD & TICKET ─────────────────────────── */
.cart-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.4);
}

@media (min-width: 1024px) {
  .cart-card {
    position: sticky;
    top: 5rem;
    max-height: calc(100vh - 6rem);
  }
}

.cart-card-header {
  padding: 0.85rem 1rem;
  background: #1e293b;
  border-bottom: 1px solid #334155;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.cart-title-row {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.cart-heading {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.cart-heading h2 {
  font-size: 1.05rem;
  font-weight: 700;
}

.item-count-badge {
  background: #0f172a;
  border: 1px solid #334155;
  color: #38bdf8;
  font-size: 0.72rem;
  font-weight: 700;
  padding: 0.15rem 0.5rem;
  border-radius: 9999px;
}

.cart-header-actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.mobile-back-to-scan-btn {
  background: #0f172a;
  border: 1px solid #334155;
  color: #38bdf8;
  padding: 0.35rem 0.75rem;
  border-radius: 8px;
  font-size: 0.78rem;
  font-weight: 700;
  cursor: pointer;
}

.cart-items-wrapper {
  flex: 1;
  overflow-y: auto;
  padding: 0.85rem 1rem;
  min-height: 200px;
  max-height: 420px;
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
  font-size: 2.8rem;
  opacity: 0.6;
}

.empty-title {
  font-size: 1rem;
  font-weight: 600;
  color: #94a3b8;
}

.cart-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.cart-item-row {
  display: flex;
  align-items: center;
  gap: 0.6rem;
  background: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 12px;
  padding: 0.6rem 0.75rem;
}

.item-img-box {
  width: 40px;
  height: 40px;
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
  font-size: 1.25rem;
}

.item-main {
  flex: 1;
  min-width: 0;
}

.item-name {
  font-size: 0.85rem;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  color: #f8fafc;
}

.item-sub-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.item-barcode {
  font-size: 0.7rem;
  color: #64748b;
}

.item-unit-price {
  font-size: 0.75rem;
  color: #94a3b8;
  font-weight: 600;
}

.qty-stepper {
  display: flex;
  align-items: center;
  gap: 0.25rem;
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
  min-width: 18px;
  text-align: center;
  font-size: 0.82rem;
  font-weight: 700;
}

.item-subtotal {
  font-size: 0.9rem;
  font-weight: 700;
  color: #38bdf8;
  min-width: 52px;
  text-align: right;
}

.remove-item-btn {
  background: none;
  border: none;
  cursor: pointer;
  opacity: 0.45;
  padding: 0.2rem;
}

.remove-item-btn:hover {
  opacity: 1;
}

/* ── Cart Financial Summary ── */
.cart-summary-footer {
  padding: 1rem;
  background: #111827;
  border-top: 1px solid #334155;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.summary-line {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #94a3b8;
}

.summary-divider {
  height: 1px;
  background: #334155;
  margin: 0.15rem 0;
}

.summary-line.grand-total {
  font-size: 1.15rem;
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
  padding: 0.85rem 1.1rem;
  border-radius: 12px;
  font-size: 1rem;
  font-weight: 800;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 0.4rem;
  transition: all 0.2s ease;
  box-shadow: 0 4px 14px rgba(56, 189, 248, 0.35);
  min-height: 48px;
}

.checkout-submit-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  box-shadow: none;
}

/* ─────────────────────────── SALES HISTORY (MOBILE CARDS) ─────────────────────────── */
.history-layout {
  width: 100%;
}

.history-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 1.25rem;
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.danger-btn {
  border-color: #ef4444;
  color: #ef4444;
}

.history-list-mobile {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.history-item-card {
  background: #0f172a;
  border: 1px solid #1e293b;
  border-radius: 14px;
  padding: 0.9rem;
  display: flex;
  flex-direction: column;
  gap: 0.45rem;
  cursor: pointer;
  transition: border-color 0.15s ease;
}

.history-item-card:hover, .history-item-card:active {
  border-color: #38bdf8;
}

.history-card-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-card-mid {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-card-bot {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 0.35rem;
  border-top: 1px dashed #1e293b;
}

.history-total-amt {
  font-weight: 800;
  font-size: 1.1rem;
  color: #38bdf8;
}

.method-tag {
  font-size: 0.72rem;
  padding: 0.15rem 0.5rem;
  border-radius: 6px;
  background: #1e293b;
  font-weight: 600;
}

/* ─────────────────────────── MOBILE BOTTOM NAVIGATION BAR ─────────────────────────── */
.mobile-bottom-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 50;
  background: rgba(15, 23, 42, 0.96);
  backdrop-filter: blur(14px);
  border-top: 1px solid #334155;
  display: flex;
  justify-content: space-around;
  padding: 0.45rem 0.5rem calc(0.45rem + env(safe-area-inset-bottom, 8px));
}

@media (min-width: 1024px) {
  .mobile-bottom-nav {
    display: none;
  }
}

.bottom-nav-item {
  flex: 1;
  background: none;
  border: none;
  color: #94a3b8;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.2rem;
  cursor: pointer;
  font-size: 0.72rem;
  font-weight: 600;
  padding: 0.3rem 0;
  transition: all 0.15s ease;
  -webkit-tap-highlight-color: transparent;
}

.bottom-nav-item.active {
  color: #38bdf8;
}

.bottom-icon {
  font-size: 1.35rem;
}

.icon-badge-wrapper {
  position: relative;
  display: inline-block;
}

.bottom-badge {
  position: absolute;
  top: -4px;
  right: -10px;
  background: #38bdf8;
  color: #0f172a;
  border-radius: 9999px;
  font-size: 0.65rem;
  font-weight: 800;
  padding: 0.05rem 0.35rem;
}

/* ─────────────────────────── MODALS & MOBILE BOTTOM SHEETS ─────────────────────────── */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(11, 17, 32, 0.85);
  backdrop-filter: blur(8px);
  z-index: 100;
  display: flex;
  align-items: flex-end; /* Slide from bottom on mobile */
  justify-content: center;
  padding: 0;
}

@media (min-width: 640px) {
  .modal-backdrop {
    align-items: center;
    padding: 1rem;
  }
}

.modal-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 24px 24px 0 0; /* Bottom sheet styling on mobile */
  width: 100%;
  max-width: 520px;
  box-shadow: 0 -10px 40px rgba(0, 0, 0, 0.7);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  max-height: 85vh;
  animation: slideUp 0.25s ease-out;
}

@media (min-width: 640px) {
  .modal-card {
    border-radius: 20px;
    box-shadow: 0 20px 50px rgba(0, 0, 0, 0.6);
    max-height: 90vh;
  }
}

.modal-sheet-handle {
  width: 40px;
  height: 4px;
  background: #475569;
  border-radius: 9999px;
  margin: 0.6rem auto 0.2rem;
}

@media (min-width: 640px) {
  .mobile-only {
    display: none !important;
  }
}

.modal-header {
  padding: 0.85rem 1.15rem;
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
  font-size: 1.25rem;
  cursor: pointer;
  padding: 0.2rem;
}

.modal-footer {
  padding: 0.85rem 1.15rem;
  background: #111827;
  border-top: 1px solid #334155;
  display: flex;
  justify-content: flex-end;
  gap: 0.65rem;
}

/* ── Payment Body ── */
.payment-body {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow-y: auto;
}

.pay-total-callout {
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 14px;
  padding: 0.85rem 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.callout-value {
  font-size: 1.5rem;
  font-weight: 800;
  color: #38bdf8;
}

.payment-methods-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.5rem;
}

.method-btn {
  background: #1e293b;
  border: 2px solid transparent;
  color: #f8fafc;
  padding: 0.75rem 0.4rem;
  border-radius: 12px;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.35rem;
  transition: all 0.15s ease;
}

.method-btn.selected {
  border-color: #38bdf8;
  background: rgba(56, 189, 248, 0.15);
}

.method-icon {
  font-size: 1.4rem;
}

.method-name {
  font-size: 0.75rem;
  font-weight: 700;
}

.cash-section {
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
}

.cash-input-row {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.field-input.large {
  font-size: 1.35rem;
  font-weight: 700;
  width: 100%;
}

.quick-cash-row {
  display: flex;
  gap: 0.35rem;
  flex-wrap: wrap;
}

.cash-chip {
  background: #1e293b;
  border: 1px solid #334155;
  color: #f8fafc;
  padding: 0.35rem 0.65rem;
  border-radius: 8px;
  font-size: 0.78rem;
  font-weight: 600;
  cursor: pointer;
}

.cash-chip:active {
  background: #38bdf8;
  color: #0f172a;
}

.change-callout {
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 12px;
  padding: 0.75rem 1rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.change-amount {
  font-size: 1.25rem;
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
  padding: 1.25rem;
  gap: 0.65rem;
}

.qr-badge {
  background: #03496b;
  color: #ffffff;
  padding: 0.2rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 800;
}

.qr-code-placeholder {
  background: #ffffff;
  padding: 0.65rem;
  border-radius: 12px;
}

.qr-image {
  width: 140px;
  height: 140px;
  display: block;
}

.qr-hint {
  font-size: 0.8rem;
  color: #94a3b8;
}

.qr-amount {
  font-size: 1.3rem;
  font-weight: 800;
  color: #38bdf8;
}

.card-terminal-mock {
  text-align: center;
  padding: 2rem 1rem;
  background: #0f172a;
  border: 1px solid #334155;
  border-radius: 16px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4rem;
}

.terminal-icon {
  font-size: 2.8rem;
}

/* ─────────────────────────── PAPER RECEIPT ─────────────────────────── */
.receipt-paper {
  background: #ffffff;
  color: #111827;
  padding: 1.25rem 1rem;
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.82rem;
  line-height: 1.4;
  overflow-y: auto;
  max-height: 55vh;
}

.receipt-brand {
  text-align: center;
}

.receipt-brand h2 {
  font-size: 1.2rem;
  font-weight: 900;
}

.receipt-separator {
  text-align: center;
  margin: 0.4rem 0;
  color: #64748b;
  overflow: hidden;
}

.receipt-line-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.35rem;
}

.receipt-line-left {
  display: flex;
  flex-direction: column;
}

.receipt-item-title {
  font-weight: bold;
}

.receipt-item-calc {
  font-size: 0.72rem;
  color: #475569;
}

.receipt-totals {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.receipt-row {
  display: flex;
  justify-content: space-between;
}

.receipt-row.grand-line {
  font-size: 1.05rem;
  font-weight: 900;
  margin: 0.3rem 0;
  border-top: 1px dashed #111;
  border-bottom: 1px dashed #111;
  padding: 0.2rem 0;
}

.receipt-footer {
  text-align: center;
  font-size: 0.72rem;
  color: #64748b;
}

/* ─────────────────────────── UTILITY STYLES ─────────────────────────── */
.text-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 0.78rem;
  color: #94a3b8;
}

.text-btn.danger {
  color: #ef4444;
}

.mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}

.muted {
  color: #94a3b8;
  font-size: 0.82rem;
}

.text-sm {
  font-size: 0.78rem;
}

.spinner {
  width: 28px;
  height: 28px;
  border: 3px solid rgba(255, 255, 255, 0.2);
  border-top-color: #38bdf8;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.spinner.large {
  width: 44px;
  height: 44px;
  border-width: 4px;
}

.loading-card {
  background: #151f32;
  border: 1px solid #334155;
  border-radius: 20px;
  padding: 2.5rem 1.25rem;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
}

.alert-banner {
  padding: 0.65rem 0.85rem;
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
  font-size: 2.8rem;
  display: block;
  margin-bottom: 0.4rem;
}

.mt-3 {
  margin-top: 0.75rem;
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
  from { opacity: 0; transform: translateY(-8px); }
  to { opacity: 1; transform: translateY(0); }
}

@keyframes slideUp {
  from { transform: translateY(100%); }
  to { transform: translateY(0); }
}
</style>
