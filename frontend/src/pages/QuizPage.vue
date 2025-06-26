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
		<button @click="toggleChatGPT" class="btn btn-primary">Решить с ChatGPT (режим "solve-together")</button>
	</div>
	<div v-if="showChat" class="chat-container mt-4">
		<h2>AI-тьютор (режим "solve-together")</h2>
		<div class="chat-window">
			<div id="chat-box" class="chat-box">
				<div v-for="(message, index) in chatHistory" :key="index" class="message">
					<span :class="{ 'user-message': message.isUser, 'ai-message': !message.isUser }">
						{{ message.text }}
					</span>
				</div>
				<div v-if="chatHistory.length === 0" class="placeholder-text">Ожидаю вашего первого шага...</div>
			</div>
		</div>
		<div class="chat-input mt-4">
			<input
				v-model="userInput"
				type="text"
				placeholder="Опишите задачу или ответьте на вопрос..."
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
				system_prompt: `Ты — опытный, доброжелательный и очень терпеливый учитель. Твоя задача — сопровождать ученика в самостоятельном разборе заданий (домашних заданий, ЕГЭ или олимпиадных), не раскрывая ему готовое решение и не выдавая правильного ответа напрямую. 
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
			chatResponse.value = 'Загрузка данных квиза...';
			console.log('[DEBUG] Данные квиза загружаются');
			return;
		}

		if (quizData.error) {
			throw new Error(`Ошибка загрузки квиза: ${quizData.error.message}`);
		}

		const quiz = quizData.data;
		const quiz_title = quiz.title;
		console.log('[DEBUG] Получен quiz:', quiz, quiz_title);
		if (!quiz || !quiz.questions || quiz.questions.length === 0) {
			chatHistory.value.push({ text: 'Вопросы не найдены', isUser: false });
			console.log('[DEBUG] Вопросы не найдены в quiz:', quiz);
			return;
		}

		// Используем текущий индекс
		const currentQuestion = quiz.questions[currentQuestionIndex.value];
		console.log('[DEBUG] currentQuestion:', currentQuestion);
		if (!currentQuestion) {
			chatHistory.value.push({ text: 'Вопрос не найден', isUser: false });
			console.log('[DEBUG] Текущий вопрос отсутствует:', quiz.questions);
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

		let question, prompt, correct_answer, options = [], possibilitys = [];

		if (questionData.type === "Choices") {
			question = currentQuestion.question_detail;
			options = [
				questionData.option_1,
				questionData.option_2,
				questionData.option_3,
				questionData.option_4,
			].filter(Boolean); // Убираем undefined/null
			correct_answer = null;
			for (let i = 1; i <= 4; i++) {
				if (questionData[`is_correct_${i}`] === 1) {
					correct_answer = questionData[`option_${i}`];
					console.log(`Правильный ответ ${i}:`, correct_answer);
					break;
				}
			}
			console.log('[DEBUG] Получен вопрос, варианты и ответ:', { question, options, correct_answer });
			prompt = generatePrompt(question, { type: "Choices", options, correct_answer });
		} else if (questionData.type === "User Input") {
			question = currentQuestion.question_detail;
			possibilitys = [
				questionData.possibility_1,
				questionData.possibility_2,
				questionData.possibility_3,
				questionData.possibility_4,
			].filter(Boolean);
			console.log('[DEBUG] Получен вопрос и возможные варианты:', { question, possibilitys });
			prompt = generatePrompt(question, { type: "User Input", possibilitys });
		} else {
			question = currentQuestion.question_detail;
			console.log('[DEBUG] Получен вопрос:', { question });
			prompt = generatePrompt(question, { type: "Other" });
		}

		if (!question) {
			chatHistory.value.push({ text: 'Текст вопроса не найден', isUser: false });
			console.log('[DEBUG] Текст вопроса отсутствует в currentQuestion:', currentQuestion);
			return;
		}

		// Генерируем первый наводящий вопрос
		console.log('[DEBUG] Сформирован начальный промпт:', prompt);
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
				system_prompt: `Ты — опытный, доброжелательный и очень терпеливый учитель. Твоя задача — сопровождать ученика в самостоятельном разборе заданий (домашних заданий, ЕГЭ или олимпиадных), не раскрывая ему готовое решение и не выдавая правильного ответа напрямую. 
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
		console.error('[DEBUG] Ошибка в toggleChatGPT:', err);
		chatHistory.value.push({ text: `Ошибка: ${err.message}`, isUser: false });
	}
};

// Функция для генерации промпта с историей и контекстом задачи
const generatePrompt = (userMsg, taskContext) => {
	let sys = `Текущая задача: "${userMsg}".\n`;
	if (taskContext.type === "Choices") {
		sys += `Тип задачи: Выбор ответа. Варианты ответа: "${taskContext.options.join('", "')}". Правильный ответ известен учителю, но не должен раскрываться.`;
	} else if (taskContext.type === "User Input") {
		sys += `Тип задачи: Ввод ответа. Возможные варианты ответа: "${taskContext.possibilitys.join('", "')}".`;
	} else {
		sys += `Тип задачи: Произвольный ввод.`;
	}
	sys += '\nИстория диалога:\n';
	chatHistory.value.forEach(msg => {
		sys += `${msg.isUser ? 'Ученик' : 'Учитель'}: ${msg.text}\n`;
	});
	sys += `\nНовое сообщение от ученика: ${userMsg}`;
	return sys;
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