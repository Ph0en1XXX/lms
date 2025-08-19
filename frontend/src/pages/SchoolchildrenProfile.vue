<template>
  <div>
    <NoPermission v-if="!$user.data" />

    <div v-else-if="profile.data">
      <header class="sticky top-0 z-10 flex items-center justify-between border-b bg-surface-white px-3 py-2.5">
        <Breadcrumbs class="h-7" :items="breadcrumbs" />
      </header>

      <div class="mx-auto -mt-10 max-w-4xl px-5">
        <div class="flex items-center min-h-[100px]">
          <div class="ml-6">
            <h2 class="mt-2 text-3xl font-semibold text-ink-gray-9">{{ displayName }}</h2>
            <div class="mt-2 text-base text-ink-gray-7">{{ profile.data.headline || '' }}</div>
          </div>
          <div class="ml-auto">
            <Button v-if="isSessionUser()" @click="toggleEdit()">
              <template #prefix><Edit class="w-4 h-4 stroke-1.5 text-ink-gray-7" /></template>
              {{ editMode ? 'Отмена' : 'Редактировать' }}
            </Button>
          </div>
        </div>

        <!-- Убрана кнопка About -->
        <!-- <div class="mt-6">
          <TabButtons class="inline-block" :buttons="[{label:'About'}]" v-model="activeTab" />
        </div> -->

        <!-- VIEW MODE -->
        <div v-if="!editMode" class="mt-4 space-y-3">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <b>Фамилия:</b> {{ schoolProfile.value.last_name || '-' }}<br/>
              <b>Имя:</b> {{ schoolProfile.value.first_name || '-' }}<br/>
              <b>Отчество:</b> {{ schoolProfile.value.middle_name || '-' }}<br/>
              <b>Дата рождения:</b> {{ formattedDate(schoolProfile.value.birth_date) || '-' }}
            </div>
            <div>
              <b>Школа:</b> {{ schoolProfile.value.school_name || schoolProfile.value.school || '-' }}<br/>
              <b>Класс:</b> {{ schoolProfile.value.grade || '-' }}<br/>
              <b>Телефон:</b> {{ maskPrivate(schoolProfile.value.phone) }}<br/>
              <b>Email (приватный):</b> {{ maskPrivate(schoolProfile.value.email_private) }}<br/>
              <b>Telegram:</b>
              <a v-if="schoolProfile.value.telegram" :href="formatTelegram(schoolProfile.value.telegram)" target="_blank">{{ schoolProfile.value.telegram }}</a>
            </div>
          </div>

          <div>
            <b>ЕГЭ, планируется:</b> {{ (schoolProfile.value.exams || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Чему хочется научиться:</b> {{ (schoolProfile.value.learn_subjects || []).join(', ') || '-' }}
          </div>

          <div>
            <b>Коротко о интересах:</b>
            <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value.interests || '-' }}</div>
          </div>

          <div class="grid md:grid-cols-2 gap-4">
            <div>
              <b>Коротко о себе:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value.about_me || '-' }}</div>
            </div>
            <div>
              <b>Коротко о мечтах:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.value.dreams || '-' }}</div>
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
import { ref, computed, inject } from 'vue'
import { Breadcrumbs, Button, TabButtons, Input, DatePicker, Select, Textarea, createResource } from 'frappe-ui'
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

const profile = ref({
  data: {
    full_name: 'Иван Иванов',
    user_image: '',
    headline: 'Тестовый заголовок',
    email: 'test@example.com'
  }
})

const schoolProfile = ref({
  value: {
    first_name: 'Иван',
    last_name: 'Иванов',
    middle_name: 'Иванович',
    birth_date: '2005-01-01',
    school: 'Школа №1',
    school_name: 'Школа №1',
    grade: '10',
    phone: '+79991234567',
    email_private: 'private@example.com',
    telegram: '@testuser',
    exams: ['Математика', 'Русский язык'],
    learn_subjects: ['Программирование'],
    interests: 'Интересы: спорт, музыка',
    about_me: 'О себе: люблю учиться',
    dreams: 'Мечтаю стать программистом'
  }
})

const editMode = ref(false)
const saving = ref(false)
const activeTab = ref('About')

const examOptions = [
  'Русский язык','Математика(базовый)','Математика(профильный)','Физика','Химия','Информатика',
  'Биология','История','География','Английский язык','Немецкий язык','Французский язык','Испанский язык',
  'Китайский язык','Обществознание','Литература'
]

const learnOptions = [
  'Программирование','Дизайн','Актерское мастерство','Риторика','Робототехника','Иностранные языки',
  'Математика углубленно','Физика углубленно'
]

const form = ref({ ...schoolProfile.value })

const breadcrumbs = computed(() => [
  { label: 'People' },
  { label: profile.value.data?.full_name || form.value.first_name || '', route: { name: 'Profile', params: { username: 'test' } } }
])

const displayName = computed(() => profile.value.data?.full_name || `${form.value.first_name || ''} ${form.value.last_name || ''}`)

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

function isSessionUser(){
  return true // всегда true для теста
}

function fillFormFromProfile(){
  form.value = { ...schoolProfile.value }
}

function toggleEdit(){
  editMode.value = !editMode.value
  if(editMode.value) fillFormFromProfile()
}

async function saveProfile(){
  saving.value = true
  setTimeout(() => {
    // Копируем данные из формы в schoolProfile (имитация сохранения)
    schoolProfile.value = { ...form.value }
    saving.value = false
    editMode.value = false
    alert('Профиль сохранён (тестово, без бэка)')
  }, 500)
}

const schoolQuery = ref('')
const schoolResults = ref([])

async function searchSchool(q){
  if(!q) { schoolResults.value = []; return }
  // тестовые школы
  schoolResults.value = [
    { name: 'school1', school_name: 'Школа №1', region: 'Москва', license_number: '12345' },
    { name: 'school2', school_name: 'Школа №2', region: 'СПб', license_number: '67890' }
  ]
}

const debouncedSearchSchool = debounce(()=> searchSchool(schoolQuery.value), 300)

function selectSchool(s){
  form.value.school = s.name
  form.value.school_name = s.school_name
  schoolResults.value = []
  schoolQuery.value = s.school_name
}
</script>

<style scoped>
/* Небольшие правки стилей (можно добавить Tailwind кастомизации) */
</style>
