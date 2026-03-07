<template>
  <div class="join-section">
    <h3>Entrar na sala</h3>

    <!-- Campo de usuário ou exibição do usuário salvo -->
    <div v-if="!savedUsername" class="username-input-section">
      <label for="playerUsername">Usuário</label>
      <input
        id="playerUsername"
        :value="usernameInput"
        type="text"
        placeholder="Escolha um nome de usuário"
        @input="$emit('update:usernameInput', $event.target.value)"
      />
      <p class="form-hint">Este será o nome da sua conta</p>
    </div>

    <div v-else class="username-display-section">
      <label>Usuário (Conta)</label>
      <div class="username-display">
        <div class="username-value">{{ savedUsername }}</div>
        <button class="change-username-btn" @click="$emit('changeUsername')">Alterar</button>
      </div>
    </div>

    <label for="playerDisplayName">Nome exibido</label>
    <input
      id="playerDisplayName"
      :value="displayNameInput"
      type="text"
      placeholder="Nome exibido no jogo"
      @input="$emit('update:displayNameInput', $event.target.value)"
    />
    <p class="form-hint">Este é o nome que os outros jogadores verão</p>

    <label for="roomCodeManual">Código da sala</label>
    <input
      id="roomCodeManual"
      :value="roomCodeInput"
      type="text"
      placeholder="Digite o código da sala"
      @input="$emit('update:roomCodeInput', $event.target.value)"
    />

    <button class="btn-primary" @click="$emit('joinRoom')">Entrar na sala</button>
    <button class="btn-secondary" @click="$emit('manageAccount')">Gerenciar conta</button>
  </div>
</template>

<script setup>
defineProps({
  savedUsername: { type: String, default: null },
  usernameInput: { type: String, required: true },
  displayNameInput: { type: String, required: true },
  roomCodeInput: { type: String, required: true }
});

defineEmits([
  'update:usernameInput',
  'update:displayNameInput',
  'update:roomCodeInput',
  'changeUsername',
  'joinRoom',
  'manageAccount'
]);
</script>

<style scoped>
.join-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.join-section h3 {
  margin: 0;
  color: var(--info-light);
  font-size: 1.1rem;
}

.username-input-section,
.username-display-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.username-display {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.username-value {
  flex: 1;
  padding: 0.75rem;
  background: var(--bg-overlay-10);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  color: var(--info-light);
  font-weight: 500;
}

.change-username-btn {
  padding: 0.75rem 1rem;
  background: var(--bg-overlay-10);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  color: var(--text-tertiary);
  cursor: pointer;
  font-size: 0.85rem;
  white-space: nowrap;
  transition: all 0.2s;
}

.change-username-btn:hover {
  background: var(--bg-overlay-20);
  color: var(--text-primary);
}

.form-hint {
  font-size: 0.85rem;
  color: var(--text-tertiary);
  margin: 0;
}

.join-section input {
  padding: 0.75rem;
  background: var(--bg-overlay-10);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  color: var(--text-primary);
  font-size: 1rem;
}

.join-section label {
  color: var(--text-tertiary);
  font-size: 0.9rem;
  font-weight: 500;
}

.btn-primary,
.btn-secondary {
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-primary {
  background: var(--info-bg-30);
  border: 1px solid var(--info-light);
  color: var(--info-light);
}

.btn-primary:hover {
  background: var(--info-bg-50);
}

.btn-secondary {
  background: var(--info-bg-20);
  border: 1px solid var(--info-light);
  color: var(--info-light);
}

.btn-secondary:hover {
  background: var(--info-bg-30);
}
</style>
