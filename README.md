# RV32I-functions-in-C
Требуется написать для RV32I функции на Си. Функции должны покрывать хотя бы по одной инструкции RISC-V каждого типа. 
```
int add(int a, int b)
{
  if(a == b) // B-type: beq - branch if equal
    return a + b; // R-type: add
  else 
    return a - b; // R-type: sub
}

int funct(int a, int b) // J-type: вызов add (jal)
{
  return add(a, b) << 3; // I-type: slli и ret (jalr)
}

int upper(void)
{
    return 0x12345000; // U-type: загрузка старшей константы (lui)
}
```
