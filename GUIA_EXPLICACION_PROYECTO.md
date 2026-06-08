# GUÍA PARA EXPLICAR EL PROYECTO DEL COMPILADOR MINILANG ESPAÑOL

## INTRODUCCIÓN - ¿QUÉ ES ESTE PROYECTO?

Este proyecto es un **compilador completo** para un lenguaje de programación simple llamado MiniLang. Un compilador es un programa que traduce código escrito en un lenguaje de programación a código que la computadora puede ejecutar.

### Analogía Simple

Imagina que el compilador es como un traductor:

- **Entrada**: Código en MiniLang (lenguaje humano)
- **Salida**: Instrucciones que una máquina virtual puede ejecutar

---

## ESTRUCTURA DEL PROYECTO - VISIÓN GENERAL

El compilador tiene **8 fases principales** que trabajan en secuencia:

Código Fuente (texto)
    ↓
[1. LEXER] → Tokens
    ↓
[2. PARSER] → Árbol AST
    ↓
[3. SEMANTIC] → AST Validado
    ↓
[4. OPTIMIZER] → AST Optimizado
    ↓
[5. IR GENERATOR] → Código Intermedio
    ↓
[6. ASM GENERATOR] → Código Ensamblador
    ↓
[7. MACHINE CODE] → Bytecode
    ↓
[8. VIRTUAL MACHINE] → Ejecución
```

---

## EXPLICACIÓN FASE POR FASE

### FASE 1: LEXER (Análisis Léxico)

**Archivo**: `lexer.py`

**¿Qué hace?**
Convierte el texto del programa en "tokens" (piezas significativas).

**Analogía**: Como cuando lees una oración y separas las palabras:

- Texto: `"x = 5 + 3"`
- Tokens: `[IDENT("x"), ASSIGN("="), NUMBER("5"), PLUS("+"), NUMBER("3")]`

**Puntos clave para explicar**:

1. Lee el código carácter por carácter
2. Identifica patrones (números, identificadores, operadores)
3. Ignora espacios y comentarios
4. Reporta errores si encuentra caracteres inválidos

**Ejemplo práctico**:
```
Entrada: "read x; print x;"
Salida: 

- Token(READ, "read", línea=1, col=1)
  - Token(IDENT, "x", línea=1, col=6)
  - Token(SEMI, ";", línea=1, col=7)
  - Token(PRINT, "print", línea=1, col=9)
  - Token(IDENT, "x", línea=1, col=15)
  - Token(SEMI, ";", línea=1, col=16)

```

---

### FASE 2: PARSER (Análisis Sintáctico)

**Archivo**: `parser.py`

**¿Qué hace?**
Convierte la lista de tokens en un árbol que representa la estructura del programa.

**Analogía**: Como analizar la gramática de una oración:

- "Juan come manzanas" → [Sujeto: Juan] [Verbo: come] [Objeto: manzanas]

**Puntos clave para explicar**:

1. Verifica que los tokens sigan las reglas de gramática del lenguaje
2. Construye un Árbol de Sintaxis Abstracta (AST)
3. Maneja precedencia de operadores (ej: `*` antes que `+`)
4. Reporta errores sintácticos (ej: falta punto y coma)

**Ejemplo práctico**:
```
Entrada (tokens): [IDENT("x"), ASSIGN("="), NUMBER("5")]
Salida (AST):
  Assign(
    name="x",
    expr=Number(5)
  )
```

**Precedencia de operadores**:
```
Expresión: 2 + 3 * 4
AST correcto:
  BinaryOp(
    left=Number(2),
    op="+",
    right=BinaryOp(
      left=Number(3),
      op="*",
      right=Number(4)
    )
  )
Resultado: 2 + (3 * 4) = 14```

### FASE 3: SEMANTIC (Análisis Semántico)

**Archivo**: `semantic.py`

**¿Qué hace?**
Verifica que el programa tenga sentido lógico (que las variables se usen correctamente).

**Analogía**: Como revisar que una oración tenga sentido lógico:

- ✓ "El gato come pescado" (correcto)
- ✗ "El come gato pescado" (sintaxis incorrecta)
- ✗ "Uso el coche antes de comprarlo" (error semántico)

**Puntos clave para explicar**:

1. **Tabla de símbolos**: Registra todas las variables del programa
2. **Análisis de inicialización**: Verifica que las variables se asignen antes de usarse
3. **Análisis de flujo**: Rastrea qué variables están inicializadas en cada punto del código

**Ejemplo práctico**:
```
Código problemático:
  print x;    // ❌ Error: x no está inicializada
  x = 5;

