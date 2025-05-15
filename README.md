const acorn = require("acorn");
const walk = require("acorn-walk");

function detectErrors(code) {
  const errors = [];
  
  try {
    // Пытаемся распарсить код
    const ast = acorn.parse(code, {
      ecmaVersion: 2022,
      sourceType: "script"
    });
  } catch (e) {
    errors.push({
      type: "Синтаксическая ошибка",
      message: e.message,
      position: e.pos
    });
    return errors; // Не можем анализировать дальше при синтаксических ошибках
  }

  // Анализ AST для поиска распространённых ошибок
  const ast = acorn.parse(code, {ecmaVersion: 2022, sourceType: "script"});

  // 1. Проверка на использование == вместо ===
  walk.simple(ast, {
    BinaryExpression(node) {
      if (node.operator === "==") {
        errors.push({
          type: "Строгое сравнение",
          message: "Использование == вместо ===, позиция: " + node.start,
          position: node.start
        });
      }
    }
  });

  // 2. Проверка на необработанные промисы
  walk.simple(ast, {
    CallExpression(node) {
      if (node.callee.type === "MemberExpression" && 
          node.callee.property.name === "then" &&
          !node.arguments.some(arg => arg.type === "FunctionExpression")) {
        errors.push({
          type: "Промисы",
          message: "Отсутствует обработчик ошибок для промиса, позиция: " + node.start,
          position: node.start
        });
      }
    }
  });

  // 3. Проверка на console.log
  walk.simple(ast, {
    MemberExpression(node) {
      if (node.object.name === "console" && node.property.name === "log") {
        errors.push({
          type: "Отладочный код",
          message: "Обнаружен console.log, позиция: " + node.start,
          position: node.start
        });
      }
    }
  });

  return errors;
}

// Пример использования
const code = 
function test() {
  let x = 5;
  if (x == "5") {
    console.log('test');
    fetch('/data').then();
  }
}
;

const errors = detectErrors(code);
console.log("Найдены ошибки:");
errors.forEach(err => console.log([${err.type}] ${err.message}));
