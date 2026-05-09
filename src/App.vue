<template>
  <div class="menu-app">
    <header class="navbar">
      <div class="navbar-content">
        <div class="brand">
          <h1>El Sazón</h1>
          <span class="location">📍 San Gil, Santander</span>
        </div>
      </div>
    </header>

    <main class="menu-grid">
      <div v-for="item in platos" :key="item.id" class="card" :class="{ 'sin-stock': item.stock === 0 }">
        <div class="card-image">
          <img :src="item.imagen" :alt="item.nombre" loading="lazy">
          <div v-if="item.stock === 0" class="overlay-agotado">AGOTADO</div>
        </div>
        
        <div class="card-body">
          <div class="card-info">
            <h3>{{ item.nombre }}</h3>
            <p class="precio">${{ item.precio.toLocaleString() }}</p>
            <small class="stock-label" :class="{ 'low-stock': item.stock > 0 && item.stock < 5 }">
              Disponibles: {{ item.stock }}
            </small>
          </div>
          
          <button 
            @click="agregarAlCarrito(item)" 
            class="btn-add" 
            :disabled="item.stock === 0"
          >
            {{ item.stock === 0 ? 'Agotado' : 'Añadir' }}
          </button>
        </div>
      </div>
    </main>

    <div class="carrito-container-fijo">
      <transition-group name="notification">
        <div v-for="nota in notificaciones" :key="nota.id" class="toast-notificacion">
          +1 {{ nota.nombre }}
        </div>
      </transition-group>

      <button class="btn-carrito-flotante" @click="toggleCarrito">
        <span class="icon">🛒</span>
        <span v-if="totalItems > 0" class="badge-notif">{{ totalItems }}</span>
      </button>
    </div>

    <transition name="slide">
      <aside v-if="mostrarCarrito" class="sidebar-cart">
        <div class="sidebar-header">
          <h2>Tu Pedido</h2>
          <button @click="toggleCarrito" class="btn-close">×</button>
        </div>

        <div class="cart-list">
          <div v-if="carrito.length === 0" class="empty-msg">Tu carrito está vacío 🍕</div>
          <div v-for="item in carrito" :key="item.id" class="cart-item">
            <img :src="item.imagen" class="mini-img">
            <div class="item-details">
              <strong>{{ item.nombre }}</strong>
              <div class="quantity-controls">
                <button @click="modificarCantidad(item, -1)">-</button>
                <span>{{ item.cantidad }}</span>
                <button @click="modificarCantidad(item, 1)" :disabled="platos.find(p => p.id === item.id).stock === 0">+</button>
              </div>
            </div>
            <div class="item-price">
              ${{ (item.precio * item.cantidad).toLocaleString() }}
            </div>
          </div>
        </div>

        <div class="sidebar-footer" v-if="carrito.length > 0">
          <div class="total-info">
            <span>TOTAL:</span>
            <strong>${{ calcularTotal().toLocaleString() }}</strong>
          </div>
          <button @click="generarPDF" class="btn-checkout">Finalizar Pedido</button>
        </div>
      </aside>
    </transition>

    <div v-if="mostrarCarrito" class="overlay" @click="toggleCarrito"></div>

    <div class="pdf-hidden">
      <div id="factura-pdf" class="pdf-design">
        <div class="pdf-header">
          <h2>RECIBO DE PEDIDO</h2>
          <p><strong>El Sazón - San Gil</strong></p>
          <small>{{ fechaActual }}</small>
        </div>
        <table class="pdf-table">
          <thead>
            <tr>
              <th>Cant.</th>
              <th>Descripción</th>
              <th>Total</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in carrito" :key="'pdf-' + item.id">
              <td>{{ item.cantidad }}</td>
              <td>{{ item.nombre }}</td>
              <td>${{ (item.precio * item.cantidad).toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
        <div class="pdf-total">
          <strong>TOTAL: ${{ calcularTotal().toLocaleString() }}</strong>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import html2pdf from 'html2pdf.js';

const mostrarCarrito = ref(false);
const carrito = ref([]);
const notificaciones = ref([]);

const platos = ref([
  { id: 1, nombre: 'Papas Nativas en Cascos', precio: 15000, stock: 12, imagen: 'https://www.elespectador.com/resizer/v2/53S4LN44IBAIDI3PGEF3RRKYX4.jpg?auth=058adc68fe263502ae7aa3f860010f73226cf756df3b9ed6477f5de52ca64c4d&width=920&height=613&smart=true&quality=60' },
  { id: 2, nombre: 'Empanaditas de Copetín', precio: 12000, stock: 20, imagen: 'https://oreganatto.com.ar/v/wp-content/uploads/2024/01/empanadascopetin-600x600.png' },
  { id: 3, nombre: 'Ceviche de Mango', precio: 18000, stock: 5, imagen: 'https://cdn.colombia.com/gastronomia/2011/07/27/ceviche-de-mango-1521.webp' },
  { id: 4, nombre: 'Nachos El Sazón', precio: 22000, stock: 8, imagen: 'https://dynamic-media-cdn.tripadvisor.com/media/photo-o/23/e6/91/71/steak-supreme-nachos.jpg?w=900&h=500&s=1' },
  { id: 5, nombre: 'Deditos de Mozzarella', precio: 10000, stock: 15, imagen: 'https://www.kelloggs.com.co/content/dam/latinamerica/kelloggs_co/img/recetas/cena-deditos-de-mozarella-588x510.jpg' },
  { id: 6, nombre: 'Crema de Tomate Rostizado', precio: 25000, stock: 10, imagen: 'https://cheforopeza.com.mx/wp-content/uploads/2017/07/crema-de-tomates-rostizados_recetas_chef-oropeza-jpg' },
  { id: 7, nombre: 'Punta de Anca a la Parrilla', precio: 14000, stock: 6, imagen: 'https://www.sanpalesteparrilla.com/wp-content/uploads/CARTA-SAN-PALESTE3370.png' },
  { id: 8, nombre: 'Pollo en Salsa de Champiñones', precio: 19500, stock: 10, imagen: 'https://campollo.com/wp-content/uploads/2024/05/GettyImages-1142667686.jpg' },
  { id: 9, nombre: 'Costillitas BBQ', precio: 21000, stock: 4, imagen: 'https://cdn7.kiwilimon.com/recetaimagen/26648/640x640/23963.jpg.webp' },
  { id: 10, nombre: 'Lomo de Cerdo al Caramelo', precio: 16000, stock: 12, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTB3S3N9nRR00I-Q2ijpqOYebhOitk4dk-sgw&s' },
  { id: 11, nombre: 'Arroz Especial de la Casa', precio: 15000, stock: 20, imagen: 'https://media-cdn.tripadvisor.com/media/photo-s/14/2d/47/be/el-tesoro-arroz-especial.jpg' },
  { id: 12, nombre: 'Sobrebarriga al Horno', precio: 12000, stock: 8, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQlf8gZ1BJxz_uZXPZEymYDAKd9OnByFAcZCQ&s' },
  { id: 13, nombre: 'Pechuga Gratinada', precio: 18000, stock: 10, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcR-zFC7I2oKIeVDmbxdC_6-PGcp5uUoDA7MYw&s' },
  { id: 14, nombre: 'Carne Asada "El Sazón"', precio: 22000, stock: 5, imagen: 'https://mojo.generalmills.com/api/public/content/oKDBX8mQfkSqraA1_GfVGw_gmi_hi_res_jpeg.jpeg?v=6cac91ad&t=16e3ce250f244648bef28c5949fb99ff' },
  { id: 15, nombre: 'Filete de Trucha al Ajillo', precio: 10000, stock: 7, imagen: 'https://www.cocinacaserayfacil.net/wp-content/uploads/2020/10/Trucha-al-ajillo.jpg' },
  { id: 16, nombre: 'Salmón en Costra de Hierbas', precio: 25000, stock: 3, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTSvCUTacPw5F6e-uPDa2J5CGXyZLpYZmgzmw&s' },
  { id: 17, nombre: 'Cazuela de Mariscos', precio: 14000, stock: 6, imagen: 'https://www.cocinadelirante.com/800x600/filters:format(webp):quality(75)/sites/default/files/images/2025/01/cazuela-de-mariscos.jpg' },
  { id: 18, nombre: 'Arroz Marinero', precio: 19500, stock: 10, imagen: 'https://www.recetasnestle.com.ec/sites/default/files/styles/recipe_detail_desktop_new/public/srh_recipes/7283831430f0ad722c8fad598b4d8851.jpg?itok=qlUvHH5l' },
  { id: 19, nombre: 'Hamburguesa Artesanal', precio: 21000, stock: 15, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTCoUhMnPmRh4ILmy00z6avGA5amn6L_eCCRg&s' },
  { id: 20, nombre: 'Pizza de Carnes', precio: 16000, stock: 10, imagen: 'https://imag.bonviveur.com/pizza-de-carne-picada.jpg' },
  { id: 21, nombre: 'Perro Caliente Especial', precio: 15000, stock: 12, imagen: 'https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcR0FSPzoZa4gZjp8XhCpAeYV1PokPlAvlhWRw&s' },
  { id: 22, nombre: 'Tacos de Birria', precio: 12000, stock: 20, imagen: 'https://www.elespectador.com/resizer/v2/SDYQQJZH2REZXJHMP72B4F6KVU.jpg?auth=f96280713131aa938200eb4dda94f6d6e206c242992d7ef15f25f03a3f73f9c0&width=920&height=613&smart=true&quality=60' },
  { id: 23, nombre: 'Club Sandwich', precio: 18000, stock: 8, imagen: 'https://www.tasteofhome.com/wp-content/uploads/2025/04/Turkey-Club-Sandwich_EXPS_FT25_278726_EC_0408_5.jpg' },
  { id: 24, nombre: 'Wrap de Pollo Crispy', precio: 22000, stock: 10, imagen: 'https://chefeel.com/chefgeneralfiles/2023/11/shawarma-5893935_1280-880x772.jpg' },
  { id: 25, nombre: 'Brownie con Helado', precio: 10000, stock: 15, imagen: 'https://www.elespectador.com/resizer/v2/BS7XK3UP25GMJO3G5IXXDXCRCE.png?auth=3c129e8844974311269f7548856b579d7adf8ec3046e1d427974c8334f911f67&width=920&height=613&smart=true&quality=60' },
  { id: 26, nombre: 'Limonada de Coco', precio: 25000, stock: 30, imagen: 'https://osojimix.com/wp-content/uploads/2023/02/Limonada-de-coco-frappe.jpeg.jpg' },
  { id: 27, nombre: 'Jugos Naturales de Temporada', precio: 14000, stock: 20, imagen: 'https://www.prensalibre.com/wp-content/uploads/2025/05/Jugos-naturales-01.jpg?quality=52' },
  { id: 28, nombre: 'Gaseosa', precio: 19500, stock: 50, imagen: 'https://confirmado.com.ve/conf/conf-upload/uploads/2024/09/cu1-12.jpg' },
  { id: 29, nombre: 'Porcion de Papas', precio: 21000, stock: 15, imagen: 'https://ninina.com/cdn/shop/files/IMG_9549-Foto_miserereok_0f4de11c-6b91-4fef-b0f9-bf7ca4130116.jpg?v=1686253182&width=1445' },
  { id: 30, nombre: 'Porcion de Arroz', precio: 16000, stock: 25, imagen: 'https://cloudfront-us-east-1.images.arcpublishing.com/infobae/DFN6EUB2SFC7BPEAQWDCKUJUIA.jpg' }
]);

const toggleCarrito = () => mostrarCarrito.value = !mostrarCarrito.value;
const totalItems = computed(() => carrito.value.reduce((acc, item) => acc + item.cantidad, 0));

const agregarAlCarrito = (item) => {
  if (item.stock > 0) {
    const existe = carrito.value.find(prod => prod.id === item.id);
    if (existe) {
      existe.cantidad++;
    } else {
      carrito.value.push({ ...item, cantidad: 1 });
    }
    
    item.stock--;

    const idNota = Date.now();
    notificaciones.value.push({ id: idNota, nombre: item.nombre });
    setTimeout(() => {
      notificaciones.value = notificaciones.value.filter(n => n.id !== idNota);
    }, 2000);
  }
};

const modificarCantidad = (item, valor) => {
  const platoOriginal = platos.value.find(p => p.id === item.id);
  
  if (valor === 1) { 
    if (platoOriginal.stock > 0) {
      item.cantidad++;
      platoOriginal.stock--;
    }
  } else { 
    item.cantidad--;
    platoOriginal.stock++;
    if (item.cantidad <= 0) {
      carrito.value = carrito.value.filter(prod => prod.id !== item.id);
    }
  }
};

const calcularTotal = () => carrito.value.reduce((acc, item) => acc + (item.precio * item.cantidad), 0);
const fechaActual = computed(() => new Date().toLocaleString());

const generarPDF = () => {
  const elemento = document.getElementById('factura-pdf');
  const opciones = {
    margin: 0.5,
    html2canvas: { scale: 3, useCORS: true },
    jsPDF: { unit: 'cm', format: 'a6', orientation: 'portrait' }
  };
  html2pdf().set(opciones).from(elemento).output('bloburl').then(url => window.open(url, '_blank'));
};
</script>

<style scoped>
.menu-app{font-family:'Inter',sans-serif;background:#f9fafb;min-height:100vh;padding-bottom:120px}

.navbar{background:#fff;padding:15px 20px;position:sticky;top:0;z-index:90;box-shadow:0 2px 10px rgba(0,0,0,.05)}
.navbar-content{max-width:1200px;margin:auto;display:flex;justify-content:space-between;align-items:center}
.brand h1{margin:0;font-size:1.4rem;color:#111827}
.location{font-size:.8rem;color:#6b7280}

.menu-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(360px,1fr));gap:20px;max-width:1200px;margin:auto;padding:30px 20px}

.card{display:flex;height:145px;background:#fff;border-radius:20px;overflow:hidden;border:1px solid #f3f4f6;transition:.3s}
.card.sin-stock{opacity:.6;filter:grayscale(.5)}
.card-image{width:35%;position:relative}
.card-image img{width:100%;height:100%;object-fit:cover}
.overlay-agotado{position:absolute;inset:0;display:flex;justify-content:center;align-items:center;background:rgba(0,0,0,.5);color:#fff;font-weight:700;font-size:.9rem}
.card-body{width:65%;padding:15px;display:flex;flex-direction:column;justify-content:space-between}
.card-body h3{margin:0;font-size:1rem;color:#111827}
.precio{margin:4px 0;font-size:1.1rem;font-weight:800;color:#10b981}
.stock-label{font-size:.75rem;color:#6b7280}
.stock-label.low-stock{color:#f59e0b;font-weight:700}

.btn-add,.btn-checkout{border:none;color:#fff;font-weight:600;cursor:pointer}
.btn-add{padding:8px 16px;border-radius:50px;background:#3b82f6;align-self:flex-start}
.btn-add:disabled{background:#9ca3af;cursor:not-allowed}
.btn-checkout{width:100%;padding:15px;border-radius:12px;background:#111827}

.carrito-container-fijo{position:fixed;bottom:30px;right:30px;z-index:1000;display:flex;flex-direction:column;align-items:flex-end;gap:10px}
.btn-carrito-flotante{width:65px;height:65px;border:none;border-radius:50%;background:#111827;color:#fff;font-size:1.8rem;cursor:pointer;display:flex;justify-content:center;align-items:center;position:relative;box-shadow:0 8px 25px rgba(0,0,0,.3)}
.badge-notif{position:absolute;top:0;right:0;padding:4px 8px;font-size:.8rem;background:#ef4444;color:#fff;border-radius:50%;border:2px solid #111827}
.toast-notificacion{background:#10b981;color:#fff;padding:8px 15px;border-radius:12px;font-size:.9rem;font-weight:600}

.sidebar-cart{position:fixed;top:0;right:0;width:380px;height:100vh;background:#fff;z-index:2000;display:flex;flex-direction:column;box-shadow:-10px 0 30px rgba(0,0,0,.1)}
.sidebar-header{padding:25px;border-bottom:1px solid #f3f4f6;display:flex;justify-content:space-between;align-items:center}
.cart-list{flex:1;overflow-y:auto;padding:20px}
.cart-item{display:flex;align-items:center;gap:12px;margin-bottom:15px;padding-bottom:10px;border-bottom:1px solid #f9f9f9}
.mini-img{width:50px;height:50px;border-radius:10px;object-fit:cover}

.quantity-controls{display:flex;align-items:center;gap:10px;margin-top:5px}
.quantity-controls button{width:28px;height:28px;border:1px solid #e5e7eb;background:#fff;border-radius:6px;cursor:pointer}
.quantity-controls button:disabled{opacity:.3;cursor:not-allowed}

.sidebar-footer{padding:25px;background:#f9fafb}
.total-info{display:flex;justify-content:space-between;font-size:1.2rem;margin-bottom:15px}

.overlay{position:fixed;inset:0;z-index:1500;background:rgba(0,0,0,.4);backdrop-filter:blur(2px)}

.notification-enter-active,.notification-leave-active{transition:.4s}
.notification-enter-from{opacity:0;transform:translateY(20px)}
.notification-leave-to{opacity:0;transform:translateY(-20px)}
.slide-enter-active,.slide-leave-active{transition:transform .4s cubic-bezier(.4,0,.2,1)}
.slide-enter-from,.slide-leave-to{transform:translateX(100%)}

.pdf-hidden{position:absolute;left:-9999px}
</style>