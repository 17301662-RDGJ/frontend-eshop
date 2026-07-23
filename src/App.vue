<template>
  <div class="store-container">
    <!-- Navbar / Encabezado de Tienda -->
    <header class="navbar">
      <div class="logo">
        <span>🛒</span> Tienda Sol
      </div>
      <div class="cart-icon-btn" @click="mostrarCarrito = !mostrarCarrito">
        <span>🛒 Carrito</span>
        <span class="badge" v-if="itemsCarrito.length > 0">{{ totalItems }}</span>
      </div>
    </header>

    <!-- Banner / Promoción -->
    <section class="hero-banner">
      <h1>¡Encuentra los mejores productos al mejor precio!</h1>
      <p>Aprovecha nuestros descuentos de temporada</p>
    </section>

    <!-- Catálogo de Productos -->
    <main class="main-content">
      <h2>Nuestros Productos</h2>
      
      <div v-if="cargando" class="loading">Cargando catálogo...</div>

      <div v-else class="product-grid">
        <div v-for="producto in productos" :key="producto.id" class="product-card">
          <div class="card-image">
            <span>📦</span>
          </div>
          <div class="card-body">
            <h3>{{ producto.name }}</h3>
            <p class="description">{{ producto.description || 'Sin descripción disponible.' }}</p>
            <div class="card-footer">
              <span class="price">${{ producto.price }}</span>
              <button class="btn-add-cart" @click="agregarAlCarrito(producto)">
                + Agregar
              </button>
            </div>
          </div>
        </div>
      </div>
    </main>

    <!-- Modal / Drawer del Carrito de Compras -->
    <div class="cart-modal" :class="{ 'is-open': mostrarCarrito }">
      <div class="cart-header">
        <h3>Tu Carrito de Compras</h3>
        <button class="close-btn" @click="mostrarCarrito = false">✕</button>
      </div>

      <div class="cart-body">
        <div v-if="itemsCarrito.length === 0" class="empty-cart">
          Tu carrito está vacío.
        </div>

        <div v-else class="cart-items-list">
          <div v-for="item in itemsCarrito" :key="item.productId" class="cart-item">
            <div class="item-info">
              <h4>{{ item.productName }}</h4>
              <span class="item-price">${{ item.price }} c/u</span>
            </div>
            <div class="item-actions">
              <input 
                type="number" 
                v-model.number="item.quantity" 
                min="1" 
                @change="guardarCarritoEnBackend"
                class="qty-input"
              />
              <button class="remove-btn" @click="eliminarDelCarrito(item.productId)">🗑️</button>
            </div>
          </div>
        </div>
      </div>

      <div class="cart-footer" v-if="itemsCarrito.length > 0">
        <div class="total-row">
          <span>Total:</span>
          <strong>${{ totalPrecio.toFixed(2) }}</strong>
        </div>
        <button class="btn-checkout" @click="realizarCompra">Procesar Compra</button>
      </div>
    </div>
  </div>
</template>
<script setup>
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

// Configuración de URLs de Microservicios (Ajusta las variables de entorno o endpoints de Render)
const CATALOG_API = 'https://eshop-services-az0q.onrender.com' // Tu API de catálogo
const BASKET_API = 'https://tu-basket-api.onrender.com'       // Tu API de Basket (Ajustar URL real)

const USER_NAME = 'sol_cliente' // Identificador de usuario para guardar el Basket en .NET

// Estados
const productos = ref([])
const itemsCarrito = ref([])
const cargando = ref(true)
const mostrarCarrito = ref(false)

// Computados
const totalItems = computed(() => {
  return itemsCarrito.value.reduce((acc, item) => acc + item.quantity, 0)
})

const totalPrecio = computed(() => {
  return itemsCarrito.value.reduce((acc, item) => acc + (item.price * item.quantity), 0)
})

// Cargar datos iniciales
onMounted(async () => {
  await obtenerProductos()
  await cargarCarritoDesdeBackend()
})

// 1. Obtener catálogo de productos
const obtenerProductos = async () => {
  try {
    const res = await axios.get(`${CATALOG_API}/products`)
    productos.value = res.data.products || res.data
  } catch (err) {
    console.error('Error al cargar catálogo:', err)
  } finally {
    cargando.value = false
  }
}

// 2. Obtener carrito existente desde el Microservicio de Basket (.NET)
const cargarCarritoDesdeBackend = async () => {
  try {
    const res = await axios.get(`${BASKET_API}/basket/${USER_NAME}`)
    if (res.data && res.data.items) {
      itemsCarrito.value = res.data.items
    }
  } catch (err) {
    console.warn('No se encontró carrito previo en Basket API o ocurrió un error:', err)
  }
}

// 3. Guardar cambios del carrito en .NET Basket API
const guardarCarritoEnBackend = async () => {
  try {
    // Formato estándar DTO para el ShoppingCartItem/ShoppingCartBasket en .NET
    const payload = {
      userName: USER_NAME,
      items: itemsCarrito.value.map(item => ({
        productId: item.productId,
        productName: item.productName,
        price: item.price,
        quantity: item.quantity,
        color: item.color || 'Default'
      }))
    }

    await axios.post(`${BASKET_API}/basket`, payload)
  } catch (err) {
    console.error('Error al guardar carrito en el backend:', err)
  }
}

// 4. Agregar producto al carrito
const agregarAlCarrito = async (producto) => {
  const existe = itemsCarrito.value.find(item => item.productId === producto.id)

  if (existe) {
    existe.quantity++
  } else {
    itemsCarrito.value.push({
      productId: producto.id,
      productName: producto.name,
      price: producto.price,
      quantity: 1
    })
  }

  mostrarCarrito.value = true
  await guardarCarritoEnBackend()
}

// 5. Eliminar un producto del carrito
const eliminarDelCarrito = async (productId) => {
  itemsCarrito.value = itemsCarrito.value.filter(item => item.productId !== productId)
  await guardarCarritoEnBackend()
}

// 6. Finalizar la compra
const realizarCompra = async () => {
  alert('¡Compra procesada con éxito!')
  itemsCarrito.value = []
  
  try {
    // Limpiar basket en backend
    await axios.delete(`${BASKET_API}/basket/${USER_NAME}`)
  } catch (err) {
    console.error('Error al limpiar el basket:', err)
  }
  mostrarCarrito.value = false
}
</script>