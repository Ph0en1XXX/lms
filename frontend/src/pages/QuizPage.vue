<template>
	<header
		v-if="!fromLesson"
		class="sticky top-0 z-10 flex items-center justify-between border-b bg-surface-white px-3 py-2.5 sm:px-5"
	>
		<Breadcrumbs :items="breadcrumbs" />
	</header>
	<div
		class="md:w-7/12 md:mx-auto mx-4 py-10"
		:class="{ 'pt-4 md:w-full': fromLesson }"
	>
		<Quiz :quizName="quizID" @current-question="handleCurrentQuestion" />
	</div>
	<div>
		<button @click="toggleChatGPT" class="btn btn-primary">Решить с ChatGPT</button>
	</div>
	<div v-if="showChat" class="chat-container mt-4">
		<h2>AI-тьютор</h2>
		<div class="chat-window">
			<div id="chat-box" class="chat-box">
				<div v-for="(message, index) in chatHistory" :key="index" class="message">
					<span :class="{ 'user-message': message.isUser, 'ai-message': !message.isUser }">
						{{ message.text }}
					</span>
				</div>
				<div v-if="chatHistory.length === 0" class="placeholder-text">Загрузка ответа...</div>
			</div>
		</div>
		<div class="chat-input mt-4">
			<input
				v-model="userInput"
				type="text"
				placeholder="Введите ваше сообщение..."
				class="w-full p-2 border rounded-md"
				@keyup.enter="sendMessage"
			/>
			<button @click="sendMessage" class="btn btn-primary ml-2">Отправить</button>
		</div>
	</div>
</template>

<script setup>
import Quiz from '@/components/Quiz.vue'
import { createResource, Breadcrumbs } from 'frappe-ui'
import { computed, inject, onMounted, ref } from 'vue'
import { useRouter } from 'vue-router'
import { updateDocumentTitle } from '@/utils'

const user = inject('$user')
const router = useRouter()
const fromLesson = ref(false)
const showChat = ref(false)
const chatResponse = ref(null) // Временная переменная для текущего ответа
const currentQuestionIndex = ref(0) // Индекс текущего вопроса
const userInput = ref('') // Поле для ввода сообщения
const chatHistory = ref([]) // История сообщений

const props = defineProps({
	quizID: {
		type: String,
		required: true,
	},
})

const quizData = createResource({
	url: 'frappe.client.get',
	params: {
		doctype: 'LMS Quiz',
		name: props.quizID,
	},
	auto: true,
	onSuccess: (data) => {
		console.log('[DEBUG] quizData onSuccess:', data)
	},
	onError: (err) => {
		console.error('[DEBUG] quizData onError:', err)
	},
})

const title = createResource({
	url: 'frappe.client.get_value',
	params: {
		doctype: 'LMS Quiz',
		fieldname: 'title',
		filters: {
			name: props.quizID,
		},
	},
	auto: true,
})

const breadcrumbs = computed(() => {
	return [{ label: __('Quiz Submission') }, { label: title.data?.title }]
})

const pageMeta = computed(() => {
	return {
		title: title.data?.title,
		description: __('Quiz Submission'),
	}
})

const handleCurrentQuestion = (index) => {
	currentQuestionIndex.value = index
	console.log('[DEBUG] Текущий индекс вопроса из Quiz.vue:', index)
}

const sendMessage = async () => {
	if (!userInput.value.trim()) return;

	// Добавляем сообщение пользователя в историю
	chatHistory.value.push({ text: userInput.value, isUser: true });
	console.log('[DEBUG] Пользовательское сообщение:', userInput.value);

	// Сбрасываем поле ввода
	const userMessage = userInput.value;
	userInput.value = '';

	try {
		console.log('[DEBUG] Отправка запроса к прокси с пользовательским сообщением...');
		const res = await fetch('https://openai.enlightrussia.ru/chat', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				'X-API-KEY': 'a38ed6248c82e31f014b7459479d4c75154e41a331f8a4d9b4afc4b10cbc884a'
			},
			body: JSON.stringify({
				prompt: generatePrompt(userMessage),
				model: 'gpt-4o',
				system_prompt: `Ты — опытный, доброжелательный и очень терпеливый учитель. Твоя задача — сопровождать ученика в самостоятельном разборе задачи, не раскрывая ему готовое решение и не выдавая правильного ответа напрямую. 
Ни при каких условиях нельзя выдавать готовый ответ. Ученик должен сам дойти до него. Обязательно нужно разбирать процесс решений по шагам, задавая ученику по одному вопросу за один шаг и ожидая от него ответа. 
Говори и отвечай на русском языке, простыми и понятными формулировками. Будь вежлив, отзывчив и дружелюбен. Показывай искреннее желание помочь. Подбадривай ученика, если он ошибается или затрудняется.`
			})
		});

		console.log('[DEBUG] Ответ от прокси:', res.status, res.statusText);
		if (!res.ok) {
			const errorText = await res.text();
			console.log('[DEBUG] Ошибка ответа от прокси:', errorText);
			throw new Error(`Ошибка API: ${res.status} ${errorText}`);
		}

		const data = await res.json();
		console.log('[DEBUG] Получен ответ от GPT:', data);
		chatHistory.value.push({ text: data.answer, isUser: false });
	} catch (err) {
		console.error('[DEBUG] Ошибка в sendMessage:', err);
		chatHistory.value.push({ text: `Ошибка: ${err.message}`, isUser: false });
	}
};

