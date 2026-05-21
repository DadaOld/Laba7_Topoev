<img width="1284" height="796" alt="image" src="https://github.com/user-attachments/assets/1dd875b8-315f-4955-82c4-a0c6a9ed5c8a" /># Лабораторная работа №7

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

<img width="886" height="776" alt="image" src="https://github.com/user-attachments/assets/b134d32d-4a91-4794-9ed9-79322f1a79ad" />

После установки были проверены версии основных инструментов. Проверка выполнялась командами `clang --version`, `opt --version`, `dot -V` и `llvm-config --version`.

```bash
clang --version
opt --version
dot -V
llvm-config --version
```

**Рисунок 2 — проверка версий Clang, opt, Graphviz и создание рабочей папки**

<img width="856" height="579" alt="image" src="https://github.com/user-attachments/assets/4ec0cbe4-1c93-4970-b0b3-f4a193a35593" />

**Рисунок 3 — дополнительная проверка команды llvm-config --version**

<img width="591" height="62" alt="image" src="https://github.com/user-attachments/assets/89dcd118-95d3-4d1b-9740-d4d7709c9386" />

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

<img width="886" height="700" alt="image" src="https://github.com/user-attachments/assets/df1ae7af-e1f0-4541-9237-2ee99be319e4" />

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

**Рисунок 5 — LLVM IR без оптимизации для main.c**

<img width="886" height="813" alt="image" src="https://github.com/user-attachments/assets/2887ebc4-e18c-48da-bfd8-ac5bf7e6e240" />

**Рисунок 6 — сравнение ключевых инструкций в IR без оптимизации и после -O2**

<img width="886" height="778" alt="image" src="https://github.com/user-attachments/assets/2909afc1-1443-4a58-ba09-e0ee3d09d055" />

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

**Рисунок 7 — создание dot-файлов и png-файлов CFG для main.c без оптимизации**

<img width="886" height="474" alt="image" src="https://github.com/user-attachments/assets/69390a72-a327-4c5c-a0d9-bfeaec609e07" />

**Рисунок 8 — CFG функций main и square без оптимизации**

<img width="886" height="783" alt="image" src="https://github.com/user-attachments/assets/1d63972b-2c4f-41e1-a7a3-745b8b813a31" />

После оптимизации `-O2` CFG был построен аналогичным способом.

```bash
opt -passes=dot-cfg -disable-output main_02.ll
dot -Tpng .main.dot -o cfg_main_02.png
dot -Tpng .square.dot -o cfg_square_02.png
```

**Рисунок 9 — создание CFG для main.c после оптимизации -O2**

<img width="886" height="497" alt="image" src="https://github.com/user-attachments/assets/dc5e9072-f248-4e54-b18f-ae4e225d28ec" />

**Рисунок 10 — CFG функции main после оптимизации -O2**

<img width="827" height="754" alt="image" src="https://github.com/user-attachments/assets/91ecf4b8-7b02-4ebb-94ab-3346f2a388a2" />

**Рисунок 11 — CFG функции square после оптимизации -O2**

<img width="768" height="593" alt="image" src="https://github.com/user-attachments/assets/4435093b-b387-4d5c-968e-df80bf1605ed" />

Так как программа не содержит условных операторов и циклов, CFG для каждой функции состоит из одного базового блока. После оптимизации меняется содержимое блока, но структура управления остается линейной.

---
