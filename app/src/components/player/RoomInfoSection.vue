<template>
  <div class="room-info">
    <h3>Informações da sala</h3>

    <div>
      <strong>Código da sala:</strong>
      <span>{{ currentRoomCode }}</span>
    </div>

    <div>
      <strong>Seu nome:</strong>
      <span>{{ currentDisplayName }}</span>
    </div>

    <button class="btn-info" @click="$emit('showProgress')">
      📊 Ver progresso
    </button>

    <button class="btn-danger" @click="$emit('leaveRoom')">
      Sair da sala
    </button>
  </div>
</template>

<script setup>
defineProps({
  currentRoomCode: { type: String, required: true },
  currentDisplayName: { type: String, required: true }
});

defineEmits(['showProgress', 'leaveRoom']);
</script>

<style scoped>
.room-info {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.room-info h3 {
  margin: 0;
  color: var(--info-light);
  font-size: 1.1rem;
}

.room-info div {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem;
  background: var(--bg-overlay-10);
  border-radius: 8px;
  color: var(--text-primary);
}

.room-info strong {
  color: var(--text-tertiary);
}

.btn-info,
.btn-danger {
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-info {
  background: var(--info-bg-20);
  border: 1px solid var(--info-light);
  color: var(--info-light);
}

.btn-info:hover {
  background: var(--info-bg-30);
}

.btn-danger {
  background: var(--danger-bg-20);
  border: 1px solid var(--danger-light);
  color: var(--danger-light);
}

.btn-danger:hover {
  background: var(--danger-bg-30);
}
</style>
