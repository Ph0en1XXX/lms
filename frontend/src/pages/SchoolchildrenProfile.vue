<template>
  <div>
    <NoPermission v-if="!$user.data" />

    <div v-else-if="profileLoaded">
      <header class="sticky top-0 z-10 flex items-center justify-between border-b bg-surface-white px-3 py-2.5">
        <Breadcrumbs class="h-7" :items="breadcrumbs" />
      </header>

      <div class="mx-auto -mt-10 max-w-4xl px-5">
        <div class="flex items-center">
          <div>
            <img
              v-if="userDoc.user_image"
              :src="userDoc.user_image"
              class="object-cover h-[100px] w-[100px] rounded-full border-4 border-white"
            />
            <UserAvatar v-else :user="userDoc" class="object-cover h-[100px] w-[100px] rounded-full border-4 border-white" />
          </div>

          <div class="ml-6">
            <h2 class="mt-2 text-3xl font-semibold text-ink-gray-9">{{ displayName }}</h2>
            <div class="mt-2 text-base text-ink-gray-7">{{ schoolProfile.headline || '' }}</div>
          </div>

          <div class="ml-auto">
            <Button v-if="isSessionUser()" @click="toggleEdit()">
              <template #prefix><Edit class="w-4 h-4 stroke-1.5 text-ink-gray-7" /></template>
              {{ editMode ? 'Отмена' : 'Редактировать' }}
            </Button>
          </div>
        </div>

        <div class="mt-6">
          <TabButtons class="inline-block" :buttons="[{label:'About'}]" v-model="activeTab" />
        </div>

        <!-- VIEW MODE -->
        <div v-if="!editMode" class="mt-4 space-y-3">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <b>Фамилия:</b> {{ schoolProfile.last_name || '-' }}<br/>
              <b>Имя:</b> {{ schoolProfile.first_name || '-' }}<br/>
              <b>Отчество:</b> {{ schoolProfile.middle_name || '-' }}<br/>
              <b>Дата рождения:</b> {{ formattedDate(schoolProfile.birth_date) || '-' }}
            </div>
            <div>
              <b>Школа:</b> {{ schoolProfile.school_name || schoolProfile.school || '-' }}<br/>
              <b>Класс:</b> {{ schoolProfile.grade || '-' }}<br/>
              <b>Телефон:</b> {{ maskPrivate(schoolProfile.phone) }}<br/>
              <b>Email (приватный):</b> {{ maskPrivate(schoolProfile.email_private) }}<br/>
              <b>Telegram:</b> <a v-if="schoolProfile.telegram" :href="formatTelegram(schoolProfile.telegram)" target="_blank">{{ schoolProfile.telegram }}</a>
            </div>
          </div>

          <div>
            <b>ЕГЭ, планируется:</b> {{ (schoolProfile.exams || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Чему хочется научиться:</b> {{ (schoolProfile.learn_subjects || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Коротко о интересах:</b>
            <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{
              schoolProfile.interests || '-'
            }}</div>
          </div>

          <div class="grid md:grid-cols-2 gap-4">
            <div>
              <b>Коротко о себе:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.about_me || '-' }}</div>
            </div>
            <div>
              <b>Коротко о мечтах:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.dreams || '-' }}</div>
            </div>
          </div>
        </div>

        <!-- EDIT MODE -->
        <div v-else class="mt-4">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <Input v-model="form.last_name" label="Фамилия" />
            <Input v-model="form.first_name" label="Имя" />
            <Input v-model="form.middle_name" label="Отчество" />
            <DatePicker v-model="form.birth_date" label="Дата рождения" />
            <!-- Школа: Link field к School List: реализуем через typeahead -->
            <div>
              <label class="block text-sm font-medium text-ink-gray-7 mb-1">Школа</label>
              <input
                type="text"
                v-model="schoolQuery"
                @input="debouncedSearchSchool"
                class="w-full border rounded p-2"
                placeholder="Начните вводить название школы"
              />
              <div v-if="schoolResults.length" class="border rounded mt-1 max-h-44 overflow-auto bg-white">
                <div
                  v-for="s in schoolResults"
                  :key="s.name"
                  class="p-2 cursor-pointer hover:bg-surface-gray-2"
                  @click="selectSchool(s)"
                >
                  <div class="font-medium">{{ s.school_name }}</div>
                  <div class="text-xs text-ink-gray-6">{{ s.region }} {{ s.license_number ? '• лиц: ' + s.license_number : '' }}</div>
                </div>
              </div>
              <div v-if="form.school_name && !schoolResults.length" class="text-xs text-ink-gray-6 mt-1">
                Выбрана: {{ form.school_name }}
              </div>
            </div>

            <Select v-model="form.grade" :options="['10','11']" label="Класс" />
            <Input v-model="form.phone" label="Телефон (не публиковать)" />
            <Input v-model="form.email_private" label="Email (не публиковать)" />
            <Input v-model="form.telegram" label="Telegram (например t.me/username)" />
          </div>

          <!-- ЕГЭ: чекбоксы -->
          <div class="mt-4">
            <label class="block mb-2 font-medium">ЕГЭ (отметьте):</label>
            <div class="grid grid-cols-2 md:grid-cols-4 gap-2">
              <label v-for="e in examOptions" :key="e" class="flex items-center space-x-2">
                <input type="checkbox" :value="e" v-model="form.exams" />
                <span>{{ e }}</span>
              </label>
            </div>
          </div>

          <!-- Чему хочется научиться -->
          <div class="mt-4">
            <label class="block mb-2 font-medium">Чему хочется научиться (выбери):</label>
            <div class="grid grid-cols-2 md:grid-cols-4 gap-2">
              <label v-for="s in learnOptions" :key="s" class="flex items-center space-x-2">
                <input type="checkbox" :value="s" v-model="form.learn_subjects" />
                <span>{{ s }}</span>
              </label>
            </div>
          </div>

          <div class="mt-4 grid md:grid-cols-2 gap-4">
            <Textarea v-model="form.interests" label="Коротко о своих интересах (2-3 предложения)" />
            <Textarea v-model="form.about_me" label="Коротко о себе" />
            <Textarea class="md:col-span-2" v-model="form.dreams" label="Коротко о своих мечтах" />
          </div>

          <div class="mt-4 flex space-x-2">
            <Button @click="saveProfile" :loading="saving">Сохранить</Button>
            <Button variant="outline" @click="toggleEdit()">Отмена</Button>
          </div>
        </div>
      </div>
    </div>

    <!-- EditProfile modal placeholder (если нужно) -->
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, inject } from 'vue'
import { Breadcrumbs, Button, TabButtons, Input, DatePicker, Select, Textarea } from 'frappe-ui'
import { sessionStore } from '@/stores/session'
import UserAvatar from '@/components/UserAvatar.vue'
import NoPermission from '@/components/NoPermission.vue'
import { Edit } from 'lucide-vue-next'
import debounce from 'lodash/debounce'

