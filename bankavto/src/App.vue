<script setup>
import { ref, computed, watch } from 'vue'

const STORAGE_KEY = 'bankavto-cars'
const USERS_KEY = 'bankavto-users'
const SESSION_KEY = 'bankavto-session'
const COLORS = ['#1b2a41', '#6b4f3a', '#2f4a43', '#5b3a4a', '#3f4f6b', '#7a6a4f']
const color = (i) => COLORS[Math.abs(i) % COLORS.length]

const defaultCars = [
  { id: 1, brand: 'Chevrolet', model: 'Gentra', year: 2019, mileage: 78000, price: 105, bank: 'Namuna Bank', city: 'Toshkent', color: 'Oq', description: "Bank garovga olgan avtomobil. Holati yaxshi, hujjatlari to'liq.", address: "Toshkent sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
  { id: 2, brand: 'Chevrolet', model: 'Cobalt', year: 2021, mileage: 42000, price: 125, bank: 'Demo Bank', city: 'Samarqand', color: 'Kulrang', description: "Bitta egasi bo'lgan, avtomat uzatmalar qutisi.", address: "Samarqand sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
  { id: 3, brand: 'Kia', model: 'K5', year: 2020, mileage: 61000, price: 290, bank: 'Namuna Bank', city: 'Toshkent', color: 'Qora', description: "Salon toza, texnik ko'rikdan o'tgan.", address: "Toshkent sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
  { id: 4, brand: 'Chevrolet', model: 'Malibu', year: 2018, mileage: 95000, price: 175, bank: 'Demo Bank', city: "Farg'ona", color: 'Oq', description: 'Garovdan chiqarilgan, sotuvga tayyor.', address: "Farg'ona sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
  { id: 5, brand: 'Hyundai', model: 'Sonata', year: 2019, mileage: 70000, price: 230, bank: 'Namuna Bank', city: 'Andijon', color: 'Kumush', description: 'Yaxshi holatda, ehtiyot qismlari almashtirilgan.', address: "Andijon sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
  { id: 6, brand: 'Chevrolet', model: 'Tracker', year: 2022, mileage: 25000, price: 210, bank: 'Demo Bank', city: 'Toshkent', color: 'Qizil', description: 'Deyarli yangi, kam yurgan.', address: "Toshkent sh., [ko'cha va uy raqami]", phone: '+998 [xx xxx xx xx]', image: '' },
]

function readJSON(key, fallback) {
  try {
    const v = localStorage.getItem(key)
    return v ? JSON.parse(v) : fallback
  } catch {
    return fallback
  }
}

/* ---------- Mashinalar ---------- */
const cars = ref(readJSON(STORAGE_KEY, null) || structuredClone(defaultCars))

watch(
  cars,
  (v) => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(v))
    } catch {
      alert("Saqlab bo'lmadi. Rasm hajmi katta bo'lishi mumkin.")
    }
  },
  { deep: true }
)

/* ---------- Foydalanuvchilar, rollar va sessiya ---------- */
// Rollar: 'admin' (e'lon qo'sha oladi) va 'superadmin' (hammasini boshqaradi, faqat bitta bo'ladi)
const users = ref(readJSON(USERS_KEY, []))
const currentUserId = ref(readJSON(SESSION_KEY, null))
const currentUser = computed(() => users.value.find((u) => u.id === currentUserId.value) || null)
const isAdmin = computed(() => ['admin', 'superadmin'].includes(currentUser.value?.role))
const isSuper = computed(() => currentUser.value?.role === 'superadmin')
const hasSuper = computed(() => users.value.some((u) => u.role === 'superadmin'))
const roleLabel = (r) => (r === 'superadmin' ? 'Superadmin' : r === 'admin' ? 'Admin' : 'Foydalanuvchi')

watch(users, (v) => { try { localStorage.setItem(USERS_KEY, JSON.stringify(v)) } catch {} }, { deep: true })
watch(currentUserId, (v) => {
  try {
    if (v) localStorage.setItem(SESSION_KEY, JSON.stringify(v))
    else localStorage.removeItem(SESSION_KEY)
  } catch {}
})

const randomSalt = () => [...crypto.getRandomValues(new Uint8Array(16))].map((b) => b.toString(16).padStart(2, '0')).join('')

async function hashPassword(password, salt) {
  const data = new TextEncoder().encode(salt + password)
  const buf = await crypto.subtle.digest('SHA-256', data)
  return [...new Uint8Array(buf)].map((b) => b.toString(16).padStart(2, '0')).join('')
}

function normPhone(s) {
  let d = s.replace(/\D/g, '')
  if (d.length === 9) d = '998' + d
  return d
}

/* ---------- Sahifalar ---------- */
const view = ref('home') // 'home' | 'car' | 'admin' | 'login' | 'signup'
const selected = ref(null)

function goHome() {
  view.value = 'home'
  selected.value = null
}
function goAdmin() {
  view.value = 'admin'
  window.scrollTo(0, 0)
}
function goLogin() {
  authError.value = ''
  view.value = 'login'
  window.scrollTo(0, 0)
}
function goSignup() {
  authError.value = ''
  view.value = 'signup'
  window.scrollTo(0, 0)
}
function openCar(car) {
  selected.value = car
  view.value = 'car'
  window.scrollTo(0, 0)
}

/* ---------- Ro'yxatdan o'tish va kirish ---------- */
const su = ref({ name: '', phone: '', password: '', password2: '', role: 'admin' })
const li = ref({ phone: '', password: '' })
const authError = ref('')
const busy = ref(false)

function afterAuth() {
  view.value = selected.value ? 'car' : 'home'
  window.scrollTo(0, 0)
}

async function signUp() {
  authError.value = ''
  const phone = normPhone(su.value.phone)
  if (phone.length < 12) return (authError.value = "Telefon raqamni to'liq kiriting.")
  if (su.value.password.length < 6) return (authError.value = "Parol kamida 6 ta belgidan iborat bo'lsin.")
  if (su.value.password !== su.value.password2) return (authError.value = 'Parollar bir xil emas.')
  if (users.value.some((u) => u.phone === phone)) {
    return (authError.value = "Bu raqam bilan ro'yxatdan o'tilgan. Kirish sahifasiga o'ting.")
  }
  if (su.value.role === 'superadmin' && hasSuper.value) {
    return (authError.value = 'Superadmin allaqachon mavjud. Admin sifatida ro\'yxatdan o\'ting.')
  }
  busy.value = true
  try {
    const salt = randomSalt()
    const user = {
      id: Date.now(),
      name: su.value.name.trim(),
      phone,
      salt,
      hash: await hashPassword(su.value.password, salt),
      role: su.value.role === 'superadmin' ? 'superadmin' : 'admin',
    }
    users.value.push(user)
    currentUserId.value = user.id
    su.value = { name: '', phone: '', password: '', password2: '', role: 'admin' }
    afterAuth()
  } catch {
    authError.value = 'Xatolik yuz berdi. Sayt localhost yoki HTTPS orqali ochilganini tekshiring.'
  } finally {
    busy.value = false
  }
}

async function logIn() {
  authError.value = ''
  const phone = normPhone(li.value.phone)
  const user = users.value.find((u) => u.phone === phone)
  busy.value = true
  try {
    if (!user || (await hashPassword(li.value.password, user.salt)) !== user.hash) {
      return (authError.value = "Telefon yoki parol noto'g'ri.")
    }
    currentUserId.value = user.id
    li.value = { phone: '', password: '' }
    afterAuth()
  } catch {
    authError.value = 'Xatolik yuz berdi. Sayt localhost yoki HTTPS orqali ochilganini tekshiring.'
  } finally {
    busy.value = false
  }
}

function logOut() {
  currentUserId.value = null
  cancelEdit()
  goHome()
}

/* ---------- Bosh sahifa: qidiruv va filtrlar ---------- */
const brands = computed(() => [...new Set(cars.value.map((c) => c.brand))])
const banks = computed(() => [...new Set(cars.value.map((c) => c.bank))])
const suggestions = computed(() => [...new Set([...cars.value.map((c) => c.brand), ...cars.value.map((c) => c.model)])])
const query = ref('')
const filters = ref({ brand: '', year: '', maxPrice: '', bank: '' })
const visible = ref(8)

// Qidiruvni soddalashtirish: kichik harf, apostrof va belgilarsiz
const norm = (s) =>
  String(s ?? '')
    .toLowerCase()
    .replace(/['’‘`ʻʼ]/g, '')
    .replace(/[^\p{L}\p{N}]+/gu, ' ')
    .trim()

const filtered = computed(() => {
  const { brand, year, maxPrice, bank } = filters.value
  const tokens = norm(query.value).split(' ').filter(Boolean)
  return cars.value.filter((c) => {
    const hay = norm(`${c.brand} ${c.model}`) // marka (emblema) va model nomi
    return (
      tokens.every((t) => hay.includes(t)) &&
      (!brand || c.brand === brand) &&
      (!year || c.year >= Number(year)) &&
      (!maxPrice || c.price <= Number(maxPrice)) &&
      (!bank || c.bank === bank)
    )
  })
})
const shown = computed(() => filtered.value.slice(0, visible.value))

watch(query, (v) => {
  visible.value = 8
  if (v) filters.value.brand = '' // qidirganda marka tugmasi filtri xalaqit bermasin
})

function pickBrand(b) {
  filters.value.brand = filters.value.brand === b ? '' : b
  query.value = ''
  visible.value = 8
}
function resetFilters() {
  query.value = ''
  filters.value = { brand: '', year: '', maxPrice: '', bank: '' }
}
function toList() {
  document.getElementById('royxat')?.scrollIntoView({ behavior: 'smooth' })
}

/* ---------- Admin panel: qo'shish, tahrirlash, o'chirish ---------- */
const emptyForm = () => ({
  brand: '', model: '', year: 2020, mileage: 0, price: 0,
  bank: '', city: '', color: '', address: '', phone: '', description: '', image: '',
})
const form = ref(emptyForm())
const editingId = ref(null)
const fileKey = ref(0)
const notice = ref('')

function flash(msg) {
  notice.value = msg
  setTimeout(() => (notice.value = ''), 2500)
}

function onImage(e) {
  const file = e.target.files[0]
  if (!file) return
  if (file.size > 1.5 * 1024 * 1024) {
    alert("Rasm 1.5 MB dan kichik bo'lsin.")
    e.target.value = ''
    return
  }
  const reader = new FileReader()
  reader.onload = () => (form.value.image = reader.result)
  reader.readAsDataURL(file)
}

// Faqat Superadmin
function startEdit(car) {
  if (!isSuper.value) return
  form.value = { ...emptyForm(), ...car }
  editingId.value = car.id
  fileKey.value++
  view.value = 'admin'
  window.scrollTo(0, 0)
}

function cancelEdit() {
  editingId.value = null
  form.value = emptyForm()
  fileKey.value++
}

function saveCar() {
  const data = {
    ...form.value,
    year: Number(form.value.year),
    mileage: Number(form.value.mileage),
    price: Number(form.value.price),
  }
  if (editingId.value) {
    if (!isSuper.value) return // tahrirlash faqat Superadmin uchun
    const i = cars.value.findIndex((c) => c.id === editingId.value)
    if (i !== -1) {
      cars.value[i] = { ...data, id: editingId.value }
      if (selected.value?.id === editingId.value) selected.value = cars.value[i]
    }
    cancelEdit()
    flash("E'lon yangilandi!")
  } else {
    if (!isAdmin.value) return // qo'shish Admin va Superadmin uchun
    cars.value.unshift({ ...data, id: Date.now() })
    cancelEdit()
    flash("E'lon qo'shildi!")
  }
}

// Faqat Superadmin
function removeCar(car) {
  if (!isSuper.value) return
  if (confirm(`${car.brand} ${car.model} o'chirilsinmi?`)) {
    cars.value = cars.value.filter((c) => c.id !== car.id)
    if (editingId.value === car.id) cancelEdit()
  }
}

function removeUser(u) {
  if (!isSuper.value) return
  if (u.role === 'superadmin' || u.id === currentUser.value.id) return
  if (confirm(`${u.name} (+${u.phone}) hisobi o'chirilsinmi?`)) {
    users.value = users.value.filter((x) => x.id !== u.id)
  }
}

function resetDemo() {
  if (!isSuper.value) return
  if (confirm("Hamma e'lonlar o'chirilib, namuna ma'lumotlar qaytariladi. Davom etasizmi?")) {
    cars.value = structuredClone(defaultCars)
    cancelEdit()
  }
}

const km = (n) => Number(n).toLocaleString('ru-RU')
const initial = (s) => (s || '?').trim().charAt(0).toUpperCase()
</script>

<template>
  <header class="header">
    <a href="#" class="logo" @click.prevent="goHome">
      <svg viewBox="0 0 64 64" width="36" height="36" aria-hidden="true">
  <path d="M32 3 L57 11.5 V32 C57 46.5 46.5 56.5 32 61 C17.5 56.5 7 46.5 7 32 V11.5 Z" fill="#1b2a41" />
  <path d="M32 7 L53 14 V32 C53 44.5 44.2 53 32 57 C19.8 53 11 44.5 11 32 V14 Z" fill="none" stroke="#b08d57" stroke-width="1.2" />
  <path d="M15 38 V33 Q15 31.5 16.5 31 L22 29.5 L26 24.5 Q27 23.5 28.5 23.5 H37 Q39 23.5 40.2 24.7 L44.5 29.5 L47.5 31 Q49 31.5 49 33 V38 Q49 39.5 47.5 39.5 H16.5 Q15 39.5 15 38 Z" fill="#b08d57" />
  <path d="M26.2 29 L28.3 26 H36.2 L39.5 29 Z" fill="#1b2a41" />
  <circle cx="23" cy="39.5" r="4.4" fill="#b08d57" stroke="#1b2a41" stroke-width="2" />
  <circle cx="23" cy="39.5" r="1.6" fill="#1b2a41" />
  <circle cx="41" cy="39.5" r="4.4" fill="#b08d57" stroke="#1b2a41" stroke-width="2" />
  <circle cx="41" cy="39.5" r="1.6" fill="#1b2a41" />
  <path d="M20 46 H44" stroke="#b08d57" stroke-width="1.4" stroke-linecap="round" />
</svg>
      <span>Bank<em>Avto</em>.uz</span>
    </a>
    <nav class="nav">
      <a href="#" @click.prevent="goHome">Avtomobillar</a>
      <a v-if="isAdmin" href="#" @click.prevent="goAdmin">{{ isSuper ? 'Superadmin' : 'Admin' }}</a>
    </nav>
    <div class="auth">
      <template v-if="currentUser">
        <span class="hello">Salom, {{ currentUser.name }}</span>
        <span class="role" :class="currentUser.role">{{ roleLabel(currentUser.role) }}</span>
        <button class="btn ghost small" @click="logOut">Chiqish</button>
      </template>
      <template v-else>
        <button class="btn ghost small" @click="goLogin">Kirish</button>
        <button class="btn small" @click="goSignup">Ro'yxatdan o'tish</button>
      </template>
    </div>
  </header>

  <!-- RO'YXATDAN O'TISH -->
  <section v-if="view === 'signup'" class="wrap authpage">
    <form class="box authbox" @submit.prevent="signUp">
      <h2>Ro'yxatdan o'tish</h2>

      <div>
        <div class="seg" role="group" aria-label="Hisob turi">
          <button type="button" class="segbtn" :class="{ on: su.role === 'admin' }" :aria-pressed="su.role === 'admin'" @click="su.role = 'admin'">Admin</button>
          <button type="button" class="segbtn" :class="{ on: su.role === 'superadmin' }" :aria-pressed="su.role === 'superadmin'" :disabled="hasSuper" @click="su.role = 'superadmin'">Superadmin</button>
        </div>
        <p class="m hint">
          <template v-if="hasSuper">Superadmin allaqachon mavjud, faqat Admin bo'lib ro'yxatdan o'tish mumkin.</template>
          <template v-else-if="su.role === 'superadmin'">Superadmin faqat bitta bo'ladi: e'lonlarni tahrirlaydi, o'chiradi va foydalanuvchilarni boshqaradi.</template>
          <template v-else>Admin yangi avtomobil e'lonlarini qo'sha oladi.</template>
        </p>
      </div>

      <label class="f">Ismingiz<input v-model="su.name" autocomplete="name" required /></label>
      <label class="f">Telefon raqam<input v-model="su.phone" type="tel" autocomplete="tel" placeholder="+998 90 123 45 67" required /></label>
      <label class="f">Parol<input v-model="su.password" type="password" autocomplete="new-password" minlength="6" required /></label>
      <label class="f">Parolni takrorlang<input v-model="su.password2" type="password" autocomplete="new-password" required /></label>
      <p v-if="authError" class="err">{{ authError }}</p>
      <button class="btn" type="submit" :disabled="busy">Ro'yxatdan o'tish</button>
      <p class="m center">Hisobingiz bormi? <button type="button" class="link" @click="goLogin">Kirish</button></p>
    </form>
  </section>

  <!-- KIRISH -->
  <section v-else-if="view === 'login'" class="wrap authpage">
    <form class="box authbox" @submit.prevent="logIn">
      <h2>Kirish</h2>
      <label class="f">Telefon raqam<input v-model="li.phone" type="tel" autocomplete="tel" placeholder="+998 90 123 45 67" required /></label>
      <label class="f">Parol<input v-model="li.password" type="password" autocomplete="current-password" required /></label>
      <p v-if="authError" class="err">{{ authError }}</p>
      <button class="btn" type="submit" :disabled="busy">Kirish</button>
      <p class="m center">Hisobingiz yo'qmi? <button type="button" class="link" @click="goSignup">Ro'yxatdan o'tish</button></p>
    </form>
  </section>

  <!-- ADMIN / SUPERADMIN PANEL -->
  <section v-else-if="view === 'admin'" class="wrap admin">
    <div v-if="!isAdmin" class="box authbox">
      <h2>Faqat admin uchun</h2>
      <p class="m">Bu sahifani ko'rish uchun Admin yoki Superadmin hisobi bilan kiring.</p>
      <button class="btn" @click="goLogin">Kirish</button>
    </div>

    <template v-else>
      <div class="admin-top">
        <h2>{{ isSuper ? 'Superadmin panel' : 'Admin panel' }}</h2>
        <button v-if="isSuper" class="btn ghost" @click="resetDemo">Namuna ma'lumotlarni qaytarish</button>
      </div>
      <p v-if="!isSuper" class="m">Siz Admin sifatida yangi e'lon qo'sha olasiz. Tahrirlash, o'chirish va foydalanuvchilarni boshqarish faqat Superadmin uchun.</p>

      <form class="box" @submit.prevent="saveCar">
        <h3>{{ editingId ? 'Avtomobilni tahrirlash' : "Yangi avtomobil qo'shish" }}</h3>
        <div class="form-grid">
          <label class="f">Marka<input v-model="form.brand" list="brandList" required placeholder="Chevrolet" /></label>
          <label class="f">Model<input v-model="form.model" required placeholder="Gentra" /></label>
          <label class="f">Yil<input v-model="form.year" type="number" min="1990" max="2035" required /></label>
          <label class="f">Probeg (km)<input v-model="form.mileage" type="number" min="0" required /></label>
          <label class="f">Narx (mln so'm)<input v-model="form.price" type="number" min="0" step="any" required /></label>
          <label class="f">Bank<input v-model="form.bank" list="bankList" required placeholder="Bank nomi" /></label>
          <label class="f">Shahar<input v-model="form.city" required /></label>
          <label class="f">Rang<input v-model="form.color" required /></label>
          <label class="f">Bank manzili<input v-model="form.address" required placeholder="Shahar, ko'cha, uy" /></label>
          <label class="f">Bank telefoni<input v-model="form.phone" type="tel" required placeholder="+998 ..." /></label>
        </div>
        <label class="f">Tavsif<textarea v-model="form.description" rows="3"></textarea></label>
        <label class="f">Rasm (ixtiyoriy)<input :key="fileKey" type="file" accept="image/*" @change="onImage" /></label>
        <div v-if="form.image" class="row">
          <img :src="form.image" class="preview" alt="" />
          <button type="button" class="btn ghost small" @click="form.image = ''; fileKey++">Rasmni olib tashlash</button>
        </div>
        <div class="row">
          <button class="btn" type="submit">{{ editingId ? 'Saqlash' : "Qo'shish" }}</button>
          <button v-if="editingId" type="button" class="btn ghost" @click="cancelEdit">Bekor qilish</button>
          <span v-if="notice" class="ok">{{ notice }}</span>
        </div>
        <datalist id="brandList"><option v-for="b in brands" :key="b" :value="b" /></datalist>
        <datalist id="bankList"><option v-for="b in banks" :key="b" :value="b" /></datalist>
      </form>

      <h3 class="subtitle">Hamma e'lonlar ({{ cars.length }})</h3>
      <div class="rows">
        <div v-for="car in cars" :key="car.id" class="rowitem" :class="{ editing: editingId === car.id }">
          <img v-if="car.image" :src="car.image" class="thumb" alt="" />
          <div v-else class="thumb ph">Rasm</div>
          <div class="grow">
            <strong>{{ car.brand }} {{ car.model }}</strong>
            <span class="m">{{ car.year }} · {{ km(car.mileage) }} km · {{ car.bank }} · {{ car.price }} mln so'm</span>
          </div>
          <div v-if="isSuper" class="row">
            <button class="btn ghost small" @click="startEdit(car)">Tahrirlash</button>
            <button class="btn danger" @click="removeCar(car)">O'chirish</button>
          </div>
        </div>
      </div>

      <template v-if="isSuper">
        <h3 class="subtitle">Foydalanuvchilar ({{ users.length }})</h3>
        <div class="rows">
          <div v-for="u in users" :key="u.id" class="rowitem">
            <div class="grow">
              <strong>{{ u.name }} <span class="role" :class="u.role">{{ roleLabel(u.role) }}</span></strong>
              <span class="m">+{{ u.phone }}</span>
            </div>
            <button v-if="u.role !== 'superadmin' && u.id !== currentUser.id" class="btn danger" @click="removeUser(u)">O'chirish</button>
          </div>
        </div>
      </template>
    </template>
  </section>

  <!-- MASHINA SAHIFASI -->
  <section v-else-if="view === 'car' && selected" class="wrap detail">
    <div class="row between">
      <button class="link" @click="goHome">&larr; Orqaga</button>
      <button v-if="isSuper" class="btn ghost small" @click="startEdit(selected)">Tahrirlash</button>
    </div>
    <div class="detail-grid">
      <img v-if="selected.image" :src="selected.image" class="img big" alt="" />
      <div v-else class="ph big" :style="{ background: color(selected.id) }">{{ initial(selected.brand) }}</div>
      <div class="in">
        <h1>{{ selected.brand }} {{ selected.model }}</h1>
        <span class="p">{{ selected.price }} mln so'm</span>
        <ul class="specs">
          <li>Yil: {{ selected.year }}</li>
          <li>Probeg: {{ km(selected.mileage) }} km</li>
          <li>Rang: {{ selected.color }}</li>
          <li>Shahar: {{ selected.city }}</li>
          <li>Sotuvchi bank: {{ selected.bank }}</li>
        </ul>
        <p>{{ selected.description }}</p>

        <div class="box visit">
          <h3>Qanday sotib olinadi?</h3>
          <p class="m">Avtomobil saytda sotilmaydi. Sotuv bank ofisida amalga oshiriladi, shuning uchun to'g'ridan-to'g'ri bankka murojaat qiling.</p>
          <dl class="contact">
            <dt>Bank</dt><dd>{{ selected.bank }}</dd>
            <dt>Manzil</dt><dd>{{ selected.address || "Manzil ko'rsatilmagan" }}</dd>
            <dt>Telefon</dt><dd>{{ selected.phone || "Ko'rsatilmagan" }}</dd>
          </dl>
        </div>
      </div>
    </div>
  </section>

  <!-- BOSH SAHIFA -->
  <template v-else>
    <section class="hero">
      <div class="wrap hero-in">
        <span class="eyebrow">Bank garovidagi avtomobillar</span>
        <h1>Ishonchli banklardan tanlangan avtomobillar</h1>
        <p>Banklar sotayotgan mashinalarni bir joyda ko'ring, solishtiring va bank manziliga o'zingiz tashrif buyuring.</p>
        <form class="search" @submit.prevent="toList">
          <input v-model="query" list="suggest" placeholder="Mashina nomi yoki markasi: Gentra, Kia..." aria-label="Qidiruv" />
          <button v-if="query" type="button" class="clear" aria-label="Tozalash" @click="query = ''">&times;</button>
          <button class="btn" type="submit">Qidirish</button>
        </form>
        <datalist id="suggest"><option v-for="s in suggestions" :key="s" :value="s" /></datalist>
      </div>

      <div class="wrap tiles">
        <button
          v-for="(b, i) in brands"
          :key="b"
          type="button"
          class="tile"
          :class="{ on: filters.brand === b }"
          @click="pickBrand(b)"
        >
          <span class="ico" :style="{ background: color(i) }">{{ initial(b) }}</span>
          {{ b }}
        </button>
      </div>
    </section>

    <section id="royxat" class="wrap list">
      <div class="list-top">
        <div>
          <h2>E'lonlar ({{ filtered.length }})</h2>
          <p v-if="query.trim()" class="m">&laquo;{{ query.trim() }}&raquo; bo'yicha {{ filtered.length }} ta natija.</p>
          <p v-else class="m">Banklar tomonidan sotuvga qo'yilgan eng so'nggi avtomobillar.</p>
        </div>
        <div class="mini">
          <select v-model="filters.year" aria-label="Yil">
            <option value="">Yil: barchasi</option>
            <option v-for="y in [2015, 2018, 2020, 2022]" :key="y" :value="y">{{ y }} dan</option>
          </select>
          <select v-model="filters.maxPrice" aria-label="Narx">
            <option value="">Narx: barchasi</option>
            <option v-for="p in [120, 200, 300, 500]" :key="p" :value="p">{{ p }} mln gacha</option>
          </select>
          <select v-model="filters.bank" aria-label="Bank">
            <option value="">Bank: barchasi</option>
            <option v-for="b in banks" :key="b" :value="b">{{ b }}</option>
          </select>
        </div>
      </div>

      <div v-if="filtered.length" class="grid">
        <a v-for="car in shown" :key="car.id" href="#" class="card" @click.prevent="openCar(car)">
          <div class="thumbwrap">
            <span class="badge">BANK TAKLIFI</span>
            <img v-if="car.image" :src="car.image" class="img" alt="" />
            <div v-else class="ph" :style="{ background: color(car.id) }">{{ initial(car.brand) }}</div>
          </div>
          <div class="in">
            <h3>{{ car.brand }} {{ car.model }}</h3>
            <span class="m">{{ car.year }} · {{ km(car.mileage) }} km · {{ car.city }}</span>
            <span class="m bank">{{ car.bank }}</span>
            <span class="p">{{ car.price }} mln so'm</span>
          </div>
        </a>
      </div>
      <p v-else class="m">
        Hech narsa topilmadi. <button class="link" @click="resetFilters">Filtrlarni tozalash</button>
      </p>

      <div v-if="filtered.length > visible" class="more">
        <button class="btn ghost" @click="visible += 8">Ko'proq ko'rsatish</button>
      </div>
    </section>
  </template>

  <footer class="footer">BankAvto.uz · Avtomobillar saytda sotilmaydi, sotuv bank ofisida amalga oshiriladi.</footer>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Manrope:wght@400;600;800&family=Playfair+Display:wght@500;700&display=swap');

/* Vue standart stilini bekor qilish */
body { display: block !important; min-height: 0 !important; }
#app { display: block !important; max-width: none !important; margin: 0 !important; padding: 0 !important; grid-template-columns: none !important; }

:root {
  --navy: #1b2a41;
  --gold: #b08d57;
  --gold-dark: #8a6a36;
  --ink: #1f2430;
  --muted: #6b6559;
  --line: #e7e1d6;
  --bg: #faf7f2;
  --danger: #9b2c2c;
}
* { box-sizing: border-box; }
body { margin: 0; font-family: Manrope, sans-serif; background: var(--bg); color: var(--ink); }
a { color: var(--ink); text-decoration: none; }
a:hover { color: var(--gold-dark); }
h1, h2, h3 { margin: 0; font-family: 'Playfair Display', Georgia, serif; font-weight: 700; }
.wrap { max-width: 1180px; margin: 0 auto; padding: 0 24px; }

.header { display: flex; flex-wrap: wrap; gap: 12px; align-items: center; justify-content: space-between; padding: 16px 48px; background: #fff; border-bottom: 1px solid var(--line); }
.logo { display: flex; align-items: center; gap: 10px; font-family: 'Playfair Display', Georgia, serif; font-weight: 700; font-size: 24px; color: var(--navy); }
.logo em { font-style: normal; color: var(--gold-dark); }
.nav { display: flex; gap: 28px; font-weight: 600; font-size: 15px; }
.auth { display: flex; flex-wrap: wrap; gap: 10px; align-items: center; }
.hello { font-weight: 600; font-size: 14px; }
.role { display: inline-block; font-size: 11px; font-weight: 800; letter-spacing: .06em; text-transform: uppercase; padding: 3px 8px; border-radius: 4px; background: #efe8da; color: var(--gold-dark); }
.role.superadmin { background: var(--navy); color: #f1e3c4; }

.btn { display: inline-flex; align-items: center; justify-content: center; height: 46px; padding: 0 24px; border: 0; border-radius: 8px; background: var(--navy); color: #fff; font: 800 15px Manrope, sans-serif; cursor: pointer; }
.btn:hover { color: #fff; background: #2b3f5f; }
.btn:disabled { opacity: .6; cursor: wait; }
.btn.small { height: 38px; padding: 0 16px; font-size: 14px; }
.btn.ghost { background: #fff; color: var(--navy); border: 1px solid #d6cdbb; }
.btn.ghost:hover { color: var(--navy); background: #f3ede2; }
.btn.danger { background: var(--danger); height: 38px; padding: 0 16px; font-size: 14px; }
.link { background: none; border: 0; padding: 0; color: var(--gold-dark); font: inherit; font-weight: 700; cursor: pointer; }

.hero { background: linear-gradient(180deg, #f1e9da 0%, var(--bg) 100%); padding: 72px 0 24px; }
.hero-in { text-align: center; display: flex; flex-direction: column; align-items: center; gap: 14px; }
.eyebrow { font-size: 13px; font-weight: 800; letter-spacing: .14em; text-transform: uppercase; color: var(--gold-dark); }
.hero h1 { font-size: 46px; line-height: 1.15; max-width: 760px; color: var(--navy); }
.hero p { margin: 0; color: var(--muted); font-size: 17px; max-width: 580px; }
.search { display: flex; align-items: center; gap: 10px; width: 100%; max-width: 640px; margin-top: 12px; background: #fff; border: 1px solid #d6cdbb; border-radius: 12px; padding: 8px; box-shadow: 0 8px 28px rgba(27, 42, 65, .1); }
.search input { flex: 1; border: 0; outline: 0; font: inherit; font-size: 16px; padding: 0 12px; min-width: 0; background: transparent; }
.clear { border: 0; background: #efe8da; color: var(--navy); width: 30px; height: 30px; border-radius: 50%; font-size: 20px; line-height: 1; cursor: pointer; flex-shrink: 0; }

.tiles { display: flex; flex-wrap: wrap; justify-content: center; gap: 12px; margin-top: 36px; }
.tile { display: flex; flex-direction: column; align-items: center; gap: 8px; width: 112px; padding: 14px 8px; background: #fff; border: 1px solid var(--line); border-radius: 12px; font: 600 13px Manrope, sans-serif; color: var(--ink); cursor: pointer; }
.tile:hover, .tile.on { border-color: var(--gold); box-shadow: 0 4px 14px rgba(176, 141, 87, .25); }
.tile.on { background: #f6efe0; }
.ico { width: 40px; height: 40px; border-radius: 10px; color: #fff; display: flex; align-items: center; justify-content: center; font-family: 'Playfair Display', Georgia, serif; font-weight: 700; font-size: 20px; }

.list { padding-top: 32px; padding-bottom: 56px; }
.list-top { display: flex; flex-wrap: wrap; gap: 16px; align-items: flex-end; justify-content: space-between; margin-bottom: 20px; }
.list h2 { font-size: 28px; color: var(--navy); }
.mini { display: flex; flex-wrap: wrap; gap: 10px; }
.mini select, .f select, .f input, .f textarea { border: 1px solid #d6cdbb; border-radius: 8px; padding: 0 12px; font: inherit; background: #fff; color: var(--ink); }
.mini select { height: 42px; font-size: 14px; }

.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); gap: 20px; }
.card { background: #fff; border: 1px solid var(--line); border-radius: 12px; overflow: hidden; display: flex; flex-direction: column; box-shadow: 0 2px 10px rgba(27, 42, 65, .05); transition: transform .15s, box-shadow .15s; }
.card:hover { transform: translateY(-3px); box-shadow: 0 12px 26px rgba(27, 42, 65, .14); }
.thumbwrap { position: relative; }
.badge { position: absolute; top: 10px; left: 10px; z-index: 1; background: var(--navy); color: #f1e3c4; font-weight: 800; font-size: 11px; letter-spacing: .08em; padding: 4px 8px; border-radius: 4px; }
.ph { height: 170px; color: rgba(255, 255, 255, .92); display: flex; align-items: center; justify-content: center; font-family: 'Playfair Display', Georgia, serif; font-size: 64px; font-weight: 700; }
.img { width: 100%; height: 170px; object-fit: cover; display: block; }
.ph.big, .img.big { height: 100%; min-height: 320px; border-radius: 12px; }
.in { padding: 14px 16px 16px; display: flex; flex-direction: column; gap: 6px; }
.in h3 { font-size: 18px; color: var(--navy); }
.m { font-size: 14px; color: var(--muted); margin: 0; }
.m.bank { font-weight: 600; }
.m.center { text-align: center; }
.m.hint { margin-top: 10px; }
.p { font-weight: 800; font-size: 19px; color: var(--gold-dark); margin-top: 4px; }
.more { text-align: center; margin-top: 28px; }

.detail { padding-top: 32px; padding-bottom: 56px; }
.detail-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 32px; margin-top: 20px; }
.detail h1 { color: var(--navy); font-size: 32px; }
.specs { padding-left: 18px; margin: 0; color: var(--muted); line-height: 1.8; }
.visit { border-left: 4px solid var(--gold); }
.contact { display: grid; grid-template-columns: 80px 1fr; gap: 8px 12px; margin: 4px 0 0; }
.contact dt { font-weight: 700; color: var(--muted); font-size: 14px; }
.contact dd { margin: 0; font-weight: 600; }

.box { display: flex; flex-direction: column; gap: 14px; background: #fff; border: 1px solid var(--line); border-radius: 12px; padding: 20px; }
.authpage { padding-top: 48px; padding-bottom: 64px; }
.authbox { max-width: 430px; margin: 0 auto; width: 100%; }
.seg { display: grid; grid-template-columns: 1fr 1fr; border: 1px solid #d6cdbb; border-radius: 10px; overflow: hidden; }
.segbtn { height: 46px; border: 0; background: #fff; color: var(--navy); font: 800 15px Manrope, sans-serif; cursor: pointer; }
.segbtn + .segbtn { border-left: 1px solid #d6cdbb; }
.segbtn.on { background: var(--navy); color: #f1e3c4; }
.segbtn:disabled { background: #f1eee7; color: #a39b8b; cursor: not-allowed; }
.f { display: flex; flex-direction: column; gap: 6px; flex: 1; min-width: 150px; font-size: 13px; font-weight: 600; }
.f select, .f input { height: 46px; }
.f input[type='file'] { padding: 10px 12px; height: auto; }
.f textarea { padding: 12px; resize: vertical; }
.ok { color: #2f6f4f; font-weight: 700; }
.err { color: var(--danger); font-weight: 600; margin: 0; }

.admin { padding-top: 32px; padding-bottom: 56px; display: flex; flex-direction: column; gap: 20px; }
.admin-top { display: flex; flex-wrap: wrap; gap: 12px; align-items: center; justify-content: space-between; }
.row { display: flex; flex-wrap: wrap; gap: 12px; align-items: center; }
.row.between { justify-content: space-between; }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 14px; }
.preview { max-width: 240px; border-radius: 10px; border: 1px solid var(--line); }
.subtitle { font-size: 20px; margin-top: 8px; color: var(--navy); }
.rows { display: flex; flex-direction: column; gap: 10px; }
.rowitem { display: flex; flex-wrap: wrap; gap: 14px; align-items: center; background: #fff; border: 1px solid var(--line); border-radius: 10px; padding: 10px; }
.rowitem.editing { border-color: var(--gold); background: #fbf6ea; }
.rowitem .grow { flex: 1; display: flex; flex-direction: column; gap: 4px; min-width: 200px; }
.thumb { width: 84px; height: 60px; object-fit: cover; border-radius: 8px; flex-shrink: 0; }
.thumb.ph { height: 60px; font-family: Manrope, sans-serif; font-size: 12px; font-weight: 600; background: #efe8da; color: var(--muted); }

.footer { padding: 28px 48px; font-size: 14px; color: var(--muted); text-align: center; border-top: 1px solid var(--line); background: #fff; }
@media (max-width: 700px) { .header { padding: 14px 20px; } .hero h1 { font-size: 30px; } .tile { width: 96px; } }
</style>
