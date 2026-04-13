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