const { user } = sessionStore()
const $user = inject('$user')

const props = defineProps({
  username: { type: String, required: true }
})

const profileLoaded = ref(false)
const editMode = ref(false)
const saving = ref(false)
const activeTab = ref('About')

const userDoc = ref({})
const schoolProfile = reactive({}) // merged view object
const form = reactive({
  first_name: '',
  last_name: '',
  middle_name: '',
  birth_date: '',
  school: '',         // School List docname
  school_name: '',    // friendly name for view
  grade: '',
  phone: '',
  email_private: '',
  telegram: '',
  exams: [],          // array
  learn_subjects: [], // array
  interests: '',
  about_me: '',
  dreams: ''
})

const examOptions = [
  'Русский язык','Математика(базовый)','Математика(профильный)','Физика','Химия','Информатика',
  'Биология','История','География','Английский язык','Немецкий язык','Французский язык','Испанский язык',
  'Китайский язык','Обществознание','Литература'
]

const learnOptions = [
  // образец: можно расширить списком с profi.ru
  'Программирование','Дизайн','Актерское мастерство','Риторика','Робототехника','Иностранные языки',
  'Математика углубленно','Физика углубленно'
]

const breadcrumbs = computed(()=>[
  { label: 'People' },
  { label: `${userDoc.value.full_name || form.first_name || ''}`, route: { name: 'Profile', params: { username: props.username } } }
])