Código correcto:
  x = 5;
  print x;    // ✓ Correcto: x está inicializada
```

**Análisis de flujo en if-else**:
```
if condicion {
    x = 1;
} else {
    x = 2;
}
print x;  // ✓ x está garantizada en ambas ramas

if condicion {
    y = 1;
} else {
    // y no se asigna aquí
}
print y;  // ⚠️ Warning: y puede no estar inicializada
```

---

### FASE 4: OPTIMIZER (Optimización)

**Archivo**: `optimizer.py`

**¿Qué hace?**
Mejora el código eliminando cálculos innecesarios.

**Técnica principal: Plegado de Constantes (Constant Folding)**

**Puntos clave para explicar**:

1. Evalúa expresiones constantes en tiempo de compilación
2. Simplifica el código resultante
3. Reduce el trabajo en tiempo de ejecución

**Ejemplos prácticos**:

**Ejemplo 1 - Aritmética constante**:
```
Código original:  x = 5 + 3 * 2;
Después de optimizar: x = 11;

Explicación:
  3 * 2 = 6  (calculado en compilación)
  5 + 6 = 11 (calculado en compilación)
```

**Ejemplo 2 - Condiciones constantes**:
```
Código original:
  if 1 > 0 {
      print 100;
  } else {
      print 200;
  }

Después de optimizar:
  print 100;

Explicación:
  1 > 0 es siempre verdadero → eliminar rama else
```

**Ejemplo 3 - Sin optimización posible**:
```
Código: x = a + b;
No se puede optimizar porque a y b son variables (valores desconocidos)
```

---

### FASE 5: IR GENERATOR (Generación de Código Intermedio)

**Archivo**: `ir.py`

**¿Qué hace?**
Convierte el AST a un formato más simple llamado "Three Address Code" (Código de Tres Direcciones).

**Formato TAC**: Cada instrucción tiene máximo 3 operandos:
```
resultado = operando1 operador operando2
```

**Puntos clave para explicar**:

1. Simplifica expresiones complejas usando variables temporales
2. Convierte estructuras de control (if, while) en saltos con etiquetas
3. Es independiente de la arquitectura (puede usarse para diferentes máquinas)

**Ejemplos prácticos**:

**Ejemplo 1 - Expresión simple**:
```
Código: x = a + b;

IR:
  t1 = a + b
  x = t1
```

**Ejemplo 2 - Expresión compleja**:
```
Código: result = (a + b) * (c - d);

IR:
  t1 = a + b
  t2 = c - d
  t3 = t1 * t2
  result = t3

Explicación:
  Se crean temporales (t1, t2, t3) para dividir la expresión
```

**Ejemplo 3 - Estructura if-else**:
```
Código:
  if x > 0 {
      y = 1;
  } else {
      y = 2;
  }

IR:
  t1 = x > 0
  ifnz t1 L1      // Si t1 ≠ 0, saltar a L1 (rama verdadera)
  y = 2           // Rama falsa
  goto L2
  label L1        // Rama verdadera
  y = 1
  label L2        // Fin del if
```

**Ejemplo 4 - Bucle while**:
```
Código:
  while i < 10 {
      i = i + 1;
  }

IR:
  label L1        // Inicio del bucle
  t1 = i < 10
  ifnz t1 L2      // Si verdadero, ejecutar cuerpo
  goto L3         // Si falso, salir
  label L2        // Cuerpo del bucle
  t2 = i + 1
  i = t2
  goto L1         // Volver al inicio
  label L3        // Fin del bucle
