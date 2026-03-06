<template>
  <div class="login-container">
    <div class="login-background-glow login-background-glow-1"></div>
    <div class="login-background-glow login-background-glow-2"></div>

    <div class="login-box">
      <div class="brand-area">

  <img
    :src="unimarLogo"
    alt="Universidade de Marília"
    class="unimar-logo"
  />

  <h1 class="login-title">UNIMAR Quiz</h1>

  <p class="login-subtitle">
    Plataforma interativa para aulas e atividades acadêmicas
  </p>

</div>

      <!-- Formulário de login -->
      <div v-if="!isLoggedIn && !requires2FA" class="login-form">
        <FormInput
          v-model="username"
          label="Usuário"
          type="text"
          placeholder="Digite seu usuário"
          :error="error"
          @keypress.enter="attemptLogin"
        />

        <FormInput
          v-model="password"
          label="Senha"
          type="password"
          placeholder="Digite sua senha"
          @keypress.enter="attemptLogin"
        />

        <Button
          :is-loading="isLoading"
          full-width
          @click="attemptLogin"
        >
          Entrar
        </Button>
      </div>

      <!-- Formulário de verificação em duas etapas -->
      <div v-else-if="requires2FA" class="two-factor-form">
        <div class="totp-header">
          <h3>Verificação em duas etapas</h3>
          <p>Digite o código de 6 dígitos do seu aplicativo autenticador</p>
        </div>

        <FormInput
          v-model="totpCode"
          label="Código de verificação"
          type="text"
          placeholder="000000"
          maxlength="6"
          inputmode="numeric"
          autocomplete="one-time-code"
          :error="totpError"
          @keypress.enter="verifyTOTP"
        />

        <Button
          :is-loading="isVerifying"
          :disabled="totpCode.length !== 6"
          full-width
          @click="verifyTOTP"
        >
          Verificar código
        </Button>

        <div class="totp-options">
          <button
            type="button"
            class="link-button"
            @click="showBackupCodeInput = !showBackupCodeInput"
          >
            {{ showBackupCodeInput ? 'Usar aplicativo autenticador' : 'Usar código de recuperação' }}
          </button>
        </div>

        <div v-if="showBackupCodeInput" class="backup-code-section">
          <FormInput
            v-model="backupCode"
            label="Código de recuperação"
            type="text"
            placeholder="XXXX-XXXX"
            maxlength="9"
            @keypress.enter="verifyBackupCode"
          />
          <Button
            :is-loading="isVerifying"
            :disabled="backupCode.length < 8"
            full-width
            variant="secondary"
            @click="verifyBackupCode"
          >
            Usar código de recuperação
          </Button>
        </div>

        <button type="button" class="link-button cancel-link" @click="cancelTwoFactor">
          Cancelar e voltar ao início
        </button>
      </div>

      <!-- Acessos após login -->
      <div v-else class="access-buttons">
        <h3 class="success-message">
          <AppIcon name="check" size="lg" />
          Acesso liberado
        </h3>

        <div class="button-group">
          <RouterLink to="/admin" class="access-link admin-link">
            <AppIcon name="clipboard-list" size="lg" />
            Painel administrativo - Gerenciar quizzes
          </RouterLink>

          <RouterLink to="/presenter" class="access-link presenter-link">
            <AppIcon name="presentation" size="lg" />
            Apresentador - Conduzir atividade
          </RouterLink>

          <RouterLink to="/display" class="access-link display-link">
            <AppIcon name="monitor" size="lg" />
            Tela de exibição - Público / projetor
          </RouterLink>
        </div>

        <Button
          variant="danger"
          full-width
          @click="performLogout"
        >
          Sair
        </Button>
      </div>

      <hr class="divider">

      <div class="public-links">
        <RouterLink to="/player" class="player-link">
          <AppIcon name="users" size="lg" />
          Entrar como aluno
        </RouterLink>

        <RouterLink to="/solo" class="solo-link">
          <AppIcon name="user" size="lg" />
          Prática individual
        </RouterLink>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth.js'
import { useUIStore } from '@/stores/ui.js'
import { useApi } from '@/composables/useApi.js'
import { useTheme } from '@/composables/useTheme.js'
import Button from '@/components/common/Button.vue'
import FormInput from '@/components/common/FormInput.vue'
import AppIcon from '@/components/common/AppIcon.vue'
import unimarLogo from '@/assets/unimar.png'

const router = useRouter()
const authStore = useAuthStore()
const uiStore = useUIStore()
const { get, post } = useApi()

const { initTheme } = useTheme('LOGIN')
initTheme()

const username = ref('admin')
const password = ref('')
const error = ref(null)
const isLoading = ref(false)
const isLoggedIn = ref(false)

