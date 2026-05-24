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

# 3. Индивидуальное задание

## 3.1. Исходная конструкция

Индивидуальное задание относится к варианту **2.13: Условные операторы**. Для анализа был создан файл `max.c`.

```c
#include <stdio.h>

int max(int a, int b){
    if (a > b)
        return a;
    else
        return b;
}

int main(){
    int x = 10;
    int y = 20;
    int result = max(x, y);
    printf("max = %d\n", result);
    return 0;
}
```

**Рисунок 10 — исходный код max.c**

<img width="1279" height="663" alt="image" src="https://github.com/user-attachments/assets/8ebbe197-0313-4e9b-a51b-dd06fe6330f0" />


Функция max принимает два целых числа и возвращает большее из них с помощью условного оператора if-else. В функции main значения 10 и 20 передаются в max, после чего результат выводится через printf.

---

## 3.2. AST индивидуального задания

```bash
clang -Xclang -ast-dump -fsyntax-only max.c > ast_max.txt
grep -A60 "FunctionDecl.*main" ast_max.txt
```

**Рисунок 11 — AST функции main для индивидуального задания**

<img width="1277" height="798" alt="image" src="https://github.com/user-attachments/assets/b0342a33-d6fc-4952-90bc-15af65bebc89" />


В AST функция main представлена узлом FunctionDecl. Внутри видны объявления переменных x, y, result, вызов max (CallExpr) и вызов printf.

---

## 3.3. LLVM IR индивидуального задания

```bash
clang -O0 -S -emit-llvm max.c -o max_00.ll
clang -O2 -S -emit-llvm max.c -o max_02.ll
```

Для просмотра ключевых инструкций были использованы команды `grep` по словам `define`, `alloca`, `store`, `load`, `strtod` и `printf`.

```bash
grep -n "define\|alloca\|store\|load\|strtod\|printf" max_00.ll
grep -n "define\|alloca\|store\|load\|strtod\|printf" max_02.ll
```

**Рисунок 12 — IR max.c без оптимизации**

<img width="1280" height="784" alt="image" src="https://github.com/user-attachments/assets/19b60b00-25ca-44ea-b1f2-a9507b73daee" />


**Рисунок 13 — сравнение ключевых инструкций IR max.c до и после оптимизации**

<img width="1280" height="797" alt="image" src="https://github.com/user-attachments/assets/72cc6429-b6e9-402c-b3ff-0485d24e8537" />


В IR без оптимизации присутствуют alloca, store, load, а также условный переход br i1 с тремя метками (%9 — then, %11 — else, %13 — merge). Это классическое представление ветвления с phi-узлом (результат выбирается через память %3).


После оптимизации -O2:
- alloca, store, load полностью удалены;
- условный переход br i1 заменён на встроенную функцию llvm.smax.i32 (встроенный intrinsic для signed maximum);
- функция max не встроена в main, но её тело упрощено до одной инструкции;
- в main результат max(10, 20) вычислен на этапе компиляции — в printf передаётся константа 20.
---

## 3.4. Сравнение IR индивидуального задания

```bash
diff notation_00.ll notation_02.ll
```

**Рисунок 14 — сравнение max_00.ll и max_02.ll через diff**

<img width="1280" height="798" alt="image" src="https://github.com/user-attachments/assets/9597f0d6-c253-4d50-ae8c-f59d48b82ccd" />


В оптимизированном IR лишние операции работы с памятью удалены, ветвление заменено на встроенную функцию llvm.smax.i32, а в main результат max(10, 20) вычислен статически.
---

## 3.5. CFG индивидуального задания

```bash
opt -passes=dot-cfg -disable-output max_00.ll
dot -Tpng .main.dot -o cfg_max_00.png
xdg-open cfg_max_00.png
```

**Рисунок 15 — создание CFG для max.c без оптимизации**

<img width="966" height="109" alt="image" src="https://github.com/user-attachments/assets/9fcdef3c-c9f6-42db-a7b9-b4080788add4" />


**Рисунок 16 — CFG функции main для max.c без оптимизации**

<img width="1279" height="799" alt="image" src="https://github.com/user-attachments/assets/34b79116-d1ad-4f20-b141-cb92c1d656e5" />


```bash
opt -passes=dot-cfg -disable-output max_02.ll
dot -Tpng .main.dot -o cfg_max_02.png
xdg-open cfg_max_02.png
```

**Рисунок 17 — CFG функции main для max.c после оптимизации -O2**

<img width="1279" height="797" alt="image" src="https://github.com/user-attachments/assets/a7c63152-212f-4844-97dd-7fdf6f76deea" />


Ветвление полностью устранено благодаря замене if-else на intrinsic llvm.smax.i32 и последующей константной свёртке.

---

## 3.6. Вывод по индивидуальному заданию