```

---

### FASE 6: ASM GENERATOR (Generación de Ensamblador)

**Archivo**: `codegen_asm.py`

**¿Qué hace?**
Convierte el código intermedio a ensamblador para una máquina virtual basada en acumulador.

**Concepto clave: Máquina de Acumulador**

- Tiene UN solo registro (el acumulador)
- Todas las operaciones usan el acumulador

**Instrucciones principales**:

- `LOAD x`: Cargar valor de memoria en acumulador
- `STORE x`: Guardar acumulador en memoria
- `ADD x`: Sumar memoria al acumulador
- `SUB x`: Restar memoria del acumulador
- `JMP label`: Saltar a etiqueta
- `JLT label`: Saltar si acumulador < 0

**Ejemplos prácticos**:

**Ejemplo 1 - Suma simple**:
```
IR: c = a + b

Ensamblador:
  LOAD a      // ACC = valor de 'a'
  ADD b       // ACC = ACC + valor de 'b'
  STORE c     // guardar ACC en 'c'
```

**Ejemplo 2 - Resta**:
```
IR: c = a - b

Ensamblador:
  LOAD a      // ACC = a
  SUB b       // ACC = ACC - b
  STORE c     // c = ACC
```

**Ejemplo 3 - Negación**:
```
IR: y = -x

Ensamblador:
  LOAD const_0   // ACC = 0
  SUB x          // ACC = 0 - x = -x
  STORE y        // y = ACC
```

**Ejemplo 4 - Comparación**:
```
IR: t1 = x < y

Ensamblador:
  LOAD x           // ACC = x
  SUB y            // ACC = x - y
  JLT L_true       // Si ACC < 0, saltar a verdadero
  LOAD const_0     // Falso: ACC = 0
  STORE t1
  JMP L_end
  LABEL L_true     // Verdadero: ACC = 1
  LOAD const_1
  STORE t1
  LABEL L_end
```

---

### FASE 7: MACHINE CODE (Generación de Código Máquina)

**Archivo**: `codegen_machine.py`

**¿Qué hace?**
Convierte el ensamblador a bytecode (números que la máquina virtual puede ejecutar directamente).

**Concepto: Bytecode**
Array de enteros donde cada instrucción ocupa 2 posiciones:
```
[opcode, operando, opcode, operando, ...]
```

**Tabla de opcodes**:
```
LOAD:  1    STORE: 2    ADD:   3    SUB:   4
MUL:   5    DIV:   6    JMP:   7    JLT:   8
JGT:   9    JLE:  10    JGE:  11    JEQ:  12
JNE:  13    IN:   14    OUT:  15    HALT: 16
```

**Puntos clave para explicar**:

1. **Fase de ensamblado**: Convierte texto a instrucciones
2. **Fase de enlace**: Asigna direcciones de memoria y resuelve etiquetas
3. **Asignación de memoria**: Variables, temporales y constantes

**Ejemplo práctico**:
```
Ensamblador:
  LOAD x
  ADD y
  STORE z
  HALT

Bytecode:
  [1, 0,    // LOAD (opcode=1) de dirección 0 (x)
   3, 1,    // ADD (opcode=3) de dirección 1 (y)
   2, 2,    // STORE (opcode=2) en dirección 2 (z)
   16, -1]  // HALT (opcode=16)

Asignación de memoria:
  Dirección 0: x
  Dirección 1: y
  Dirección 2: z
```

---

### FASE 8: VIRTUAL MACHINE (Máquina Virtual)

**Archivo**: `runtime_vm.py`

**¿Qué hace?**
Ejecuta el bytecode simulando una computadora simple.

**Componentes de la VM**:

- **PC (Program Counter)**: Apunta a la instrucción actual
- **ACC (Acumulador)**: Registro para cálculos
- **MEM (Memoria)**: Array para variables
- **Outputs**: Lista de valores impresos

**Ciclo de ejecución**:
```
mientras PC < longitud_código:
    1. Leer opcode en posición PC
    2. Leer operando en posición PC+1
    3. PC = PC + 2
    4. Ejecutar la instrucción
    5. Si es HALT, terminar
```

**Ejemplo de ejecución paso a paso**:
```
Bytecode: [1, 0, 3, 1, 2, 2, 16, -1]
Memoria inicial: [5, 3, 0]  // x=5, y=3, z=0

Paso 1: PC=0
  Ejecutar: LOAD 0
  ACC = MEM[0] = 5
  PC = 2

Paso 2: PC=2
  Ejecutar: ADD 1
  ACC = ACC + MEM[1] = 5 + 3 = 8
  PC = 4

Paso 3: PC=4
  Ejecutar: STORE 2
  MEM[2] = ACC = 8
  PC = 6

