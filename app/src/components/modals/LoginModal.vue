<template>
  <Modal :isOpen="isOpen" @close="$emit('cancel')" title="Login necessário">
    <template #default>
      <p class="modal-description">
        Este usuário pertence a uma conta registrada. Digite sua senha para continuar.
      </p>

      <form @submit.prevent="handleSubmit">
        <FormInput
          v-model="usernameDisplay"
          label="Usuário"
          type="text"
          readonly
        />

        <FormInput
          v-model="passwordInput"
          label="Senha"
          type="password"
          placeholder="Digite sua senha"
          :error="error"
        />
      </form>
    </template>

    <template #footer>
      <button class="btn-success" @click="handleSubmit">Entrar</button>
      <button class="btn-secondary" @click="$emit('cancel')">Cancelar</button>
    </template>
  </Modal>
</template>

<script setup>
import { ref, watch } from 'vue'
import Modal from '@/components/common/Modal.vue'
import FormInput from '@/components/common/FormInput.vue'

/**
 * LoginModal - Modal para login de usuários registrados
 *
 * @emits submit - Quando o usuário envia a senha
 * @emits cancel - Quando o usuário cancela
 */

const props = defineProps({
  isOpen: {
    type: Boolean,
    required: true
  },

  username: {
    type: String,
    required: true
  },

  error: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['submit', 'cancel'])

const usernameDisplay = ref(props.username)
const passwordInput = ref('')

watch(() => props.username, (newVal) => {
  usernameDisplay.value = newVal
})

watch(() => props.isOpen, (isOpen) => {
  if (!isOpen) {
    passwordInput.value = ''
  }
})

function handleSubmit() {
  if (passwordInput.value.trim()) {
    emit('submit', passwordInput.value)
  }
}
</script>

<style scoped>
.modal-description {
  color: var(--text-tertiary);
  margin-bottom: 1.5rem;
}
</style>
