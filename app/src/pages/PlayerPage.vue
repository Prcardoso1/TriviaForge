<template>
  <div class="player-page">
    <!-- Navbar -->
    <PlayerNavbar
      :inRoom="inRoom"
      :currentRoomCode="currentRoomCode"
      :connectionStateClass="connectionStateClass"
      :connectionStateSymbol="connectionStateSymbol"
      :menuOpen="menuOpen"
      :nonSpectatorPlayers="nonSpectatorPlayers"
      :loginUsername="loginUsername"
      :revealedCount="revealedCount"
      :totalQuestions="totalQuestions"
      @showProgress="showProgressModal"
      @toggleMenu="toggleMenu"
      @leaveRoom="handleLeaveRoomClick"
      @logout="handleLogout"
    />

    <!-- Main Content -->
    <div class="player-container">
      <!-- Banner de conexão perdida -->
      <div v-if="showConnectionLostBanner" class="connection-lost-banner">
        <div class="banner-content">
          <AppIcon name="alert-triangle" size="lg" class="icon" />
          <div class="text">
            <strong>Conexão perdida</strong>
            <p>Tentando reconectar... ({{ reconnectionAttempts }} tentativa{{ reconnectionAttempts !== 1 ? 's' : '' }})</p>
          </div>
          <Button @click="forceReconnect" variant="danger" size="small" class="reconnect-btn">Reconectar agora</Button>
        </div>
      </div>

      <!-- Banner de questões perdidas -->
      <div v-if="missedQuestionsBanner" class="missed-questions-banner">
        <AppIcon name="alert-triangle" size="md" /> Você perdeu algumas questões enquanto estava ausente
      </div>

      <!-- Esquerda/Topo: Exibição da questão -->
      <div class="question-area">
        <!-- Pódio de resultados finais -->
        <GameResults
          v-if="quizResultsData && !questionDisplaying"
          :players="quizResultsData.players"
          :totalQuestions="quizResultsData.totalQuestions"
          :classAverage="quizResultsData.classAverage"
        />

        <!-- Tela de quiz concluído -->
        <div v-else-if="quizCompleted && !questionDisplaying" class="quiz-complete-screen">
          <AppIcon name="check-circle" size="2xl" class="complete-icon" />
          <h2 class="complete-title">Quiz concluído!</h2>
          <p v-if="resultsCountdown > 0" class="complete-subtitle">
            Resultados em <span class="countdown-number">{{ resultsCountdown }}</span>...
          </p>
          <p v-else class="complete-subtitle">
            Consulte seu <strong>Progresso</strong> para ver seu desempenho!
          </p>
        </div>

        <!-- Tela de espera -->
        <WaitingDisplay
          v-else-if="!questionDisplaying"
          :inRoom="inRoom"
          :recentRooms="recentRooms"
          @quickJoin="quickJoinRoom"
        />

        <!-- Exibição da questão -->
        <QuestionDisplay
          v-else
          :currentQuestion="currentQuestion"
          :selectedAnswer="selectedAnswer"
          :answerRevealed="answerRevealed"
          :playerGotCorrect="playerGotCorrect"
          :answeredCurrentQuestion="answeredCurrentQuestion"
          :autoMode="autoMode"
          :timerStartedAt="timerStartedAt"
          :timerDuration="timerDuration"
          :timerPaused="timerPaused"
          @selectAnswer="selectAnswer"
        />
      </div>

      <!-- Direita/Base: Sala / Barra lateral -->
      <div class="sidebar" :class="{ 'hidden-mobile-in-room': inRoom }">
        <!-- Área de entrada -->
        <JoinRoomSection
          v-if="!inRoom"
          :savedUsername="savedUsername"
          v-model:usernameInput="usernameInput"
          v-model:displayNameInput="displayNameInput"
          v-model:roomCodeInput="roomCodeInput"
          @changeUsername="handleChangeUsername"
          @joinRoom="handleJoinRoom"
          @manageAccount="handleManageAccount"
        />

        <!-- Informações da sala -->
        <RoomInfoSection
          v-else
          :currentRoomCode="currentRoomCode"
          :currentDisplayName="currentDisplayName"
          @showProgress="showProgressModal"
          @leaveRoom="handleLeaveRoomClick"
        />

        <!-- Lista de jogadores -->
        <PlayersList :nonSpectatorPlayers="nonSpectatorPlayers" />

        <!-- Mensagem de status -->
        <StatusMessage :message="statusMessage" :messageType="statusMessageType" />
      </div>
    </div>

    <!-- Modais -->
    <LoginModal
      :isOpen="showLoginModal"
      :username="loginUsername"
      :error="loginError"
      @submit="handleLoginSubmit"
      @cancel="loginCancelled"
    />

    <SetPasswordModal
      :isOpen="showSetPasswordModal"
      :username="setPasswordUsername"
      :error="setPasswordError"
      @submit="handleSetPasswordSubmit"
      @cancel="passwordSetupCancelled"
    />

    <ProgressModal
      :isOpen="showProgressModalFlag"
      :questionHistory="questionHistory"
      @close="showProgressModalFlag = false"
    />

    <LogoutConfirmModal
      :isOpen="showLogoutConfirmModal"
      @confirm="confirmLogout"
      @close="showLogoutConfirmModal = false"
    />

    <LeaveRoomConfirmModal
      :isOpen="showLeaveRoomConfirmModal"
      @confirm="confirmLeaveRoom"
      @close="showLeaveRoomConfirmModal = false"
    />

    <ChangeUsernameConfirmModal
      :isOpen="showChangeUsernameModal"
      @confirm="confirmChangeUsername"
      @close="showChangeUsernameModal = false"
    />

    <AnswerConfirmModal
      :isOpen="showAnswerConfirmModal"
      :selectedIndex="pendingAnswerIndex"
      :choices="currentQuestion?.choices || []"
      @confirm="confirmAnswer"
      @cancel="cancelAnswer"
    />

    <WakeLockIndicator
      :inRoom="inRoom"
      :wakeLockActive="wakeLock.isActive.value"
      :error="wakeLock.error.value"
      :isSupported="wakeLock.isSupported.value"
    />
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useSocket } from '@/composables/useSocket.js'
import { useWakeLock } from '@/composables/useWakeLock.js'
import { useApi } from '@/composables/useApi.js'
import { useUIStore } from '@/stores/ui.js'
import { useTheme } from '@/composables/useTheme.js'
import Button from '@/components/common/Button.vue'
import LoginModal from '@/components/modals/LoginModal.vue'
import SetPasswordModal from '@/components/modals/SetPasswordModal.vue'
import LogoutConfirmModal from '@/components/modals/LogoutConfirmModal.vue'
import LeaveRoomConfirmModal from '@/components/modals/LeaveRoomConfirmModal.vue'
import ChangeUsernameConfirmModal from '@/components/modals/ChangeUsernameConfirmModal.vue'
import ProgressModal from '@/components/modals/ProgressModal.vue'
import AnswerConfirmModal from '@/components/modals/AnswerConfirmModal.vue'
import PlayerNavbar from '@/components/player/PlayerNavbar.vue'
import GameResults from '@/components/player/GameResults.vue'
import QuestionDisplay from '@/components/player/QuestionDisplay.vue'
import WaitingDisplay from '@/components/player/WaitingDisplay.vue'
import JoinRoomSection from '@/components/player/JoinRoomSection.vue'
import RoomInfoSection from '@/components/player/RoomInfoSection.vue'
import PlayersList from '@/components/player/PlayersList.vue'
import StatusMessage from '@/components/player/StatusMessage.vue'
import WakeLockIndicator from '@/components/player/WakeLockIndicator.vue'
import AppIcon from '@/components/common/AppIcon.vue'