Paso 4: PC=6
  Ejecutar: HALT
  Terminar

Resultado: z = 8
```

---

## CÓMO EXPLICAR EL FLUJO COMPLETO

### Usa este ejemplo de principio a fin:

**Código fuente**:
```
read x;
y = x * 2;
print y;
end
```

**PASO 1 - LEXER**:
```
Tokens: [READ, IDENT(x), SEMI, IDENT(y), ASSIGN, IDENT(x), STAR, NUMBER(2), SEMI, PRINT, IDENT(y), SEMI, END]
```

**PASO 2 - PARSER**:
```
Program([
  Read("x"),
  Assign("y", BinaryOp(Var("x"), "*", Number(2))),
  Print(Var("y"))
])
```

**PASO 3 - SEMANTIC**:
```
✓ x está inicializada por read
✓ y está inicializada por asignación
✓ No hay errores
```

**PASO 4 - OPTIMIZER**:
```
No hay optimizaciones (x es variable, no constante)
```

**PASO 5 - IR**:
```
read x
t1 = 2
t2 = x * t1
y = t2
print y
label END
```

**PASO 6 - ASM**:
```
IN x
LOAD const_2
STORE t1
LOAD x
MUL t1
STORE t2
LOAD t2
STORE y
OUT y
HALT
```

**PASO 7 - MACHINE CODE**:
```
Bytecode: [14, 0, 1, 1, 2, 2, 1, 0, 5, 2, 2, 3, 1, 3, 2, 4, 15, 4, 16, -1]
Memoria: [x=?, const_2=2, t1=?, t2=?, y=?]
```

**PASO 8 - EJECUCIÓN**:
```
Input: 5
Resultado:
  x = 5
  t1 = 2
  t2 = 10
  y = 10
Output: 10
```

---

## PREGUNTAS FRECUENTES Y CÓMO RESPONDERLAS

### P1: "¿Por qué necesitamos tantas fases?"

**R**: Cada fase tiene una responsabilidad específica:

- Lexer: Reconocer caracteres
- Parser: Verificar estructura
- Semantic: Verificar significado
- Optimizer: Mejorar eficiencia
- IR: Simplificar
- ASM: Traducir a instrucciones
- Machine: Generar código ejecutable
- VM: Ejecutar

Es como cocinar: comprar ingredientes → lavar → cortar → cocinar → servir

### P2: "¿Por qué usamos código intermedio?"

**R**: 

1. Simplifica optimizaciones
2. Permite generar código para diferentes arquitecturas
3. Separa análisis de generación de código

### P3: "¿Qué es un acumulador?"

**R**: Un registro especial donde se hacen todos los cálculos.
Analogía: Una calculadora donde el resultado queda en pantalla.

### P4: "¿Para qué sirve la tabla de símbolos?"

**R**: Llevar registro de todas las variables:

- Nombre
- Si está inicializada
- Dirección de memoria (más tarde)

### P5: "¿Cómo funciona la máquina virtual?"

**R**: Simula una computadora simple:

1. Lee instrucciones una por una
2. Ejecuta operaciones sobre memoria y acumulador
3. Mantiene un contador de programa (PC)

---

## ARQUITECTURA DEL PROYECTO - ARCHIVOS

```
project_final/
├── minilang_compiler/
│   ├── tokens.py          # Definición de tokens
│   ├── ast_nodes.py       # Nodos del AST
│   ├── errors.py          # Formateo de errores
│   ├── lexer.py          # Análisis léxico
│   ├── parser.py         # Análisis sintáctico
│   ├── semantic.py       # Análisis semántico
│   ├── optimizer.py      # Optimizaciones
│   ├── ir.py             # Código intermedio
│   ├── codegen_asm.py    # Generación de ensamblador
│   ├── codegen_machine.py # Generación de bytecode
│   ├── runtime_vm.py     # Máquina virtual
│   └── compiler.py       # Orquestador principal
├── tests/                # Pruebas unitarias
├── web_app.py           # Interfaz web
└── README.md            # Documentación
```

---

## CARACTERÍSTICAS DEL LENGUAJE MINILANG

### Tipos de datos

- **Solo enteros**: No hay flotantes, strings, ni booleanos

### Declaraciones

- `read variable;` - Leer valor
- `print expresion;` - Imprimir valor
- `variable = expresion;` - Asignación
- `if condicion { ... } else { ... }` - Condicional (obligatorio else)
- `while condicion { ... }` - Bucle

### Operadores

- **Aritméticos**: `+`, `-`, `*`, `/`
- **Relacionales**: `<`, `>`, `<=`, `>=`, `==`, `!=`
- **Unario**: `-` (negación)

### Ejemplo completo

```
// Programa que calcula factorial
read n;
fact = 1;
i = 1;
while i <= n {
    fact = fact * i;
    i = i + 1;
}
print fact;
end
```

---

## CÓMO DEMOSTRAR EL PROYECTO

### Demo 1: Interfaz Web

1. Ejecutar: `python web_app.py`
2. Abrir navegador: `http://localhost:5000`
3. Mostrar ejemplos predefinidos
4. Compilar y ver todas las etapas

