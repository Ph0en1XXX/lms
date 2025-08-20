<template>
  <div>
    <NoPermission v-if="!$user.data" />

    <div v-else-if="profile.data">
      <header class="sticky top-0 z-10 flex items-center justify-between border-b bg-surface-white px-3 py-2.5">
        <Breadcrumbs class="h-7" :items="breadcrumbs" />
      </header>

      <div class="mx-auto -mt-10 max-w-4xl px-5">
        <div class="ml-auto" v-if="ready && $user.data && profile.data && isSessionUser()">
          <Button @click="toggleEdit()">
            <template #prefix><Edit class="w-4 h-4 stroke-1.5 text-ink-gray-7" /></template>
            {{ editMode ? 'Отмена' : 'Редактировать' }}
          </Button>
        </div>

        <!-- Убрана кнопка About -->
        <!-- <div class="mt-6">
          <TabButtons class="inline-block" :buttons="[{label:'About'}]" v-model="activeTab" />
        </div> -->

        <!-- VIEW MODE -->
        <div v-if="!editMode" class="mt-4 space-y-3">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <b>Фамилия:</b> {{ schoolProfile.value?.last_name || '-' }}<br/>
              <b>Имя:</b> {{ schoolProfile.value?.first_name || '-' }}<br/>
              <b>Отчество:</b> {{ schoolProfile.value?.middle_name || '-' }}<br/>
              <b>Дата рождения:</b> {{ formattedDate(schoolProfile.value?.birth_date) || '-' }}
            </div>
            <div>
              <b>Школа:</b> {{ schoolProfile.value?.school_name || schoolProfile.value?.school || '-' }}<br/>
              <b>Класс:</b> {{ schoolProfile.value?.grade || '-' }}<br/>
              <b>Телефон:</b> {{ maskPrivate(schoolProfile.value?.phone) }}<br/>
              <b>Email (приватный):</b> {{ maskPrivate(schoolProfile.value?.email_private) }}<br/>
              <b>Telegram:</b>
              <a v-if="schoolProfile.value?.telegram" :href="formatTelegram(schoolProfile.value.telegram)" target="_blank">{{ schoolProfile.value.telegram }}</a>
            </div>
          </div>

          <div>
            <b>ЕГЭ, планируется:</b> {{ (schoolProfile.value?.exams || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Чему хочется научиться:</b> {{ (schoolProfile.value?.learn_subjects || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Коротко о интересах:</b>
            <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value?.interests || '-' }}</div>
          </div>

          <div class="grid md:grid-cols-2 gap-4">
            <div>
              <b>Коротко о себе:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value?.about_me || '-' }}</div>
            </div>
            <div>
              <b>Коротко о мечтах:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value?.dreams || '-' }}</div>
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

          <div class="mt-4">
            <label class="block mb-2 font-medium">ЕГЭ (отметьте):</label>
            <div class="grid grid-cols-2 md:grid-cols-4 gap-2">
              <label v-for="e in examOptions" :key="e" class="flex items-center space-x-2">
                <input type="checkbox" :value="e" v-model="form.exams" />
                <span>{{ e }}</span>
              </label>
            </div>
          </div>

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
  </div>
</template>

<script setup>
import { ref, computed, inject, onMounted, watch } from 'vue' // добавили watch
import { Breadcrumbs, Button, Input, DatePicker, Select, Textarea, createResource } from 'frappe-ui'
import { sessionStore } from '@/stores/session'
import NoPermission from '@/components/NoPermission.vue'
import { Edit } from 'lucide-vue-next'
import debounce from 'lodash/debounce'

const { user } = sessionStore()
const $user = inject('$user')

const props = defineProps({
  username: { type: String, required: true }
})

const editMode = ref(false)
const saving = ref(false)

const examOptions = [
  'Русский язык','Математика(базовый)','Математика(профильный)','Физика','Химия','Информатика',
  'Биология','История','География','Английский язык','Немецкий язык','Французский язык','Испанский язык',
  'Китайский язык','Обществознание','Литература'
]

