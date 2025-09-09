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
            <p>Загрузка данных профиля родителя...</p>
          </div>
          <div v-else-if="schoolProfile.error">
            <p class="text-red-500">Ошибка загрузки данных родителя: {{ schoolProfile.error.message }}</p>
          </div>
          <div v-else-if="schoolProfile.data">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <b>Фамилия:</b> {{ schoolProfile.data.last_name || '-' }}<br/>
                <b>Имя:</b> {{ schoolProfile.data.first_name || '-' }}<br/>
                <b>Отчество:</b> {{ schoolProfile.data.middle_name || '-' }}<br/>
                <b>Дата рождения:</b> {{ formattedDate(schoolProfile.data.birth_date) || '-' }}
                <b>Телефон (приватный):</b> {{ maskPrivate(schoolProfile.data.phone) }}<br/>
                <b>Email (приватный):</b> {{ maskPrivate(schoolProfile.data.email_private) }}<br/>
                <b>Telegram:</b>
                <a v-if="schoolProfile.data.telegram" :href="formatTelegram(schoolProfile.data.telegram)" target="_blank">{{ schoolProfile.data.telegram }}</a>
              </div>
              <div>

              </div>
            </div>
          </div>
          <div v-else>
            <p>Данные профиля родителя не найдены.</p>
          </div>
        </div>

        <!-- EDIT MODE -->
        <div v-else class="mt-4">
          <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <Input v-model="form.last_name" label="Фамилия" />
            <Input v-model="form.first_name" label="Имя" />
            <Input v-model="form.middle_name" label="Отчество" />
            <label class="block text-sm font-medium text-ink-gray-7 mb-1">Дата рождения</label>
            <DatePicker v-model="form.birth_date" label="Дата рождения" />
            <Input v-model="form.phone" label="Телефон (не публиковать)" />
            <Input v-model="form.email_private" label="Email (не публиковать)" />
            <Input v-model="form.telegram" label="Telegram (например t.me/username)" />
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
  phone: '',
  email_private: '',
  telegram: '',

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
  form.value.phone = schoolProfile.data?.phone || '';
  form.value.email_private = schoolProfile.data?.email_private || '';
  form.value.telegram = schoolProfile.data?.telegram || '';

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
      phone: formData.phone,
      email_private: formData.email_private,
      telegram: formData.telegram,

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