<template>
  <div class="waiting-display">
    <h2 class="waiting-title">
      {{ inRoom ? 'Aguardando pergunta' : 'Entre em uma sala para começar' }}
    </h2>

    <p class="waiting-message">
      {{ inRoom ? 'Aguardando o apresentador iniciar o quiz...' : 'Digite seu nome e o código da sala para começar a jogar.' }}
    </p>

    <!-- Link de prática individual -->
    <div v-if="!inRoom" class="solo-practice-section">
      <p class="solo-hint">ou pratique sozinho</p>

      <RouterLink to="/solo" class="solo-practice-btn">
        <span class="solo-icon">🎯</span>
        <span class="solo-text">Modo prática individual</span>
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
              Entrou {{ getTimeAgo(room.timestamp) }}
            </div>
          </div>

          <div class="recent-room-arrow">→ Entrar rapidamente</div>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
defineProps({
  inRoom: { type: Boolean, required: true },
  recentRooms: { type: Array, required: true }
});

defineEmits(['quickJoin']);

const getTimeAgo = (timestamp) => {
  const seconds = Math.floor((Date.now() - timestamp) / 1000);

  if (seconds < 60) return 'agora mesmo';

  const minutes = Math.floor(seconds / 60);
  if (minutes < 60) return `há ${minutes} min`;

  const hours = Math.floor(minutes / 60);
  if (hours < 24) return `há ${hours} hora${hours > 1 ? 's' : ''}`;

  const days = Math.floor(hours / 24);
  return `há ${days} dia${days > 1 ? 's' : ''}`;
};
</script>