const router = useRouter()
const route = useRoute()
const socket = useSocket()
const wakeLock = useWakeLock()
const { post } = useApi()
const uiStore = useUIStore()

const { initTheme } = useTheme('PLAYER')
initTheme()

const DEBUG = import.meta.env.DEV
const debugLog = (...args) => {
  if (DEBUG) console.log('[PLAYER DEBUG]', ...args)
}

const menuOpen = ref(false)
const questionDisplaying = ref(false)
const inRoom = ref(false)
const isConnected = ref(false)
const answerRevealed = ref(false)

const autoMode = ref(false)
const timerStartedAt = ref(null)
const timerDuration = ref(null)
const timerPaused = ref(false)

const showLoginModal = ref(false)
const showSetPasswordModal = ref(false)
const showProgressModalFlag = ref(false)
const showLogoutConfirmModal = ref(false)
const showLeaveRoomConfirmModal = ref(false)
const showChangeUsernameModal = ref(false)
const showAnswerConfirmModal = ref(false)
const pendingAnswerIndex = ref(null)

const currentRoomCode = ref(null)
const currentUsername = ref(null)
const currentDisplayName = ref(null)
const currentPlayers = ref([])
const currentQuestion = ref(null)
const selectedAnswer = ref(null)
const playerGotCorrect = ref(false)
const answeredCurrentQuestion = ref(false)
const answeredQuestions = new Set()

const revealedCount = ref(0)
const totalQuestions = ref(0)

const quizResultsData = ref(null)
const quizCompleted = ref(false)
const resultsCountdown = ref(0)
let countdownInterval = null

const questionHistory = ref([])
const recentRooms = ref([])
const activeRoomCodes = ref(null)

const usernameInput = ref('')
const displayNameInput = ref('')
const roomCodeInput = ref('')
const savedUsername = ref(localStorage.getItem('playerUsername'))
const savedDisplayName = ref(localStorage.getItem('playerDisplayName'))
const savedAccountType = ref(localStorage.getItem('playerAccountType'))

const connectionState = ref('connected')
const isPageVisible = ref(true)
const visibilitySwitchCount = ref(0)
const visibilitySwitchTimestamps = ref([])
const awayTimeout = ref(null)
const disconnectTimeout = ref(null)
const warningClearTimeout = ref(null)
const answerDisplayTimeout = ref(null)
const missedQuestionsBanner = ref(false)
const lastJoinRoomAttempt = ref(0)
const joinRoomInProgress = ref(false)
const showConnectionLostBanner = ref(false)
const reconnectionAttempts = ref(0)

const loginUsername = ref('')
const loginError = ref('')

const setPasswordUsername = ref('')
const setPasswordError = ref('')

const statusMessage = ref('Pronto para entrar')
const statusMessageType = ref('')

const nonSpectatorPlayers = computed(() => {
  return currentPlayers.value.filter(p => !p.isSpectator)
})

const connectionStateClass = computed(() => {
  switch (connectionState.value) {
    case 'connected': return 'status-connected'
    case 'away': return 'status-away'
    case 'disconnected': return 'status-disconnected'
    case 'warning': return 'status-warning'
    default: return 'status-disconnected'
  }
})

const connectionStateSymbol = computed(() => {
  switch (connectionState.value) {
    case 'connected': return '●'
    case 'away': return '●'
    case 'disconnected': return '●'
    case 'warning': return '!'
    default: return '○'
  }
})

const updateConnectionState = (newState) => {
  if (connectionState.value === newState) return

  const oldState = connectionState.value
  connectionState.value = newState

  console.log(`[CONNECTION] State changed: ${oldState} → ${newState}`)

  if (inRoom.value && currentRoomCode.value) {
    socket.emit('playerStateChange', {
      roomCode: currentRoomCode.value,
      username: currentUsername.value,
      state: newState
    })
  }

  if (newState === 'connected') {
    clearTimeout(awayTimeout.value)
    clearTimeout(disconnectTimeout.value)
  }
}

const detectRapidSwitching = () => {
  const now = Date.now()
  const tenSecondsAgo = now - 10000

  visibilitySwitchTimestamps.value = visibilitySwitchTimestamps.value.filter(
    timestamp => timestamp > tenSecondsAgo
  )

  visibilitySwitchTimestamps.value.push(now)

  if (visibilitySwitchTimestamps.value.length >= 5) {
    console.log('[CONNECTION] Rapid switching detected!')
    updateConnectionState('warning')

    clearTimeout(warningClearTimeout.value)
    warningClearTimeout.value = setTimeout(() => {
      if (connectionState.value === 'warning' && isPageVisible.value) {
        updateConnectionState('connected')
      }
    }, 10000)

    return true
  }

  return false
}

