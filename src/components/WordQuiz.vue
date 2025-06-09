<script setup>
import { ref, computed } from 'vue'

// 状态管理
const words = ref([])
const currentWordIndex = ref(0)
const userAnswer = ref('')
const quizStarted = ref(false)
const quizFinished = ref(false)
const wrongAnswers = ref([])
const shuffledWords = ref([])

// 计算属性
const currentWord = computed(() => {
    return shuffledWords.value[currentWordIndex.value] || {}
})

const progress = computed(() => {
    return shuffledWords.value.length > 0
        ? Math.round((currentWordIndex.value / shuffledWords.value.length) * 100)
        : 0
})

// CSV文件导入
const handleFileUpload = (event) => {
    const file = event.target.files[0]
    if (!file) return

    const reader = new FileReader()
    reader.onload = (e) => {
        const csv = e.target.result
        const lines = csv.split('\n')
        const parsedWords = []

        for (let i = 1; i < lines.length; i++) { // 跳过标题行
            const line = lines[i].trim()
            if (line) {
                const columns = line.split(',')
                if (columns.length >= 3) {
                    parsedWords.push({
                        id: columns[0].trim(),
                        word: columns[1].trim(),
                        meaning: columns[2].trim()
                    })
                }
            }
        }

        words.value = parsedWords
        alert(`成功导入 ${parsedWords.length} 个单词！`)
    }
    reader.readAsText(file)
}

// 导出错题为CSV
const exportWrongAnswers = () => {
    if (wrongAnswers.value.length === 0) {
        alert('没有错题可以导出！')
        return
    }

    // 构建CSV内容
    const csvContent = [
        // CSV标题行
        '序号,单词,释义',
        // 错题数据行
        ...wrongAnswers.value.map((item, index) =>
            `${index + 1},${item.word},${item.meaning}`
        )
    ].join('\n')

    // 创建Blob对象
    const blob = new Blob(['\ufeff' + csvContent], { type: 'text/csv;charset=utf-8;' })

    // 创建下载链接
    const link = document.createElement('a')
    const url = URL.createObjectURL(blob)
    link.setAttribute('href', url)

    // 生成文件名（包含时间戳）
    const now = new Date()
    const timestamp = now.toISOString().slice(0, 19).replace(/[:-]/g, '').replace('T', '_')
    link.setAttribute('download', `错题集_${timestamp}.csv`)

    // 触发下载
    link.style.visibility = 'hidden'
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)

    // 释放URL对象
    URL.revokeObjectURL(url)

    alert(`已导出 ${wrongAnswers.value.length} 个错题到CSV文件！`)
}

// 随机排序并开始测验
const startQuiz = () => {
    if (words.value.length === 0) {
        alert('请先导入单词库！')
        return
    }

    // 随机排序
    shuffledWords.value = [...words.value].sort(() => Math.random() - 0.5)

    // 重置状态
    currentWordIndex.value = 0
    userAnswer.value = ''
    wrongAnswers.value = []
    quizStarted.value = true
    quizFinished.value = false
}

// 提交答案
const submitAnswer = () => {
    const correct = userAnswer.value.trim().toLowerCase() === currentWord.value.word.toLowerCase()

    if (!correct) {
        wrongAnswers.value.push({
            ...currentWord.value,
            userAnswer: userAnswer.value.trim()
        })
    }

    // 清空输入并移动到下一个单词
    userAnswer.value = ''
    currentWordIndex.value++

    // 检查是否完成测验
    if (currentWordIndex.value >= shuffledWords.value.length) {
        quizFinished.value = true
        quizStarted.value = false
    }
}

// 重新开始测验
const restartQuiz = () => {
    quizStarted.value = false
    quizFinished.value = false
    currentWordIndex.value = 0
    userAnswer.value = ''
    wrongAnswers.value = []
}

// 键盘事件处理
const handleKeyPress = (event) => {
    if (event.key === 'Enter') {
        submitAnswer()
    }
}
</script>

