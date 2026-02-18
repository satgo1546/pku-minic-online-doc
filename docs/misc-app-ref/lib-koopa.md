# Libkoopa C 接口

`koopa.h` 头文件中定义了 Koopa IR 相关的 C 语言接口。通过这些接口，你可以解析文本形式的 Koopa IR，将其转换为内存中的数据结构（Raw Program），或者生成 LLVM IR 等。

使用该库时，需要在源文件中包含头文件：

```c
#include "koopa.h"
```

并在编译时链接 `libkoopa` 库（如果你使用模板 makefile/cmake 文件，我们已经帮你链接好了）。

所有接口的声明均在 `koopa.h` 中。此外，库中涉及的所有内存（如 `koopa_program_t` 和 `koopa_raw_program_builder_t`）都需要手动管理释放，具体请参考各函数的文档说明。

## 结构体

### koopa_raw_program_t

**描述**

Koopa IR 的原始程序结构体。包含全局变量定义和函数定义。

**定义**

```c
typedef struct {
  /// Global values (global allocations only).
  koopa_raw_slice_t values;
  /// Function definitions.
  koopa_raw_slice_t funcs;
} koopa_raw_program_t;
```

**成员**

- `values`: 全局变量定义的切片。切片中的元素类型为 `koopa_raw_value_t`。
- `funcs`: 函数定义的切片。切片中的元素类型为 `koopa_raw_function_t`。

**说明**

对应整个 Koopa IR 文件：
```koopa
// values 对应这里
global @num = alloc i32, 0

// funcs 对应这里
fun @main(): i32 {
  %entry:
    ret 0
}
```

### koopa_raw_slice_t

**描述**

Koopa IR 中的切片结构体。类似于 C++ 的 `std::span` 或 Rust 的 slice，用于表示一段连续的内存区域。

**定义**

```c
typedef struct {
  /// Buffer of slice items.
  const void **buffer;
  /// Length of slice.
  uint32_t len;
  /// Kind of slice items.
  koopa_raw_slice_item_kind_t kind;
} koopa_raw_slice_t;
```

**成员**

- `buffer`: 指向数据指针数组的指针。`buffer[i]` 即为第 `i` 个元素的指针。
- `len`: 切片的长度（元素个数）。
- `kind`: 切片中元素的类型。

**说明**

LibKoopa 中大部分列表（如参数列表、基本块列表、指令列表）都使用此结构存储。

### koopa_raw_function_t

**描述**

Koopa IR 中的函数结构体。

**定义**

```c
typedef struct {
  /// Type of function.
  koopa_raw_type_t ty;
  /// Name of function.
  const char *name;
  /// Parameters.
  koopa_raw_slice_t params;
  /// Basic blocks, empty if is a function declaration.
  koopa_raw_slice_t bbs;
} koopa_raw_function_data_t;

typedef const koopa_raw_function_data_t *koopa_raw_function_t;
```

**成员**

- `ty`: 函数类型。
- `name`: 函数名称（包含 `@` 前缀）。
- `params`: 函数参数列表。元素为 `koopa_raw_value_t`。
- `bbs`: 基本块列表。元素为 `koopa_raw_basic_block_t`。如果为空，表示这是函数声明。

**说明**

```koopa
// name: "@add"
// params:包含 "%a", "%b"
// bbs: 包含 "%entry"
fun @add(%a: i32, %b: i32): i32 {
%entry:
  %0 = add %a, %b
  ret %0
}
```

### koopa_raw_basic_block_t

**描述**

Koopa IR 中的基本块结构体。

**定义**

```c
typedef struct {
  /// Name of basic block, null if no name.
  const char *name;
  /// Parameters.
  koopa_raw_slice_t params;
  /// Values that this basic block is used by.
  koopa_raw_slice_t used_by;
  /// Instructions in this basic block.
  koopa_raw_slice_t insts;
} koopa_raw_basic_block_data_t;

typedef const koopa_raw_basic_block_data_t *koopa_raw_basic_block_t;
```

**成员**

- `name`: 基本块名称（包含 `%` 前缀）。如果是匿名基本块可能为 null。
- `params`: 基本块参数。
- `used_by`: 使用了该基本块的 User 列表。
- `insts`: 指令列表。元素为 `koopa_raw_value_t`。

**说明**

```koopa
// name: "%entry"
// insts: 包含 "%0 = add ...", "ret %0"
%entry:
  %0 = add %a, %b
  ret %0
```

### koopa_raw_value_t

**描述**

Koopa IR 中的值（指令/常量/参数）结构体。

**定义**

```c
struct koopa_raw_value_data {
  /// Type of value.
  koopa_raw_type_t ty;
  /// Name of value, null if no name.
  const char *name;
  /// Values that this value is used by.
  koopa_raw_slice_t used_by;
  /// Kind of value.
  koopa_raw_value_kind_t kind;
};

typedef struct koopa_raw_value_data koopa_raw_value_data_t;

typedef const koopa_raw_value_data_t *koopa_raw_value_t;
```

