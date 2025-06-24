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
				<div v-if="chatResponse" v-html="chatResponse"></div>
				<div v-else class="placeholder-text">
					Загрузка ответа...
				</div>
			</div>
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
const chatResponse = ref(null)
const currentQuestionIndex = ref(0) // Индекс текущего вопроса

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

const toggleChatGPT = async () => {
	console.log('[DEBUG] toggleChatGPT вызван, showChat:', showChat.value);
	showChat.value = !showChat.value;
	if (!showChat.value) {
		chatResponse.value = null;
		console.log('[DEBUG] Чат закрыт, chatResponse сброшен');
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
		const quiz_title = quizData.title;
		console.log('[DEBUG] Получен quiz:', quiz, quiz_title);
		if (!quiz || !quiz.questions || quiz.questions.length === 0) {
			chatResponse.value = 'Вопросы не найдены';
			console.log('[DEBUG] Вопросы не найдены в quiz:', quiz);
			return;
		}

		// Используем текущий индекс
		const currentQuestion = quiz.questions[currentQuestionIndex.value];
		console.log('[DEBUG] currentQuestion:', currentQuestion);
		if (!currentQuestion) {
			chatResponse.value = 'Вопрос не найден';
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
			chatResponse.value = 'Данные вопроса не загружены';
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
			prompt = `Вопрос из квиза: "${quiz_title}". Решите задачу: "${question}". Вопрос имеет выбор ответов: "${options.join('", "')}". Правильный ответ: "${correct_answer || 'не указан'}". 
			Дайте пошаговое решение, убедившись, что оно соответствует правильному ответу. Проверьте решение на логические и фактические ошибки.`;
		} else if (questionData.type === "User Input") {
			question = currentQuestion.question_detail;
			possibilitys = [
				questionData.possibility_1,
				questionData.possibility_2,
				questionData.possibility_3,
				questionData.possibility_4,
			].filter(Boolean);
			console.log('[DEBUG] Получен вопрос и возможные варианты:', { question, options });
			prompt = `Вопрос из квиза: "${quiz_title}". Решите задачу: "${question}". Вопрос предполагает пользовательский ввод, есть возможные варианты ответа: 
			"${possibilitys.join('", "')}". 
			Дайте пошаговое решение, убедившись, что оно соответствует возможным вариантам ответа. Проверьте решение на логические и фактические ошибки.`;
		} else {
			question = currentQuestion.question_detail;
			prompt = `Вопрос из квиза: "${quiz_title}". Решите задачу: "${question}". Дайте пошаговое решение. Проверьте решение на логические и фактические ошибки.`;
			console.log('[DEBUG] Получен вопрос:', { question });
		}

		if (!question) {
			chatResponse.value = 'Текст вопроса не найден';
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
				system_prompt: 'Ты — эксперт, решающий задачи для школьников. Дай полное пошаговое решение, проверяя его на соответствие правильному ответу.'
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
		chatResponse.value = data.answer;
	} catch (err) {
		console.error('[DEBUG] Ошибка в toggleChatGPT:', err);
		chatResponse.value = `Ошибка: ${err.message}`;
	}
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
.btn-primary {
	background-color: #2563eb;
	color: white;
	padding: 8px 16px;
	border-radius: 4px;
}
</style>