<template>
    <div class="quiz-container">
        <!-- 文件导入阶段 -->
        <div v-if="!quizStarted && !quizFinished" class="import-section">
            <div class="card">
                <h2>导入单词库</h2>
                <p>请选择CSV文件（格式：序号,单词,释义）</p>
                <input type="file" accept=".csv" @change="handleFileUpload" class="file-input" />

                <div v-if="words.length > 0" class="word-count">
                    <p>已导入 {{ words.length }} 个单词</p>
                    <button @click="startQuiz" class="start-btn">开始测验</button>
                </div>
            </div>
        </div>

        <!-- 测验进行阶段 -->
        <div v-if="quizStarted" class="quiz-section">
            <div class="progress-bar">
                <div class="progress" :style="{ width: progress + '%' }"></div>
            </div>

            <div class="quiz-info">
                <span>{{ currentWordIndex + 1 }} / {{ shuffledWords.length }}</span>
                <span>进度: {{ progress }}%</span>
            </div>

            <div class="quiz-card">
                <h2>请写出以下释义对应的单词：</h2>
                <div class="meaning">{{ currentWord.meaning }}</div>

                <input v-model="userAnswer" @keypress="handleKeyPress" placeholder="输入单词并按回车提交" class="answer-input"
                    autofocus />

                <button @click="submitAnswer" class="submit-btn">提交答案</button>
            </div>
        </div>

        <!-- 结果显示阶段 -->
        <div v-if="quizFinished" class="result-section">
            <div class="result-card">
                <h2>测验完成！</h2>

                <div class="score-summary">
                    <div class="score-item">
                        <span class="label">总题数：</span>
                        <span class="value">{{ shuffledWords.length }}</span>
                    </div>
                    <div class="score-item">
                        <span class="label">正确：</span>
                        <span class="value correct">{{ shuffledWords.length - wrongAnswers.length }}</span>
                    </div>
                    <div class="score-item">
                        <span class="label">错误：</span>
                        <span class="value wrong">{{ wrongAnswers.length }}</span>
                    </div>
                    <div class="score-item">
                        <span class="label">准确率：</span>
                        <span class="value">{{ Math.round(((shuffledWords.length - wrongAnswers.length) /
                            shuffledWords.length) * 100) }}%</span>
                    </div>
                </div>

                <div v-if="wrongAnswers.length > 0" class="wrong-answers">
                    <div class="wrong-answers-header">
                        <h3>错误单词列表：</h3>
                        <button @click="exportWrongAnswers" class="export-btn">
                            📄 导出错题CSV
                        </button>
                    </div>

                    <div class="wrong-list">
                        <div v-for="item in wrongAnswers" :key="item.id" class="wrong-item">
                            <div class="wrong-meaning">{{ item.meaning }}</div>
                            <div class="wrong-details">
                                <span class="your-answer">你的答案: {{ item.userAnswer || '(空)' }}</span>
                                <span class="correct-answer">正确答案: {{ item.word }}</span>
                            </div>
                        </div>
                    </div>
                </div>

                <div v-else class="perfect-score">
                    <h3>🎉 完美答题！所有单词都答对了！</h3>
                </div>

                <div class="result-actions">
                    <button @click="restartQuiz" class="restart-btn">重新开始</button>
                    <button @click="startQuiz" class="retry-btn">再次测验</button>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.quiz-container {
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem;
}