**成员**

- `ty`: 值的类型。
- `name`: 值的名称（如指令结果 `%0`，参数 `%a`）。临时值或常量可能为 null。
- `used_by`: 使用了该值的 User 列表。
- `kind`: values的具体种类（包含 tag 和 data）。

**说明**

```koopa
// name: "%0"
// ty: i32
// kind.tag: KOOPA_RVT_BINARY
// kind.data.binary: { op: ADD, lhs: %a, rhs: %b }
%0 = add %a, %b
```

### koopa_raw_type_t

**描述**

Koopa IR 中的类型结构体。

**定义**

```c
typedef struct koopa_raw_type_kind {
  koopa_raw_type_tag_t tag;
  union {
    struct {
      const struct koopa_raw_type_kind *base;
      size_t len;
    } array;
    struct {
      const struct koopa_raw_type_kind *base;
    } pointer;
    struct {
      koopa_raw_slice_t params;
      const struct koopa_raw_type_kind *ret;
    } function;
  } data;
} koopa_raw_type_kind_t;

typedef const koopa_raw_type_kind_t *koopa_raw_type_t;

```

**成员**

- `tag`: 类型标签。
- `data`: 类型数据联合体。根据 `tag` 访问不同成员。

**说明**

- `i32`: `tag` = `KOOPA_RTT_INT32`
- `[i32, 4]`: `tag` = `KOOPA_RTT_ARRAY`, `data.array` = `{ base: i32, len: 4 }`
- `*i32`: `tag` = `KOOPA_RTT_POINTER`, `data.pointer` = `{ base: i32 }`

### koopa_raw_value_kind_t

**描述**

值的具体种类，包含标签和联合体数据。

**定义**

```c
typedef struct {
  koopa_raw_value_tag_t tag;
  union {
    koopa_raw_integer_t integer;
    koopa_raw_aggregate_t aggregate;
    koopa_raw_func_arg_ref_t func_arg_ref;
    koopa_raw_block_arg_ref_t block_arg_ref;
    koopa_raw_global_alloc_t global_alloc;
    koopa_raw_load_t load;
    koopa_raw_store_t store;
    koopa_raw_get_ptr_t get_ptr;
    koopa_raw_get_elem_ptr_t get_elem_ptr;
    koopa_raw_binary_t binary;
    koopa_raw_branch_t branch;
    koopa_raw_jump_t jump;
    koopa_raw_call_t call;
    koopa_raw_return_t ret;
  } data;
} koopa_raw_value_kind_t;
```

**成员**

- `tag`: 值的标签，决定了 `data` 中哪个成员有效。
- `data`: 具体的数据结构。

### koopa_raw_integer_t

**描述**

整数常量。

**定义**
```c
typedef struct {
  /// Value of integer.
  int32_t value;
} koopa_raw_integer_t;
```

**成员**

- `value`: 整数值。

**说明**

```koopa
123
// tag: KOOPA_RVT_INTEGER
// value: 123
```

### koopa_raw_aggregate_t

**描述**

聚合常量（如数组初始化列表）。

**定义**
```c
typedef struct {
  /// Elements.
  koopa_raw_slice_t elems;
} koopa_raw_aggregate_t;
```

**成员**

- `elems`: 元素列表，元素类型为 `koopa_raw_value_t`。

**说明**

```koopa
{1, 2, 3}
// tag: KOOPA_RVT_AGGREGATE
// elems: [1, 2, 3] (皆为 koopa_raw_value_t)
```

### koopa_raw_func_arg_ref_t

**描述**

函数参数引用。

**定义**
```c
typedef struct {
  /// Index.
  size_t index;
} koopa_raw_func_arg_ref_t;
```

**成员**

- `index`: 参数在函数参数列表中的索引。

**说明**

```koopa
fun @foo(@x: i32) {
  %0 = add @x, 1  // @x 对应 func_arg_ref, index=0
}
```

### koopa_raw_block_arg_ref_t

**定义**
```c
typedef struct {
  /// Index.
  size_t index;
} koopa_raw_block_arg_ref_t;
```

### koopa_raw_global_alloc_t

**描述**

全局变量内存分配。

**定义**
```c
typedef struct {
  /// Initializer.
  koopa_raw_value_t init;
} koopa_raw_global_alloc_t;
```

**成员**

- `init`: 初始化值。

**说明**

```koopa
global @x = alloc i32, 0 
// init: 0 (INTEGER value)
```

### koopa_raw_load_t

**描述**

内存加载指令。

**定义**
```c
typedef struct {
  /// Source.
  koopa_raw_value_t src;
} koopa_raw_load_t;
```

**成员**

- `src`: 源地址（指针类型的值）。

**说明**

```koopa
%0 = load @x
// src: @x
```

### koopa_raw_store_t

**描述**

内存存储指令。

**定义**
```c
typedef struct {
  /// Value.
  koopa_raw_value_t value;
  /// Destination.
  koopa_raw_value_t dest;
} koopa_raw_store_t;
```

