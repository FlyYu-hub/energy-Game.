<template>
  <div class="app">
    <header class="topbar">
      <div class="title-row">
        <h1>能源發電方式小遊戲</h1>

        <label class="demo">
          <input type="checkbox" v-model="demoMode" />
          DEMO 模式
        </label>
      </div>

      <nav class="nav">
        <button @click="go('home')" :class="{ active: page === 'home' }">首頁</button>
        <button @click="go('memory')" :class="{ active: page === 'memory' }">翻牌配對</button>
        <button @click="go('quiz')" :class="{ active: page === 'quiz' }">問答</button>
        <button @click="go('result')" :class="{ active: page === 'result' }">結果</button>
      </nav>
    </header>

    <main class="content">
      <!-- HOME -->
      <section v-if="page === 'home'">
        <h2>能源發電方式</h2>
        <p class="muted">
          點卡片查看簡介；也可以直接開始「翻牌配對」或「問答」。
        </p>

        <div class="home-grid">
          <button v-for="e in energies" :key="e.id" class="energy-card" @click="openEnergy(e.id)">
            <div class="energy-icon">{{ e.icon }}</div>
            <div class="energy-name">{{ e.name }}</div>
            <div class="energy-intro">{{ e.intro }}</div>
            <div class="kw">
              <span v-for="k in e.keywords" :key="k" class="chip">{{ k }}</span>
            </div>
          </button>
        </div>

        <div class="actions">
          <button class="primary" @click="go('memory')">開始翻牌</button>
          <button class="primary" @click="go('quiz')">開始問答</button>
        </div>

        <!-- 詳情彈窗 -->
        <div v-if="selectedEnergy" class="modal-backdrop" @click.self="closeEnergy">
          <div class="modal">
            <div class="row">
              <h3 style="margin:0">{{ selectedEnergy.icon }} {{ selectedEnergy.name }}</h3>
              <button class="ghost" @click="closeEnergy">關閉</button>
            </div>

            <p class="muted">{{ selectedEnergy.intro }}</p>

            <div class="two-col">
              <div class="box">
                <strong>優點</strong>
                <ul>
                  <li v-for="p in selectedEnergy.pros" :key="p">{{ p }}</li>
                </ul>
              </div>
              <div class="box">
                <strong>限制 / 缺點</strong>
                <ul>
                  <li v-for="c in selectedEnergy.cons" :key="c">{{ c }}</li>
                </ul>
              </div>
            </div>

            <div class="actions">
              <button class="primary" @click="go('memory'); closeEnergy()">用翻牌學這些</button>
              <button class="primary" @click="go('quiz'); closeEnergy()">用問答測看看</button>
            </div>
          </div>
        </div>
      </section>

      <!-- MEMORY GAME -->
      <section v-else-if="page === 'memory'">
        <div class="row">
          <h2>翻牌配對</h2>
          <div class="row-actions">
            <button class="ghost" @click="resetMemory">重新開始</button>
            <button class="ghost" v-if="demoMode" @click="demoFinishMemory">DEMO：快速完成</button>
          </div>
        </div>

        <p class="muted">規則：每次翻兩張牌，如果是同一種能源就配對成功（圖卡 + 關鍵字卡）。</p>

        <div class="status" v-if="memoryStatus">
          {{ memoryStatus }}
        </div>

        <div class="grid">
          <button
            v-for="card in cards"
            :key="card.uid"
            class="card"
            :class="{ flipped: card.flipped || card.matched, matched: card.matched }"
            @click="flipCard(card)"
          >
            <div class="inner">
              <div class="front">?</div>
              <div class="back">
                <div class="name">{{ card.name }}</div>

                <div v-if="card.cardType === 'img'" class="bigicon">
                  {{ card.icon }}
                </div>

                <div v-else class="kw">
                  <span v-for="k in card.keywords" :key="k" class="chip">{{ k }}</span>
                </div>

                <div class="hint">{{ card.intro }}</div>
              </div>
            </div>
          </button>
        </div>

        <div class="done" v-if="allMatched">
          <strong>完成！</strong> 你已經把 6 種能源都配對成功了 🎉
        </div>
      </section>

      <!-- QUIZ GAME -->
      <section v-else-if="page === 'quiz'">
        <div class="row">
          <h2>問答遊戲</h2>
          <div class="row-actions">
            <button class="ghost" @click="resetQuiz">重新開始</button>
          </div>
        </div>

        <p class="muted">
          不計分，可跳過或返回；最後會顯示答對、答錯與跳過數量。
          <span v-if="demoMode">（DEMO 模式：題目順序固定 + 可一鍵示範）</span>
        </p>

        <div class="quizbox" v-if="currentQ">
          <div class="quiz-top">
            <div class="pill">第 {{ quizIndex + 1 }} / {{ quizTotal }} 題</div>
            <div class="pill">題目類型：{{ currentQ.type === 'tf' ? '是非題' : '單選題' }}</div>
          </div>

          <h3 class="q-title">{{ currentQ.question }}</h3>

          <!-- True/False -->
          <div v-if="currentQ.type === 'tf'" class="options">
            <button
              class="opt"
              :class="{ chosen: userAnswers[currentQ.id]?.choice === true }"
              @click="chooseTF(true)"
            >
              O（是）
            </button>
            <button
              class="opt"
              :class="{ chosen: userAnswers[currentQ.id]?.choice === false }"
              @click="chooseTF(false)"
            >
              X（否）
            </button>
          </div>

          <!-- Single choice -->
          <div v-else class="options">
            <button
              v-for="(op, idx) in currentQ.options"
              :key="idx"
              class="opt"
              :class="{ chosen: userAnswers[currentQ.id]?.choice === idx }"
              @click="chooseSingle(idx)"
            >
              {{ op }}
            </button>
          </div>

          <div class="explain" v-if="userAnswers[currentQ.id]?.locked">
            <div class="tag" :class="{ ok: userAnswers[currentQ.id]?.correct, bad: !userAnswers[currentQ.id]?.correct }">
              {{ userAnswers[currentQ.id]?.correct ? "答對 ✅" : "答錯 ❌" }}
            </div>
            <div class="exp-text">{{ currentQ.explain }}</div>
          </div>

          <div class="quiz-actions">
            <button class="ghost" @click="prevQuiz" :disabled="quizIndex === 0">返回</button>
            <button class="ghost" @click="skipQuiz">跳過</button>
            <button class="primary" @click="submitQuiz" :disabled="!hasSelected || userAnswers[currentQ.id]?.locked">
              送出答案
            </button>
            <button class="primary" @click="nextQuiz" :disabled="quizIndex === quizTotal - 1">
              下一題
            </button>
            <button class="primary" @click="finishQuiz" :disabled="quizTotal === 0">
              看結果
            </button>

            <button class="ghost" v-if="demoMode" @click="demoAnswerCorrect" :disabled="!currentQ">
              DEMO：自動答對
            </button>
            <button class="ghost" v-if="demoMode" @click="demoRunToResult" :disabled="!currentQ">
              DEMO：跑到結果
            </button>
          </div>
        </div>

        <div v-else class="status">
          目前沒有題目，請檢查題庫檔案（src/data/quiz.js）。
        </div>
      </section>

      <!-- RESULT -->
      <section v-else>
        <div class="row">
          <h2>結果</h2>
          <div class="row-actions">
            <button class="ghost" @click="go('quiz')">回到問答</button>
          </div>
        </div>

        <div class="result-grid">
          <div class="result-card">
            <div class="big">{{ quizStats.correct }}</div>
            <div class="label">答對</div>
          </div>
          <div class="result-card">
            <div class="big">{{ quizStats.wrong }}</div>
            <div class="label">答錯</div>
          </div>
          <div class="result-card">
            <div class="big">{{ quizStats.skipped }}</div>
            <div class="label">跳過</div>
          </div>
        </div>

        <div class="status" v-if="skippedList.length">
          <strong>你跳過的題目：</strong>
          <ul>
            <li v-for="q in skippedList" :key="q.id">{{ q.question }}</li>
          </ul>
        </div>

        <div class="status" v-else>
          你沒有跳過任何題目 👍
        </div>
      </section>
    </main>

    <footer class="footer">
      <small>© 期末分組專題：能源發電方式</small>
    </footer>
  </div>
