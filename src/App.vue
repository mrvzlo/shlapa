<template>
  <main class="game-shell">
    <header class="topbar">
      <div class="scoreboard" aria-label="Счёт команд">
        <div class="team-card team-card--one">
          <strong class="team-score">{{ scores[0] }}</strong>
        </div>
        <div class="score-divider" aria-hidden="true"><span>:</span></div>
        <div class="team-card team-card--two">
          <strong class="team-score">{{ scores[1] }}</strong>
        </div>
        <div class="score-divider" aria-hidden="true"><span>:</span></div>
        <div class="team-card team-card--three">
          <strong class="team-score">{{ scores[2] }}</strong>
        </div>
      </div>
      <div class="game-meta">
        <span class="phase-pill">ФАЗА {{ phase }}</span>
        <span class="round-label">Раунд {{ roundNumber }}</span>
      </div>
      <button class="phase-action" type="button" :disabled="gameFinished" @click="showPhaseDialog = true">
        Завершить фазу <span aria-hidden="true">↗</span>
      </button>
    </header>

    <section class="game-content">
      <div class="timer-area" :class="{ 'timer-area--urgent': timeLeft <= 5 && roundActive }">
        <div class="timer-caption">
          {{ roundActive ? "" : currentWord ? "ВРЕМЯ ВЫШЛО" : roundFinished ? "РАУНД ЗАВЕРШЁН" : "" }}
        </div>
        <div class="timer-track" aria-hidden="true">
          <span :style="{ width: `${(timeLeft / ROUND_LENGTH) * 100}%` }"></span>
        </div>
      </div>

      <section class="word-stage" aria-live="polite">
        <div class="word-content">
          <template v-if="currentWord">
            <h1 class="word">{{ currentWord }}</h1>
          </template>
          <template v-else-if="roundFinished">
            <h1 class="word word--message">{{ wordsLeft === 0 ? "Слова закончились" : "Раунд завершён" }}</h1>
          </template>
          <template v-else-if="gameFinished">
            <span class="word-kicker">ИГРА ОКОНЧЕНА</span>
          </template>
          <template v-else>
            <span class="word-kicker">ФАЗА {{ phase }}</span>
          </template>
        </div>

        <div v-if="currentWord" class="answer-actions">
          <button class="pass-button" type="button" @click="passWord">Пас</button>
          <button class="answer-button answer-button--one" type="button" @click="awardWord(0)">
            <span class="button-team">01</span>
          </button>
          <button class="answer-button answer-button--two" type="button" @click="awardWord(1)">
            <span class="button-team">02</span>
          </button>
          <button class="answer-button answer-button--three" type="button" @click="awardWord(2)">
            <span class="button-team">03</span>
          </button>
        </div>
        <div v-else class="round-actions">
          <button v-if="!gameFinished && roundFinished && wordsLeft > 0" class="primary-button" type="button" @click="startRound">
            Следующий раунд <span aria-hidden="true">→</span>
          </button>
          <button
            v-else-if="!gameFinished && !roundFinished"
            class="primary-button"
            type="button"
            :disabled="wordsLeft === 0"
            @click="startRound"
          >
            Начать раунд <span aria-hidden="true">→</span>
          </button>
          <button
            v-else-if="!gameFinished && wordsLeft === 0 && guessedWords.length > 0"
            class="primary-button"
            type="button"
            @click="showPhaseDialog = true"
          >
            Начать следующую фазу <span aria-hidden="true">→</span>
          </button>
          <button v-else-if="!gameFinished && wordsLeft === 0" class="primary-button" type="button" disabled>
            Нет слов для следующей фазы
          </button>
          <button v-else class="primary-button" type="button" @click="restartGame">Играть ещё раз <span aria-hidden="true">↻</span></button>
          <span v-if="!gameFinished" class="round-note"> {{ wordsLeft }} {{ wordForm(wordsLeft) }} в шляпе </span>
        </div>
      </section>
    </section>

    <div v-if="showPhaseDialog" class="dialog-backdrop" @click.self="showPhaseDialog = false">
      <section class="phase-dialog" role="dialog" aria-modal="true" aria-labelledby="dialog-title">
        <button class="dialog-close" type="button" aria-label="Закрыть" @click="showPhaseDialog = false">×</button>
        <span class="dialog-kicker">СМЕНА ФАЗЫ</span>
        <h2 id="dialog-title">Закончить фазу {{ phase }}?</h2>
        <p>
          {{
            guessedWords.length
              ? `Неугаданные слова исчезнут. ${guessedWords.length} угаданных слов перемешаются для фазы ${phase + 1}.`
              : "Угаданных слов пока нет. Если закончить фазу, игра завершится."
          }}
        </p>
        <div class="dialog-actions">
          <button class="dialog-cancel" type="button" @click="showPhaseDialog = false">Продолжить игру</button>
          <button class="dialog-confirm" type="button" @click="startNextPhase">
            {{ guessedWords.length ? "Начать следующую фазу" : "Закончить игру" }}
          </button>
        </div>
      </section>
    </div>
  </main>