**成员**

- `value`: 要存储的值。
- `dest`: 目标地址。

**说明**

```koopa
store 1, @x
// value: 1
// dest: @x
```

### koopa_raw_get_ptr_t

**描述**

指针运算指令（`getptr`）。

**定义**
```c
typedef struct {
  /// Source.
  koopa_raw_value_t src;
  /// Index.
  koopa_raw_value_t index;
} koopa_raw_get_ptr_t;
```

**成员**

- `src`: 源指针。
- `index`: 偏移量（索引）。

**说明**

```koopa
%0 = getptr @arr, 1
// src: @arr
// index: 1
```

### koopa_raw_get_elem_ptr_t

**描述**

元素指针运算指令（`getelemptr`）。

**定义**
```c
typedef struct {
  /// Source.
  koopa_raw_value_t src;
  /// Index.
  koopa_raw_value_t index;
} koopa_raw_get_elem_ptr_t;
```

**成员**

- `src`: 源指针（通常指向数组或聚合体）。
- `index`: 元素各索引。

**说明**

```koopa
%0 = getelemptr @arr, 1
// src: @arr
// index: 1
```

### koopa_raw_binary_t

**描述**

二元运算指令。

**定义**
```c
typedef struct {
  /// Operator.
  koopa_raw_binary_op_t op;
  /// Left-hand side value.
  koopa_raw_value_t lhs;
  /// Right-hand side value.
  koopa_raw_value_t rhs;
} koopa_raw_binary_t;
```

**成员**

- `op`: 运算符。
- `lhs`: 左操作数。
- `rhs`: 右操作数。

**说明**

```koopa
%0 = add %a, %b
// op: KOOPA_RBO_ADD
// lhs: %a
// rhs: %b
```

### koopa_raw_branch_t

**描述**

条件分支指令。

**定义**
```c
typedef struct {
  /// Condition.
  koopa_raw_value_t cond;
  /// Target if condition is `true`.
  koopa_raw_basic_block_t true_bb;
  /// Target if condition is `false`.
  koopa_raw_basic_block_t false_bb;
  /// Arguments of `true` target..
  koopa_raw_slice_t true_args;
  /// Arguments of `false` target..
  koopa_raw_slice_t false_args;
} koopa_raw_branch_t;
```

**成员**

- `cond`: 条件值。
- `true_bb`: 条件为真时跳转的基本块。
- `false_bb`: 条件为假时跳转的基本块。
- `true_args`: 跳转到 true_bb 时传递的参数列表；
- `false_args`: 跳转到 false_bb 时传递的参数列表。

**说明**

```koopa
br %cond, %then, %else
// cond: %cond
// true_bb: %then
// false_bb: %else
```

### koopa_raw_jump_t

**描述**

无条件跳转指令。

**定义**
```c
typedef struct {
  /// Target.
  koopa_raw_basic_block_t target;
  /// Arguments of target..
  koopa_raw_slice_t args;
} koopa_raw_jump_t;
```

**成员**

- `target`: 目标基本块。
- `args`: 传递参数列表。

**说明**

```koopa
jump %target
// target: %target
```

### koopa_raw_call_t

**描述**

函数调用指令。

**定义**
```c
typedef struct {
  /// Callee.
  koopa_raw_function_t callee;
  /// Arguments.
  koopa_raw_slice_t args;
} koopa_raw_call_t;
```

**成员**

- `callee`: 被调用的函数。
- `args`: 实参列表。

**说明**

```koopa
call @func(1, 2)
// callee: @func
// args: [1, 2]
```

### koopa_raw_return_t

**描述**

函数返回指令。

**定义**
```c
typedef struct {
  /// Return value, null if no return value.
  koopa_raw_value_t value;
} koopa_raw_return_t;
```

**成员**

- `value`: 返回值。

**说明**

```koopa
ret 0
// value: 0
```

## 枚举

### koopa_error_code_t

**描述**

Koopa IR 库函数的错误码枚举。

**定义**

```c
enum koopa_error_code {
  /// No errors occurred.
  KOOPA_EC_SUCCESS = 0,
  /// UTF-8 string conversion error.
  KOOPA_EC_INVALID_UTF8_STRING,
  /// File operation error.
  KOOPA_EC_INVALID_FILE,
  /// Koopa IR program parsing error.
  KOOPA_EC_INVALID_KOOPA_PROGRAM,
  /// IO operation error.
  KOOPA_EC_IO_ERROR,
  /// Byte array to C string conversion error.
  KOOPA_EC_NULL_BYTE_ERROR,
  /// Insufficient buffer length.
  KOOPA_EC_INSUFFICIENT_BUFFER_LENGTH,
  /// Mismatch of item kind in raw slice.
  KOOPA_EC_RAW_SLICE_ITEM_KIND_MISMATCH,
  /// Passing null pointers to `libkoopa`.
  KOOPA_EC_NULL_POINTER_ERROR,
  /// Mismatch of type.
  KOOPA_EC_TYPE_MISMATCH,
  /// Mismatch of function parameter number.
  KOOPA_EC_FUNC_PARAM_NUM_MISMATCH,
};

typedef enum koopa_error_code koopa_error_code_t;
```