const learnOptions = [
  'Программирование','Дизайн','Актерское мастерство','Риторика','Робототехника','Иностранные языки',
  'Математика углубленно','Физика углубленно'
]

const resolvedUsername = computed(() => props.username || $user.data?.username || null)

const profile = createResource({
  url: 'frappe.client.get',
  params: {
    doctype: 'User',
    name: '' // сюда подставим username позже
  },
  auto: false
})

const schoolProfile = createResource({
  url: 'frappe.client.get_list',
  params: {
    doctype: 'Schoolchildren Profile',
    filters: { user: '' },   // заполним позже
    limit_page_length: 1
  },
  auto: false,
  transform(data) {
    let doc = data && data.length ? data[0] : {}
    try { doc.exams = JSON.parse(doc.exams) } catch(e){ doc.exams = doc.exams ? doc.exams.split(',').map(s=>s.trim()) : [] }
    try { doc.learn_subjects = JSON.parse(doc.learn_subjects) } catch(e){ doc.learn_subjects = doc.learn_subjects ? doc.learn_subjects.split(',').map(s=>s.trim()) : [] }
    return doc
  }
})

const ready = ref(false)

async function loadAll(u) {
  ready.value = false
  try {
    // Подтягиваем User (через get)
    await profile.reload({
      params: {
        doctype: 'User',
        name: u
      }
    })

    // Подтягиваем Schoolchildren Profile
    await schoolProfile.reload({
      params: {
        doctype: 'Schoolchildren Profile',
        filters: { user: u },
        limit_page_length: 1
      }
    })

    if (schoolProfile.value && Object.keys(schoolProfile.value).length) {
      fillFormFromProfile()
    }
  } finally {
    ready.value = true
  }
}

watch(
  [() => props.username, () => $user.data && $user.data.username],
  ([propU, sessU]) => {
    const u = propU || sessU
    if (!u) return
    loadAll(u)
  },
  { immediate: true }
)




const form = ref({
  first_name: '',
  last_name: '',
  middle_name: '',
  birth_date: '',
  school: '',
  school_name: '',
  grade: '',
  phone: '',
  email_private: '',
  telegram: '',
  exams: [],
  learn_subjects: [],
  interests: '',
  about_me: '',
  dreams: ''
})

const breadcrumbs = computed(() => {
  const items = [{ label: 'People' }]
  if (props.username) {
    items.push({
      label: profile.data?.full_name || form.value.first_name || '',
      route: { name: 'Profile', params: { username: props.username } }
    })
  } else {
    items.push({
      label: profile.data?.full_name || form.value.first_name || ''
    })
  }
  return items
})


const displayName = computed(() => profile.data?.full_name || `${form.value.first_name || ''} ${form.value.last_name || ''}`)

function formattedDate(d){
  if(!d) return ''
  try{
    return new Date(d).toLocaleDateString('ru-RU')
  }catch(e){ return d }
}