</template>

<script setup>
import { computed, ref } from "vue";
import { energies } from "./data/energy";
import { quizQuestions } from "./data/quiz";

// -------------------- DEMO MODE --------------------
const demoMode = ref(false);

// -------------------- Page --------------------
const page = ref("home");
function go(p) {
  page.value = p;
  if (p === "memory") resetMemory();
  if (p === "quiz") resetQuizIfEmpty();
}

// -------------------- Home modal --------------------
const selectedEnergyId = ref(null);
const selectedEnergy = computed(() => energies.find((e) => e.id === selectedEnergyId.value) || null);
function openEnergy(id) {
  selectedEnergyId.value = id;
}
function closeEnergy() {
  selectedEnergyId.value = null;
}

// -------------------- Helpers --------------------
function shuffle(arr) {
  const a = [...arr];
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

// -------------------- Memory Game --------------------
const cards = ref([]);
const lock = ref(false);
const memoryStatus = ref("");

function buildCards() {
  // 每種能源 2 張：一張圖卡、一張關鍵字卡
  const duplicated = energies.flatMap((e) => [
    { ...e, uid: `${e.id}-img`, cardType: "img" },
    { ...e, uid: `${e.id}-kw`, cardType: "kw" },
  ]);

  cards.value = shuffle(
    duplicated.map((c) => ({
      ...c,
      flipped: false,
      matched: false,
    }))
  );
}

function resetMemory() {
  lock.value = false;
  memoryStatus.value = "";
  buildCards();
}

const flippedUnmatched = computed(() => cards.value.filter((c) => c.flipped && !c.matched));
const allMatched = computed(() => cards.value.length > 0 && cards.value.every((c) => c.matched));

function flipCard(card) {
  if (lock.value) return;
  if (card.matched || card.flipped) return;
  if (flippedUnmatched.value.length >= 2) return;

  card.flipped = true;

  if (flippedUnmatched.value.length === 2) {
    const [a, b] = flippedUnmatched.value;
    lock.value = true;

    if (a.id === b.id) {
      setTimeout(() => {
        a.matched = true;
        b.matched = true;
        memoryStatus.value = `✅ 配對成功：${a.name}`;
        lock.value = false;
      }, 450);
    } else {
      setTimeout(() => {
        a.flipped = false;
        b.flipped = false;
        memoryStatus.value = "❌ 配對失敗，再試一次！";
        lock.value = false;
      }, 750);
    }
  }
}

function demoFinishMemory() {
  for (const c of cards.value) {
    c.flipped = true;
    c.matched = true;
  }
  memoryStatus.value = "✅ DEMO：已快速完成全部配對";
}

// -------------------- Quiz Game --------------------
const quiz = ref([]); // 題目列表
const quizIndex = ref(0);
const userAnswers = ref({}); // { [questionId]: { choice, locked, correct, skipped } }

const quizTotal = computed(() => quiz.value.length);
const currentQ = computed(() => quiz.value[quizIndex.value] || null);

function resetQuizIfEmpty() {
  if (quiz.value.length === 0) resetQuiz();
}

function resetQuiz() {
  // DEMO 模式：固定順序；非 DEMO：洗牌
  quiz.value = demoMode.value ? [...quizQuestions] : shuffle(quizQuestions);
  quizIndex.value = 0;
  userAnswers.value = {};
}

const hasSelected = computed(() => {
  const q = currentQ.value;
  if (!q) return false;
  const ua = userAnswers.value[q.id];
  return ua && ua.choice !== null && ua.choice !== undefined;
});

function chooseTF(val) {
  const q = currentQ.value;
  if (!q) return;
  if (userAnswers.value[q.id]?.locked) return;
  userAnswers.value[q.id] = { choice: val, locked: false, correct: false, skipped: false };
}

function chooseSingle(idx) {
  const q = currentQ.value;
  if (!q) return;
  if (userAnswers.value[q.id]?.locked) return;
  userAnswers.value[q.id] = { choice: idx, locked: false, correct: false, skipped: false };
}

function submitQuiz() {
  const q = currentQ.value;
  if (!q) return;
  const ua = userAnswers.value[q.id];
  if (!ua || ua.choice === null || ua.choice === undefined) return;
  if (ua.locked) return;

  let correct = false;
  if (q.type === "tf") correct = ua.choice === q.answer;
  else correct = ua.choice === q.answerIndex;

  userAnswers.value[q.id] = { ...ua, locked: true, correct, skipped: false };
}

function skipQuiz() {
  const q = currentQ.value;
  if (!q) return;

  const ua = userAnswers.value[q.id];

  // ✅ BUG 修正：若已送出答案（locked=true），不允許再被跳過覆蓋
  if (ua && ua.locked) return;

  userAnswers.value[q.id] = { choice: null, locked: false, correct: false, skipped: true };
  nextQuiz();
}

function prevQuiz() {
  if (quizIndex.value > 0) quizIndex.value -= 1;
}

function nextQuiz() {
  if (quizIndex.value < quizTotal.value - 1) quizIndex.value += 1;
}

const quizStats = computed(() => {
  let correct = 0,
    wrong = 0,
    skipped = 0;
  for (const q of quiz.value) {
    const ua = userAnswers.value[q.id];
    if (!ua) continue;
    if (ua.skipped) skipped += 1;
    else if (ua.locked && ua.correct) correct += 1;
    else if (ua.locked && !ua.correct) wrong += 1;
  }
  return { correct, wrong, skipped };
});

const skippedList = computed(() => quiz.value.filter((q) => userAnswers.value[q.id]?.skipped));

function finishQuiz() {
  page.value = "result";
}

// DEMO actions
function demoAnswerCorrect() {
  const q = currentQ.value;
  if (!q) return;

  if (q.type === "tf") chooseTF(q.answer);
  else chooseSingle(q.answerIndex);

  submitQuiz();
  nextQuiz();
}

function demoRunToResult() {
  // 連續答到最後
  while (quizIndex.value < quizTotal.value - 1) {
    demoAnswerCorrect();
  }

  // 最後一題也確保答對 + 送出
  const q = currentQ.value;
  if (q) {
    if (q.type === "tf") chooseTF(q.answer);
    else chooseSingle(q.answerIndex);
    submitQuiz();
  }

  finishQuiz();
}

// 初始化（避免第一次進入沒資料）
resetMemory();
resetQuiz();
</script>

<style scoped>
.app {
  min-height: 100vh;
  display: grid;
  grid-template-rows: auto 1fr auto;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans TC", sans-serif;
}

.topbar {
  padding: 16px 20px;
  border-bottom: 1px solid #eee;
  background: white;
  position: sticky;
  top: 0;
}

.title-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  flex-wrap: wrap;
  margin-bottom: 10px;
}

