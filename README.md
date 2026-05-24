# Лабораторная работа №7

## Анализ и преобразование кода с использованием Clang и LLVM

**Выполнил:** Топоев М.А.  
**Год:** 2026  
**Индивидуальный вариант:** 2.13 — Условные операторы  

---

---

# 1. Постановка задачи

Познакомиться с инструментарием Clang и LLVM, освоить получение абстрактного синтаксического дерева (AST) и промежуточного представления (LLVM IR) для кода на C/C++, научиться применять базовые оптимизации, строить графы потока управления (CFG), а также анализировать влияние оптимизаций на различные синтаксические конструкции языка.

Установка среды Установить Clang, LLVM, opt и Graphviz (например, в Ubuntu 26.04).
Работа с AST Сгенерировать абстрактное синтаксическое дерево для заданного C/C++‑файла.

Генерация LLVM IR Получить промежуточное представление кода без оптимизаций (-O0) и с оптимизациями (-O2).

Оптимизация IR Применить оптимизации с помощью opt и/или флагов Clang, сравнить изменения.

Построение CFG Построить граф потока управления для одной или нескольких функций.

Индивидуальное задание (по варианту) Выполнить анализ конкретной синтаксической конструкции в соответствии с вариантом. Сформулировать, как LLVM обрабатывает выбранную конструкцию, какие оптимизации применяются.

Выводы Ответить на контрольные вопросы

1. Построить AST и IR для -O0.
2. Примените -O2. Изменилось ли условие на cmov?
3. Постройте CFG.
4. Исследуйте, меняется ли CFG при использовании -branch-prob.
5. Сделайте вывод о том, как LLVM оптимизирует условные
переходы.
---

# 2. Общее задание

## 2.1. Установка и подготовка среды

Работа выполнялась в **Ubuntu**, запущенной в **Oracle VirtualBox**. Для выполнения лабораторной работы были установлены `clang`, `llvm`, `opt` и `graphviz`.

```bash
sudo apt update
sudo apt install -y clang llvm llvm-dev llvm-runtime graphviz xdg-utils
```

**Рисунок 1 — установка пакетов LLVM, Clang и Graphviz**

<img width="1165" height="556" alt="image" src="https://github.com/user-attachments/assets/dc77bcde-e9fc-4b64-89d9-8d1e3f1f7942" />


После установки были проверены версии основных инструментов. Проверка выполнялась командами `clang --version`, `opt --version`, `dot -V` и `llvm-config --version`.

```bash
clang --version
opt --version
dot -V
llvm-config --version
```

**Рисунок 2 — проверка версий Clang, opt**

<img width="697" height="217" alt="image" src="https://github.com/user-attachments/assets/de704096-71db-4620-b835-a1fefe502c0c" />



**Рисунок 3 — проверка версий dot, llvm-config**

<img width="625" height="90" alt="image" src="https://github.com/user-attachments/assets/7844c142-0a30-420e-a85b-a7d1c89d21af" />

---

## 2.2. Исходный код main.c

Для освоения инструментов был создан файл `main.c` с функцией `square` и функцией `main`.

```c
#include <stdio.h>

int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    printf("%d\n", b);
    return 0;
}
```

Функция `square` принимает целое число и возвращает результат умножения аргумента на самого себя. В функции `main` значение `5` передается в `square`, после чего результат выводится через `printf`.

---

## 2.3. Работа с AST

Для получения AST использовалась команда `clang` с параметром `-ast-dump`. Результат был сохранен в файл `ast_main.txt`.

```bash
clang -Xclang -ast-dump -fsyntax-only main.c > ast_main.txt
grep -A25 "FunctionDecl.*square" ast_main.txt
grep -A45 "FunctionDecl.*main" ast_main.txt
```

**Рисунок 4 — фрагмент AST для функций square и main**

<img width="1279" height="797" alt="image" src="https://github.com/user-attachments/assets/bac039c2-7365-4ac0-a69d-0d3c439438d5" />


