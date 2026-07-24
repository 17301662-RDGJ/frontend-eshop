<script setup>
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

// Configuración de URLs de las APIs
const CATALOG_API = 'https://eshop-services-az0q.onrender.com'
const BASKET_API = 'https://eshop-services-az0q.onrender.com'
const USER_NAME = 'sol_cliente'

// Estados reactivos principales
const productos = ref([])
const itemsCarrito = ref([])
const cargando = ref(true)
const mostrarCarrito = ref(false)

// Estados para Modales de Administración (CRUD)
const mostrarModalCrear = ref(false)
const mostrarModalEditar = ref(false)
const textoBusqueda = ref('')

// Formulario de Producto (Crear / Editar)
const productoForm = ref({
  id: null,
  name: '',
  description: '',
  price: 0,
  category: '',
  imageFile: ''
})

// --- PROPIEDADES COMPUTADAS ---
const totalItems = computed(() => itemsCarrito.value.reduce((acc, item) => acc + item.quantity, 0))
const totalPrecio = computed(() => itemsCarrito.value.reduce((acc, item) => acc + (item.price * item.quantity), 0))

// Filtrar productos para BÚSQUEDA en tiempo real
const productosFiltrados = computed(() => {
  if (!textoBusqueda.value.trim()) return productos.value
  return productos.value.filter(p => 
    p.name && p.name.toLowerCase().includes(textoBusqueda.value.toLowerCase())
  )
})

// Función auxiliar para obtener la URL de imagen del producto (soporta imageFile, imagesFiles o image)
const obtenerUrlImagen = (producto) => {
  return producto.imageFile || producto.imagesFiles || producto.image || ''
}

// Función auxiliar para obtener la descripción del producto (soporta description o descripcion)
const obtenerDescripcion = (producto) => {
  return producto.description || producto.descripcion || 'Sin descripción disponible.'
}

// --- FUNCIONES CRUD (CATÁLOGO) ---

// 1. OBTENER / CONSULTAR PRODUCTOS
const obtenerProductos = async () => {
  cargando.value = true
  try {
    const res = await axios.get(`${CATALOG_API}/products`)
    productos.value = res.data.products || res.data
  } catch (err) {
    console.error('Error al cargar catálogo:', err)
  } finally {
    cargando.value = false
  }
}

// 2. INSERTAR PRODUCTO
const guardarNuevoProducto = async () => {
  try {
    const payload = {
      name: productoForm.value.name,
      description: productoForm.value.description, // Enviamos ambos para C#
      descripcion: productoForm.value.description,
      price: Number(productoForm.value.price),
      category: [productoForm.value.category || 'General'],
      imageFile: productoForm.value.imageFile || 'default.png',
      imagesFiles: productoForm.value.imageFile || 'default.png'
    }

    await axios.post(`${CATALOG_API}/products`, payload)
    
    alert('¡Producto creado exitosamente!')
    mostrarModalCrear.value = false // Corregido: antes decía mostrarModalNuevo
    limpiarFormulario()
    await obtenerProductos() // Corregido: antes decía cargarProductos
  } catch (error) {
    console.error('Error al crear producto:', error)
    alert('Error al crear el producto')
  }
}

// 3. EDITAR PRODUCTO
const abrirEditar = (prod) => {
  productoForm.value = {
    id: prod.id,
    name: prod.name,
    description: prod.description || prod.descripcion || '', // Lee cualquiera de las dos
    price: prod.price,
    category: Array.isArray(prod.category) ? prod.category[0] : (prod.category || ''),
    imageFile: prod.imageFile || prod.imagesFiles || '' // Lee cualquiera de las dos
  }
  mostrarModalEditar.value = true
}