.topbar h1 {
  margin: 0;
  font-size: 20px;
}

.demo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: #333;
  user-select: none;
}

.nav {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.nav button {
  padding: 8px 12px;
  border: 1px solid #ddd;
  background: #fafafa;
  border-radius: 10px;
  cursor: pointer;
}

.nav button.active {
  border-color: #333;
  background: #fff;
}

.content {
  padding: 20px;
  max-width: 980px;
  width: 100%;
  margin: 0 auto;
}

.actions {
  margin-top: 14px;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.primary {
  padding: 10px 16px;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  background: #111;
  color: white;
}

.footer {
  padding: 14px 20px;
  border-top: 1px solid #eee;
  background: white;
  text-align: center;
}

.row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.row-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.ghost {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 10px;
  background: white;
  cursor: pointer;
}

.ghost:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.muted {
  color: #555;
  margin: 8px 0 14px;
}

/* chips reuse */
.kw {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.chip {
  border: 1px solid #eee;
  background: #fafafa;
  padding: 4px 8px;
  border-radius: 999px;
  font-size: 12px;
}

.status {
  margin: 10px 0 14px;
  padding: 10px 12px;
  border: 1px solid #eee;
  border-radius: 12px;
  background: #fafafa;
}

/* HOME cards */
.home-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  margin: 14px 0;
}

.energy-card {
  border: 1px solid #eee;
  border-radius: 16px;
  padding: 14px;
  background: #fff;
  text-align: left;
  cursor: pointer;
}

.energy-icon {
  font-size: 28px;
}

.energy-name {
  font-weight: 800;
  margin-top: 6px;
}

.energy-intro {
  color: #555;
  font-size: 13px;
  margin-top: 6px;
  line-height: 1.35;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  display: grid;
  place-items: center;
  padding: 18px;
}

.modal {
  width: min(860px, 96vw);
  background: white;
  border-radius: 16px;
  padding: 14px;
  border: 1px solid #eee;
}

.two-col {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
  margin-top: 10px;
}

.box {
  border: 1px solid #eee;
  border-radius: 14px;
  padding: 12px;
  background: #fafafa;
}

/* Memory grid */
.grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
}

.card {
  height: 160px;
  border: 1px solid #e6e6e6;
  border-radius: 14px;
  background: white;
  cursor: pointer;
  perspective: 900px;
  padding: 0;
}

.card.matched {
  opacity: 0.55;
}

.inner {
  position: relative;
  width: 100%;
  height: 100%;
  transform-style: preserve-3d;
  transition: transform 0.35s;
  border-radius: 14px;
}

.card.flipped .inner {
  transform: rotateY(180deg);
}

.front,
.back {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  backface-visibility: hidden;
  border-radius: 14px;
}

.front {
  font-size: 28px;
  font-weight: 700;
  background: #111;
  color: white;
}

.back {
  transform: rotateY(180deg);
  padding: 10px;
  text-align: center;
}

.name {
  font-weight: 800;
}

.bigicon {
  font-size: 34px;
  margin-top: 6px;
}

.hint {
  margin-top: 6px;
  font-size: 12px;
  color: #555;
  line-height: 1.35;

  display: -webkit-box;
  -webkit-line-clamp: 2;       /* 最多兩行 */
  -webkit-box-orient: vertical;
  overflow: hidden;
}


.done {
  margin-top: 14px;
  padding: 12px 14px;
  border: 1px solid #eee;
  border-radius: 12px;
  background: #fafafa;
}

/* Quiz */
.quizbox {
  border: 1px solid #eee;
  border-radius: 14px;
  padding: 14px;
  background: white;
}

.quiz-top {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-bottom: 10px;
}

.pill {
  border: 1px solid #eee;
  background: #fafafa;
  padding: 6px 10px;
  border-radius: 999px;
  font-size: 12px;
}

.q-title {
  margin: 10px 0 12px;
  font-size: 18px;
}

.options {
  display: grid;
  gap: 10px;
  margin-bottom: 12px;
}

.opt {
  text-align: left;
  padding: 10px 12px;
  border: 1px solid #e6e6e6;
  border-radius: 12px;
  background: #fff;
  cursor: pointer;
}

.opt.chosen {
  border-color: #111;
}

.explain {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #eee;
  display: grid;
  gap: 8px;
}

.tag {
  display: inline-block;
  width: fit-content;
  padding: 6px 10px;
  border-radius: 999px;
  border: 1px solid #eee;
  background: #fafafa;
  font-size: 12px;
}

.tag.ok {
  border-color: #111;
}

.tag.bad {
  border-color: #999;
}

.exp-text {
  color: #444;
}

.quiz-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-top: 12px;
}

/* Result */
.result-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  margin: 12px 0;
}

.result-card {
  border: 1px solid #eee;
  border-radius: 14px;
  background: #fff;
  padding: 14px;
  text-align: center;
}

.big {
  font-size: 34px;
  font-weight: 800;
}

.label {
  color: #555;
}

/* Responsive */
@media (max-width: 860px) {
  .home-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
  }
  .two-col {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 520px) {
  .home-grid {
    grid-template-columns: 1fr;
  }
  .grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}
</style>