### Demo 2: Línea de comandos

```bash
# Compilar y ejecutar
python -m minilang_compiler.compiler programa.minilang --run --inputs 5

# Ver código intermedio
python -m minilang_compiler.compiler programa.minilang --trace-ir

# Ver ensamblador
python -m minilang_compiler.compiler programa.minilang --trace-asm

# Ver ejecución paso a paso
python -m minilang_compiler.compiler programa.minilang --run --trace-vm
```

### Demo 3: Mostrar tests

```bash
pytest -v
```

---

## PUNTOS FUERTES DEL PROYECTO

1. **Compilador completo**: No es solo un parser, cubre todas las fases
2. **Bien documentado**: Código comentado y especificación técnica
3. **Testing**: Suite de pruebas automatizadas
4. **Interfaz web**: Fácil de usar y demostrar
5. **Optimizaciones**: Incluye plegado de constantes
6. **Análisis semántico**: Detecta errores de inicialización
7. **Máquina virtual funcional**: Ejecuta programas reales

---

## CONSEJOS PARA LA PRESENTACIÓN

### Preparación

1. Practica ejecutar los demos
2. Ten ejemplos preparados
3. Conoce bien el flujo de una fase
4. Prepara respuestas a preguntas comunes

### Durante la presentación

1. **Empieza simple**: Muestra un ejemplo pequeño
2. **Ve paso a paso**: Explica cada fase con su entrada/salida
3. **Usa la interfaz web**: Es visual y fácil de entender
4. **Muestra el código**: Pero solo secciones pequeñas
5. **Demuestra que funciona**: Ejecuta programas

### Estructura sugerida

1. **Introducción** (2 min): ¿Qué es un compilador?
2. **Arquitectura** (3 min): Las 8 fases con diagrama
3. **Demo** (10 min): Ejemplo completo paso a paso
4. **Características** (3 min): Puntos fuertes
5. **Q&A** (variable): Preguntas

---

## GLOSARIO DE TÉRMINOS

- **Token**: Unidad mínima con significado (palabra)
- **AST**: Árbol que representa la estructura del programa
- **TAC**: Código de tres direcciones (instrucciones simples)
- **Bytecode**: Código en formato binario/numérico
- **Acumulador**: Registro para operaciones aritméticas
- **Opcode**: Código de operación (número que identifica instrucción)
- **Symbol Table**: Tabla que registra las variables
- **Constant Folding**: Evaluar expresiones constantes en compilación
- **VM**: Virtual Machine (computadora simulada en software)

---

## RECURSOS ADICIONALES

- README.md: Guía de instalación y uso
- docs/especificacion_tecnica.md: Documentación técnica completa
- tests/: Ejemplos de código y casos de prueba
- web_app.py: Código de la interfaz web

---

## NOTAS FINALES

**Recuerda**: No necesitas memorizar cada línea de código. Lo importante es:

1. Entender el propósito de cada fase
2. Poder explicar el flujo general
3. Demostrar que el proyecto funciona
4. Responder preguntas conceptuales

**Si te preguntan algo que no sabes**:

- "Déjame verificar el código para darte la respuesta exacta"
- "El concepto general es X, pero puedo mostrarte la implementación"
- "Esa es una pregunta interesante, revisemos juntos cómo está implementado"

**¡Éxito en tu presentación!**