// Estado da autenticação em duas etapas
const requires2FA = ref(false)
const tempToken = ref(null)
const pendingUser = ref(null)
const totpCode = ref('')
const totpError = ref(null)
const isVerifying = ref(false)
const showBackupCodeInput = ref(false)
const backupCode = ref('')

// Verifica se já está autenticado
onMounted(async () => {
  if (authStore.token && authStore.userRole === 'admin') {
    isLoggedIn.value = true
  } else {
    authStore.logout()
  }
})

const attemptLogin = async () => {
  error.value = null

  if (!username.value.trim() || !password.value.trim()) {
    error.value = 'Por favor, informe usuário e senha'
    return
  }

  isLoading.value = true

  try {
    const response = await post('/api/auth/login', {
      username: username.value.trim(),
      password: password.value.trim()
    })

    const data = response.data

    if (data.requires2FA) {
      requires2FA.value = true
      tempToken.value = data.tempToken
      pendingUser.value = data.user
      password.value = ''
      isLoading.value = false
      return
    }

    if (data.user?.account_type !== 'admin') {
      error.value = 'É necessário ter acesso de administrador'
      password.value = ''
      isLoading.value = false
      return
    }

    await completeLogin(data)
  } catch (err) {
    const message = err.response?.data?.message || 'Usuário ou senha inválidos'
    error.value = message
    password.value = ''
    uiStore.addNotification(message, 'error')
  } finally {
    isLoading.value = false
  }
}

const verifyTOTP = async () => {
  if (totpCode.value.length !== 6) return

  totpError.value = null
  isVerifying.value = true

  try {
    const response = await post('/api/auth/totp/verify', {
      tempToken: tempToken.value,
      code: totpCode.value,
      isBackupCode: false
    })

    await completeLogin(response.data)
    resetTwoFactorState()
  } catch (err) {
    totpError.value = err.response?.data?.message || 'Código de verificação inválido'
    totpCode.value = ''
  } finally {
    isVerifying.value = false
  }
}

const verifyBackupCode = async () => {
  if (backupCode.value.length < 8) return

  totpError.value = null
  isVerifying.value = true

  try {
    const response = await post('/api/auth/totp/verify', {
      tempToken: tempToken.value,
      code: backupCode.value.replace('-', ''),
      isBackupCode: true
    })

    await completeLogin(response.data)
    resetTwoFactorState()
  } catch (err) {
    totpError.value = err.response?.data?.message || 'Código de recuperação inválido'
    backupCode.value = ''
  } finally {
    isVerifying.value = false
  }
}

const completeLogin = async (data) => {
  authStore.setAuth(data.token, data.user.username, 'admin', data.user.id)
  isLoggedIn.value = true

  try {
    const adminInfoResponse = await get('/api/auth/admin-info')
    if (adminInfoResponse.data?.user?.is_root_admin) {
      authStore.setAuth(data.token, data.user.username, 'admin', data.user.id, true)
    }
  } catch (adminErr) {
    console.warn('Não foi possível carregar os dados do administrador:', adminErr)
  }

  uiStore.addNotification(`Bem-vindo novamente, ${data.user.username}!`, 'success')
}

const cancelTwoFactor = () => {
  resetTwoFactorState()
  password.value = ''
}

const resetTwoFactorState = () => {
  requires2FA.value = false
  tempToken.value = null
  pendingUser.value = null
  totpCode.value = ''
  totpError.value = null
  showBackupCodeInput.value = false
  backupCode.value = ''
}

const performLogout = async () => {
  isLoading.value = true

  try {
    if (authStore.token) {
      try {
        await post('/api/auth/logout', {})
      } catch (err) {
        console.error('Falha ao notificar logout:', err)
      }
    }
  } finally {
    authStore.logout()
    isLoggedIn.value = false
    username.value = 'admin'
    password.value = ''
    isLoading.value = false

    uiStore.addNotification('Saída realizada com sucesso', 'info')
  }
}
</script>

<style scoped>
  .unimar-logo{
  width:180px;
  margin-bottom:18px;
  display:block;
  margin-left:auto;
  margin-right:auto;
  filter: drop-shadow(0 6px 14px rgba(0,0,0,0.35));
}
.login-container {
  position: relative;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 24px;
  background:
    radial-gradient(circle at top left, rgba(34, 84, 160, 0.30), transparent 28%),
    radial-gradient(circle at bottom right, rgba(27, 108, 187, 0.24), transparent 32%),
    linear-gradient(135deg, #07152b 0%, #0b2347 45%, #103164 100%);
  z-index: 100;
}

.login-background-glow {
  position: absolute;
  border-radius: 50%;
  filter: blur(80px);
  opacity: 0.35;
  pointer-events: none;
}

.login-background-glow-1 {
  width: 280px;
  height: 280px;
  background: #1a63c7;
  top: -50px;
  left: -40px;
}

.login-background-glow-2 {
  width: 320px;
  height: 320px;
  background: #0ea5e9;
  right: -80px;
  bottom: -90px;
}

.login-box {
  position: relative;
  z-index: 2;
  width: 100%;
  max-width: 520px;
  padding: 40px 36px;
  border-radius: 24px;
  background: rgba(8, 20, 43, 0.74);
  border: 1px solid rgba(255, 255, 255, 0.10);
  box-shadow:
    0 20px 60px rgba(0, 0, 0, 0.38),
    inset 0 1px 0 rgba(255, 255, 255, 0.06);
  backdrop-filter: blur(14px);
}

.brand-area {
  text-align: center;
  margin-bottom: 28px;
}

.brand-badge {
  display: inline-block;
  margin-bottom: 14px;
  padding: 6px 14px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.10);
  border: 1px solid rgba(255, 255, 255, 0.14);
  color: #dbeafe;
  font-size: 0.78rem;
  font-weight: 700;
  letter-spacing: 0.12em;
}