**成员**

- `KOOPA_EC_SUCCESS`: 操作成功，无错误。
- `KOOPA_EC_INVALID_UTF8_STRING`: 无效的 UTF-8 字符串。
- `KOOPA_EC_INVALID_FILE`: 文件操作失败（如文件不存在或无法打开）。
- `KOOPA_EC_INVALID_KOOPA_PROGRAM`: Koopa IR 程序解析失败（如语法错误）。
- `KOOPA_EC_IO_ERROR`: I/O 操作错误（如读写失败）。
- `KOOPA_EC_NULL_BYTE_ERROR`: 字节数组转换 C 字符串时出错（通常涉及 null 终止符问题）。
- `KOOPA_EC_INSUFFICIENT_BUFFER_LENGTH`: 提供的缓冲区长度不足。
- `KOOPA_EC_RAW_SLICE_ITEM_KIND_MISMATCH`: Raw Slice 中的元素类型不匹配。
- `KOOPA_EC_NULL_POINTER_ERROR`: 传递了空指针给库函数。
- `KOOPA_EC_TYPE_MISMATCH`: 类型不匹配错误。
- `KOOPA_EC_FUNC_PARAM_NUM_MISMATCH`: 函数参数数量不匹配。

### koopa_raw_slice_item_kind_t

**描述**

Raw Slice 中元素的类型枚举。用于运行时判断 `koopa_raw_slice_t` 中存储的具体数据类型。

**定义**

```c
enum koopa_raw_slice_item_kind {
  /// Unknown.
  KOOPA_RSIK_UNKNOWN = 0,
  /// Type.
  KOOPA_RSIK_TYPE,
  /// Function.
  KOOPA_RSIK_FUNCTION,
  /// Basic block.
  KOOPA_RSIK_BASIC_BLOCK,
  /// Value.
  KOOPA_RSIK_VALUE,
};

typedef enum koopa_raw_slice_item_kind koopa_raw_slice_item_kind_t;
```

**成员**

- `KOOPA_RSIK_UNKNOWN`: 未知类型。
- `KOOPA_RSIK_TYPE`: 元素是类型 (`koopa_raw_type_t`)。
- `KOOPA_RSIK_FUNCTION`: 元素是函数 (`koopa_raw_function_t`)。
- `KOOPA_RSIK_BASIC_BLOCK`: 元素是基本块 (`koopa_raw_basic_block_t`)。
- `KOOPA_RSIK_VALUE`: 元素是值/指令 (`koopa_raw_value_t`)。

### koopa_raw_type_tag_t

**描述**

Koopa IR 中类型的标签枚举。用于区分 `koopa_raw_type_t` 具体是哪种类型（如整数、数组、指针等）。

**定义**

```c
typedef enum {
  /// 32-bit integer.
  KOOPA_RTT_INT32,
  /// Unit (void).
  KOOPA_RTT_UNIT,
  /// Array (with base type and length).
  KOOPA_RTT_ARRAY,
  /// Pointer (with base type).
  KOOPA_RTT_POINTER,
  /// Function (with parameter types and return type).
  KOOPA_RTT_FUNCTION,
} koopa_raw_type_tag_t;
```

**成员**

- `KOOPA_RTT_INT32`: 32 位整数类型 (`i32`)。
- `KOOPA_RTT_UNIT`: 单元类型 (`void` / unit)。
- `KOOPA_RTT_ARRAY`: 数组类型。
- `KOOPA_RTT_POINTER`: 指针类型。
- `KOOPA_RTT_FUNCTION`: 函数原型类型。

### koopa_raw_value_tag_t

**描述**

Koopa IR 中值（指令）的标签枚举。用于区分 `koopa_raw_value_t` 具体是哪种指令或值（如加载、存储、二元运算等）。

**定义**

```c
typedef enum {
  /// Integer constant.
  KOOPA_RVT_INTEGER,
  /// Zero initializer.
  KOOPA_RVT_ZERO_INIT,
  /// Undefined value.
  KOOPA_RVT_UNDEF,
  /// Aggregate constant.
  KOOPA_RVT_AGGREGATE,
  /// Function argument reference.
  KOOPA_RVT_FUNC_ARG_REF,
  /// Basic block argument reference.
  KOOPA_RVT_BLOCK_ARG_REF,
  /// Local memory allocation.
  KOOPA_RVT_ALLOC,
  /// Global memory allocation.
  KOOPA_RVT_GLOBAL_ALLOC,
  /// Memory load.
  KOOPA_RVT_LOAD,
  /// Memory store.
  KOOPA_RVT_STORE,
  /// Pointer calculation.
  KOOPA_RVT_GET_PTR,
  /// Element pointer calculation.
  KOOPA_RVT_GET_ELEM_PTR,
  /// Binary operation.
  KOOPA_RVT_BINARY,
  /// Conditional branch.
  KOOPA_RVT_BRANCH,
  /// Unconditional jump.
  KOOPA_RVT_JUMP,
  /// Function call.
  KOOPA_RVT_CALL,
  /// Function return.
  KOOPA_RVT_RETURN,
} koopa_raw_value_tag_t;
```