</template>

<script>
import wordList from "./data/words.json";

const ROUND_LENGTH = 30;

function shuffle(items) {
  const shuffled = [...items];

  for (let index = shuffled.length - 1; index > 0; index -= 1) {
    const randomIndex = Math.floor(Math.random() * (index + 1));
    [shuffled[index], shuffled[randomIndex]] = [shuffled[randomIndex], shuffled[index]];
  }

  return shuffled;
}

export default {
  name: "App",
  data() {
    return {
      ROUND_LENGTH,
      phase: 1,
      roundNumber: 1,
      scores: [0, 0, 0],
      remainingWords: shuffle(wordList),
      guessedWords: [],
      currentWord: null,
      timeLeft: ROUND_LENGTH,
      roundActive: false,
      roundFinished: false,
      gameFinished: false,
      showPhaseDialog: false,
      timerId: null,
    };
  },
  computed: {
    wordsLeft() {
      return this.remainingWords.length + (this.currentWord ? 1 : 0);
    },
  },
  beforeUnmount() {
    this.clearTimer();
  },
  methods: {
    wordForm(count) {
      const remainder = count % 10;
      const lastTwoDigits = count % 100;
      if (lastTwoDigits >= 11 && lastTwoDigits <= 14) return "слов";
      if (remainder === 1) return "слово";
      if (remainder >= 2 && remainder <= 4) return "слова";
      return "слов";
    },
    drawWord() {
      if (this.remainingWords.length === 0) {
        this.currentWord = null;
        this.finishRound();
        return;
      }

      const [nextWord, ...words] = this.remainingWords;
      this.currentWord = nextWord;
      this.remainingWords = words;
    },
    startRound() {
      if (this.roundActive || this.gameFinished || this.remainingWords.length === 0) return;

      this.timeLeft = ROUND_LENGTH;
      this.roundFinished = false;
      this.roundActive = true;
      this.drawWord();
      if (this.roundActive) {
        this.timerId = window.setInterval(() => {
          if (this.timeLeft > 0) {
            this.timeLeft -= 1;
            if (this.timeLeft === 0) {
              this.roundActive = false;
              this.clearTimer();
            }
          }
        }, 1000);
      }
    },
    awardWord(teamIndex) {
      if (!this.currentWord) return;

      this.scores[teamIndex] += 1;
      this.guessedWords.push(this.currentWord);
      this.currentWord = null;
      if (this.roundActive) {
        this.drawWord();
      } else {
        this.finishRound();
      }
    },
    passWord() {
      if (!this.currentWord) return;

      this.remainingWords.push(this.currentWord);
      this.currentWord = null;
      if (this.roundActive) {
        this.drawWord();
      } else {
        this.finishRound();
      }
    },
    finishRound() {
      this.clearTimer();
      this.roundActive = false;
      this.roundFinished = true;
      if (this.wordsLeft === 0 && this.guessedWords.length === 0) {
        this.gameFinished = true;
      }
    },
    clearTimer() {
      if (this.timerId !== null) {
        window.clearInterval(this.timerId);
        this.timerId = null;
      }
    },
    startNextPhase() {
      this.clearTimer();
      this.showPhaseDialog = false;
      this.currentWord = null;
      this.roundActive = false;
      this.roundFinished = false;

      if (this.guessedWords.length === 0) {
        this.remainingWords = [];
        this.gameFinished = true;
        return;
      }

      this.phase += 1;
      this.roundNumber = 1;
      this.remainingWords = shuffle(this.guessedWords);
      this.guessedWords = [];
      this.timeLeft = ROUND_LENGTH;
      this.gameFinished = false;
    },
    restartGame() {
      this.clearTimer();
      this.phase = 1;
      this.roundNumber = 1;
      this.scores = [0, 0];
      this.remainingWords = shuffle(wordList);
      this.guessedWords = [];
      this.currentWord = null;
      this.timeLeft = ROUND_LENGTH;
      this.roundActive = false;
      this.roundFinished = false;
      this.gameFinished = false;
    },
  },
  watch: {
    roundFinished(finished) {
      if (finished && this.wordsLeft > 0) this.roundNumber += 1;
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=Manrope:wght@400;500;600;700;800&display=swap");

:root {
  touch-action: pan-x pan-y;
  font-family: "DM Sans", sans-serif;
  color: #242323;
  background: #f7f5f0;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  font-optical-sizing: auto;
  font-weight: 400;
  --ink: #242323;
  --muted: #898781;
  --paper: #f7f5f0;
  --coral: #e97150;
  --violet: #7765bd;
}

* {
  box-sizing: border-box;
}

body {
  min-width: 320px;
  min-height: 100vh;
  margin: 0;
}

button {
  font: inherit;
}

button:focus-visible,
a:focus-visible {
  outline: 3px solid #e97150;
  outline-offset: 3px;
}

.game-shell {
  min-height: 100vh;
  background: radial-gradient(ellipse at 50% 40%, rgba(255, 255, 255, 0.9), transparent 58%), var(--paper);
}

.topbar {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  align-items: center;
  min-height: 82px;
  padding: 0 6.1%;
  border-bottom: 1px solid #e9e6df;
}

.game-meta {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 15px;
}

.phase-pill {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 8px 11px;
  border: 1px solid #e8e4dc;
  border-radius: 30px;
  color: #55524c;
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.08em;
}

.round-label {
  color: var(--muted);
  font-size: 12px;
}

.phase-action {
  justify-self: end;
  padding: 9px 0;
  border: 0;
  color: #6d6962;
  background: transparent;
  cursor: pointer;
  font-size: 12px;
  transition: color 160ms ease;
}

.phase-action:hover:not(:disabled) {
  color: var(--coral);
}

.phase-action span {
  margin-left: 5px;
}

.phase-action:disabled {
  cursor: not-allowed;
  opacity: 0.45;
}

.game-content {
  width: calc(100% - 40px);
  margin: 0 auto;
}

.timer-area {
  width: 100%;
  margin: 15px auto;
  text-align: center;
}

.timer-caption {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 7px;
  color: #89857f;
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.15em;
}

.timer-track {
  height: 15px;
  overflow: hidden;
  border-radius: 5px;
  background: #e8e4dc;
}

.timer-track span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: var(--coral);
  transition: width 0.9s linear;
}

.timer-area--urgent .timer {
  color: var(--coral);
}

.scoreboard {
  display: grid;
  grid-template-columns: 1fr 24px 1fr 24px 1fr;
  align-items: center;
  gap: 9px;
  margin: 4px 0;
}

.team-card {
  padding: 4px 4px;
  border: 1px solid #eae7e0;
  border-radius: 14px;
  text-align: center;
  background: rgba(255, 255, 255, 0.68);
}

.team-score {
  font-family: "Manrope", sans-serif;
  font-size: 29px;
  font-weight: 700;
  letter-spacing: -0.06em;
}

.team-card--one .team-score {
  color: #d97153;
}

.team-card--two .team-score {
  color: #7866be;
}

.team-card--three .team-score {
  color: #66be83;
}

.score-divider {
  color: #c3beb4;
  font-size: 19px;
  text-align: center;
}

.word-stage {
  overflow: hidden;
  border: 1px solid #e6e2d9;
  border-radius: 19px;
  background: #fffefa;
  box-shadow: 0 12px 40px rgba(62, 53, 38, 0.045);
}

.word-content {
  display: flex;
  min-height: 60vh;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 29px 20px 24px;
  text-align: center;
}

.word-kicker,
.dialog-kicker {
  color: #c0a184;
  font-size: 9px;
  font-weight: 700;
  letter-spacing: 0.18em;
}

.word {
  margin: 8px 0 5px;
  color: #252321;
  font-family: "Manrope", sans-serif;
  font-size: 100px;
  font-weight: 800;
  letter-spacing: -0.065em;
  line-height: 1.16;
}

.word--message {
  font-size: clamp(30px, 5vw, 46px);
}

.word-hint {
  margin: 4px 0 0;
  color: #99948b;
  font-size: 12px;
}

.answer-actions {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 10px;
  padding: 0 22px 18px;
}

.answer-button {
  min-height: 51px;
  padding: 0 13px;
  border: 1px solid transparent;
  border-radius: 10px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  text-align: center;
  transition: transform 160ms ease, filter 160ms ease, box-shadow 160ms ease;
}

.answer-button--one {
  color: #9e4e38;
  border-color: #f3d9d0;
  background: #fdf2ed;
}

.answer-button--two {
  color: #5e4d9d;
  border-color: #e2dcf6;
  background: #f4f1fc;
}

.answer-button--three {
  color: #4d9d7e;
  border-color: #dcf6ed;
  background: #f1fcf8;
}

.button-arrow {
  margin-left: auto;
  font-size: 16px;
}

.pass-button {
  padding: 4px 12px;
  border: 0;
  border-radius: 10px;
  color: #222;
  background: #ddd;
  cursor: pointer;
  font-size: 14px;
  transition: color 160ms ease;
}

.pass-button span {
  margin-left: 3px;
  font-size: 14px;
}

.round-actions {
  display: flex;
  min-height: 74px;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 0 20px 16px;
}

.primary-button {
  display: inline-flex;
  min-height: 43px;
  align-items: center;
  justify-content: center;
  gap: 14px;
  padding: 0 20px;
  border: 0;
  border-radius: 9px;
  color: white;
  background: var(--ink);
  cursor: pointer;
  font-size: 12px;
  font-weight: 600;
  transition: transform 160ms ease, background 160ms ease;
}

.primary-button:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.primary-button span {
  font-size: 16px;
}

.round-note {
  color: #aaa59c;
  font-size: 10px;
}

.dialog-backdrop {
  position: fixed;
  z-index: 2;
  display: grid;
  padding: 20px;
  place-items: center;
  background: rgba(34, 32, 29, 0.35);
  inset: 0;
}

.phase-dialog {
  position: relative;
  width: min(100%, 430px);
  padding: 34px;
  border: 1px solid #e8e3da;
  border-radius: 17px;
  background: #fffefa;
  box-shadow: 0 24px 80px rgba(38, 34, 27, 0.2);
}

.dialog-close {
  position: absolute;
  top: 13px;
  right: 16px;
  border: 0;
  color: #9a958d;
  background: transparent;
  cursor: pointer;
  font-size: 24px;
}

.phase-dialog h2 {
  margin: 10px 0 8px;
  font-family: "Manrope", sans-serif;
  font-size: 24px;
  letter-spacing: -0.04em;
}

.phase-dialog p {
  margin: 0;
  color: #858078;
  font-size: 13px;
  line-height: 1.6;
}

.dialog-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 9px;
  margin-top: 25px;
}

.dialog-cancel,
.dialog-confirm {
  min-height: 40px;
  padding: 0 14px;
  border: 1px solid #e9e5dd;
  border-radius: 8px;
  cursor: pointer;
  font-size: 11px;
  font-weight: 600;
}

.dialog-cancel {
  color: #706b63;
  background: transparent;
}

.dialog-confirm {
  margin-left: auto;
  border-color: var(--ink);
  color: white;
  background: var(--ink);
}
</style>