На уровне AST условный оператор if-else представлен узлом IfStmt с условием (BinaryOperator >), then-веткой (ReturnStmt с a) и else-веткой (ReturnStmt с b). Вызов функции max в main отображается как CallExpr с аргументами x и y.
В IR без оптимизации (-O0) компилятор сохраняет локальные переменные через alloca, store и load. Условный оператор реализован через icmp sgt и br i1 с тремя метками (then, else, merge), что создаёт разветвлённый CFG из четырёх базовых блоков.
После оптимизации -O2 лишние alloca, store и load удаляются, код переводится в SSA-форму. Ветвление br i1 заменено на встроенную функцию llvm.smax.i32 (intrinsic для signed maximum), которая выбирает большее из двух значений без условных переходов. В функции main результат max(10, 20) вычислен на этапе компиляции и подставлен как константа 20 в вызов printf.
Следовательно, LLVM выполняет локальное упрощение IR: устраняет ветвления, переводит в SSA-форму, применяет встроенные intrinsics и сворачивает константы. Однако функция max не исчезает полностью — она преобразуется в llvm.smax.i32, что позволяет сохранить её семантику при полном отсутствии ветвлений в CFG.

---

# 4. Ответы на контрольные вопросы

## 4.1. Что такое Clang, и какова его роль в процессе компиляции программ?

**Clang** — это компилятор языков C, C++ и Objective-C. Он выполняет роль фронтенда: анализирует исходный код, строит AST, выполняет проверки и генерирует LLVM IR.

---

## 4.2. Что представляет собой LLVM и как он используется в современных компиляторах?

**LLVM** — это инфраструктура для построения компиляторов. Она содержит промежуточное представление LLVM IR, набор оптимизаций и средства генерации машинного кода для разных архитектур.

---

## 4.3. Чем отличается AST от LLVM IR?

**AST** отражает синтаксическую структуру исходной программы, а **LLVM IR** является более низкоуровневым промежуточным представлением, удобным для оптимизации и генерации машинного кода.

---

## 4.4. Для чего необходимо промежуточное представление IR?

IR позволяет отделить анализ исходного языка от оптимизации и генерации машинного кода. Благодаря этому одни и те же оптимизации можно применять к программам, написанным на разных языках.

---

## 4.5. Что делает инструкция alloca в LLVM IR?

Инструкция `alloca` выделяет память в стеке функции для локальной переменной. При оптимизации такие размещения часто удаляются или переводятся в регистровое представление.

---

## 4.6. Зачем нужна оптимизация кода?

Оптимизация нужна для уменьшения количества инструкций, удаления лишних вычислений, сокращения обращений к памяти, упрощения потока управления и повышения скорости выполнения программы.

---

## 4.7. Что такое SSA-форма?

**SSA-форма** — это форма представления программы, в которой каждая переменная получает значение только один раз. Она упрощает анализ зависимостей и применение оптимизаций.

---

## 4.8. Что такое CFG?

**CFG** — это граф потока управления. Его вершинами являются базовые блоки, а ребра показывают возможные переходы между ними.

---

## 4.9. Как представляются арифметические операции в LLVM IR?

Арифметические операции представлены отдельными инструкциями. Например, умножение целых чисел может быть записано как:

```llvm
mul i32 %a, %b
```

---

## 4.10. Почему функции в LLVM IR являются отдельными единицами анализа?

Функция имеет собственные аргументы, локальные переменные, базовые блоки и CFG, поэтому ее удобно анализировать и оптимизировать отдельно.

---

## 4.11. Что происходит с короткой функцией, если она вызывается один раз?

Оптимизатор может встроить такую функцию в место вызова. После этого становятся возможны дополнительные оптимизации, например свертка констант.

---

## 4.12. Какие преимущества дают IR и CFG по сравнению с анализом исходного текста на C?

IR и CFG имеют формальную структуру, поэтому в них проще анализировать зависимости, поток управления, использование переменных и достижимость кода.

---
```bash
sudo apt update
sudo apt install -y clang llvm llvm-dev llvm-runtime graphviz xdg-utils

clang --version
opt --version
dot -V
llvm-config --version

mkdir -p ~/lab7
cd ~/lab7

clang -Xclang -ast-dump -fsyntax-only main.c > ast_main.txt

clang -O0 -S -emit-llvm main.c -o main_00.ll
clang -O2 -S -emit-llvm main.c -o main_02.ll

opt -passes=dot-cfg -disable-output main_00.ll
dot -Tpng .main.dot -o cfg_main_00.png
dot -Tpng .square.dot -o cfg_square_00.png

opt -passes=dot-cfg -disable-output main_02.ll
dot -Tpng .main.dot -o cfg_main_02.png

mkdir ~/lab7/individual_part
cd ~/lab7/individual_part

clang -Xclang -ast-dump -fsyntax-only max.c > ast_max.txt
grep -A60 "FunctionDecl.*main" ast_max.txt

clang -O0 -S -emit-llvm max.c -o max_00.ll
clang -O2 -S -emit-llvm max.c -o max_02.ll

grep -n "define\|alloca\|store\|load\|icmp\|br\|ret\|printf" max_00.ll
grep -n "define\|alloca\|store\|load\|icmp\|br\|select\|ret\|printf\|smax" max_02.ll

diff max_00.ll max_02.ll

opt -passes=dot-cfg -disable-output max_00.ll
dot -Tpng .main.dot -o cfg_max_00.png
xdg-open cfg_max_00.png

opt -passes=dot-cfg -disable-output max_02.ll
dot -Tpng .main.dot -o cfg_max_02.png
xdg-open cfg_max_02.png
```