**成员**

- `KOOPA_RVT_INTEGER`: 整数常量。
- `KOOPA_RVT_ZERO_INIT`: 零初始化常量 (`zeroinit`)。
- `KOOPA_RVT_UNDEF`: 未定义值 (`undef`)。
- `KOOPA_RVT_AGGREGATE`: 聚合常量（如数组初始化列表）。
- `KOOPA_RVT_FUNC_ARG_REF`: 函数参数引用。
- `KOOPA_RVT_BLOCK_ARG_REF`: 基本块参数引用（Koopa IR 目前主要用作 phi 节点的替代，实际上通常不会直接遇到）。
- `KOOPA_RVT_ALLOC`: 局部内存分配 (`alloc`)。
- `KOOPA_RVT_GLOBAL_ALLOC`: 全局内存分配 (`global alloc`)。
- `KOOPA_RVT_LOAD`: 内存读取 (`load`)。
- `KOOPA_RVT_STORE`: 内存写入 (`store`)。
- `KOOPA_RVT_GET_PTR`: 指针计算 (`getptr`)。
- `KOOPA_RVT_GET_ELEM_PTR`: 元素指针计算 (`getelemptr`)。
- `KOOPA_RVT_BINARY`: 二元运算。
- `KOOPA_RVT_BRANCH`: 条件分支 (`br`)。
- `KOOPA_RVT_JUMP`: 无条件跳转 (`jump`)。
- `KOOPA_RVT_CALL`: 函数调用 (`call`)。
- `KOOPA_RVT_RETURN`: 函数返回 (`ret`)。

### koopa_raw_binary_op_t

**描述**

二元运算的操作符枚举。

**定义**

```c
enum koopa_raw_binary_op {
  /// Not equal to.
  KOOPA_RBO_NOT_EQ,
  /// Equal to.
  KOOPA_RBO_EQ,
  /// Greater than.
  KOOPA_RBO_GT,
  /// Less than.
  KOOPA_RBO_LT,
  /// Greater than or equal to.
  KOOPA_RBO_GE,
  /// Less than or equal to.
  KOOPA_RBO_LE,
  /// Addition.
  KOOPA_RBO_ADD,
  /// Subtraction.
  KOOPA_RBO_SUB,
  /// Multiplication.
  KOOPA_RBO_MUL,
  /// Division.
  KOOPA_RBO_DIV,
  /// Modulo.
  KOOPA_RBO_MOD,
  /// Bitwise AND.
  KOOPA_RBO_AND,
  /// Bitwise OR.
  KOOPA_RBO_OR,
  /// Bitwise XOR.
  KOOPA_RBO_XOR,
  /// Shift left logical.
  KOOPA_RBO_SHL,
  /// Shift right logical.
  KOOPA_RBO_SHR,
  /// Shift right arithmetic.
  KOOPA_RBO_SAR,
};

typedef enum koopa_raw_binary_op koopa_raw_binary_op_t;
```

**成员**

- `KOOPA_RBO_NOT_EQ`: 不等于 (`ne`).
- `KOOPA_RBO_EQ`: 等于 (`eq`).
- `KOOPA_RBO_GT`: 大于 (`gt`).
- `KOOPA_RBO_LT`: 小于 (`lt`).
- `KOOPA_RBO_GE`: 大于等于 (`ge`).
- `KOOPA_RBO_LE`: 小于等于 (`le`).
- `KOOPA_RBO_ADD`: 加法 (`add`).
- `KOOPA_RBO_SUB`: 减法 (`sub`).
- `KOOPA_RBO_MUL`: 乘法 (`mul`).
- `KOOPA_RBO_DIV`: 除法 (`div`).
- `KOOPA_RBO_MOD`: 取模 (`mod`).
- `KOOPA_RBO_AND`: 按位与 (`and`).
- `KOOPA_RBO_OR`: 按位或 (`or`).
- `KOOPA_RBO_XOR`: 按位异或 (`xor`).
- `KOOPA_RBO_SHL`: 左移 (`shl`).
- `KOOPA_RBO_SHR`: 逻辑右移 (`shr`).
- `KOOPA_RBO_SAR`: 算术右移 (`sar`).

## 函数

### koopa_parse_from_file

**描述**

从给定文件路径读取并解析文本形式的 Koopa IR 程序。如果未发生错误，将更新 `program` 指针。

**定义**

```c
koopa_error_code_t koopa_parse_from_file(const char *path,
                                         koopa_program_t *program);
```

**参数**