.login-title {
  margin: 0 0 8px 0;
  font-size: 1.6rem;
  line-height: 1.1;
  font-weight: 400;
  text-align: center;
  color: #e2e8f0;
  letter-spacing: 0.05em;
}
  
.login-subtitle {
  margin: 0;
  text-align: center;
  color: #cbd5e1;
  font-size: 0.98rem;
  line-height: 1.5;
}

.login-form,
.two-factor-form {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.totp-header {
  text-align: center;
  margin-bottom: 4px;
}

.totp-header h3 {
  color: #ffffff;
  margin: 0 0 8px 0;
  font-size: 1.24rem;
}

.totp-header p {
  color: #cbd5e1;
  margin: 0;
  font-size: 0.92rem;
  line-height: 1.5;
}

.totp-options {
  text-align: center;
  margin-top: -4px;
}

.link-button {
  background: none;
  border: none;
  color: #7dd3fc;
  cursor: pointer;
  text-decoration: none;
  font-size: 0.92rem;
  padding: 8px;
  transition: color 0.2s ease, opacity 0.2s ease;
}

.link-button:hover {
  color: #ffffff;
}

.cancel-link {
  color: #cbd5e1;
  margin-top: 6px;
}

.backup-code-section {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 16px;
  background: rgba(255, 255, 255, 0.06);
  border-radius: 14px;
  border: 1px solid rgba(255, 255, 255, 0.10);
}

.access-buttons {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.success-message {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  color: #bbf7d0;
  text-align: center;
  margin: 0 0 8px 0;
  font-size: 1.3rem;
}

.button-group {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.access-link {
  display: block;
  text-align: center;
  text-decoration: none;
  font-weight: 700;
  padding: 16px 18px;
  border-radius: 14px;
  color: #f8fafc;
  transition: transform 0.2s ease, box-shadow 0.2s ease, opacity 0.2s ease;
}

.access-link:hover {
  transform: translateY(-2px);
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.22);
}

.admin-link {
  background: linear-gradient(135deg, rgba(30, 64, 175, 0.9), rgba(37, 99, 235, 0.9));
  border: 1px solid rgba(147, 197, 253, 0.35);
}

.presenter-link {
  background: linear-gradient(135deg, rgba(10, 80, 153, 0.92), rgba(14, 116, 144, 0.92));
  border: 1px solid rgba(125, 211, 252, 0.35);
}

.display-link {
  background: linear-gradient(135deg, rgba(8, 47, 73, 0.95), rgba(3, 105, 161, 0.92));
  border: 1px solid rgba(103, 232, 249, 0.28);
}

.divider {
  margin: 28px 0 22px;
  border: none;
  border-top: 1px solid rgba(255, 255, 255, 0.10);
}

.public-links {
  display: flex;
  gap: 14px;
  justify-content: center;
}

.player-link,
.solo-link {
  flex: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  min-height: 54px;
  padding: 12px 16px;
  border-radius: 14px;
  color: #f8fafc;
  text-decoration: none;
  text-align: center;
  transition: transform 0.2s ease, background 0.2s ease, border-color 0.2s ease;
}

.player-link {
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.12);
}

.player-link:hover {
  background: rgba(255, 255, 255, 0.14);
  transform: translateY(-2px);
}

.solo-link {
  background: rgba(37, 99, 235, 0.22);
  border: 1px solid rgba(96, 165, 250, 0.38);
}

.solo-link:hover {
  background: rgba(37, 99, 235, 0.34);
  transform: translateY(-2px);
}

@media (max-width: 640px) {
  .login-container {
    padding: 16px;
  }

  .login-box {
    padding: 28px 22px;
    border-radius: 20px;
  }

  .login-title {
    font-size: 2rem;
  }

  .login-subtitle {
    font-size: 0.92rem;
  }

  .public-links {
    flex-direction: column;
  }
}
</style>