В AST функция `square` представлена узлом `FunctionDecl`. Параметр `x` отображается как `ParmVarDecl`, а операция `x * x` — как `BinaryOperator`. Функция `main` также представлена как `FunctionDecl`, внутри которого видны объявления переменных, вызов `square` и вызов `printf`.

---

## 2.4. Генерация LLVM IR

LLVM IR был создан в двух вариантах: без оптимизации и с оптимизацией уровня `-O2`.

```bash
clang -O0 -S -emit-llvm main.c -o main_00.ll
clang -O2 -S -emit-llvm main.c -o main_02.ll
```

Для просмотра ключевых инструкций использовалась фильтрация через `grep`.

```bash
grep -n "define\|alloca\|load\|store\|call" main_00.ll
grep -n "define\|alloca\|load\|store\|call\|printf" main_02.ll
```

**Рисунок 5 — сравнение ключевых инструкций в IR без оптимизации и после -O2**

<img width="1157" height="707" alt="image" src="https://github.com/user-attachments/assets/4caaa616-2b37-4dcc-928e-1b087150bb68" />


В IR без оптимизации присутствуют инструкции `alloca`, `store` и `load`. Это значит, что локальные переменные размещаются в памяти, а значения явно записываются и считываются. Также видно, что функция `square` вызывается отдельно.

После оптимизации `-O2` лишние операции работы с памятью исчезают. Вызов `square(5)` фактически упрощается до готового значения `25`, которое передается в `printf`. Это показывает работу таких оптимизаций, как встраивание короткой функции, свертка констант и удаление лишних операций.

---

## 2.5. Оптимизация IR

Сравнение IR до и после оптимизации показывает, что оптимизированное представление становится короче и ближе к SSA-форме. Переменные, которые при `-O0` размещались через `alloca`, после `-O2` не требуют явного хранения в памяти.

Основные изменения после оптимизации:

- удалены лишние `alloca`, `store` и `load`;
- короткая функция `square` была встроена в место вызова;
- выражение `square(5)` было вычислено заранее;
- вызов `printf` получает уже готовый аргумент `25`;
- поток управления остался линейным, поскольку в программе нет ветвлений и циклов.

---

## 2.6. Построение CFG

CFG строился с помощью инструмента `opt`. Для IR без оптимизации были созданы dot-файлы для функций `main` и `square`.

```bash
opt -passes=dot-cfg -disable-output main_00.ll
dot -Tpng .main.dot -o cfg_main_00.png
dot -Tpng .square.dot -o cfg_square_00.png
```

**Рисунок 6 — создание dot-файлов и png-файлов CFG для main.c без оптимизации**

<img width="840" height="111" alt="image" src="https://github.com/user-attachments/assets/590c9b04-a1c5-4919-92d0-32822286cb24" />



**Рисунок 7 — CFG функций main и square без оптимизации**

<img width="1277" height="541" alt="image" src="https://github.com/user-attachments/assets/3733c04e-5059-4b33-8adc-c9ac4ee95e0b" />



После оптимизации `-O2` CFG был построен аналогичным способом.

```bash
opt -passes=dot-cfg -disable-output main_02.ll
dot -Tpng .main.dot -o cfg_main_02.png
dot -Tpng .square.dot -o cfg_square_02.png
```

**Рисунок 8 — создание CFG для main.c после оптимизации -O2**

<img width="873" height="112" alt="image" src="https://github.com/user-attachments/assets/5088f33b-60b0-4a14-b9c8-68a2d399d593" />


**Рисунок 9 — CFG функции main и square после оптимизации -O2**

<img width="1281" height="540" alt="image" src="https://github.com/user-attachments/assets/0f433563-9330-443f-871e-74a26485d14e" />


Так как программа не содержит условных операторов и циклов, CFG для каждой функции состоит из одного базового блока. После оптимизации меняется содержимое блока, но структура управления остается линейной.

---