- `path`: 输入文件的路径字符串。
- `program`: 指向 `koopa_program_t` 的指针。解析成功后，该指针指向的内存将被更新为新创建的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
koopa_error_code_t ret = koopa_parse_from_file("test.koopa", &program);
assert(ret == KOOPA_EC_SUCCESS);
// 使用 program ...
koopa_delete_program(program);
```

### koopa_parse_from_string

**描述**

从给定的字符串中解析文本形式的 Koopa IR 程序。如果未发生错误，将更新 `program` 指向的内容。

**定义**

```c
koopa_error_code_t koopa_parse_from_string(const char *str,
                                           koopa_program_t *program);
```

**参数**

- `str`: 包含 Koopa IR 程序代码的字符串。
- `program`: 指向 `koopa_program_t` 的指针。解析成功后，该指针指向的内存将被更新为新创建的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
const char *str = "fun @main(): i32 { %entry: ret 0 }";
koopa_program_t program;
koopa_error_code_t ret = koopa_parse_from_string(str, &program);
assert(ret == KOOPA_EC_SUCCESS);
koopa_delete_program(program);
```

### koopa_parse_from_stdin

**描述**

从标准输入 (stdin) 读取并解析文本形式的 Koopa IR 程序。如果未发生错误，将更新 `program` 指向的内容。

**定义**

```c
koopa_error_code_t koopa_parse_from_stdin(koopa_program_t *program);
```

**参数**

- `program`: 指向 `koopa_program_t` 的指针。解析成功后，该指针指向的内存将被更新为新创建的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
koopa_error_code_t ret = koopa_parse_from_stdin(&program);
// 此时程序会等待用户从标准输入输入 Koopa IR 代码
// 输入完毕后（需要发送 EOF，如 Linux 下 Ctrl+D），解析开始
assert(ret == KOOPA_EC_SUCCESS);
koopa_delete_program(program);
```

### koopa_parse_from_raw

**描述**

从给定的文件描述符 (UNIX/Linux) 或文件句柄 (Windows) 读取并解析文本形式的 Koopa IR 程序。如果未发生错误，将更新 `program` 指向的内容。

**定义**

```c
koopa_error_code_t koopa_parse_from_raw(koopa_raw_file_t file,
                                        koopa_program_t *program);
```

**参数**

- `file`: 已经打开的文件描述符 (`FILE*` 或 `int` 等，取决于具体平台定义 `koopa_raw_file_t`)。
- `program`: 指向 `koopa_program_t` 的指针。解析成功后，该指针指向的内存将被更新为新创建的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_raw_file_t fd = open("hello.koopa", O_RDONLY);
if (fd != -1) {
  koopa_program_t program;
  koopa_error_code_t ret = koopa_parse_from_raw(fd, &program);
  assert(ret == KOOPA_EC_SUCCESS && "parsing koopa ir failure");

  close(fd);
  koopa_delete_program(program);
} else {
  assert(false && "Failed to open test file!");
}
```

### koopa_delete_program

**描述**

删除给定的 Koopa IR 程序对象，释放其占用的所有内存。
所有由 `koopa_parse_*` 系列函数返回的程序对象都必须在使用完毕后手动调用此函数删除，以避免内存泄漏。

**定义**

```c
void koopa_delete_program(koopa_program_t program);
```

**参数**

- `program`: 要删除的 Koopa IR 程序对象。

**示例**

```c
koopa_program_t program;
koopa_parse_from_string("...", &program);
// ... 处理 program ...
koopa_delete_program(program);
```

?> 为了避免忘记释放内存或者你觉得手动释放内存很麻烦，可以把 koopa 构造程序对象的过程封装进一个类使用 RAII。

### koopa_dump_to_file

**描述**

将 Koopa IR 程序以文本形式输出到指定文件。

路径可以是绝对路径，也可以是相对于当前工作目录的相对路径。

**定义**

```c
koopa_error_code_t koopa_dump_to_file(koopa_program_t program,
                                      const char *path);
```

**参数**

- `program`: 要输出的 Koopa IR 程序对象。
- `path`: 输出文件的路径。如果文件已存在，将被覆盖。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
koopa_parse_from_string("...", &program);
koopa_dump_to_file(program, "output.koopa");
koopa_delete_program(program);
```

### koopa_dump_to_string

**描述**

将 Koopa IR 程序以文本形式输出到给定的字符缓冲区，并在末尾添加 null 终止符。

如果 `buffer` 参数为 null，函数将计算生成字符串所需的长度（**不包含** null 终止符），并将结果存储在 `len` 指向的变量中。这通常用于确定缓冲区所需的最小大小。

`len` 同时代表了最多可以往 buffer 里写多少个字节（input）和实际上写了多少个字节（output）。

**定义**

```c
koopa_error_code_t koopa_dump_to_string(koopa_program_t program, char *buffer,
                                        size_t *len);