const handleVisibilityChange = () => {
  const wasVisible = isPageVisible.value
  isPageVisible.value = !document.hidden

  console.log(`[CONNECTION] Page visibility changed: ${wasVisible} → ${isPageVisible.value}`)

  const isRapidSwitching = detectRapidSwitching()
  if (isRapidSwitching) {
    return
  }

  if (isPageVisible.value) {
    console.log('[CONNECTION] Page visible - returning from away')

    clearTimeout(awayTimeout.value)
    clearTimeout(disconnectTimeout.value)

    const missedQuestions = questionHistory.value.filter(q => q.revealed && q.playerChoice === null && q.missedWhileAway)
    if (missedQuestions.length > 0) {
      missedQuestionsBanner.value = true
      setTimeout(() => {
        missedQuestionsBanner.value = false
      }, 5000)
      console.log(`[CONNECTION] Player missed ${missedQuestions.length} question(s) while away`)
    }

    if (/iPhone|iPad|iPod/i.test(navigator.userAgent)) {
      console.log('[CONNECTION] iOS device detected - forcing immediate reconnection')

      if (!socket.isConnected.value) {
        socket.connect()
      }

      if (currentRoomCode.value && currentUsername.value && currentDisplayName.value) {
        const rejoinInterval = setInterval(() => {
          if (socket.isConnected.value) {
            clearInterval(rejoinInterval)
            emitJoinRoom(currentRoomCode.value, currentUsername.value, currentDisplayName.value, 'iOS visibility change')
            updateConnectionState('connected')
          }
        }, 100)

        setTimeout(() => clearInterval(rejoinInterval), 3000)
      }

      if (wakeLock.isSupported.value && !wakeLock.isActive.value) {
        wakeLock.requestWakeLock()
      }

      return
    }

    if (/Android.*Chrome/i.test(navigator.userAgent)) {
      console.log('[CONNECTION] Android Chrome detected - delayed reconnection check')
      setTimeout(() => {
        if (!socket.isConnected.value) {
          socket.connect()
        }
      }, 500)
    }

    if (wakeLock.isSupported.value && !wakeLock.isActive.value) {
      wakeLock.requestWakeLock()
    }

    if (connectionState.value === 'disconnected') {
      if (currentRoomCode.value && currentUsername.value && currentDisplayName.value && socket.isConnected.value) {
        emitJoinRoom(currentRoomCode.value, currentUsername.value, currentDisplayName.value, 'visibility change (disconnected)')
        updateConnectionState('connected')
      }
    } else if (connectionState.value === 'away') {
      console.log('[CONNECTION] Returned from away - updating state')
      updateConnectionState('connected')
    } else if (connectionState.value === 'warning') {
      console.log('[CONNECTION] Returned while in warning state')
    }
  } else {
    console.log('[CONNECTION] Page hidden - starting away timer (30s debounce)')

    clearTimeout(awayTimeout.value)

    awayTimeout.value = setTimeout(() => {
      if (!isPageVisible.value) {
        console.log('[CONNECTION] Page still hidden after 30s - marking as away')
        updateConnectionState('away')

        disconnectTimeout.value = setTimeout(() => {
          console.log('[CONNECTION] Away timeout (2 min) - marking as disconnected')
          updateConnectionState('disconnected')

          setTimeout(() => {
            console.log('[CONNECTION] Disconnect timeout (5 min) - leaving room')
            if (inRoom.value) {
              handleLeaveRoom()
            }
          }, 5 * 60 * 1000)
        }, 2 * 60 * 1000)
      } else {
        console.log('[CONNECTION] Page became visible again - canceling away state')
      }
    }, 30 * 1000)
  }
}

