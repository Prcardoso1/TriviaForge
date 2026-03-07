<template>
  <div class="waiting-display">
    <div class="brand-area">
      <img
        :src="unimarLogo"
        alt="Universidade de Marília"
        class="unimar-logo"
      />
      <h1 class="brand-title">UNIMAR Quiz</h1>
    </div>

    <h2 class="waiting-title">
      {{ inRoom ? 'Aguardando a próxima pergunta' : 'Entre em uma sala para começar' }}
    </h2>

    <p class="waiting-message">
      {{ inRoom
        ? 'Aguarde o apresentador iniciar a atividade.'
        : 'Digite seu nome e o código da sala para participar do quiz.' }}
    </p>

    <!-- Link de prática individual -->
    <div v-if="!inRoom" class="solo-practice-section">
      <p class="solo-hint">ou, se preferir, pratique individualmente</p>

      <RouterLink to="/solo" class="solo-practice-btn">
        <span class="solo-icon">🎯</span>
        <span class="solo-text">Modo de prática individual</span>
      </RouterLink>
    </div>

    <!-- Salas recentes -->
    <div v-if="!inRoom && recentRooms.length > 0" class="recent-rooms-section">
      <h3>Salas recentes</h3>

      <div class="recent-rooms-list">
        <button
          v-for="room in recentRooms"
          :key="room.code"
          class="recent-room-btn"
          @click="$emit('quickJoin', room.code)"
        >
          <div class="recent-room-content">
            <div class="recent-room-code">Sala {{ room.code }}</div>
            <div class="recent-room-time">
              Acessada {{ getTimeAgo(room.timestamp) }}
            </div>
          </div>

          <div class="recent-room-arrow">→ Entrar agora</div>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import unimarLogo from '@/assets/unimar.png'

defineProps({
  inRoom: { type: Boolean, required: true },
  recentRooms: { type: Array, required: true }
})

defineEmits(['quickJoin'])

const getTimeAgo = (timestamp) => {
  const seconds = Math.floor((Date.now() - timestamp) / 1000)

  if (seconds < 60) return 'agora mesmo'

  const minutes = Math.floor(seconds / 60)
  if (minutes < 60) return `há ${minutes} min`

  const hours = Math.floor(minutes / 60)
  if (hours < 24) return `há ${hours} hora${hours > 1 ? 's' : ''}`

  const days = Math.floor(hours / 24)
  return `há ${days} dia${days > 1 ? 's' : ''}`
}
</script>

<style scoped>
.waiting-display {
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 1.2rem;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  padding: 2rem;
}

.brand-area {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.5rem;
}

.unimar-logo {
  width: 170px;
  max-width: 80%;
  display: block;
  filter: drop-shadow(0 6px 14px rgba(0, 0, 0, 0.28));
}

.brand-title {
  margin: 0;
  font-size: 1.3rem;
  font-weight: 400;
  letter-spacing: 0.04em;
  color: #dbeafe;
}

.waiting-title {
  font-size: 2rem;
  margin: 0;
  color: var(--text-primary);
}

.waiting-message {
  font-size: 1.05rem;
  color: var(--text-tertiary);
  margin: 0;
  max-width: 620px;
  line-height: 1.6;
}

.solo-practice-section {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.65rem;
}

.solo-hint {
  font-size: 0.92rem;
  color: var(--text-tertiary);
  margin: 0;
}

.solo-practice-btn {
  display: flex;
  align-items: center;
  gap: 0.65rem;
  padding: 0.9rem 1.6rem;
  background: linear-gradient(135deg, rgba(30, 64, 175, 0.25), rgba(37, 99, 235, 0.18));
  border: 1px solid rgba(96, 165, 250, 0.38);
  border-radius: 10px;
  color: var(--text-primary);
  text-decoration: none;
  font-weight: 700;
  transition: all 0.2s ease;
}

.solo-practice-btn:hover {
  background: linear-gradient(135deg, rgba(30, 64, 175, 0.34), rgba(37, 99, 235, 0.26));
  transform: translateY(-2px);
}

.solo-icon {
  font-size: 1.2rem;
}

.solo-text {
  color: #bfdbfe;
}

.recent-rooms-section {
  margin-top: 2rem;
  width: 100%;
  max-width: 560px;
}

.recent-rooms-section h3 {
  font-size: 1.15rem;
  margin-bottom: 1rem;
  color: #93c5fd;
}

.recent-rooms-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.recent-room-btn {
  width: 100%;
  padding: 1rem 1.1rem;
  background: rgba(14, 30, 58, 0.62);
  border: 1px solid rgba(96, 165, 250, 0.34);
  border-radius: 12px;
  color: var(--text-primary);
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.2s ease;
  text-align: left;
}

.recent-room-btn:hover {
  background: rgba(22, 42, 79, 0.82);
  transform: translateY(-2px);
}

.recent-room-content {
  text-align: left;
}

.recent-room-code {
  font-size: 1.05rem;
  font-weight: 700;
  color: #bfdbfe;
}

.recent-room-time {
  font-size: 0.85rem;
  color: var(--text-tertiary);
  margin-top: 0.25rem;
}

.recent-room-arrow {
  font-size: 0.92rem;
  color: #93c5fd;
  font-weight: 600;
}

@media (max-width: 768px) {
  .waiting-display {
    padding: 1.2rem;
  }

  .unimar-logo {
    width: 140px;
  }

  .brand-title {
    font-size: 1.1rem;
  }

  .waiting-title {
    font-size: 1.6rem;
  }

  .waiting-message {
    font-size: 0.98rem;
  }

  .recent-room-btn {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.6rem;
  }

  .recent-room-arrow {
    align-self: flex-start;
  }
}
</style>
