<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import { io, type Socket } from 'socket.io-client';

type Word = {
  word: string;
  difficulty: 'easy' | 'medium' | 'hard';
};

const socket = ref<Socket | null>(null);
const currentWord = ref<Word | null>(null);
const guess = ref('');
const result = ref<boolean | null>(null);
const isLoading = ref(true);
const error = ref<string | null>(null);

// Initialize Socket.IO
onMounted(() => {
  try {
    socket.value = io({
      reconnectionAttempts: 3,
      timeout: 5000
    });

    socket.value.on('connect', () => {
      isLoading.value = false;
      socket.value?.emit('request-word');
    });

    socket.value.on('new-word', (word: Word) => {
      currentWord.value = word;
      guess.value = '';
      result.value = null;
    });

    socket.value.on('guess-result', (data: { isCorrect: boolean }) => {
      result.value = data.isCorrect;
    });

    socket.value.on('connect_error', (err) => {
      error.value = `Connection failed: ${err.message}`;
      isLoading.value = false;
    });

  } catch (err) {
    error.value = 'Failed to initialize socket';
    isLoading.value = false;
  }
});

// Clean up on unmount
onBeforeUnmount(() => {
  socket.value?.disconnect();
});

const submitGuess = () => {
  if (socket.value?.connected && guess.value.trim()) {
    socket.value.emit('submit-guess', { word: guess.value.trim() });
  }
};

const nextWord = () => {
  if (socket.value?.connected) {
    socket.value.emit('request-word');
  }
};
</script>

<template>
  <div>
    <h1>Spelling Bee</h1>
    
    <div v-if="isLoading">Loading game...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    
    <div v-else>
      <div v-if="currentWord">
        <p>Spell this word: <strong>{{ currentWord.word }}</strong></p>
        <input 
          v-model="guess" 
          @keyup.enter="submitGuess" 
          placeholder="Type the word..."
          :disabled="result !== null"
        />
        <button @click="submitGuess" :disabled="!guess.trim() || result !== null">
          Submit
        </button>
        <button @click="nextWord">
          Next Word
        </button>
      </div>
      <p v-if="result !== null" :class="result ? 'correct' : 'wrong'">
        {{ result ? '✅ Correct!' : '❌ Wrong! Try again.' }}
      </p>
    </div>
  </div>
</template>

<style scoped>
.correct { color: green; }
.wrong { color: red; }
.error { color: crimson; }
input { margin: 0 10px; }
button:disabled { opacity: 0.5; }
</style>