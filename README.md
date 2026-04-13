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
	addi	sp, sp, -32
	sw	ra, 28(sp)                      # 4-byte Folded Spill
	sw	s0, 24(sp)                      # 4-byte Folded Spill
	addi	s0, sp, 32
	sw	a0, -16(s0)
	sw	a1, -20(s0)
	lw	a0, -16(s0)
	lw	a1, -20(s0)
	bne	a0, a1, .LBB0_2
	j	.LBB0_1
.LBB0_1:
	lw	a0, -16(s0)
	lw	a1, -20(s0)
	add	a0, a0, a1
	sw	a0, -12(s0)
	j	.LBB0_3
.LBB0_2:
	lw	a0, -16(s0)
	lw	a1, -20(s0)
	sub	a0, a0, a1
	sw	a0, -12(s0)
	j	.LBB0_3
.LBB0_3:
	lw	a0, -12(s0)
	lw	ra, 28(sp)                      # 4-byte Folded Reload
	lw	s0, 24(sp)                      # 4-byte Folded Reload
	addi	sp, sp, 32
	ret
.Lfunc_end0:
	.size	add, .Lfunc_end0-add
                                        # -- End function
	.globl	funct                           # -- Begin function funct
	.p2align	2
	.type	funct,@function
funct:                                  # @funct
# %bb.0:
	addi	sp, sp, -16
	sw	ra, 12(sp)                      # 4-byte Folded Spill
	sw	s0, 8(sp)                       # 4-byte Folded Spill
	addi	s0, sp, 16
	sw	a0, -12(s0)
	sw	a1, -16(s0)
	lw	a0, -12(s0)
	lw	a1, -16(s0)
	call	add
	slli	a0, a0, 3
	lw	ra, 12(sp)                      # 4-byte Folded Reload
	lw	s0, 8(sp)                       # 4-byte Folded Reload
	addi	sp, sp, 16
	ret
.Lfunc_end1:
	.size	funct, .Lfunc_end1-funct
                                        # -- End function
	.globl	upper                           # -- Begin function upper
	.p2align	2
	.type	upper,@function
upper:                                  # @upper
# %bb.0:
	addi	sp, sp, -16
	sw	ra, 12(sp)                      # 4-byte Folded Spill
	sw	s0, 8(sp)                       # 4-byte Folded Spill
	addi	s0, sp, 16
	lui	a1, %hi(global)
	lui	a0, 74565
	sw	a0, %lo(global)(a1)
	lw	ra, 12(sp)                      # 4-byte Folded Reload
	lw	s0, 8(sp)                       # 4-byte Folded Reload
	addi	sp, sp, 16
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
	.addrsig_sym add
	.addrsig_sym global
```
# Оптимизация О1
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
# Оптимизация О2
```

```