const displayName = computed(()=> userDoc.value.full_name || `${form.first_name || ''} ${form.last_name || ''}`)

function formattedDate(d){
  if(!d) return ''
  try{
    return new Date(d).toLocaleDateString('ru-RU')
  }catch(e){ return d }
}

function maskPrivate(val){
  if(!val) return '-'
  // небольшой маскирование, если это телефон/почта
  if(val.includes('@')){
    const parts = val.split('@')
    return parts[0].slice(0,1) + '***@' + parts[1]
  }
  return val.slice(0,3) + '***' + val.slice(-2)
}

function formatTelegram(t){
  if(!t) return ''
  if(t.startsWith('t.me/') || t.startsWith('https://t.me/')) return (t.startsWith('http')? t : 'https://' + t)
  return 'https://t.me/' + t.replace(/^@/, '')
}

function isSessionUser(){
  return $user.data?.email === userDoc.value?.email || $user.data?.username === props.username
}

// Загрузка данных: User и Schoolchildren Profile
async function loadAll(){
  // 1) user doc по username
  const userRes = await frappe.call({
    method: 'frappe.client.get',
    args: { doctype: 'User', name: props.username }
  })
  userDoc.value = userRes.message || {}

  // 2) попытка получить Schoolchildren Profile по link user
  let profileRes = null
  try {
    profileRes = await frappe.call({
      method: 'frappe.client.get_list',
      args: {
        doctype: 'Schoolchildren Profile',
        filters: { user: userDoc.value.name },
        limit_page_length: 1
      }
    })
  } catch(e){
    // игнор
    profileRes = []
  }

  let profileDoc = profileRes && profileRes.message && profileRes.message.length ? profileRes.message[0] : null
  if(profileDoc){
    // Если exams/learn_subjects хранятся как JSON в текстовом поле, десериализуем
    try { profileDoc.exams = JSON.parse(profileDoc.exams) } catch(e){ profileDoc.exams = profileDoc.exams ? profileDoc.exams.split(',').map(s=>s.trim()) : [] }
    try { profileDoc.learn_subjects = JSON.parse(profileDoc.learn_subjects) } catch(e){ profileDoc.learn_subjects = profileDoc.learn_subjects ? profileDoc.learn_subjects.split(',').map(s=>s.trim()) : [] }
    // если school — link, получим display name
    if(profileDoc.school){
      try {
        const s = await frappe.call({ method: 'frappe.client.get', args: { doctype: 'School List', name: profileDoc.school }})
        profileDoc.school_name = s.message?.school_name || profileDoc.school
        profileDoc.region = s.message?.region
      } catch(e){
        profileDoc.school_name = profileDoc.school
      }
    }
  } else {
    // пустой профиль
    profileDoc = {}
  }

  // заполним reactive view-schoolProfile
  Object.assign(schoolProfile, profileDoc)

  // если профиль пустой — попробуем заполнить фамилию/имя из User
  if(!schoolProfile.first_name && userDoc.value.first_name) schoolProfile.first_name = userDoc.value.first_name
  if(!schoolProfile.last_name && userDoc.value.last_name) schoolProfile.last_name = userDoc.value.last_name

  // подготовим форму для редактирования
  fillFormFromProfile()

  profileLoaded.value = true
}

function fillFormFromProfile(){
  form.first_name = schoolProfile.first_name || ''
  form.last_name = schoolProfile.last_name || ''
  form.middle_name = schoolProfile.middle_name || ''
  form.birth_date = schoolProfile.birth_date || ''
  form.school = schoolProfile.school || ''
  form.school_name = schoolProfile.school_name || ''
  form.grade = schoolProfile.grade || ''
  form.phone = schoolProfile.phone || ''
  form.email_private = schoolProfile.email_private || ''
  form.telegram = schoolProfile.telegram || ''
  form.exams = Array.isArray(schoolProfile.exams) ? [...schoolProfile.exams] : (schoolProfile.exams||[])
  form.learn_subjects = Array.isArray(schoolProfile.learn_subjects) ? [...schoolProfile.learn_subjects] : (schoolProfile.learn_subjects||[])
  form.interests = schoolProfile.interests || ''
  form.about_me = schoolProfile.about_me || ''
  form.dreams = schoolProfile.dreams || ''
}

