<template>
  <Modal
    :isOpen="isOpen"
    @close="$emit('close')"
    size="large"
    title="Seu progresso"
  >
    <template #default>
      <div class="progress-content">

        <!-- Estatísticas -->
        <div v-if="presentedQuestions.length > 0" class="progress-stats">
          <div class="stat-card correct">
            <div class="stat-value">{{ correctCount }}</div>
            <div class="stat-label">Corretas</div>
          </div>

          <div class="stat-card incorrect">
            <div class="stat-value">{{ incorrectCount }}</div>
            <div class="stat-label">Incorretas</div>
          </div>

          <div class="stat-card">
            <div class="stat-value">{{ accuracy }}%</div>
            <div class="stat-label">Precisão</div>
          </div>

          <div class="stat-card answered">
            <div class="stat-value">{{ answeredCount }}</div>
            <div class="stat-label">Respondidas</div>
          </div>
        </div>

        <!-- Histórico das perguntas -->
        <div v-if="presentedQuestions.length > 0" class="question-history">
          <h4>Histórico de questões</h4>

          <div
            v-for="(q, idx) in presentedQuestions"
            :key="q.index"
            class="history-item"
            :class="getQuestionStatusClass(q)"
          >

            <div class="history-content">

              <div class="history-question">
                Pergunta {{ q.index + 1 }}. {{ q.text }}
              </div>

              <div v-if="q.imageUrl" class="history-image">
                <img :src="q.imageUrl" alt="Imagem da pergunta" @error="handleImageError" />
              </div>

              <div class="history-answer">
                <strong class="answer-label">Sua resposta:</strong>

                {{
                  q.playerChoice !== null
                    ? `${String.fromCharCode(65 + q.playerChoice)}. ${q.choices[q.playerChoice]}`
                    : 'Nenhuma resposta enviada'
                }}
              </div>

              <div v-if="q.revealed" class="history-correct">
                <strong class="correct-label">Resposta correta:</strong>

                {{
                  `${String.fromCharCode(65 + q.correctChoice)}. ${q.choices[q.correctChoice]}`
                }}
              </div>

            </div>

            <div class="history-status" :class="getQuestionStatusClass(q)">
              <AppIcon :name="getQuestionStatusIcon(q)" size="xl" class="status-icon" />
              <div class="status-text">{{ getQuestionStatusText(q) }}</div>
            </div>

          </div>
        </div>

        <!-- Estado vazio -->
        <div v-else class="progress-empty">
          <AppIcon name="clipboard-list" size="2xl" class="empty-icon" />
          <p>Ainda não há perguntas respondidas.</p>
          <p class="empty-hint">
            Seu progresso aparecerá aqui quando o apresentador iniciar o quiz.
          </p>
        </div>

      </div>
    </template>
  </Modal>
</template>

<script setup>
import { computed } from 'vue'
import Modal from '@/components/common/Modal.vue'
import AppIcon from '@/components/common/AppIcon.vue'

const props = defineProps({
  isOpen: { type: Boolean, required: true },
  questionHistory: { type: Array, required: true }
})

defineEmits(['close'])

const presentedQuestions = computed(() =>
  props.questionHistory.filter(q => q.presented)
)

const correctCount = computed(() =>
  presentedQuestions.value.filter(q => q.revealed && q.isCorrect).length
)

const incorrectCount = computed(() =>
  presentedQuestions.value.filter(q => q.revealed && !q.isCorrect).length
)

const answeredCount = computed(() =>
  presentedQuestions.value.filter(q => q.revealed).length
)

const accuracy = computed(() =>
  answeredCount.value > 0
    ? ((correctCount.value / answeredCount.value) * 100).toFixed(1)
    : '0.0'
)

const getQuestionStatusClass = (q) => {
  if (q.missedWhileAway && q.revealed) return 'missed-away'
  if (q.revealed && q.isCorrect) return 'correct'
  if (q.revealed && !q.isCorrect) return 'incorrect'
  if (q.playerChoice !== null) return 'answered'
  return 'unanswered'
}

const getQuestionStatusIcon = (q) => {
  if (q.missedWhileAway && q.revealed) return 'alert-triangle'
  if (q.revealed && q.isCorrect) return 'check'
  if (q.revealed && !q.isCorrect) return 'x'
  if (q.playerChoice !== null) return 'clock'
  return 'circle'
}

const getQuestionStatusText = (q) => {
  if (q.missedWhileAway && q.revealed) return 'Não respondeu (ausente)'
  if (q.revealed && q.isCorrect) return 'Correta'
  if (q.revealed && !q.isCorrect) return 'Incorreta'
  if (q.playerChoice !== null) return 'Aguardando resultado'
  return 'Não respondida'
}

const handleImageError = (event) => {
  event.target.style.display = 'none'
}
</script>

<style scoped>
.progress-content {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.progress-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1rem;
}

.stat-card {
  background: var(--bg-overlay-10);
  border: 1px solid var(--border-color);
  border-radius: 10px;
  padding: 1rem;
  text-align: center;
}

.stat-card.correct {
  background: var(--secondary-bg-10);
  border-color: var(--secondary-light);
}

.stat-card.incorrect {
  background: var(--danger-bg-10);
  border-color: var(--danger-light);
}

.stat-card.answered {
  background: var(--warning-bg-10);
  border-color: var(--warning-light);
}

.stat-value {
  font-size: 2rem;
  font-weight: bold;
  color: var(--info-light);
  margin-bottom: 0.5rem;
}

.stat-card.correct .stat-value {
  color: var(--secondary-light);
}

.stat-card.incorrect .stat-value {
  color: var(--danger-light);
}

.stat-card.answered .stat-value {
  color: var(--warning-light);
}

.stat-label {
  font-size: 0.9rem;
  color: var(--text-tertiary);
}

.question-history h4 {
  margin: 0 0 1rem 0;
  color: var(--text-tertiary);
  font-size: 1.1rem;
  border-bottom: 1px solid var(--border-color);
  padding-bottom: 0.5rem;
}

.history-item {
  background: var(--bg-overlay-10);
  border: 1px solid var(--border-color);
  border-radius: 10px;
  padding: 1rem;
  display: flex;
  justify-content: space-between;
  gap: 1rem;
}

.history-content {
  flex: 1;
  text-align: left;
}

.history-question {
  font-weight: bold;
  color: var(--text-primary);
  margin-bottom: 0.5rem;
}

.history-answer,
.history-correct {
  font-size: 0.85rem;
  color: var(--text-secondary);
  margin-top: 0.3rem;
}

.history-answer .answer-label {
  color: var(--info-light);
}

.history-correct .correct-label {
  color: var(--secondary-light);
}

.history-status {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.status-text {
  font-size: 0.75rem;
  color: var(--text-tertiary);
  text-align: center;
}

.progress-empty {
  text-align: center;
  padding: 3rem 1rem;
  color: var(--text-tertiary);
}

.empty-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}
</style>
