<template>
  <div>
    <NoPermission v-if="!$user.data" />
    <div v-else-if="profile.error">
      <p class="text-red-500">Ошибка загрузки профиля: {{ profile.error.message }}</p>
    </div>
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
          <div class="ml-auto" v-if="$user.data && isSessionUser()">
            <Button @click="toggleEdit()">
              <template #prefix>
                <Edit class="w-4 h-4 stroke-1.5 text-ink-gray-7" />
              </template>
              {{ editMode ? 'Отмена' : 'Редактировать' }}
            </Button>
          </div>
        </div>

        <!-- VIEW MODE -->
        <div v-if="!editMode" class="mt-4 space-y-3">
          <div v-if="schoolProfile.loading">
            <p>Загрузка данных профиля школьника...</p>
          </div>
          <div v-else-if="schoolProfile.error">
            <p class="text-red-500">Ошибка загрузки данных школьника: {{ schoolProfile.error.message }}</p>
          </div>
          <div v-else-if="schoolProfile.data">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <b>Фамилия:</b> {{ schoolProfile.data.last_name || '-' }}<br/>
                <b>Имя:</b> {{ schoolProfile.data.first_name || '-' }}<br/>
                <b>Отчество:</b> {{ schoolProfile.data.middle_name || '-' }}<br/>
                <b>Дата рождения:</b> {{ formattedDate(schoolProfile.data.birth_date) || '-' }}
              </div>
              <div>
                <b>Школа:</b> {{ schoolProfile.data.school || '-' }}<br/>
                <b>Класс:</b> {{ schoolProfile.data.grade || '-' }}<br/>
                <b>Телефон:</b> {{ maskPrivate(schoolProfile.data.phone) }}<br/>
                <b>Email (приватный):</b> {{ maskPrivate(schoolProfile.data.email_private) }}<br/>
                <b>Telegram:</b>
                <a v-if="schoolProfile.data.telegram" :href="formatTelegram(schoolProfile.data.telegram)" target="_blank">{{ schoolProfile.data.telegram }}</a>
              </div>
            </div>

            <div>
              <b>ЕГЭ, планируется:</b> {{ (schoolProfile.data.exams || []).join(', ') || '-' }}
            </div>

            <div>
              <b>Чему хочется научиться:</b> {{ schoolProfile.data.learn_subjects}}
            </div>

            <div>
              <b>Коротко о интересах:</b>
              <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.data.interests || '-' }}</div>
            </div>

            <div class="grid md:grid-cols-2 gap-4">
              <div>
                <b>Коротко о себе:</b>
                <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.data.about_me || '-' }}</div>
              </div>
              <div>
                <b>Коротко о мечтах:</b>
                <div class="mt-1 p-3 bg-surface-gray-1 rounded">{{ schoolProfile.data.dreams || '-' }}</div>
              </div>
            </div>
          </div>
          <div v-else>
            <p>Данные профиля школьника не найдены.</p>
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
                  :key="s.school"
                  class="p-2 cursor-pointer hover:bg-surface-gray-2"
                  @click="selectSchool(s)"
                >
                  <div class="font-medium">{{ s.school}}</div>
                  <div class="text-xs text-ink-gray-6">{{ s.adress }}</div>
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
    <div v-else>
      <p>Загрузка профиля...</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, inject, watch, onMounted } from 'vue';
import { Breadcrumbs, createResource, Button, Input, DatePicker, Select, Textarea } from 'frappe-ui';
import { sessionStore } from '@/stores/session';
import NoPermission from '@/components/NoPermission.vue';
import { Edit } from 'lucide-vue-next';
import { convertToTitleCase, updateDocumentTitle } from '@/utils';
import debounce from 'lodash/debounce';

const { user } = sessionStore();
const $user = inject('$user');

// Логирование инициализации
console.log('[DEBUG] Инициализация компонента:', {
  user: user,
  $user: $user.data,
  username: $user.data?.username,
});

const props = defineProps({
  username: {
    type: String,
    required: false,
    default: '',
  },
});

const effectiveUsername = computed(() => {
  const username = props.username || $user.data?.username || '';
  console.log('[DEBUG] Вычисление effectiveUsername:', { propsUsername: props.username, sessionUsername: $user.data?.username, result: username });
  return username;
});