// Сохранение: создаем/обновляем Schoolchildren Profile и частично User (имя)
async function saveProfile(){
  saving.value = true
  try {
    // 1) обновление User: если изменены имя/фамилия
    if(form.first_name || form.last_name){
      await frappe.call({
        method: 'frappe.client.set_value',
        args: {
          doctype: 'User',
          name: userDoc.value.name,
          fieldname: 'full_name',
          value: `${form.first_name || ''} ${form.last_name || ''}`.trim()
        }
      })
    }

    // 2) подготовим payload для Schoolchildren Profile
    const payload = {
      doctype: 'Schoolchildren Profile',
      user: userDoc.value.name,
      first_name: form.first_name,
      last_name: form.last_name,
      middle_name: form.middle_name,
      birth_date: form.birth_date,
      school: form.school || '',
      school_name: form.school_name || '',
      grade: form.grade,
      phone: form.phone,
      email_private: form.email_private,
      telegram: form.telegram,
      exams: JSON.stringify(form.exams || []),
      learn_subjects: JSON.stringify(form.learn_subjects || []),
      interests: form.interests,
      about_me: form.about_me,
      dreams: form.dreams,
      last_updated: (new Date()).toISOString()
    }

    // Проверим, есть ли уже запись
    const existing = await frappe.call({
      method: 'frappe.client.get_list',
      args: { doctype: 'Schoolchildren Profile', filters: { user: userDoc.value.name }, limit_page_length: 1 }
    })

    if(existing.message && existing.message.length){
      const docname = existing.message[0].name
      await frappe.call({
        method: 'frappe.client.set_value',
        args: { doctype: 'Schoolchildren Profile', name: docname, fieldname: Object.keys(payload), value: undefined } // we will use save to update
      }).catch(()=>{})
      // safer: load doc + update + save
      const doc = await frappe.call({ method: 'frappe.client.get', args: { doctype: 'Schoolchildren Profile', name: docname }})
      const updated = Object.assign(doc.message || {}, payload)
      await frappe.call({ method: 'frappe.client.save', args: { doc: updated } })
    } else {
      // insert new
      await frappe.call({ method: 'frappe.client.insert', args: { doc: payload } })
    }

    // reload
    await loadAll()
    editMode.value = false
    frappe.msgprint('Профиль сохранён')
  } catch(e){
    console.error(e)
    frappe.msgprint({ title: 'Ошибка', message: (e && e.message) || 'Ошибка при сохранении', indicator: 'red' })
  } finally {
    saving.value = false
  }
}

// SEARCH SCHOOL: поиск по School List (серверный)
const schoolQuery = ref('')
const schoolResults = ref([])

async function searchSchool(q){
  if(!q) { schoolResults.value = []; return }
  try {
    const res = await frappe.call({
      method: 'frappe.client.get_list',
      args: {
        doctype: 'School List',
        fields: ['name', 'school_name', 'license_number', 'region'],
        filters: [['school_name', 'like', '%' + q + '%']],
        limit_page_length: 20
      }
    })
    schoolResults.value = res.message || []
  } catch(e){
    schoolResults.value = []
  }
}

const debouncedSearchSchool = debounce(()=> searchSchool(schoolQuery.value), 300)

function selectSchool(s){
  form.school = s.name
  form.school_name = s.school_name
  schoolResults.value = []
  schoolQuery.value = s.school_name
}

function toggleEdit(){ editMode.value = !editMode.value; if(editMode.value) fillFormFromProfile() }

onMounted(()=> loadAll())

</script>

<style scoped>
/* Небольшие правки стилей (можно добавить Tailwind кастомизации) */
</style>