const setupSocketListeners = () => {
  debugLog('setupSocketListeners called', {
    inRoom: inRoom.value,
    currentRoomCode: currentRoomCode.value,
    currentUsername: currentUsername.value,
    timestamp: new Date().toISOString()
  })

  socket.off('activeRoomsUpdate')
  socket.off('roomError')
  socket.off('roomCreated')
  socket.off('playerListUpdate')
  socket.off('questionPresented')
  socket.off('questionRevealed')
  socket.off('roomClosed')
  socket.off('quizCompleted')

  socket.on('connect', () => {
    isConnected.value = true
    console.log('[CONNECTION] Socket connected')
    debugLog('Socket CONNECT handler fired (PlayerPage)', {
      inRoom: inRoom.value,
      currentRoomCode: currentRoomCode.value,
      currentUsername: currentUsername.value,
      isPageVisible: isPageVisible.value,
      connectionState: connectionState.value,
      timestamp: new Date().toISOString()
    })

    if (inRoom.value && currentRoomCode.value && currentUsername.value && currentDisplayName.value) {
      console.log('[CONNECTION] Socket connected while in room - rejoining')
      debugLog('Auto-rejoining after connect event (app switch scenario)')
      emitJoinRoom(currentRoomCode.value, currentUsername.value, currentDisplayName.value, 'connect event (app switch)')
    }

    if (isPageVisible.value && connectionState.value !== 'warning') {
      updateConnectionState('connected')
    }
    showConnectionLostBanner.value = false
    reconnectionAttempts.value = 0
    debugLog('Emitting getActiveRooms')
    socket.emit('getActiveRooms')
  })

  socket.on('disconnect', () => {
    isConnected.value = false
    console.log('[CONNECTION] Socket disconnected (hard disconnect)')
    updateConnectionState('disconnected')

    showConnectionLostBanner.value = true
    reconnectionAttempts.value = 0

    clearTimeout(disconnectTimeout.value)
    disconnectTimeout.value = setTimeout(() => {
      console.log('[CONNECTION] Disconnect timeout (5 min) - leaving room')
      if (inRoom.value) {
        handleLeaveRoom()
      }
    }, 5 * 60 * 1000)
  })

  socket.on('reconnect_attempt', (attempt) => {
    console.log(`[CONNECTION] Reconnection attempt #${attempt}`)
    reconnectionAttempts.value = attempt
  })

  socket.on('reconnect', () => {
    console.log('[CONNECTION] Successfully reconnected!')
    debugLog('Socket RECONNECT event - checking if we need to rejoin room', {
      inRoom: inRoom.value,
      currentRoomCode: currentRoomCode.value,
      currentUsername: currentUsername.value,
      currentDisplayName: currentDisplayName.value
    })

    showConnectionLostBanner.value = false
    reconnectionAttempts.value = 0

    if (connectionState.value !== 'warning') {
      updateConnectionState('connected')
    }

    if (inRoom.value && currentRoomCode.value && currentUsername.value && currentDisplayName.value) {
      console.log('[CONNECTION] Rejoining room after reconnect')
      debugLog('Calling emitJoinRoom after reconnect')
      emitJoinRoom(currentRoomCode.value, currentUsername.value, currentDisplayName.value, 'reconnect event')
    }
  })

  socket.on('playerListUpdate', ({ roomCode, players, revealedCount: rc, totalQuestions: tq }) => {
    if (tq !== undefined) {
      revealedCount.value = rc || 0
      totalQuestions.value = tq
    }
    console.log(`[CONNECTION] playerListUpdate received - roomCode: ${roomCode}, currentRoomCode: ${currentRoomCode.value}, joinInProgress: ${joinRoomInProgress.value}, inRoom: ${inRoom.value}`)
    debugLog('playerListUpdate received', {
      roomCode,
      currentRoomCode: currentRoomCode.value,
      joinRoomInProgress: joinRoomInProgress.value,
      inRoom: inRoom.value,
      playersCount: players?.length,
      timestamp: new Date().toISOString()
    })

    if (joinRoomInProgress.value) {
      console.log('[CONNECTION] joinRoom completed - ready for interactions (cleared by playerListUpdate)')
      debugLog('Clearing joinRoomInProgress flag - join successful')
      joinRoomInProgress.value = false
    }

    if (roomCode !== currentRoomCode.value) {
      console.log(`[CONNECTION] Room code updated from ${currentRoomCode.value} to ${roomCode}`)
      debugLog('Room code mismatch - updating', {
        from: currentRoomCode.value,
        to: roomCode
      })
      currentRoomCode.value = roomCode
    }

    console.log(`[CONNECTION] ✅ Setting inRoom to TRUE (${players.length} players in room)`)
    inRoom.value = true
    currentPlayers.value = players
    const nonSpectatorCount = players.filter(p => !p.isSpectator).length
    statusMessage.value = `${nonSpectatorCount} jogador(es) na sala`
    saveRecentRoom(roomCode)

    if (wakeLock.isSupported.value && !wakeLock.isActive.value) {
      wakeLock.requestWakeLock()
    }
  })

  socket.on('questionPresented', ({ questionIndex, question, autoMode: isAutoMode, timerStartedAt: serverTimerStartedAt, timerDuration: serverTimerDuration }) => {
    if (joinRoomInProgress.value) {
      console.log('[CONNECTION] Clearing joinRoom flag (questionPresented received)')
      joinRoomInProgress.value = false
    }

    if (answerDisplayTimeout.value) {
      clearTimeout(answerDisplayTimeout.value)
      answerDisplayTimeout.value = null
    }

    autoMode.value = isAutoMode || false
    timerStartedAt.value = serverTimerStartedAt || null
    timerDuration.value = serverTimerDuration || null

    questionDisplaying.value = true
    const isAlreadyRevealed = question.revealed === true || question.isRevealed === true
    answerRevealed.value = isAlreadyRevealed

    currentQuestion.value = {
      ...question,
      correctChoice: isAlreadyRevealed ? question.correctChoice : undefined,
      index: questionIndex
    }
    selectedAnswer.value = null
    answeredCurrentQuestion.value = answeredQuestions.has(questionIndex)

    if (!questionHistory.value.find(q => q.index === questionIndex)) {
      const isMissedWhileAway = connectionState.value === 'away' || !isPageVisible.value
      questionHistory.value.push({
        index: questionIndex,
        text: question.text,
        choices: question.choices,
        imageUrl: question.imageUrl || null,
        playerChoice: null,
        correctChoice: isAlreadyRevealed ? question.correctChoice : undefined,
        presented: true,
        revealed: isAlreadyRevealed,
        isCorrect: false,
        missedWhileAway: isMissedWhileAway
      })
      if (isMissedWhileAway) {
        console.log(`[CONNECTION] Question ${questionIndex} marked as missed while away`)
      }
    }

    if (isAlreadyRevealed) {
      const existingHistory = questionHistory.value.find(q => q.index === questionIndex)
      if (existingHistory && existingHistory.playerChoice !== null) {
        playerGotCorrect.value = existingHistory.isCorrect
        statusMessage.value = existingHistory.isCorrect ? 'Você acertou! 🎉' : 'Não foi dessa vez!'
        statusMessageType.value = existingHistory.isCorrect ? 'success' : 'error'
      } else {
        playerGotCorrect.value = false
        statusMessage.value = 'Esta questão já foi revelada'
        statusMessageType.value = 'info'
      }
    } else if (answeredCurrentQuestion.value) {
      statusMessage.value = 'Você já respondeu esta questão'
      statusMessageType.value = 'warning'
    } else {
      statusMessage.value = 'Selecione sua resposta'
      statusMessageType.value = 'info'
    }
  })

  socket.on('questionRevealed', ({ questionIndex, question, results, answerDisplayTime, revealedCount: rc, totalQuestions: tq }) => {
    if (tq !== undefined) {
      revealedCount.value = rc || 0
      totalQuestions.value = tq
    }

    if (answerDisplayTimeout.value) {
      clearTimeout(answerDisplayTimeout.value)
      answerDisplayTimeout.value = null
    }

    answerRevealed.value = true
    currentQuestion.value.correctChoice = question.correctChoice

    const myResult = results.find(r => r.name === currentDisplayName.value)
    playerGotCorrect.value = myResult && myResult.is_correct

    const historyItem = questionHistory.value.find(q => q.index === questionIndex)
    if (historyItem) {
      historyItem.revealed = true
      historyItem.correctChoice = question.correctChoice
      historyItem.isCorrect = playerGotCorrect.value
    }

    statusMessage.value = playerGotCorrect.value ? 'Você acertou! 🎉' : 'Não foi dessa vez!'
    statusMessageType.value = playerGotCorrect.value ? 'success' : 'error'

    const displayTimeout = (answerDisplayTime || 30) * 1000
    answerDisplayTimeout.value = setTimeout(() => {
      questionDisplaying.value = false
      statusMessage.value = 'Aguardando a próxima questão...'
      statusMessageType.value = ''
      answerDisplayTimeout.value = null
    }, displayTimeout)
  })

  socket.on('roomClosed', () => {
    localStorage.removeItem('trivia_last_room')
    uiStore.addNotification('A sala foi encerrada pelo apresentador.', 'info')
    handleLeaveRoom()
  })

  socket.on('player-kicked', ({ message }) => {
    uiStore.addNotification(message, 'warning')
    handleLeaveRoom()
  })

  socket.on('roomError', (msg) => {
    if (joinRoomInProgress.value) {
      console.log('[CONNECTION] Clearing joinRoom flag due to room error')
      joinRoomInProgress.value = false
    }
    if (msg === 'Room not found.') {
      localStorage.removeItem('trivia_last_room')
    }
    uiStore.addNotification(msg, 'error')
  })

  socket.on('answerRejected', ({ message }) => {
    statusMessage.value = message
    statusMessageType.value = 'error'
    answeredCurrentQuestion.value = true
  })

  socket.on('answerHistoryRestored', ({ answerHistory }) => {
    console.log('[PLAYER] Answer history restored:', answerHistory)

    questionHistory.value = []
    answeredQuestions.clear()

    if (answerHistory && answerHistory.length > 0) {
      answerHistory.forEach((item) => {
        const { questionIndex, choice, isRevealed, isCorrect, text, choices, correctChoice } = item
        answeredQuestions.add(questionIndex)

        const questionData = {
          index: questionIndex,
          text: text,
          choices: choices,
          playerChoice: choice,
          presented: true,
          revealed: isRevealed,
          isCorrect: isCorrect || false
        }

        if (isRevealed && correctChoice !== undefined) {
          questionData.correctChoice = correctChoice
        }

        questionHistory.value.push(questionData)
      })
    }
  })

  socket.on('activeRoomsUpdate', (rooms) => {
    activeRoomCodes.value = Array.isArray(rooms) ? rooms.map(r => r.roomCode) : []
    loadRecentRooms()
  })

  socket.on('autoModeStateChanged', ({ enabled, state, timerStartedAt: newTimerStartedAt, timerDuration: newTimerDuration }) => {
    console.log('[AUTO-MODE] State changed:', { enabled, state, newTimerStartedAt, newTimerDuration })

    autoMode.value = enabled

    if (state === 'paused') {
      timerPaused.value = true
    } else {
      timerPaused.value = false

      if (newTimerStartedAt) {
        timerStartedAt.value = newTimerStartedAt
      }
      if (newTimerDuration) {
        timerDuration.value = newTimerDuration
      }
    }
  })

  socket.on('allPlayersAnswered', ({ waitSeconds }) => {
    console.log(`[AUTO-MODE] All players answered, waiting ${waitSeconds}s before reveal`)
    statusMessage.value = 'Todos os jogadores responderam! Revelando em instantes...'
    statusMessageType.value = 'info'
  })

  socket.on('quizCompleted', (data) => {
    console.log('[PLAYER] Quiz completed:', data)
    quizCompleted.value = true
    questionDisplaying.value = false

    if (data.showResults) {
      resultsCountdown.value = 5
      if (countdownInterval) clearInterval(countdownInterval)
      countdownInterval = setInterval(() => {
        resultsCountdown.value--
        if (resultsCountdown.value <= 0) {
          clearInterval(countdownInterval)
          countdownInterval = null
        }
      }, 1000)
    }
  })

  socket.on('quizResults', (data) => {
    if (data.showResults) {
      quizResultsData.value = data
      resultsCountdown.value = 0
      if (countdownInterval) {
        clearInterval(countdownInterval)
        countdownInterval = null
      }
    }
  })
}