const editMode = ref(false);
const saving = ref(false);

const examOptions = [
  'Русский язык', 'Математика(базовый)', 'Математика(профильный)', 'Физика', 'Химия', 'Информатика',
  'Биология', 'История', 'География', 'Английский язык', 'Немецкий язык', 'Французский язык', 'Испанский язык',
  'Китайский язык', 'Обществознание', 'Литература'
];

const learnOptions = [
  'Программирование', 'Дизайн', 'Актерское мастерство', 'Риторика', 'Робототехника', 'Иностранные языки',
  'Математика углубленно', 'Физика углубленно'
];

const profile = createResource({
  url: 'frappe.client.get',
  makeParams(values) {
    const username = effectiveUsername.value;
    console.log('[DEBUG] Запрос profile:', { doctype: 'User', filters: { username } });
    return {
      doctype: 'User',
      filters: { username },
    };
  },
  onSuccess(data) {
    console.log('[DEBUG] Профиль загружен:', data);
  },
  onError(error) {
    console.error('[DEBUG] Ошибка загрузки профиля:', error);
    window.frappe?.msgprint({
      title: 'Ошибка',
      message: 'Не удалось загрузить профиль пользователя: ' + (error.message || 'Неизвестная ошибка'),
      indicator: 'red',
    });
  },
});

const schoolProfile = createResource({
  url: 'frappe.client.get',
  params: {
    doctype: 'Schoolchildren Profile',
    filters: { user:user },
  },
  auto: false,
  transform(data) {
    let doc = data || {};
    console.log('[DEBUG] Данные schoolProfile до трансформации:', doc);
    try {
      doc.exams = doc.exams ? doc.exams.map(e => e.exam_subject) : [];
      doc.learn_subjects = doc.learn_subjects ? doc.learn_subjects.map(s => s.subject) : [];
    } catch (e) {
      console.error('[DEBUG] Ошибка трансформации данных:', e);
      doc.exams = [];
      doc.learn_subjects = [];
    }
    console.log('[DEBUG] Данные schoolProfile после трансформации:', doc);
    return doc;
  },
  onSuccess(data) {
    console.log('[DEBUG] Профиль школьника загружен:', data);
  },
  onError(error) {
    console.error('[DEBUG] Ошибка загрузки профиля школьника:', error);
    window.frappe?.msgprint({
      title: 'Ошибка',
      message: 'Не удалось загрузить профиль школьника: ' + (error.message || 'Неизвестная ошибка'),
      indicator: 'red',
    });
  },
});

const form = ref({
  first_name: '',
  last_name: '',
  middle_name: '',
  birth_date: '',
  school: '',
  //school_name: '',
  grade: '',
  phone: '',
  email_private: '',
  telegram: '',
  exams: [],
  learn_subjects: [],
  interests: '',
  about_me: '',
  dreams: ''
});

const breadcrumbs = computed(() => {
  const username = effectiveUsername.value;
  const crumbs = [
    {
      label: 'People',
      route: { name: 'People' },
    },
    {
      label: profile.data?.full_name || 'Профиль',
      route: username ? {
        name: 'Profile',
        params: { username },
      } : undefined,
    },
  ];
  console.log('[DEBUG] Хлебные крошки:', crumbs);
  return crumbs;
});

const pageMeta = computed(() => {
  const meta = {
    title: profile.data?.full_name || 'Профиль',
    description: profile.data?.headline || '',
  };
  console.log('[DEBUG] Мета-данные страницы:', meta);
  return meta;
});

const displayName = computed(() => {
  if (!profile.data) {
    console.log('[DEBUG] displayName: profile.data не загружен');
    return 'Загрузка...';
  }
  const name = profile.data?.full_name || `${form.value.first_name || ''} ${form.value.last_name || ''}`.trim();
  console.log('[DEBUG] Отображаемое имя:', name);
  return name;
});

const isSessionUser = () => {
  const sessionUser = $user.data?.username;
  const profileUser = effectiveUsername.value;
  const isSession = sessionUser === profileUser;
  console.log('[DEBUG] Проверка isSessionUser:', { sessionUser, profileUser, isSession });
  return isSession;
};