```

**参数**

- `program`: 要输出的 Koopa IR 程序对象。
- `buffer`: 指向字符缓冲区的指针，用于存储生成的字符串。可以为 null。
- `len`: 指向 `size_t` 变量的指针。
  - 如果 `buffer` 为 null，函数将写入所需的字符串长度。
  - 如果 `buffer` 不为 null，函数将写入实际写入缓冲区的字符数。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...

size_t len;
auto ret = koopa_dump_to_string(program, nullptr, &len);
assert(ret == KOOPA_EC_SUCCESS && "dumping koopa ir failure");

fmt::print("Koopa IR length: {}\n", len);

len = len + 1;
std::string buffer(len, '\0');
ret = koopa_dump_to_string(program, buffer.data(), &len);
/*
len 同时代表了最多可以往 buffer 里写多少个字节（input）和实际上写了多少个字节（output）。
所以如果这里这么写：
去除掉 len = len + 1
std::string buffer(len + 1, '\0');
ret = koopa_dump_to_string(program, buffer.data(), &len);
ret 会是 KOOPA_EC_INSUFFICIENT_BUFFER_LENGTH
*/
assert(ret == KOOPA_EC_SUCCESS && "dumping koopa ir failure");
fmt::print("{}", buffer);

koopa_delete_program(program);
```

### koopa_dump_to_stdout

**描述**

将 Koopa IR 程序以文本形式输出到标准输出 (stdout)。

**定义**

```c
koopa_error_code_t koopa_dump_to_stdout(koopa_program_t program);
```

**参数**

- `program`: 要输出的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...
koopa_dump_to_stdout(program);
koopa_delete_program(program);
```

### koopa_dump_to_raw

**描述**

将 Koopa IR 程序以文本形式输出到给定的文件描述符 (UNIX/Linux) 或文件句柄 (Windows)。

**定义**

```c
koopa_error_code_t koopa_dump_to_raw(koopa_program_t program,
                                     koopa_raw_file_t file);
```

**参数**

- `program`: 要输出的 Koopa IR 程序对象。
- `file`: 已经打开并具备写权限的文件描述符。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
koopa_parse_from_file("hello.koopa", &program);

koopa_raw_file_t fd = open("output.koopa", O_WRONLY | O_CREAT | O_TRUNC, 0644);
assert(fd >= 0 && "Failed to open output file");

auto ret = koopa_dump_to_raw(program, fd);
assert(ret == KOOPA_EC_SUCCESS && "Failed to dump program to file");

close(fd);
koopa_delete_program(program);
```

### koopa_dump_llvm_to_file

**描述**

将 Koopa IR 程序转换为 LLVM IR，并将结果以文本形式输出到指定文件。

**定义**

```c
koopa_error_code_t koopa_dump_llvm_to_file(koopa_program_t program,
                                           const char *path);
```

**参数**

- `program`: 要转换的 Koopa IR 程序对象。
- `path`: 输出文件的路径。如果文件已存在，将被覆盖。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...
koopa_dump_llvm_to_file(program, "output.ll");
koopa_delete_program(program);
```

### koopa_dump_llvm_to_string

**描述**

将 Koopa IR 程序转换为 LLVM IR，并将结果以文本形式输出到给定的字符缓冲区，并在末尾添加 null 终止符。

如果 `buffer` 参数为 null，则计算所需的缓冲区长度（**不包含** null 终止符），并存储在 `len` 中。

`len` 同时代表了最多可以往 buffer 里写多少个字节（input）和实际上写了多少个字节（output）。

**定义**

```c
koopa_error_code_t koopa_dump_llvm_to_string(koopa_program_t program,
                                             char *buffer, size_t *len);
```

**参数**

- `program`: 要转换的 Koopa IR 程序对象。
- `buffer`: 指向字符缓冲区的指针，用于存储生成的 LLVM IR 字符串。可以为 null。
- `len`: 指向 `size_t` 变量的指针，用于存储长度信息。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...

size_t len;
auto ret = koopa_dump_llvm_to_string(program, nullptr, &len);
assert(ret == KOOPA_EC_SUCCESS && "dumping koopa ir failure");

fmt::print("Koopa IR length: {}\n", len);

len = len + 1;
std::string buffer(len, '\0');
ret = koopa_dump_llvm_to_string(program, buffer.data(), &len);
/*
len 同时代表了最多可以往 buffer
里写多少个字节（input）和实际上写了多少个字节（output）。 所以如果这里这么写：
去除掉 len = len + 1
std::string buffer(len + 1, '\0');
ret = koopa_dump_llvm_to_string(program, buffer.data(), &len);
ret 会是 KOOPA_EC_INSUFFICIENT_BUFFER_LENGTH
*/
assert(ret == KOOPA_EC_SUCCESS && "dumping koopa ir failure");
fmt::print("{}", buffer);

koopa_delete_program(program);
```

### koopa_dump_llvm_to_stdout

**描述**

将 Koopa IR 程序转换为 LLVM IR，并将结果以文本形式输出到标准输出 (stdout)。

**定义**

```c
koopa_error_code_t koopa_dump_llvm_to_stdout(koopa_program_t program);
```