const actualizarProducto = async () => {
  try {
    const payload = {
      id: productoForm.value.id,
      name: productoForm.value.name,
      description: productoForm.value.description, // Enviamos tanto description como descripcion
      descripcion: productoForm.value.description,
      price: Number(productoForm.value.price),
      category: [productoForm.value.category || 'General'],
      imageFile: productoForm.value.imageFile || 'default.png',
      imagesFiles: productoForm.value.imageFile || 'default.png'
    }

    // Petición a la API (probamos Endpoint general /products)
    await axios.put(`${CATALOG_API}/products`, payload)
    alert('¡Producto actualizado!')
    mostrarModalEditar.value = false
    limpiarFormulario()
    await obtenerProductos()
  } catch (err) {
    console.error('Error al actualizar producto:', err)
    alert('Error al actualizar el producto')
  }
}

// 4. ELIMINAR PRODUCTO
const eliminarProducto = async (id) => {
  if (!confirm('¿Estás segura de que deseas eliminar este producto?')) return

  try {
    await axios.delete(`${CATALOG_API}/products/${id}`)
    alert('Producto eliminado correctamente')
    await obtenerProductos()
  } catch (err) {
    console.error('Error al eliminar producto:', err)
    alert('Error al eliminar el producto')
  }
}

const limpiarFormulario = () => {
  productoForm.value = { id: null, name: '', description: '', price: 0, category: '', imageFile: '' }
}

// --- FUNCIONES DEL CARRITO (BASKET) ---
const cargarCarritoDesdeBackend = async () => {
  try {
    const res = await axios.get(`${BASKET_API}/basket/${USER_NAME}`)
    if (res.data && res.data.items) {
      itemsCarrito.value = res.data.items
    }
  } catch (err) {
    console.warn('No se pudo cargar la cesta:', err)
  }
}

const guardarCarritoEnBackend = async () => {
  try {
    const payload = {
      userName: USER_NAME,
      items: itemsCarrito.value.map(item => ({
        productId: item.productId,
        productName: item.productName,
        price: item.price,
        quantity: item.quantity
      }))
    }
    await axios.post(`${BASKET_API}/basket`, payload)
  } catch (err) {
    console.error('Error al guardar cesta:', err)
  }
}

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

const eliminarDelCarrito = async (productId) => {
  itemsCarrito.value = itemsCarrito.value.filter(item => item.productId !== productId)
  await guardarCarritoEnBackend()
}

const realizarCompra = async () => {
  alert('¡Compra procesada con éxito!')
  itemsCarrito.value = []
  try {
    await axios.delete(`${BASKET_API}/basket/${USER_NAME}`)
  } catch (err) {
    console.error('Error al vaciar cesta:', err)
  }
  mostrarCarrito.value = false
}

// Cargar datos al iniciar
onMounted(async () => {
  await obtenerProductos()
  await cargarCarritoDesdeBackend()
})
</script>