onMounted(() => {
  debugLog('PlayerPage onMounted', {
    savedUsername: savedUsername.value,
    timestamp: new Date().toISOString()
  })

  if (savedUsername.value) {
    usernameInput.value = ''
    loginUsername.value = savedUsername.value
  } else {
    usernameInput.value = localStorage.getItem('playerUsername') || ''
  }

  displayNameInput.value = localStorage.getItem('playerDisplayName') || ''

  autoLoginRegisteredPlayer()

  debugLog('Calling socket.connect() from onMounted')
  socket.connect()

  debugLog('Calling setupSocketListeners() from onMounted')
  setupSocketListeners()

  document.addEventListener('visibilitychange', handleVisibilityChange)
  isPageVisible.value = !document.hidden
  console.log(`[CONNECTION] Initial page visibility: ${isPageVisible.value}`)

  if (socket.isConnected.value && connectionState.value !== 'connected') {
    console.log('[CONNECTION] Socket already connected on mount - updating state')
    updateConnectionState('connected')
  }

  const handleBeforeUnload = () => {
    if (inRoom.value && currentRoomCode.value) {
      localStorage.setItem('trivia_last_room', JSON.stringify({
        roomCode: currentRoomCode.value,
        username: currentUsername.value,
        displayName: currentDisplayName.value,
        timestamp: Date.now()
      }))
      console.log('[CONNECTION] Saved room state for auto-rejoin')
    }
  }
  window.addEventListener('beforeunload', handleBeforeUnload)

  const lastRoom = localStorage.getItem('trivia_last_room')
  debugLog('Checking for saved session', {
    lastRoomExists: !!lastRoom,
    timestamp: new Date().toISOString()
  })

  if (lastRoom) {
    try {
      const { roomCode, username, displayName, timestamp } = JSON.parse(lastRoom)
      const ageMinutes = (Date.now() - timestamp) / 60000

      debugLog('Parsed saved session', {
        roomCode,
        username,
        displayName,
        ageMinutes: ageMinutes.toFixed(1),
        sessionTimestamp: new Date(timestamp).toISOString()
      })

      if (ageMinutes < 5) {
        console.log(`[CONNECTION] Found recent session (${ageMinutes.toFixed(1)} min old) - attempting auto-rejoin to room ${roomCode} as ${username}/${displayName}`)
        debugLog('Session is recent - will attempt auto-rejoin', {
          roomCode,
          username,
          displayName,
          socketConnectedNow: socket.isConnected.value
        })

        roomCodeInput.value = roomCode
        currentRoomCode.value = roomCode
        currentUsername.value = username
        currentDisplayName.value = displayName

        const autoRejoinInterval = setInterval(() => {
          debugLog('Auto-rejoin interval tick', {
            socketConnected: socket.isConnected.value,
            timestamp: new Date().toISOString()
          })
          if (socket.isConnected.value) {
            clearInterval(autoRejoinInterval)
            console.log(`[CONNECTION] Socket connected - executing auto-rejoin for room ${roomCode}`)
            debugLog('Socket connected - triggering auto-rejoin now')
            emitJoinRoom(roomCode, username, displayName, 'localStorage auto-rejoin')
          }
        }, 100)

        setTimeout(() => {
          clearInterval(autoRejoinInterval)
          if (!socket.isConnected.value) {
            console.warn('[CONNECTION] Auto-rejoin timeout - socket did not connect within 10 seconds')
            debugLog('Auto-rejoin TIMEOUT - socket never connected', {
              socketConnected: socket.isConnected.value,
              timestamp: new Date().toISOString()
            })
          }
        }, 10000)
      } else {
        console.log(`[CONNECTION] Session too old (${ageMinutes.toFixed(1)} min) - clearing`)
        debugLog('Session too old - clearing localStorage', {
          ageMinutes: ageMinutes.toFixed(1)
        })
        localStorage.removeItem('trivia_last_room')
      }
    } catch (err) {
      console.error('[CONNECTION] Error parsing saved room state:', err)
      debugLog('Error parsing saved session', {
        error: err.message,
        lastRoom
      })
      localStorage.removeItem('trivia_last_room')
    }
  }

  setTimeout(() => {
    if (activeRoomCodes.value === null) {
      loadRecentRooms()
    }
  }, 3000)

  const roomFromUrl = route.query.room
  if (roomFromUrl) {
    roomCodeInput.value = roomFromUrl.toUpperCase()
    console.log(`Room code from URL: ${roomCodeInput.value}`)

    if (savedUsername.value && savedDisplayName.value) {
      setTimeout(() => {
        quickJoinRoom(roomCodeInput.value)
      }, 500)
    }
  }

  document.addEventListener('click', closeMenuIfOutside)
  document.addEventListener('touchstart', closeMenuIfOutside)
})

