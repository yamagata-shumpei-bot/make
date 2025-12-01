# make<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <title>マーケティング重要用語 一問一答（10問ランダム／分野別）</title>
  <style>
    body {
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      max-width: 900px;
      margin: 20px auto;
      padding: 0 16px;
      line-height: 1.6;
    }
    h1 {
      font-size: 1.6rem;
      margin-bottom: 0.2rem;
    }
    #description {
      font-size: 0.9rem;
      color: #555;
      margin-bottom: 0.8rem;
    }
    #controls {
      margin: 12px 0;
      padding: 8px 10px;
      border: 1px solid #ddd;
      border-radius: 8px;
      font-size: 0.95rem;
    }
    #controls label {
      margin-right: 12px;
    }
    #status {
      font-size: 0.9rem;
      color: #666;
      margin: 8px 0;
    }
    #quiz-container {
      border: 1px solid #ccc;
      border-radius: 8px;
      padding: 16px;
      margin-top: 10px;
    }
    #progress {
      font-weight: bold;
      margin-bottom: 8px;
    }
    #question-text {
      margin-bottom: 12px;
      font-weight: bold;
    }
    .choice-item {
      margin-bottom: 4px;
    }
    #feedback {
      margin-top: 12px;
      padding-top: 8px;
      border-top: 1px dashed #ccc;
      min-height: 3em;
      font-size: 0.95rem;
    }
    button {
      margin-top: 8px;
      margin-right: 8px;
      padding: 6px 12px;
      font-size: 0.95rem;
      cursor: pointer;
    }
    #restart-btn {
      margin-top: 16px;
    }
    .term-list {
      margin-top: 8px;
      padding-left: 1.2em;
    }
    .term-list li {
      margin-bottom: 4px;
    }
  </style>
