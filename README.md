<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Судоку 9×9 — найди секретное слово</title>
    <style>
        * {
            user-select: none;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: #f0f2f5;
            margin: 0;
            padding: 20px;
            text-align: center;
        }
        h1 {
            color: #2c3e50;
            font-size: 1.8rem;
        }
        .sudoku-container {
            display: flex;
            justify-content: center;
            margin: 20px auto;
            overflow-x: auto;
        }
        table.sudoku {
            border-collapse: collapse;
            background: white;
            box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        table.sudoku td {
            border: 1px solid #aaa;
            padding: 0;
        }
        table.sudoku input {
            width: 48px;
            height: 48px;
            text-align: center;
            font-size: 22px;
            font-weight: 500;
            border: none;
            box-sizing: border-box;
            font-family: monospace;
            background: #fefefe;
            transition: 0.1s;
        }
        table.sudoku input:focus {
            outline: 2px solid #3498db;
            background: #eaf6ff;
        }
        table.sudoku input[readonly] {
            background: #e8eef2;
            font-weight: bold;
            color: #1e4663;
        }
        /* Жирные границы для блоков 3x3 */
        table.sudoku tr:nth-child(3) td,
        table.sudoku tr:nth-child(6) td {
            border-bottom: 3px solid #333;
        }
        table.sudoku td:nth-child(3),
        table.sudoku td:nth-child(6) {
            border-right: 3px solid #333;
        }
        .message {
            font-size: 26px;
            font-weight: bold;
            margin: 30px auto 20px;
            padding: 15px;
            border-radius: 50px;
            display: inline-block;
            background: #2c3e50;
            color: #ecf0f1;
            transition: 0.3s;
        }
        .win-message {
            background: #27ae60;
            color: white;
            box-shadow: 0 0 15px #2ecc71;
        }
        button {
            background: #3498db;
            border: none;
            color: white;
            font-size: 18px;
            padding: 10px 25px;
            border-radius: 40px;
            cursor: pointer;
            margin-top: 15px;
            font-weight: bold;
            transition: 0.2s;
        }
        button:hover {
            background: #2980b9;
            transform: scale(1.02);
        }
        .footer {
            margin-top: 30px;
            font-size: 14px;
            color: #7f8c8d;
        }
        @media (max-width: 550px) {
            table.sudoku input {
                width: 32px;
                height: 32px;
                font-size: 18px;
            }
            .message {
                font-size: 18px;
            }
        }
    </style>
</head>
<body>

<h1>🧩 Судоку 9x9</h1>
<div class="sudoku-container">
    <table class="sudoku" id="sudoku-grid"></table>
</div>
<div id="resultMessage" class="message">🔐 Решите судоку — откроется слово</div>
<button id="resetBtn">🔄 Новая игра</button>
<div class="footer">
    ⚡ Введите числа от 1 до 9. Готовые ячейки выделены серым.
</div>

<script>
    // ██████████████████████████████████████████████████████████████████████████
    //  ПРАВИЛЬНОЕ РЕШЕНИЕ (эталон) — классическое судоку 9x9
    // ██████████████████████████████████████████████████████████████████████████
    const FULL_SOLUTION = [
        [5, 3, 4, 6, 7, 8, 9, 1, 2],
        [6, 7, 2, 1, 9, 5, 3, 4, 8],
        [1, 9, 8, 3, 4, 2, 5, 6, 7],
        [8, 5, 9, 7, 6, 1, 4, 2, 3],
        [4, 2, 6, 8, 5, 3, 7, 9, 1],
        [7, 1, 3, 9, 2, 4, 8, 5, 6],
        [9, 6, 1, 5, 3, 7, 2, 8, 4],
        [2, 8, 7, 4, 1, 9, 6, 3, 5],
        [3, 4, 5, 2, 8, 6, 1, 7, 9]
    ];

    // Начальная расстановка (то, что видит игрок, 0 — пустая ячейка)
    const START_PUZZLE = [
        [5, 3, 0, 0, 7, 0, 0, 0, 0],
        [6, 0, 0, 1, 9, 5, 0, 0, 0],
        [0, 9, 8, 0, 0, 0, 0, 6, 0],
        [8, 0, 0, 0, 6, 0, 0, 0, 3],
        [4, 0, 0, 8, 0, 3, 0, 0, 1],
        [7, 0, 0, 0, 2, 0, 0, 0, 6],
        [0, 6, 0, 0, 0, 0, 2, 8, 0],
        [0, 0, 0, 4, 1, 9, 0, 0, 5],
        [0, 0, 0, 0, 8, 0, 0, 7, 9]
    ];

    // █████████████████████████████████████████████████████████████
    //  ✨ СЕКРЕТНОЕ СЛОВО / ФРАЗА (измените на своё!)
    // █████████████████████████████████████████████████████████████
    const SECRET_WORD = "🎉 ПОБЕДА! ВАШ КОД: ГЕНИЙ 🧠";

    // --------------------------------------------------------------
    let currentGrid = JSON.parse(JSON.stringify(START_PUZZLE));   // текущее состояние (включая ввод игрока)
    let readonlyMask = JSON.parse(JSON.stringify(START_PUZZLE));  // true если ячейка была изначально фиксированной

    // Создание таблицы
    function buildGrid() {
        const gridContainer = document.getElementById("sudoku-grid");
        gridContainer.innerHTML = "";
        for (let r = 0; r < 9; r++) {
            const tr = document.createElement("tr");
            for (let c = 0; c < 9; c++) {
                const td = document.createElement("td");
                const input = document.createElement("input");
                input.type = "text";
                input.maxLength = 1;
                input.inputMode = "numeric";
                
                const value = currentGrid[r][c];
                if (value !== 0) {
                    input.value = value;
                } else {
                    input.value = "";
                }
                
                // Если это изначально заданное число — блокируем редактирование
                if (START_PUZZLE[r][c] !== 0) {
                    input.readOnly = true;
                    input.classList.add("fixed-cell");
                } else {
                    input.readOnly = false;
                }
                
                input.setAttribute("data-row", r);
                input.setAttribute("data-col", c);
                input.addEventListener("input", function(e) {
                    handleInput(e, r, c);
                });
                td.appendChild(input);
                tr.appendChild(td);
            }
            gridContainer.appendChild(tr);
        }
    }

    // Обработка ввода в ячейку
    function handleInput(event, row, col) {
        let val = event.target.value.trim();
        // Разрешаем только цифры 1-9
        if (val.length > 0) {
            const num = parseInt(val);
            if (isNaN(num) || num < 1 || num > 9) {
                event.target.value = "";
                currentGrid[row][col] = 0;
            } else {
                event.target.value = num;
                currentGrid[row][col] = num;
            }
        } else {
            currentGrid[row][col] = 0;
        }
        // после каждого изменения проверяем победу
        checkVictory();
    }

    // Проверка: заполнены ли все клетки и совпадают ли с решением
    function checkVictory() {
        // 1) все ли клетки не пустые?
        let allFilled = true;
        for (let r = 0; r < 9; r++) {
            for (let c = 0; c < 9; c++) {
                if (currentGrid[r][c] === 0) {
                    allFilled = false;
                    break;
                }
            }
        }
        if (!allFilled) {
            document.getElementById("resultMessage").innerHTML = "🔐 Решите судоку — откроется слово";
            document.getElementById("resultMessage").classList.remove("win-message");
            return;
        }
        
        // 2) сравнение с эталонным решением
        let isPerfect = true;
        for (let r = 0; r < 9; r++) {
            for (let c = 0; c < 9; c++) {
                if (currentGrid[r][c] !== FULL_SOLUTION[r][c]) {
                    isPerfect = false;
                    break;
                }
            }
        }
        
        const msgDiv = document.getElementById("resultMessage");
        if (isPerfect) {
            msgDiv.innerHTML = SECRET_WORD;
            msgDiv.classList.add("win-message");
        } else {
            msgDiv.innerHTML = "❌ Решение неверное. Проверьте числа!";
            msgDiv.classList.remove("win-message");
        }
    }

    // Сброс игры (очистить введённые пользователем поля)
    function resetGame() {
        currentGrid = JSON.parse(JSON.stringify(START_PUZZLE));
        // обновляем визуально все input
        const inputs = document.querySelectorAll("#sudoku-grid input");
        for (let i = 0; i < inputs.length; i++) {
            const input = inputs[i];
            const row = parseInt(input.getAttribute("data-row"));
            const col = parseInt(input.getAttribute("data-col"));
            const val = currentGrid[row][col];
            if (val !== 0) {
                input.value = val;
            } else {
                input.value = "";
            }
            // снимаем отметку победы
            const msgDiv = document.getElementById("resultMessage");
            msgDiv.innerHTML = "🔐 Решите судоку — откроется слово";
            msgDiv.classList.remove("win-message");
        }
    }

    // Инициализация
    buildGrid();
    document.getElementById("resetBtn").addEventListener("click", resetGame);
</script>
</body>
</html>