onUnmounted(() => {
  document.removeEventListener('click', closeMenuIfOutside)
  document.removeEventListener('touchstart', closeMenuIfOutside)
  document.removeEventListener('visibilitychange', handleVisibilityChange)

  clearTimeout(awayTimeout.value)
  clearTimeout(disconnectTimeout.value)
  clearTimeout(warningClearTimeout.value)
})

const toggleMenu = () => {
  menuOpen.value = !menuOpen.value
}

const closeMenuIfOutside = (e) => {
  const menu = document.querySelector('.menu')
  const hamburger = e.target.closest('.hamburger')
  if (menu && menu.classList && menu.classList.contains('open') && !menu.contains(e.target) && !hamburger) {
    menuOpen.value = false
  }
}

const loadRecentRooms = () => {
  const stored = localStorage.getItem('playerRecentRooms')
  let rooms = stored ? JSON.parse(stored) : []

  if (activeRoomCodes.value !== null) {
    const filteredRooms = rooms.filter(room => activeRoomCodes.value.includes(room.code))
    if (filteredRooms.length < rooms.length) {
      localStorage.setItem('playerRecentRooms', JSON.stringify(filteredRooms))
    }
    recentRooms.value = filteredRooms
  } else {
    recentRooms.value = rooms
  }
}

const saveRecentRoom = (roomCode) => {
  let rooms = recentRooms.value
  rooms = rooms.filter(room => room.code !== roomCode)
  rooms.unshift({
    code: roomCode,
    timestamp: Date.now(),
    username: currentUsername.value,
    displayName: currentDisplayName.value
  })
  rooms = rooms.slice(0, 5)
  recentRooms.value = rooms
  localStorage.setItem('playerRecentRooms', JSON.stringify(rooms))
}

const quickJoinRoom = async (roomCode) => {
  const displayName = savedDisplayName.value || displayNameInput.value.trim()

  if (!savedUsername.value) {
    uiStore.addNotification('Por favor, informe seu usuário primeiro.', 'warning')
    return
  }

  if (!displayName) {
    uiStore.addNotification('Por favor, informe o nome de exibição.', 'warning')
    return
  }

  roomCodeInput.value = roomCode
  currentUsername.value = savedUsername.value
  currentDisplayName.value = displayName
  currentRoomCode.value = roomCode
  emitJoinRoom(roomCode, savedUsername.value, displayName, 'quick join')
}

const handleChangeUsername = async () => {
  showChangeUsernameModal.value = true
}

const confirmChangeUsername = () => {
  showChangeUsernameModal.value = false
  localStorage.removeItem('playerUsername')
  localStorage.removeItem('playerAccountType')
  localStorage.removeItem('playerAuthToken')
  savedUsername.value = null
  usernameInput.value = ''
}

const selectAnswer = async (idx) => {
  if (answeredCurrentQuestion.value || answerRevealed.value) {
    console.log(`[ANSWER] Blocked - already answered (${answeredCurrentQuestion.value}) or revealed (${answerRevealed.value})`)
    return
  }

  if (joinRoomInProgress.value) {
    console.warn('[ANSWER] ❌ BLOCKED - joinRoom still in progress. User will see "Reconnecting..." message')
    statusMessage.value = 'Reconectando... aguarde'
    statusMessageType.value = 'warning'
    return
  }

  console.log(`[ANSWER] Player selected answer ${idx}, showing confirmation modal`)
  pendingAnswerIndex.value = idx
  selectedAnswer.value = idx
  showAnswerConfirmModal.value = true
}

const confirmAnswer = () => {
  const idx = pendingAnswerIndex.value
  if (idx === null) return

  console.log(`[ANSWER] ✅ Submitting confirmed answer ${idx} for question ${currentQuestion.value.index}`)
  answeredCurrentQuestion.value = true
  answeredQuestions.add(currentQuestion.value.index)

  const historyItem = questionHistory.value.find(q => q.index === currentQuestion.value.index)
  if (historyItem) {
    historyItem.playerChoice = idx
    historyItem.missedWhileAway = false
  }

  socket.emit('submitAnswer', { roomCode: currentRoomCode.value, choice: idx })
  statusMessage.value = 'Resposta enviada! ✓'
  statusMessageType.value = 'success'

  showAnswerConfirmModal.value = false
  pendingAnswerIndex.value = null
}

const cancelAnswer = () => {
  console.log('[ANSWER] Player cancelled answer selection')
  selectedAnswer.value = null
  pendingAnswerIndex.value = null
  showAnswerConfirmModal.value = false
}