.card,
.quiz-card,
.result-card {
    background: var(--color-background-soft);
    border: 1px solid var(--color-border);
    border-radius: 12px;
    padding: 2rem;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.import-section h2,
.quiz-card h2,
.result-card h2 {
    color: var(--color-heading);
    margin-bottom: 1rem;
    text-align: center;
}

.file-input {
    width: 100%;
    padding: 0.75rem;
    border: 2px dashed var(--color-border);
    border-radius: 8px;
    background: var(--color-background);
    cursor: pointer;
    margin: 1rem 0;
}

.file-input:hover {
    border-color: var(--color-border-hover);
}

.word-count {
    text-align: center;
    margin-top: 1rem;
}

.start-btn,
.submit-btn,
.restart-btn,
.retry-btn {
    background: hsla(160, 100%, 37%, 1);
    color: white;
    border: none;
    padding: 0.75rem 1.5rem;
    border-radius: 8px;
    cursor: pointer;
    font-size: 1rem;
    margin: 0.5rem;
    transition: background 0.3s ease;
}

.start-btn:hover,
.submit-btn:hover,
.restart-btn:hover,
.retry-btn:hover {
    background: hsla(160, 100%, 30%, 1);
}

.export-btn {
    background: #f59e0b;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.9rem;
    transition: background 0.3s ease;
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.export-btn:hover {
    background: #d97706;
}

.progress-bar {
    width: 100%;
    height: 8px;
    background: var(--color-background-mute);
    border-radius: 4px;
    margin-bottom: 1rem;
    overflow: hidden;
}

.progress {
    height: 100%;
    background: hsla(160, 100%, 37%, 1);
    transition: width 0.3s ease;
}

.quiz-info {
    display: flex;
    justify-content: space-between;
    margin-bottom: 1rem;
    color: var(--color-text);
}

.meaning {
    font-size: 1.5rem;
    color: var(--color-heading);
    text-align: center;
    margin: 2rem 0;
    padding: 1rem;
    background: var(--color-background);
    border-radius: 8px;
    border: 1px solid var(--color-border);
}

.answer-input {
    width: 100%;
    padding: 1rem;
    border: 2px solid var(--color-border);
    border-radius: 8px;
    font-size: 1.1rem;
    margin: 1rem 0;
    background: var(--color-background);
    color: var(--color-text);
}

.answer-input:focus {
    outline: none;
    border-color: hsla(160, 100%, 37%, 1);
}

.score-summary {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    gap: 1rem;
    margin: 1.5rem 0;
}

.score-item {
    display: flex;
    justify-content: space-between;
    padding: 0.75rem;
    background: var(--color-background);
    border-radius: 8px;
    border: 1px solid var(--color-border);
}

.label {
    font-weight: 500;
}

.value.correct {
    color: #22c55e;
    font-weight: bold;
}

.value.wrong {
    color: #ef4444;
    font-weight: bold;
}

.wrong-answers {
    margin-top: 2rem;
}

.wrong-answers-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
    flex-wrap: wrap;
    gap: 1rem;
}

.wrong-answers h3 {
    color: var(--color-heading);
    margin: 0;
}

.wrong-list {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.wrong-item {
    padding: 1rem;
    background: var(--color-background);
    border-radius: 8px;
    border-left: 4px solid #ef4444;
}

.wrong-meaning {
    font-weight: 500;
    margin-bottom: 0.5rem;
    color: var(--color-heading);
}

.wrong-details {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    font-size: 0.9rem;
}

.your-answer {
    color: #ef4444;
}

.correct-answer {
    color: #22c55e;
    font-weight: 500;
}

.perfect-score {
    text-align: center;
    color: #22c55e;
    margin: 2rem 0;
}

.result-actions {
    display: flex;
    justify-content: center;
    gap: 1rem;
    margin-top: 2rem;
}

.restart-btn {
    background: #6b7280;
}

.restart-btn:hover {
    background: #4b5563;
}

@media (max-width: 768px) {
    .quiz-container {
        padding: 1rem;
    }

    .card,
    .quiz-card,
    .result-card {
        padding: 1.5rem;
    }

    .meaning {
        font-size: 1.2rem;
    }

    .result-actions {
        flex-direction: column;
    }

    .wrong-details {
        font-size: 0.8rem;
    }

    .wrong-answers-header {
        flex-direction: column;
        align-items: flex-start;
    }
}
</style>