<template>
  <div class="store-container">
    <!-- Navbar / Encabezado -->
    <header class="navbar">
      <div class="logo">
        <span>☀️</span> Tienda Sol
      </div>

      <div class="nav-actions">
        <button class="btn-admin" @click="mostrarModalCrear = true">➕ Nuevo Producto</button>
        <div class="cart-icon-btn" @click="mostrarCarrito = !mostrarCarrito">
          <span>🛒 Carrito</span>
          <span class="badge" v-if="itemsCarrito.length > 0">{{ totalItems }}</span>
        </div>
      </div>
    </header>

    <!-- Banner Principal -->
    <section class="hero-banner">
      <h1 class="hero-title">¡Encuentra los mejores productos al mejor precio!</h1>
      <p class="hero-subtitle">Aprovecha nuestros descuentos de temporada</p>
      
      <!-- Buscador de Productos -->
      <div class="search-bar-container">
        <input 
          v-model="textoBusqueda" 
          type="text" 
          placeholder="🔍 Buscar productos por nombre..." 
          class="search-input"
        />
      </div>
    </section>

    <!-- Catálogo de Productos -->
    <main class="main-content">
      <h2>Nuestros Productos ({{ productosFiltrados.length }})</h2>
      
      <div v-if="cargando" class="loading">Cargando catálogo...</div>

      <div v-else-if="productosFiltrados.length === 0" class="no-results">
        No se encontraron productos que coincidan con "{{ textoBusqueda }}".
      </div>

      <div v-else class="product-grid">
        <div v-for="producto in productosFiltrados" :key="producto.id" class="product-card">
          
          <!-- Contenedor de la Imagen del Producto -->
          <div class="card-image-container">
            <img 
              v-if="obtenerUrlImagen(producto) && obtenerUrlImagen(producto).startsWith('http')" 
              :src="obtenerUrlImagen(producto)" 
              :alt="producto.name"
              class="product-img"
              @error="(e) => e.target.style.display = 'none'"
            />
            <div v-else class="placeholder-img">
              <span>👗</span>
            </div>
          </div>

          <div class="card-body">
            <h3>{{ producto.name }}</h3>
            <!-- Descripción del producto CORREGIDA -->
            <p class="description">{{ obtenerDescripcion(producto) }}</p>
            <div class="card-footer">
              <span class="price">${{ producto.price }}</span>
              <button class="btn-add-cart" @click="agregarAlCarrito(producto)">
                + Agregar
              </button>
            </div>
          </div>
          
          <!-- Controles Admin de Editar / Eliminar -->
          <div class="card-admin-actions">
            <button class="btn-edit" @click="abrirEditar(producto)">✏️ Editar</button>
            <button class="btn-delete" @click="eliminarProducto(producto.id)">🗑️ Eliminar</button>
          </div>
        </div>
      </div>
    </main>

    <!-- MODAL INSERTAR PRODUCTO -->
    <div class="modal-overlay" v-if="mostrarModalCrear">
      <div class="modal-box">
        <h3>➕ Insertar Nuevo Producto</h3>
        <div class="form-group">
          <label>Nombre:</label>
          <input v-model="productoForm.name" type="text" placeholder="Ej: Vestido Elegante" />
        </div>
        <div class="form-group">
          <label>Precio:</label>
          <input v-model.number="productoForm.price" type="number" placeholder="Ej: 299" />
        </div>
        <div class="form-group">
          <label>URL de la Imagen:</label>
          <input v-model="productoForm.imageFile" type="text" placeholder="Ej: https://ejemplo.com/imagen.jpg" />
        </div>
        <div class="form-group">
          <label>Descripción:</label>
          <textarea v-model="productoForm.description" placeholder="Descripción del producto"></textarea>
        </div>
        <div class="form-group">
          <label>Categoría:</label>
          <input v-model="productoForm.category" type="text" placeholder="Ej: Ropa" />
        </div>
        <div class="modal-buttons">
          <button class="btn-save" @click="guardarNuevoProducto">Guardar</button>
          <button class="btn-cancel" @click="mostrarModalCrear = false">Cancelar</button>
        </div>
      </div>
    </div>

    <!-- MODAL EDITAR PRODUCTO -->
    <div class="modal-overlay" v-if="mostrarModalEditar">
      <div class="modal-box">
        <h3>✏️ Editar Producto</h3>
        <div class="form-group">
          <label>Nombre:</label>
          <input v-model="productoForm.name" type="text" />
        </div>
        <div class="form-group">
          <label>Precio:</label>
          <input v-model.number="productoForm.price" type="number" />
        </div>
        <div class="form-group">
          <label>URL de la Imagen:</label>
          <input v-model="productoForm.imageFile" type="text" placeholder="https://ejemplo.com/imagen.jpg" />
        </div>
        <div class="form-group">
          <label>Descripción:</label>
          <textarea v-model="productoForm.description"></textarea>
        </div>
        <div class="form-group">
          <label>Categoría:</label>
          <input v-model="productoForm.category" type="text" />
        </div>
        <div class="modal-buttons">
          <button class="btn-save" @click="actualizarProducto">Actualizar</button>
          <button class="btn-cancel" @click="mostrarModalEditar = false">Cancelar</button>
        </div>
      </div>
    </div>

    <!-- Drawer Lateral del Carrito -->
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