function formattedDate(d) {
  if (!d) return '';
  try {
    return new Date(d).toLocaleDateString('ru-RU');
  } catch (e) {
    console.error('[DEBUG] Ошибка форматирования даты:', e, { date: d });
    return d;
  }
}

function maskPrivate(val) {
  if (!val) return '-';
  if (val.includes('@')) {
    const parts = val.split('@');
    return parts[0].slice(0, 1) + '***@' + parts[1];
  }
  return val.slice(0, 3) + '***' + val.slice(-2);
}

function formatTelegram(t) {
  if (!t) return '';
  if (t.startsWith('t.me/') || t.startsWith('https://t.me/')) return (t.startsWith('http') ? t : 'https://' + t);
  return 'https://t.me/' + t.replace(/^@/, '');
}

function fillFormFromProfile() {
  console.log('[DEBUG] Заполнение формы:', {
    schoolProfile: schoolProfile.data,
    profile: profile.data,
    currentForm: JSON.stringify(form.value, null, 2),
  });
  form.value.first_name = schoolProfile.data?.first_name || profile.data?.first_name || '';
  form.value.last_name = schoolProfile.data?.last_name || profile.data?.last_name || '';
  form.value.middle_name = schoolProfile.data?.middle_name || '';
  form.value.birth_date = schoolProfile.data?.birth_date || '';
  form.value.school = schoolProfile.data?.school || '';
  form.value.grade = schoolProfile.data?.grade || '';
  form.value.phone = schoolProfile.data?.phone || '';
  form.value.email_private = schoolProfile.data?.email_private || '';
  form.value.telegram = schoolProfile.data?.telegram || '';
  form.value.exams = schoolProfile.data?.exams ? schoolProfile.data.exams : [];
  form.value.learn_subjects = schoolProfile.data?.learn_subjects ? schoolProfile.data.learn_subjects : [];
  form.value.interests = schoolProfile.data?.interests || '';
  form.value.about_me = schoolProfile.data?.about_me || '';
  form.value.dreams = schoolProfile.data?.dreams || '';
  console.log('[DEBUG] Форма после заполнения:', JSON.stringify(form.value, null, 2));
}


function toggleEdit() {
  editMode.value = !editMode.value;
  if (editMode.value) fillFormFromProfile();
  console.log('[DEBUG] Переключение режима редактирования:', { editMode: editMode.value });
}

function validateExams(exams) {
  console.log('[DEBUG] Валидация exams:', { exams, validOptions: examOptions });
  return exams.every(exam => examOptions.includes(exam));
}

function validateLearnSubjects(subjects) {
  console.log('[DEBUG] Валидация learn_subjects:', { subjects, validOptions: learnOptions });
  return subjects.every(subject => learnOptions.includes(subject));
}