**参数**

- `program`: 要转换的 Koopa IR 程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...
koopa_dump_llvm_to_stdout(program);
koopa_delete_program(program);
```

### koopa_dump_llvm_to_raw

**描述**

将 Koopa IR 程序转换为 LLVM IR，并将结果以文本形式输出到给定的文件描述符 (UNIX/Linux) 或文件句柄 (Windows)。

**定义**

```c
koopa_error_code_t koopa_dump_llvm_to_raw(koopa_program_t program,
                                          koopa_raw_file_t file);
```

**参数**

- `program`: 要转换的 Koopa IR 程序对象。
- `file`: 已经打开并具备写权限的文件描述符。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program;
koopa_parse_from_file("hello.koopa", &program);

koopa_raw_file_t fd =
    open("output.ll", O_WRONLY | O_CREAT | O_TRUNC, 0644);
assert(fd >= 0 && "Failed to open output file");

auto ret = koopa_dump_llvm_to_raw(program, fd);
assert(ret == KOOPA_EC_SUCCESS && "Failed to dump program to file");

close(fd);
koopa_delete_program(program);
```

### koopa_new_raw_program_builder

**描述**

创建一个新的原始程序构建器 (`koopa_raw_program_builder_t`)。
该构建器用于将高级的 `koopa_program_t` 对象转换为更易于遍历和分析的 `koopa_raw_program_t` 结构。

**定义**

```c
koopa_raw_program_builder_t koopa_new_raw_program_builder();
```

**返回值**

- 返回新创建的构建器对象的指针。

**示例**

```c
koopa_raw_program_builder_t builder = koopa_new_raw_program_builder();
// ... 使用 builder ...
koopa_delete_raw_program_builder(builder);
```

### koopa_delete_raw_program_builder

**描述**

删除给定的原始程序构建器，并释放其占用的内存。必须在构建器不再使用后手动调用，以避免内存泄漏。

**定义**

```c
void koopa_delete_raw_program_builder(koopa_raw_program_builder_t builder);
```

**参数**

- `builder`: 要删除的构建器对象。

**示例**

```c
// 见 koopa_new_raw_program_builder 示例
```

?> 为了避免忘记释放内存或者你觉得手动释放内存很麻烦，可以把 koopa 构造程序对象的过程封装进一个类使用 RAII。

### koopa_build_raw_program

**描述**

使用给定的构建器，将 `koopa_program_t` 对象转换为 `koopa_raw_program_t` 结构。

`koopa_raw_program_t` 是 Koopa IR 的内存表示形式，包含了程序的完整结构信息（函数、基本块、指令等），便于遍历和分析。
生成的原始程序 (`koopa_raw_program_t`) 的生命周期依附于构建器 (`builder`)。只要构建器未被删除，原始程序就有效。

**定义**

```c
koopa_raw_program_t koopa_build_raw_program(koopa_raw_program_builder_t builder,
                                            koopa_program_t program);
```

**参数**

- `builder`: 已经创建好的原始程序构建器。
- `program`: 要转换的 Koopa IR 程序对象。

**返回值**

- 返回构建好的 `koopa_raw_program_t` 结构。

**示例**

```c
koopa_program_t program;
// ... 初始化 program ...

koopa_raw_program_builder_t builder = koopa_new_raw_program_builder();
koopa_raw_program_t raw = koopa_build_raw_program(builder, program);

// ... 遍历 raw program，进行分析或优化 ...

koopa_delete_raw_program_builder(builder);
koopa_delete_program(program);
```

### koopa_generate_raw_to_koopa

**描述**

将给定的原始程序 (`koopa_raw_program_t`) 逆向转换回 Koopa IR 程序对象 (`koopa_program_t`)。

`libkoopa` 的文本输出函数（如 `koopa_dump_to_string`）仅接受 `koopa_program_t`。如果你手动构建或修改了 `raw` 结构，并希望将其导出为 Koopa IR 文本形式，则可以使用此函数将其转换为 `program` 对象。

**定义**

```c
koopa_error_code_t koopa_generate_raw_to_koopa(const koopa_raw_program_t *raw,
                                               koopa_program_t *program);
```

**参数**

- `raw`: 指向原始程序结构的指针。
- `program`: 指向 `koopa_program_t` 的指针，用于存储生成的新程序对象。

**返回值**

- 返回 `KOOPA_EC_SUCCESS` 表示成功，否则返回相应的错误码。

**示例**

```c
koopa_program_t program1, program2;
// ... 初始化 program1 ...

koopa_raw_program_builder_t builder = koopa_new_raw_program_builder();
koopa_raw_program_t raw = koopa_build_raw_program(builder, program1);
// ... 修改 raw ...

// 将 raw program 转换回 program2
koopa_generate_raw_to_koopa(&raw, &program2);

// ... 使用 program2 ...

koopa_delete_program(program2);
koopa_delete_raw_program_builder(builder);
koopa_delete_program(program1);
```