const emitJoinRoom = (roomCode, username, displayName, source = '') => {
  debugLog('emitJoinRoom called', {
    roomCode,
    username,
    displayName,
    source,
    socketConnected: socket.isConnected.value,
    joinRoomInProgress: joinRoomInProgress.value,
    timestamp: new Date().toISOString()
  })

  const now = Date.now()
  if (now - lastJoinRoomAttempt.value < 2000) {
    console.log(`[CONNECTION] Skipping duplicate joinRoom from ${source} (debounced)`)
    debugLog('joinRoom DEBOUNCED', {
      timeSinceLastAttempt: now - lastJoinRoomAttempt.value
    })
    return false
  }
  lastJoinRoomAttempt.value = now

  joinRoomInProgress.value = true
  console.log(`[CONNECTION] Emitting joinRoom from ${source} - flag set to true`)
  debugLog('joinRoom emitting to server', {
    roomCode,
    username,
    displayName,
    source
  })

  setTimeout(() => {
    if (joinRoomInProgress.value) {
      console.warn('[CONNECTION] joinRoom flag still true after 5s - auto-clearing (safety mechanism)')
      debugLog('joinRoom safety timeout triggered - flag still true after 5s')
      joinRoomInProgress.value = false
    }
  }, 5000)

  const playerID = socket.getPlayerID()
  debugLog('PlayerID for joinRoom:', playerID)

  socket.emit('joinRoom', { roomCode, username, displayName, playerID })
  socket.setRoomContext(roomCode, username)

  return true
}

const handleJoinRoom = async () => {
  const username = savedUsername.value || usernameInput.value.trim()
  const displayName = displayNameInput.value.trim()
  const roomCode = roomCodeInput.value.trim()

  if (DEBUG) {
    console.log('[JOIN] handleJoinRoom called:', {
      username,
      displayName,
      roomCode,
      isMobile: /Mobile|Android|iPhone|iPad/i.test(navigator.userAgent)
    })
  }

  if (!username) {
    uiStore.addNotification('Por favor, informe um usuário.', 'warning')
    return
  }

  if (!displayName) {
    uiStore.addNotification('Por favor, informe o nome exibido.', 'warning')
    return
  }

  if (!roomCode) {
    uiStore.addNotification('Por favor, informe o código da sala.', 'warning')
    return
  }

  try {
    if (DEBUG) console.log('[JOIN] Calling check-username API')
    const response = await post('/api/auth/check-username', { username })
    if (DEBUG) console.log('[JOIN] check-username response:', response.data)

    if (response.data.requiresAuth) {
      if (DEBUG) console.log('[JOIN] Auth required, verifying token')
      const hasToken = await verifyAuthToken()
      if (!hasToken) {
        if (DEBUG) console.log('[JOIN] No valid token, showing login modal')
        loginUsername.value = username
        loginError.value = ''
        showLoginModal.value = true
        return
      }
      if (DEBUG) console.log('[JOIN] Token verified, proceeding with join')
    } else {
      if (DEBUG) console.log('[JOIN] Guest account, no auth required')
      localStorage.setItem('playerUsername', username)
      localStorage.setItem('playerAccountType', 'guest')
      savedUsername.value = username
    }

    localStorage.setItem('playerDisplayName', displayName)
    currentUsername.value = username
    currentDisplayName.value = displayName
    currentRoomCode.value = roomCode
    if (DEBUG) console.log('[JOIN] Emitting joinRoom event')
    emitJoinRoom(roomCode, username, displayName, 'manual join')
  } catch (err) {
    if (DEBUG) {
      console.error('[JOIN] Error in handleJoinRoom:', err)
      console.error('[JOIN] Error details:', {
        message: err.message,
        response: err.response,
        stack: err.stack
      })
    }
    const errorMsg = err.response?.data?.error || err.message || 'Erro desconhecido'
    uiStore.addNotification(`Falha ao entrar: ${errorMsg}`, 'error')
  }
}

const handleLoginSubmit = async (password) => {
  try {
    const response = await post('/api/auth/player-login', {
      username: loginUsername.value,
      password: password
    })

    localStorage.setItem('playerAuthToken', response.data.token)
    localStorage.setItem('playerUsername', loginUsername.value)
    localStorage.setItem('playerAccountType', 'registered')
    savedUsername.value = loginUsername.value

    showLoginModal.value = false

    const displayName = displayNameInput.value.trim()
    const roomCode = roomCodeInput.value.trim()
    currentUsername.value = loginUsername.value
    currentDisplayName.value = displayName
    currentRoomCode.value = roomCode
    localStorage.setItem('playerDisplayName', displayName)
    saveRecentRoom(roomCode)
    emitJoinRoom(roomCode, loginUsername.value, displayName, 'login submit')
  } catch (err) {
    if (err.response?.status === 428 || err.response?.data?.requiresPasswordReset) {
      console.log('[AUTH] Password reset required for user:', loginUsername.value)
      showLoginModal.value = false
      setPasswordUsername.value = loginUsername.value
      setPasswordError.value = ''
      showSetPasswordModal.value = true
      return
    }

    loginError.value = err.response?.data?.error || 'Senha inválida'
  }
}

const loginCancelled = () => {
  showLoginModal.value = false
}

const handleSetPasswordSubmit = async ({ newPassword, confirmPassword }) => {
  if (newPassword !== confirmPassword) {
    setPasswordError.value = 'As senhas não coincidem!'
    return
  }

  if (newPassword.length < 6) {
    setPasswordError.value = 'A senha deve ter pelo menos 6 caracteres'
    return
  }

  try {
    const response = await post('/api/auth/set-new-password', {
      username: setPasswordUsername.value,
      password: newPassword
    })

    localStorage.setItem('playerAuthToken', response.data.token)
    localStorage.setItem('playerUsername', setPasswordUsername.value)
    localStorage.setItem('playerAccountType', 'registered')
    savedUsername.value = setPasswordUsername.value

    showSetPasswordModal.value = false

    const displayName = displayNameInput.value.trim()
    const roomCode = roomCodeInput.value.trim()
    currentUsername.value = setPasswordUsername.value
    currentDisplayName.value = displayName
    currentRoomCode.value = roomCode
    localStorage.setItem('playerDisplayName', displayName)
    saveRecentRoom(roomCode)
    emitJoinRoom(roomCode, setPasswordUsername.value, displayName, 'password setup')
  } catch (err) {
    setPasswordError.value = err.response?.data?.error || 'Falha ao definir a senha'
  }
}

const passwordSetupCancelled = () => {
  showSetPasswordModal.value = false
}