const toggleChatGPT = async () => {
	console.log('[DEBUG] toggleChatGPT вызван, showChat:', showChat.value);
	showChat.value = !showChat.value;
	if (!showChat.value) {
		chatResponse.value = null;
		chatHistory.value = [];
		console.log('[DEBUG] Чат закрыт, chatResponse и история сброшены');
		return;
	}

	try {
		console.log('[DEBUG] Начало обработки, quizData:', quizData);
		if (quizData.loading) {
			chatHistory.value.push({ text: 'Загрузка данных квиза...', isUser: false });
			console.log('[DEBUG] Данные квиза загружаются');
			return;
		}

		if (quizData.error) {
			throw new Error(`Ошибка загрузки квиза: ${quizData.error.message}`);
		}

		const quiz = quizData.data;
		if (!quiz) {
			chatHistory.value.push({ text: 'Данные квиза не загружены', isUser: false });
			console.log('[DEBUG] quizData.data отсутствует');
			return;
		}
		const quiz_title = quiz.title || 'Без названия';
		console.log('[DEBUG] Получен quiz:', quiz, quiz_title);
		if (!quiz.questions || quiz.questions.length === 0) {
			chatHistory.value.push({ text: 'Вопросы не найдены', isUser: false });
			console.log('[DEBUG] Вопросы не найдены в quiz:', quiz);
			return;
		}

		// Используем текущий индекс
		const currentQuestion = quiz.questions[currentQuestionIndex.value];
		if (!currentQuestion?.question_detail) {
			chatHistory.value.push({ text: 'Детали вопроса не найдены', isUser: false });
			console.log('[DEBUG] Текущий вопрос отсутствует:', currentQuestion);
			return;
		}

		// Создаём и ждём загрузки данных вопроса
		const questionData = await new Promise((resolve) => {
			const resource = createResource({
				url: 'frappe.client.get',
				params: {
					doctype: 'LMS Question',
					name: currentQuestion.question,
				},
				auto: true,
				onSuccess: (data) => {
					console.log('[DEBUG] questionData onSuccess:', data);
					resolve(data);
				},
				onError: (err) => {
					console.error('[DEBUG] questionData onError:', err);
					resolve(null);
				},
			});
		});

		if (!questionData) {
			chatHistory.value.push({ text: 'Данные вопроса не загружены', isUser: false });
			console.log('[DEBUG] Данные вопроса отсутствуют');
			return;
		}

		let questionDataPrompt, question, prompt, correct_answer, options = [], possibilitys = [];

		if (questionData.type === "Choices") {
			question = currentQuestion.question_detail;
			options = [
				questionData.option_1 || '',
				questionData.option_2 || '',
				questionData.option_3 || '',
				questionData.option_4 || '',
			].filter(Boolean);
			correct_answer = null;
			for (let i = 1; i <= 4; i++) {
				if (questionData[`is_correct_${i}`] === 1) {
					correct_answer = questionData[`option_${i}`];
					console.log(`Правильный ответ ${i}:`, correct_answer);
					break;
				}
			}
			console.log('[DEBUG] Получен вопрос, варианты и ответ:', { question, options, correct_answer });
			questionDataPrompt = `Это вопрос типа: ${questionData.type}. ${question}\nВарианты ответа: ${options.join(', ')}\nПравильный ответ: ${correct_answer}`;
			console.log('[DEBUG] Данные для промта:', questionDataPrompt)
			prompt = generatePrompt(questionDataPrompt);
		} else if (questionData.type === "User Input") {
			question = currentQuestion.question_detail;
			possibilitys = [
				questionData.possibility_1 || '',
				questionData.possibility_2 || '',
				questionData.possibility_3 || '',
				questionData.possibility_4 || '',
			].filter(Boolean);
			console.log('[DEBUG] Получен вопрос и возможные варианты:', { question, possibilitys });
			questionDataPrompt = `Это вопрос типа: ${questionData.type}. ${question}\nВозможные варианты ответа: ${possibilitys.join(', ')}`;
			console.log('[DEBUG] Данные для промта:', questionDataPrompt)
			prompt = generatePrompt(questionDataPrompt);
		} else {
			question = currentQuestion.question_detail;
			console.log('[DEBUG] Получен вопрос:', { question });
			questionDataPrompt = `Это вопрос типа: ${questionData.type}. ${question}`;
			console.log('[DEBUG] Данные для промта:', questionDataPrompt)
			prompt = generatePrompt(questionDataPrompt);
		}

		if (!question) {
			chatHistory.value.push({ text: 'Текст вопроса не найден', isUser: false });
			console.log('[DEBUG] Текст вопроса отсутствует в currentQuestion:', currentQuestion);
			return;
		}

		console.log('[DEBUG] Сформирован промпт:', prompt);
		console.log('[DEBUG] Отправка запроса к прокси...');
		const res = await fetch('https://openai.enlightrussia.ru/chat', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				'X-API-KEY': 'a38ed6248c82e31f014b7459479d4c75154e41a331f8a4d9b4afc4b10cbc884a'
			},
			body: JSON.stringify({
				prompt: prompt,
				model: 'gpt-4o',
				system_prompt: `1. Роль
Ты — опытный, доброжелательный и очень терпеливый учитель. Твоя задача — сопровождать ученика в самостоятельном разборе задачи, не раскрывая ему готовое решение и не выдавая правильного ответа напрямую. 
Ни при каких условиях нельзя выдавать готовый ответ. Ученик должен сам дойти до него. Обязательно нужно разбирать процесс решений по шагам, задавая ученику по одному вопросу за один шаг и ожидая от него ответа. 
2. Стиль общения
Говори и отвечай на русском языке, простыми и понятными формулировками.
Будь вежлив, отзывчив и дружелюбен.
Показывай искреннее желание помочь.
Подбадривай ученика, если он ошибается или затрудняется.
3. Пошаговый подход
Начинай с запроса к ученику: попроси его показать или описать задание. Если возможно, пусть он загрузит скриншот, документ или текст задания.
Не говори финальный ответ сразу. Вместо этого:
1. Спроси, что ученик уже знает или какие идеи у него есть.
2. Дай наводящие вопросы, чтобы понять, в каком месте он испытывает затруднение.
3. Подсказывай принципы или формулы, которые могут помочь, но не окончательный результат.
4. Делай это пошагово, пока ученик не найдёт решение самостоятельно (или не выскажет версию, близкую к правильной).
При необходимости:
Повтори ключевые определения и формулы.
Предложи разобрать аналогичный, но более простой пример. Обязательно нужно разбирать процесс решений по шагам, задавая ученику по одному вопросу за один шаг и ожидая от него ответа. Ни в коем случае не давай сразу решение.`
			})
		});

		console.log('[DEBUG] Ответ от прокси:', res.status, res.statusText);
		if (!res.ok) {
			const errorText = await res.text();
			console.log('[DEBUG] Ошибка ответа от прокси:', errorText);
			throw new Error(`Ошибка API: ${res.status} ${errorText}`);
		}

		const data = await res.json();
		console.log('[DEBUG] Получен ответ от GPT:', data);
		chatHistory.value.push({ text: data.answer, isUser: false });
		chatResponse.value = null; // Сбрасываем chatResponse после добавления в историю
	} catch (err) {
		console.error('[DEBUG] Ошибка в toggleChatGPT:', err);
		chatHistory.value.push({ text: `Ошибка: ${err.message}`, isUser: false });
		chatResponse.value = null;
	}
};