function maskPrivate(val){
  if(!val) return '-'
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

function isSessionUser() {
  return $user.data?.username === (props.username || $user.data?.username)
}



function fillFormFromProfile(){
  const sp = schoolProfile.data || {}
  form.value.first_name = sp.first_name || profile.data?.first_name || ''
  form.value.last_name  = sp.last_name  || profile.data?.last_name  || ''
  form.value.middle_name = sp.middle_name || ''
  form.value.birth_date  = sp.birth_date  || ''
  form.value.school      = sp.school || ''
  form.value.school_name = sp.school_name || ''
  form.value.grade       = sp.grade || ''
  form.value.phone       = sp.phone || ''
  form.value.email_private = sp.email_private || ''
  form.value.telegram    = sp.telegram || ''
  form.value.exams       = Array.isArray(sp.exams) ? [...sp.exams] : (sp.exams||[])
  form.value.learn_subjects = Array.isArray(sp.learn_subjects) ? [...sp.learn_subjects] : (sp.learn_subjects||[])
  form.value.interests   = sp.interests || ''
  form.value.about_me    = sp.about_me || ''
  form.value.dreams      = sp.dreams || ''
}


function toggleEdit(){
  editMode.value = !editMode.value
  if(editMode.value) fillFormFromProfile()
}

// Сохранение профиля
async function saveProfile(){
  saving.value = true
  try {
    // Обновление User (имя/фамилия)
    if (form.value.first_name || form.value.last_name) {
      if (!profile.data?.name) {
        // если по какой-то причине name не подтянулся — выходим тихо
      } else {
        await createResource({
          url: 'frappe.client.set_value',
          params: {
            doctype: 'User',
            name: profile.data.name,
            fieldname: 'full_name',
            value: `${form.value.first_name || ''} ${form.value.last_name || ''}`.trim()
          }
        }).submit()
      }
    }

    // Проверяем, есть ли профиль школьника
    let docname = schoolProfile.value.name
    let payload = {
      doctype: 'Schoolchildren Profile',
      user: profile.data.name,
      first_name: form.value.first_name,
      last_name: form.value.last_name,
      middle_name: form.value.middle_name,
      birth_date: form.value.birth_date,
      school: form.value.school || '',
      school_name: form.value.school_name || '',
      grade: form.value.grade,
      phone: form.value.phone,
      email_private: form.value.email_private,
      telegram: form.value.telegram,
      exams: JSON.stringify(form.value.exams || []),
      learn_subjects: JSON.stringify(form.value.learn_subjects || []),
      interests: form.value.interests,
      about_me: form.value.about_me,
      dreams: form.value.dreams,
      last_updated: (new Date()).toISOString()
    }

    if(docname){
      // Обновление существующего
      await createResource({
        url: 'frappe.client.set_value',
        params: {
          doctype: 'Schoolchildren Profile',
          name: docname,
          fieldname: Object.keys(payload),
          value: undefined
        }
      }).submit().catch(()=>{})
      // Сохраняем через save
      await createResource({
        url: 'frappe.client.save',
        params: { doc: { ...schoolProfile.data, ...payload } }
      }).submit()
    } else {
      // Вставка нового
      await createResource({
        url: 'frappe.client.insert',
        params: { doc: payload }
      }).submit()
    }

    await schoolProfile.reload()
    editMode.value = false
    if(window.frappe && window.frappe.msgprint) window.frappe.msgprint('Профиль сохранён')
  } catch(e){
    console.error(e)
    if(window.frappe && window.frappe.msgprint) window.frappe.msgprint({ title: 'Ошибка', message: (e && e.message) || 'Ошибка при сохранении', indicator: 'red' })
  } finally {
    saving.value = false
  }
}

// Поиск школы
const schoolQuery = ref('')
const schoolResults = ref([])

async function searchSchool(q){
  if(!q) { schoolResults.value = []; return }
  try {
    const res = await createResource({
      url: 'frappe.client.get_list',
      params: {
        doctype: 'School List',
        fields: ['name', 'school_name', 'license_number', 'region'],
        filters: [['school_name', 'like', '%' + q + '%']],
        limit_page_length: 20
      }
    }).submit()
    schoolResults.value = res || []
  } catch(e){
    schoolResults.value = []
  }
}

const debouncedSearchSchool = debounce(()=> searchSchool(schoolQuery.value), 300)

function selectSchool(s){
  form.value.school = s.name
  form.value.school_name = s.school_name
  schoolResults.value = []
  schoolQuery.value = s.school_name
}

onMounted(() => {
  // Если профиль найден — заполняем форму для редактирования
  if (schoolProfile.value && Object.keys(schoolProfile.value).length) {
    fillFormFromProfile()
  }
})
</script>

<style scoped>
/* Небольшие правки стилей (можно добавить Tailwind кастомизации) */
</style>
