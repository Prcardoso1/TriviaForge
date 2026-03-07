<template>
  <Modal :isOpen="isOpen" @close="$emit('cancel')" title="Definir nova senha">
    <template #default>
      <p class="modal-description">
        Sua senha foi redefinida por um administrador. Defina uma nova senha para continuar.
      </p>

      <form @submit.prevent="handleSubmit">
        <FormInput
          v-model="usernameDisplay"
          label="Usuário"
          type="text"
          readonly
        />

        <FormInput
          v-model="newPasswordInput"
          label="Nova senha"
          type="password"
          placeholder="Digite a nova senha (mínimo 6 caracteres)"
          :error="error"
        />

        <FormInput
          v-model="confirmPasswordInput"
          label="Confirmar senha"
          type="password"
          placeholder="Confirme a nova senha"
        />
      </form>
    </template>

    <template #footer>
      <button class="btn-success" @click="handleSubmit">Definir senha</button>
      <button class="btn-secondary" @click="$emit('cancel')">Cancelar</button>
    </template>
  </Modal>
</template>

<script setup>
import { ref, watch } from 'vue'
import Modal from '@/components/common/Modal.vue'
import FormInput from '@/components/common/FormInput.vue'

/**
 * SetPasswordModal - Modal para definir nova senha após reset do administrador
 *
 * @emits submit - Quando usuário envia nova senha
 * @emits cancel - Quando usuário cancela
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
const newPasswordInput = ref('')
const confirmPasswordInput = ref('')

watch(() => props.username, (newVal) => {
  usernameDisplay.value = newVal
})

watch(() => props.isOpen, (isOpen) => {
  if (!isOpen) {
    newPasswordInput.value = ''
    confirmPasswordInput.value = ''
  }
})

function handleSubmit() {
  if (newPasswordInput.value.trim() && confirmPasswordInput.value.trim()) {
    emit('submit', {
      newPassword: newPasswordInput.value,
      confirmPassword: confirmPasswordInput.value
    })
  }
}
</script>

<style scoped>
.modal-description {
  color: var(--text-tertiary);
  margin-bottom: 1.5rem;
}
</style>