async function saveProfile() {
  console.log('[DEBUG] Сохранение профиля:', { form: form.value });
  saving.value = true;
  try {
    // Создаём копию данных формы
    const formData = { ...form.value };
    console.log('[DEBUG] Копия formData:', JSON.stringify(formData, null, 2));

    // Обновление full_name в User, если нужно
    if (formData.first_name || formData.last_name) {
      const fullName = `${formData.first_name || ''} ${formData.last_name || ''}`.trim();
      console.log('[DEBUG] Обновление User.full_name:', { name: profile.data?.name, fullName });
      await createResource({
        url: 'frappe.client.set_value',
        params: {
          doctype: 'User',
          name: profile.data?.name,
          fieldname: 'full_name',
          value: fullName,
        },
      }).submit();
    }

    // Получаем docname
    let docname = '';
    try {
      await schoolProfile.reload();
      console.log('[DEBUG] Schoolprofile:', { schoolProfile });
      docname = schoolProfile?.data?.name;
      console.log('[DEBUG] Выбранное имя документа:', docname);
    } catch (error) {
      console.log('[DEBUG] Ошибка загрузки schoolProfile, продолжаем с profile:', error.message);
    }

    // Формируем payload из копии данных формы
    let payload = {
      doctype: 'Schoolchildren Profile',
      user: profile.data?.name,
      first_name: formData.first_name,
      last_name: formData.last_name,
      middle_name: formData.middle_name,
      birth_date: formData.birth_date,
      school: formData.school || '',
      grade: formData.grade,
      phone: formData.phone,
      email_private: formData.email_private,
      telegram: formData.telegram,
      exams: Array.isArray(formData.exams) ? formData.exams.map(exam => ({ exam_subject: exam })) : [],
      learn_subjects: Array.isArray(formData.learn_subjects) ? formData.learn_subjects.map(subject => ({ learn_subject: subject })) : [],
      interests: formData.interests,
      about_me: formData.about_me,
      dreams: formData.dreams,
      last_updated: new Date().toISOString(),
    };
    console.log('[DEBUG] Сохранение Schoolchildren Profile (payload):', { docname, payload });

    // Сохранение или создание документа
    if (docname) {
      await createResource({
        url: 'frappe.client.save',
        params: { doc: { ...schoolProfile.data, ...payload } },
      }).submit();
    } else {
      await createResource({
        url: 'frappe.client.insert',
        params: { doc: payload },
      }).submit();
    }

    editMode.value = false;
    if (window.frappe && window.frappe.msgprint) window.frappe.msgprint('Профиль сохранён');
    console.log('[DEBUG] Профиль успешно сохранён');
  } catch (e) {
    console.error('[DEBUG] Ошибка при сохранении профиля:', e);
    if (window.frappe && window.frappe.msgprint) window.frappe.msgprint({
      title: 'Ошибка',
      message: (e && e.message) || 'Ошибка при сохранении',
      indicator: 'red',
    });
  } finally {
    saving.value = false;
  }
  await schoolProfile.reload();
}

const schoolQuery = ref('');
const schoolResults = ref([]);

async function searchSchool(q) {
  if (!q) {
    schoolResults.value = [];
    return;
  }
  try {
    console.log('[DEBUG] Поиск школы:', { query: q });
    const res = await createResource({
      url: 'frappe.client.get_list',
      params: {
        doctype: 'Schools',
        fields: ['school', 'address'],
        filters: [['school', 'like', '%' + q + '%']],
        limit_page_length: 20,
      },
    }).submit();
    schoolResults.value = res || [];
    console.log('[DEBUG] Результаты поиска школы:', schoolResults.value);
  } catch (e) {
    schoolResults.value = [];
    console.error('[DEBUG] Ошибка поиска школы:', e);
  }
}

const debouncedSearchSchool = debounce(() => searchSchool(schoolQuery.value), 300);

function selectSchool(s) {
  form.value.school = s.school;
  //form.value.school_name = s.school_name;
  schoolResults.value = [];
  schoolQuery.value = s.school;
  console.log('[DEBUG] Выбрана школа:', { school: s });
  console.log('[DEBUG] Форма после заполнения:', JSON.stringify(form.value, null, 2));
}

onMounted(() => {
  console.log('[DEBUG] Компонент смонтирован:', {
    propsUsername: props.username,
    sessionUsername: $user.data?.username,
    user: user,
    $user: $user.data,
  });
  if ($user.data) {
    console.log('[DEBUG] Запуск profile.reload()');
    profile.reload();
  }
});

watch(
  () => props.username,
  (newUsername, oldUsername) => {
    console.log('[DEBUG] Изменение props.username:', { old: oldUsername, new: newUsername });
    profile.reload();
  }
);

watch(
  () => profile.data,
  (newData, oldData) => {
    console.log('[DEBUG] Изменение profile.data:', { old: oldData, new: newData });
    if (newData) {
      console.log('[DEBUG] Запуск schoolProfile.reload()');
      schoolProfile.reload();
    }
  }
);

watch(
  () => schoolProfile.data,
  (newData, oldData) => {
    console.log('[DEBUG] Изменение schoolProfile.data:', { old: oldData, new: newData });
    if (newData && !editMode.value) {
      console.log('[DEBUG] Заполнение формы из schoolProfile');
      fillFormFromProfile();
    }
  }
);

watch(
  () => effectiveUsername.value,
  (newUsername) => {
    console.log('[DEBUG] Изменение effectiveUsername для schoolProfile:', newUsername);
    schoolProfile.update({
      params: {
        doctype: 'Schoolchildren Profile',
        filters: { user: newUsername },
      },
    });
  }
);
</script>