</head>
<body>
  <h1>マーケティング重要用語 一問一答（10問ランダム）</h1>
  <p id="description">
    Googleスプレッドシート
    <code>1Z1oaOOMUZb962OxVm2ZYvYW6Zsku7MR7AvXqdKWduS8</code>
    の内容（420問）を読み込んで出題します。<br>
    「全問題からランダム」または「分野別」を選んでからスタートしてください。
  </p>

  <div id="controls">
    <div>
      <label><input type="radio" name="mode" value="all" checked> 全問題からランダム10問</label>
      <label><input type="radio" name="mode" value="byField"> 分野別ランダム10問</label>
      <select id="field-select" style="display:none; margin-left:8px;"></select>
    </div>
    <div id="status">スプレッドシートから問題を読み込み中です…</div>
    <button type="button" id="start-btn" disabled>この条件でスタート</button>
  </div>

  <div id="quiz-container" style="display:none;">
    <div id="progress"></div>
    <div id="question-text"></div>
    <form id="choices"></form>
    <div>
      <button type="button" id="answer-btn">回答する</button>
      <button type="button" id="next-btn" disabled>次の問題へ</button>
    </div>
    <div id="feedback"></div>
  </div>

  <button type="button" id="restart-btn" style="display:none;">もう一度最初から</button>

  <script>
    // 元の編集URL:
    // https://docs.google.com/spreadsheets/d/1Z1oaOOMUZb962OxVm2ZYvYW6Zsku7MR7AvXqdKWduS8/edit?usp=sharing
    // ↓同じシートを CSV として取得するエンドポイント
    const SPREADSHEET_CSV_URL =
      "https://docs.google.com/spreadsheets/d/1Z1oaOOMUZb962OxVm2ZYvYW6Zsku7MR7AvXqdKWduS8/export?format=csv&gid=0";

    const NUM_QUESTIONS_PER_PLAY = 10;

    let QUESTION_BANK = [];   // {id, field, question, answer, explanation, expert_comment}
    let quizQuestions = [];
    let currentIndex = 0;
    let score = 0;
    let currentChoices = [];

    const progressEl = document.getElementById("progress");
    const questionTextEl = document.getElementById("question-text");
    const choicesForm = document.getElementById("choices");
    const feedbackEl = document.getElementById("feedback");
    const answerBtn = document.getElementById("answer-btn");
    const nextBtn = document.getElementById("next-btn");
    const restartBtn = document.getElementById("restart-btn");
    const quizContainer = document.getElementById("quiz-container");
    const fieldSelect = document.getElementById("field-select");
    const modeRadios = document.querySelectorAll('input[name="mode"]');
    const statusEl = document.getElementById("status");
    const startBtn = document.getElementById("start-btn");

    // --- CSVパーサ（簡易：ダブルクォート対応） ---
    function parseCsv(text) {
      const rows = [];
      let row = [];
      let cur = "";
      let insideQuotes = false;

      for (let i = 0; i < text.length; i++) {
        const c = text[i];

        if (insideQuotes) {
          if (c === '"') {
            if (i + 1 < text.length && text[i + 1] === '"') {
              // エスケープされたダブルクォート
              cur += '"';
              i++;
            } else {
              insideQuotes = false;
            }
          } else {
            cur += c;
          }
        } else {
          if (c === '"') {
            insideQuotes = true;
          } else if (c === ",") {
            row.push(cur);
            cur = "";
          } else if (c === "\r") {
            // 無視
          } else if (c === "\n") {
            row.push(cur);
            rows.push(row);
            row = [];
            cur = "";
          } else {
            cur += c;
          }
        }
      }
      if (cur.length > 0 || row.length > 0) {
        row.push(cur);
        rows.push(row);
      }
      return rows;
    }

    // --- スプレッドシートから読み込み ---
    async function loadQuestionsFromSheet() {
      try {
        const resp = await fetch(SPREADSHEET_CSV_URL);
        if (!resp.ok) {
          throw new Error("HTTPエラー: " + resp.status);
        }
        const text = await resp.text();
        const rows = parseCsv(text);
        if (rows.length < 2) {
          throw new Error("データ行がありません");
        }

        const header = rows[0];
        const idxId = header.indexOf("id");
        const idxField = header.indexOf("field");
        const idxQ = header.indexOf("question");
        const idxA = header.indexOf("answer");
        const idxExp = header.indexOf("explanation");
        const idxCom = header.indexOf("expert_comment");

        QUESTION_BANK = rows.slice(1)
          .filter(r => r.length > 0 && r[idxQ] && r[idxA])
          .map(r => ({
            id: r[idxId] ? Number(r[idxId]) : null,
            field: idxField >= 0 ? (r[idxField] || "").trim() : "",
            question: (idxQ >= 0 ? r[idxQ] : "").trim(),
            answer: (idxA >= 0 ? r[idxA] : "").trim(),
            explanation: idxExp >= 0 ? (r[idxExp] || "").trim() : "",
            expert_comment: idxCom >= 0 ? (r[idxCom] || "").trim() : ""
          }));

        if (QUESTION_BANK.length === 0) {
          throw new Error("有効な問題が読み込めませんでした。");
        }

        statusEl.textContent = `問題を読み込みました（全 ${QUESTION_BANK.length} 問）。モードを選んで「この条件でスタート」を押してください。`;
        startBtn.disabled = false;

        // 分野（field）の候補を自動生成
        populateFieldOptions();
      } catch (e) {
        console.error(e);
        statusEl.textContent = "問題の読み込みに失敗しました。ブラウザや共有設定を確認してください。";
      }
    }

    function populateFieldOptions() {
      const fields = Array.from(
        new Set(QUESTION_BANK.map(q => q.field).filter(f => f && f.length > 0))
      ).sort();

      fieldSelect.innerHTML = "";
      fields.forEach(f => {
        const opt = document.createElement("option");
        opt.value = f;
        opt.textContent = f; // もし「A 基礎」などにしたければシート側で field をそう設定
        fieldSelect.appendChild(opt);
      });
    }

    // --- 汎用ユーティリティ ---
    function shuffle(array) {
      const arr = array.slice();
      for (let i = arr.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [arr[i], arr[j]] = [arr[j], arr[i]];
      }
      return arr;
    }

    function getSelectedMode() {
      const r = Array.from(modeRadios).find(r => r.checked);
      return r ? r.value : "all";
    }

    // --- クイズの準備 ---
    function setupQuizQuestions() {
      const mode = getSelectedMode();
      let pool = [];
      if (mode === "all") {
        pool = QUESTION_BANK;
      } else {
        const fld = fieldSelect.value;
        pool = QUESTION_BANK.filter(q => q.field === fld);
      }

      if (pool.length === 0) {
        alert("選択された分野に問題がありません。");
        return false;
      }

      const limit = Math.min(NUM_QUESTIONS_PER_PLAY, pool.length);
      quizQuestions = shuffle(pool).slice(0, limit);
      currentIndex = 0;
      score = 0;
      return true;
    }

    // --- 1問表示 ---
    function showQuestion() {
      const qObj = quizQuestions[currentIndex];
      progressEl.textContent = `第 ${currentIndex + 1} 問 / 全 ${quizQuestions.length} 問`;
      questionTextEl.textContent = qObj.question;
      feedbackEl.innerHTML = "";

      const otherAnswers = QUESTION_BANK
        .filter(q => q.answer !== qObj.answer)
        .map(q => q.answer);

      const dummyChoices = shuffle(otherAnswers).slice(0, 3);
      currentChoices = shuffle([qObj.answer, ...dummyChoices]);

      choicesForm.innerHTML = "";
      currentChoices.forEach((choiceText, idx) => {
        const id = `choice-${idx}`;
        const div = document.createElement("div");
        div.className = "choice-item";
        const input = document.createElement("input");
        input.type = "radio";
        input.name = "choice";
        input.id = id;
        input.value = choiceText;
        const label = document.createElement("label");
        label.setAttribute("for", id);
        label.textContent = choiceText;
        div.appendChild(input);
        div.appendChild(label);
        choicesForm.appendChild(div);
      });

      answerBtn.disabled = false;
      nextBtn.disabled = true;
    }

    // --- 回答処理 ---
    function checkAnswer() {
      const selected = choicesForm.querySelector("input[name='choice']:checked");
      if (!selected) {
        alert("選択肢を選んでください。");
        return;
      }
      const qObj = quizQuestions[currentIndex];
      const userAnswer = selected.value;
      const isCorrect = (userAnswer === qObj.answer);

      if (isCorrect) {
        score++;
      }

      const mainExp = (qObj.explanation && qObj.explanation.length > 0)
        ? qObj.explanation
        : qObj.question;
      const mainCom = qObj.expert_comment && qObj.expert_comment.length > 0
        ? `<br><em>${qObj.expert_comment}</em>`
        : "";

      // 選択肢4つそれぞれの解説をまとめる
      let termListHtml = "<ul class=\"term-list\">";
      currentChoices.forEach(ch => {
        const termObj = QUESTION_BANK.find(q => q.answer === ch);
        if (termObj) {
          const exp = (termObj.explanation && termObj.explanation.length > 0)
            ? termObj.explanation
            : termObj.question;
          const com = termObj.expert_comment && termObj.expert_comment.length > 0
            ? `<br><em>${termObj.expert_comment}</em>`
            : "";
          termListHtml += `<li><strong>${ch}</strong>：${exp}${com}</li>`;
        } else {
          termListHtml += `<li><strong>${ch}</strong>：解説データが登録されていません。</li>`;
        }
      });
      termListHtml += "</ul>";

      feedbackEl.innerHTML =
        `<p>${isCorrect ? "◎ 正解！" : "✕ 不正解…"}</p>` +
        `<p>【正解】${qObj.answer}</p>` +
        `<p>【この問題の解説】${mainExp}${mainCom}</p>` +
        `<hr>` +
        `<p><strong>今回の選択肢に出てきた用語の解説</strong></p>` +
        termListHtml;

      answerBtn.disabled = true;
      nextBtn.disabled = false;
    }

    // --- 次の問題 ---
    function nextQuestion() {
      currentIndex++;
      if (currentIndex >= quizQuestions.length) {
        showResult();
      } else {
        showQuestion();
      }
    }

    // --- 結果表示 ---
    function showResult() {
      progressEl.textContent = "結果";
      questionTextEl.textContent = "";
      choicesForm.innerHTML = "";
      feedbackEl.innerHTML =
        `<p>終了です。${quizQuestions.length}問中 <strong>${score}問 正解</strong> でした。</p>`;

      answerBtn.disabled = true;
      nextBtn.disabled = true;
      restartBtn.style.display = "";
    }

    // --- 最初から ---
    function restartQuiz() {
      if (!setupQuizQuestions()) return;
      quizContainer.style.display = "";
      restartBtn.style.display = "none";
      showQuestion();
    }

    // --- イベント設定 ---
    answerBtn.addEventListener("click", checkAnswer);
    nextBtn.addEventListener("click", nextQuestion);
    restartBtn.addEventListener("click", restartQuiz);

    startBtn.addEventListener("click", () => {
      if (!setupQuizQuestions()) return;
      quizContainer.style.display = "";
      restartBtn.style.display = "none";
      showQuestion();
      statusEl.textContent = "";
    });

    modeRadios.forEach(r => {
      r.addEventListener("change", () => {
        const mode = getSelectedMode();
        if (mode === "byField") {
          fieldSelect.style.display = "";
        } else {
          fieldSelect.style.display = "none";
        }
      });
    });

    // --- 初期処理：シートから読み込む ---
    loadQuestionsFromSheet();
  </script>
</body>
</html>
