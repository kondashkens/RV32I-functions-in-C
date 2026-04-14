# RV32I-functions-in-C
Требуется написать для RV32I функции на Си. Функции должны покрывать хотя бы по одной инструкции RISC-V каждого типа. 
```
int g;

int funct(int a, int b) {
    if (a < b) {              // B-type (blt)
        g = (a + b) << 3;     // R-type (add), I-type (slli), S-type (sw)
    }
    return g + 1;             // I-type (lw, addi), I-type (jalr для ret), J-type (jal)
}
void start() {
    funct(0x12345000, 5); // U-type (lui для константы), J-type (jal для вызова)
}
```
# Ассемблерный код
Код компилировался с помощью компилятора clang под архитектуру rv32i с уровнем оптимизации O0. 
```
.attribute	4, 16                     # RISC-V ABI: 32-bit
.attribute	5, "rv32i2p1"             # Архитектура RV32I v2.1
.file	"hw2.c"
.text                                 # Секция кода

.globl	add
.p2align	2                         # Выравнивание по 4 байта
.type	add,@function
add:
	addi	sp, sp, -32               # Выделяем 32 байта в стеке
	sw	ra, 28(sp)                 # Сохраняем обратный адрес
	sw	s0, 24(sp)                 # Сохраняем frame pointer
	addi	s0, sp, 32                # Устанавливаем frame pointer (s0 = старая sp + 32)
	sw	a0, -16(s0)                # Сохраняем первый аргумент (a0) на стек
	sw	a1, -20(s0)                # Сохраняем второй аргумент (a1) на стек
	lw	a0, -16(s0)                # Загружаем a0 обратно (излишне, без оптимизаций)
	lw	a1, -20(s0)                # Загружаем a1 обратно
	bne	a0, a1, .LBB0_2           # Если a0 != a1, перейти к .LBB0_2
	j	.LBB0_1                    # Иначе перейти к .LBB0_1 (a0 == a1)
.LBB0_1:
	lw	a0, -16(s0)                # Загружаем a0
	lw	a1, -20(s0)                # Загружаем a1
	add	a0, a0, a1                 # a0 = a0 + a1
	sw	a0, -12(s0)                # Сохраняем результат на стек
	j	.LBB0_3                    # Переход к эпилогу
.LBB0_2:
	lw	a0, -16(s0)                # Загружаем a0
	lw	a1, -20(s0)                # Загружаем a1
	sub	a0, a0, a1                 # a0 = a0 - a1
	sw	a0, -12(s0)                # Сохраняем результат на стек
	j	.LBB0_3
.LBB0_3:
	lw	a0, -12(s0)                # Загружаем результат в a0 (возвращаемое значение)
	lw	ra, 28(sp)                 # Восстанавливаем ra
	lw	s0, 24(sp)                 # Восстанавливаем s0
	addi	sp, sp, 32                # Освобождаем стек
	ret                            # Возврат

.globl	funct
.p2align	2
.type	funct,@function
funct:
	addi	sp, sp, -16               # Выделяем 16 байт под стек
	sw	ra, 12(sp)                 # Сохраняем ra (т.к. вызываем add)
	sw	s0, 8(sp)                  # Сохраняем s0
	addi	s0, sp, 16                # Frame pointer
	sw	a0, -12(s0)                # Сохраняем первый параметр
	sw	a1, -16(s0)                # Сохраняем второй параметр
	lw	a0, -12(s0)                # Загружаем обратно (неоптимизированно)
	lw	a1, -16(s0)                #
	call	add                      # Вызов add(a0, a1), результат в a0
	slli	a0, a0, 3                 # Умножаем результат на 8 (сдвиг влево на 3)
	lw	ra, 12(sp)                 # Восстанавливаем ra
	lw	s0, 8(sp)                  # Восстанавливаем s0
	addi	sp, sp, 16                # Освобождаем стек
	ret

.globl	upper
.p2align	2
.type	upper,@function
upper:
	addi	sp, sp, -16               # Выделяем стек (хотя вызовов нет)
	sw	ra, 12(sp)                 # Сохраняем ra (избыточно)
	sw	s0, 8(sp)                  # Сохраняем s0
	addi	s0, sp, 16
	lui	a1, %hi(global)           # Загружаем старшие 20 бит адреса global в a1
	lui	a0, 74565                 # Загружаем 74565 << 12 = 0x12345000 в a0
	sw	a0, %lo(global)(a1)       # Сохраняем a0 по полному адресу global (global = 0x12345000)
	lw	ra, 12(sp)                 # Восстанавливаем ra
	lw	s0, 8(sp)                  # Восстанавливаем s0
	addi	sp, sp, 16                # Освобождаем стек
	ret

.type	global,@object
.bss                                  # Неинициализированные данные
.globl	global
.p2align	2, 0x0
global:
	.word	0                         # 4 байта, начальное значение 0
	.size	global, 4

.ident	"clang version 22.1.2 ..."
.section	".note.GNU-stack","",@progbits
.addrsig
.addrsig_sym add
.addrsig_sym global
```
# Оптимизация О1
```
.attribute	4, 16                     # RISC-V ABI: 32-bit
.attribute	5, "rv32i2p1"             # Архитектура RV32I v2.1
.file	"hw2.c"
.text
.globl	add
.p2align	2                         # Выравнивание по 4 байта
.type	add,@function
add:
    # Вход: a0 = x, a1 = y
    # Выход: a0 = (x == y) ? x + y : x - y

    beq	a0, a1, .LBB0_2            # если x == y, перейти на .LBB0_2 (пропустить neg)
    # Если x != y:
    neg	a1, a1                       # a1 = -y
.LBB0_2:
    # Теперь: если x == y, то a1 остался равным y; если x != y, то a1 = -y.
    add	a0, a1, a0                   # a0 = a0 + a1 = x + (y или -y)
    # При x == y: a0 = x + y
    # При x != y: a0 = x + (-y) = x - y
    ret
.Lfunc_end0:
    .size	add, .Lfunc_end0-add
.globl	funct
.p2align	2
.type	funct,@function
funct:
    # Вход: a0 = x, a1 = y
    # Выход: a0 = ((x == y) ? x + y : x - y) * 8

    beq	a0, a1, .LBB1_2            # если x == y, перейти (пропустить neg)
    neg	a1, a1                       # иначе a1 = -y
.LBB1_2:
    add	a0, a1, a0                   # a0 = x ± y (как в add)
    slli	a0, a0, 3                  # Логический сдвиг влево на 3 => умножение на 8
    ret
.Lfunc_end1:
    .size	funct, .Lfunc_end1-funct
.globl	upper
.p2align	2
.type	upper,@function
upper:
    # Записывает константу 0x12345000 в глобальную переменную global
    # (0x12345 << 12 = 0x12345000)

    lui	a1, %hi(global)              # a1 = старшие 20 бит адреса global
    lui	a2, 74565                    # a2 = 74565 << 12 = 0x12345000
    lui	a0, 74565                    # a0 = тоже 0x12345000 (бесполезная инструкция,
                                     # можно было использовать a2, но компилятор так сгенерировал)
    sw	a2, %lo(global)(a1)          # сохраняем a2 по полному адресу global (a1 + %lo)
    ret
.Lfunc_end2:
    .size	upper, .Lfunc_end2-upper
.type	global,@object
.bss                                  # сегмент неинициализированных данных
.globl	global
.p2align	2, 0x0
global:
    .word	0                         # 4 байта, начальное значение 0
    .size	global, 4
.ident	"clang version 22.1.2 ..."
.section	".note.GNU-stack","",@progbits
.addrsig
```
# Оптимизация О2
```
.attribute	4, 16
.attribute	5, "rv32i2p1"
.file	"hw2.c"
.text
.globl	add
.p2align	2                         # выравнивание на 4 байта
.type	add,@function
add:
    # Вход: a0 = x, a1 = y
    # Выход: a0 = (x == y) ? x + y : x - y
    beq	a0, a1, .LBB0_2            # если x == y, перейти к .LBB0_2 (пропустить neg)
    # иначе (x != y):
    neg	a1, a1                       # a1 = -y
.LBB0_2:
    add	a0, a1, a0                   # a0 = a0 + a1 = x + (y или -y)
    # Если были равны: a0 = x + y
    # Если не равны:   a0 = x + (-y) = x - y
    ret
.Lfunc_end0:
    .size	add, .Lfunc_end0-add
.globl	funct
.p2align	2
.type	funct,@function
funct:
    # Вход: a0 = x, a1 = y
    # Выход: a0 = ((x == y) ? x + y : x - y) * 8
    beq	a0, a1, .LBB1_2            # если x == y, пропустить neg
    neg	a1, a1                       # иначе a1 = -y
.LBB1_2:
    add	a0, a1, a0                   # a0 = x ± y (как в add)
    slli	a0, a0, 3                  # логический сдвиг влево на 3 -> умножение на 8
    ret
.Lfunc_end1:
    .size	funct, .Lfunc_end1-funct
.globl	upper
.p2align	2
.type	upper,@function
upper:
    # Записывает константу 0x12345000 в глобальную переменную global
    lui	a1, %hi(global)              # a1 = старшие 20 бит адреса global
    lui	a2, 74565                    # a2 = 74565 << 12 = 0x12345000
    lui	a0, 74565                    # a0 = то же самое (бесполезная инструкция)
    sw	a2, %lo(global)(a1)          # сохраняем a2 по адресу (a1 + %lo(global))
    ret
.Lfunc_end2:
    .size	upper, .Lfunc_end2-upper
.type	global,@object
.bss                                  # сегмент неинициализированных данных
.globl	global
.p2align	2, 0x0                    # выравнивание на 4 байта
global:
    .word	0                         # 4 байта, начальное значение 0
    .size	global, 4
.ident	"clang version 22.1.2 (https://github.com/llvm/llvm-project 1ab49a973e210e97d61e5db6557180dcb92c3e98)"
.section	".note.GNU-stack","",@progbits   # стек не требует исполнения
.addrsig                                      # таблица сигнатур адресов
```
# Оптимизация О3
```
	.attribute	4, 16
	.attribute	5, "rv32i2p1"
	.file	"hw2.c"
	.text
	.globl	add                             # -- Begin function add
	.p2align	2
	.type	add,@function
add:                                    # @add
# %bb.0:
	beq	a0, a1, .LBB0_2
# %bb.1:
	neg	a1, a1
.LBB0_2:
	add	a0, a1, a0
	ret
.Lfunc_end0:
	.size	add, .Lfunc_end0-add
                                        # -- End function
	.globl	funct                           # -- Begin function funct
	.p2align	2
	.type	funct,@function
funct:                                  # @funct
# %bb.0:
	beq	a0, a1, .LBB1_2
# %bb.1:
	neg	a1, a1
.LBB1_2:
	add	a0, a1, a0
	slli	a0, a0, 3
	ret
.Lfunc_end1:
	.size	funct, .Lfunc_end1-funct
                                        # -- End function
	.globl	upper                           # -- Begin function upper
	.p2align	2
	.type	upper,@function
upper:                                  # @upper
# %bb.0:
	lui	a1, %hi(global)
	lui	a2, 74565
	lui	a0, 74565
	sw	a2, %lo(global)(a1)
	ret
.Lfunc_end2:
	.size	upper, .Lfunc_end2-upper
                                        # -- End function
	.type	global,@object                  # @global
	.bss
	.globl	global
	.p2align	2, 0x0
global:
	.word	0                               # 0x0
	.size	global, 4

	.ident	"clang version 22.1.2 (https://github.com/llvm/llvm-project 1ab49a973e210e97d61e5db6557180dcb92c3e98)"
	.section	".note.GNU-stack","",@progbits
	.addrsig
```