// Функция для генерации промпта с историей
const generatePrompt = (userMsg) => {
	let prompt = `Текущая задача: "${userMsg}".\n`;
	prompt += 'История диалога:\n';
	chatHistory.value.forEach(msg => {
		prompt += `${msg.isUser ? 'Ученик' : 'Учитель'}: ${msg.text}\n`;
	});
	prompt += `\nНовое сообщение от ученика: ${userMsg}`;
	return prompt;
};

updateDocumentTitle(pageMeta)
</script>

<style scoped>
.chat-container {
	max-width: 700px;
	margin: 0 auto;
	padding: 20px;
	border: 1px solid #e5e7eb;
	border-radius: 8px;
	background: #fff;
}
.chat-window {
	max-height: 400px;
	overflow-y: auto;
	padding: 10px;
	border: 1px solid #e5e7eb;
	border-radius: 4px;
}
.chat-box {
	min-height: 100px;
}
.placeholder-text {
	color: #94a3b8;
	text-align: center;
}
.message {
	margin: 5px 0;
}
.user-message {
	background-color: #e3f2fd;
	padding: 5px 10px;
	border-radius: 5px;
	display: inline-block;
}
.ai-message {
	background-color: #f0f0f0;
	padding: 5px 10px;
	border-radius: 5px;
	display: inline-block;
}
.chat-input input {
	border: 1px solid #e5e7eb;
}
.btn-primary {
	background-color: #2563eb;
	color: white;
	padding: 8px 16px;
	border-radius: 4px;
}
</style>