const handleManageAccount = () => {
  const username = savedUsername.value || usernameInput.value.trim()

  if (!username) {
    uiStore.addNotification('Por favor, informe um usuário.', 'warning')
    return
  }

  if (!savedUsername.value) {
    localStorage.setItem('playerUsername', username)
    localStorage.setItem('playerAccountType', 'guest')
    savedUsername.value = username
  }

  router.push('/manage')
}

const handleLeaveRoomClick = async () => {
  showLeaveRoomConfirmModal.value = true
}

const confirmLeaveRoom = () => {
  showLeaveRoomConfirmModal.value = false
  handleLeaveRoom()
}

const handleLeaveRoom = () => {
  inRoom.value = false
  currentRoomCode.value = null
  currentUsername.value = null
  currentDisplayName.value = null
  questionDisplaying.value = false
  selectedAnswer.value = null
  answeredQuestions.clear()
  questionHistory.value = []
  currentPlayers.value = []
  quizResultsData.value = null
  quizCompleted.value = false
  resultsCountdown.value = 0
  if (countdownInterval) { clearInterval(countdownInterval); countdownInterval = null }
  revealedCount.value = 0
  totalQuestions.value = 0
  autoMode.value = false
  timerStartedAt.value = null
  timerDuration.value = null
  timerPaused.value = false
  statusMessage.value = 'Pronto para entrar'
  statusMessageType.value = ''
  menuOpen.value = false

  if (answerDisplayTimeout.value) {
    clearTimeout(answerDisplayTimeout.value)
    answerDisplayTimeout.value = null
  }

  if (wakeLock.isActive.value) {
    wakeLock.releaseWakeLock()
  }

  socket.clearRoomContext()
  socket.disconnect()

  setTimeout(() => {
    socket.connect()
    setupSocketListeners()
  }, 100)

  loadRecentRooms()
}

const handleLogout = () => {
  showLogoutConfirmModal.value = true
}

const confirmLogout = async () => {
  showLogoutConfirmModal.value = false

  if (inRoom.value) {
    handleLeaveRoom()
  }

  localStorage.removeItem('playerUsername')
  localStorage.removeItem('playerDisplayName')
  localStorage.removeItem('playerAccountType')
  localStorage.removeItem('playerAuthToken')
  localStorage.removeItem('playerRecentRooms')

  loginUsername.value = ''
  usernameInput.value = ''
  displayNameInput.value = ''
  roomCodeInput.value = ''
  savedUsername.value = null
  savedDisplayName.value = null
  savedAccountType.value = null
  questionHistory.value = []
  recentRooms.value = []

  uiStore.addNotification('Sessão encerrada com sucesso', 'info')
}

const showProgressModal = () => {
  showProgressModalFlag.value = true
}

const forceReconnect = () => {
  console.log('[CONNECTION] Manual reconnection triggered')
  socket.disconnect()
  setTimeout(() => {
    socket.connect()
    setupSocketListeners()
  }, 100)
}

const autoLoginRegisteredPlayer = async () => {
  const token = localStorage.getItem('playerAuthToken')
  if (token && savedAccountType.value === 'registered') {
    try {
      await post('/api/auth/verify-player', {})
    } catch (err) {
      localStorage.removeItem('playerAuthToken')
    }
  }
}

const verifyAuthToken = async () => {
  const token = localStorage.getItem('playerAuthToken')
  if (!token) return false

  try {
    await post('/api/auth/verify-player', {})
    return true
  } catch (err) {
    localStorage.removeItem('playerAuthToken')
    return false
  }
}
</script>

<style scoped>
.player-page {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  background: var(--bg-primary);
  color: var(--text-primary);
}

.player-container {
  display: flex;
  flex: 1 1 auto;
  gap: 1rem;
  padding: 1rem;
  position: relative;
  box-sizing: border-box;
}

.connection-lost-banner {
  position: fixed;
  top: 60px;
  left: 0;
  right: 0;
  background: var(--danger-color);
  color: var(--text-primary);
  padding: 1rem;
  z-index: 9999;
  animation: slideDown 0.3s ease;
}

.banner-content {
  display: flex;
  align-items: center;
  gap: 1rem;
  max-width: 1200px;
  margin: 0 auto;
}

.banner-content .icon {
  font-size: 1.5rem;
}

.banner-content .text {
  flex: 1;
}

.banner-content .text strong {
  display: block;
  font-size: 1.1rem;
  margin-bottom: 0.25rem;
}

.banner-content .text p {
  margin: 0;
  font-size: 0.9rem;
  opacity: 0.9;
}

.reconnect-btn {
  margin-left: auto;
}

.missed-questions-banner {
  position: fixed;
  top: 120px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--warning-color);
  color: var(--text-primary);
  padding: 1rem 2rem;
  border-radius: 8px;
  font-size: 1.1rem;
  font-weight: bold;
  box-shadow: var(--shadow-lg);
  z-index: 1000;
  animation: slideDown 0.3s ease-out;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.question-area {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 0;
  box-sizing: border-box;
}

.quiz-complete-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  padding: 2rem;
  text-align: center;
  animation: fadeInUp 0.5s ease;
}

.complete-icon {
  color: var(--secondary-light);
  filter: drop-shadow(0 0 10px rgba(34, 197, 94, 0.4));
  font-size: 3rem;
}

.complete-title {
  font-size: clamp(1.5rem, 5vw, 2rem);
  font-weight: 700;
  margin: 0;
  color: var(--text-primary);
}

.complete-subtitle {
  font-size: 1.1rem;
  color: var(--text-secondary);
  margin: 0;
}

.countdown-number {
  display: inline-block;
  font-size: 1.4rem;
  font-weight: 700;
  color: var(--info-light);
  min-width: 1.5ch;
  font-variant-numeric: tabular-nums;
  animation: countPulse 1s ease-in-out infinite;
}

@keyframes countPulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.15); }
}

@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.sidebar {
  width: 300px;
  min-width: 300px;
  flex-shrink: 0;
  background: var(--bg-secondary);
  padding: 1.5rem;
  border-radius: 15px;
  border: 1px solid var(--border-color);
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

@media (max-width: 768px) {
  .player-container {
    flex-direction: column;
  }

  .question-area {
    min-height: auto;
    flex: 0 0 auto;
  }

  .sidebar {
    width: 100%;
    min-width: unset;
    flex: 1;
    overflow-y: auto;
  }

  .sidebar.hidden-mobile-in-room {
    display: none;
  }
}
</style>
