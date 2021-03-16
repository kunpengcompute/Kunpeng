# Kunpeng Compute安全编程指南

本文档基于C语言提供一些安全编程建议，用于指导开发实践。



# 数据类型

## 确保有符号整数运算不溢出

**【描述】**
有符号整数溢出是未定义的行为。出于安全考虑，对外部数据中的有符号整数值在如下场景中使用时，需要确保运算不会导致溢出：

-   指针运算的整数操作数(指针偏移值)

-   数组索引

-   变长数组的长度(及长度运算表达式)

-   内存拷贝的长度

-   内存分配函数的参数

-   循环判断条件

在精度低于int的整数类型上进行运算时，需要考虑整数提升。程序员还需要掌握整数转换规则，包括隐式转换规则，以便设计安全的算术运算。

1)加法

**【错误代码示例】**(加法)

如下代码示例中，参与加法运算的整数是外部数据，在使用前未做校验，可能出现整数溢出。

```
int num_a = ... // 来自外部数据
int num_b = ... // 来自外部数据
int sum = num_a + num_b;
...
```

**【正确代码示例】**(加法)

```
int num_a = ... // 来自外部数据
int num_b = ... // 来自外部数据
int sum = 0;
if (((num_a > 0) && (num_b > (INT_MAX - num_a))) ||
	((num_a < 0) && (num_b < (INT_MIN - num_a)))) {
 	... // 错误处理
}
sum = num_a + num_b;
...
```

2)减法

**【错误代码示例】**(减法)

如下代码示例中，参与减法运算的整数是外部数据，在使用前未做校验，可能出现整数溢出，进而造成后续的内存复制操作出现缓冲区溢出。

```
unsigned char  *content = ... // 指向报文头的指针
size_t content_size = ... // 缓冲区的总长度
int total_len = ... // 报文总长度
int skip_len = ... // 从消息中解析出来的需要忽略的数据长度
// 用total_len - skip_len 计算剩余数据长度，可能出现整数溢出
(void)memmove(content, content + skip_len, total_len - skip_len);
...
```

**【正确代码示例】**(减法)

如下代码示例中，重构为使用size_t类型的变量表示数据长度，并校验外部数据长度是否在合法范围内。

```
unsigned char *content = ... //指向报文头的指针
size_t content_size = ... // 缓冲区的总长度
size_t total_len = ... // 报文总长度
size_t skip_len = ... // 从消息中解析出来的需要忽略的数据长度
if (skip_len >= total_len || total_len > content_size) {
	... // 错误处理
}

(void)memmove(content, content + skip_len, total_len - skip_len);
...
```



3)乘法

**【错误代码示例】**(乘法)

如下代码示例中，内核代码对来自用户态的数值范围做了校验，但是由于opt是int类型，而校验条件中错误的使用了ULONG_MAX进行限制，导致整数溢出。

```
int opt = ... // 来自用户态
if ((opt < 0) || (opt > (ULONG_MAX / (60 * HZ)))) { // 错误的使用了ULONG_MAX做上限校验
	return -EINVAL;
}

... = opt * 60 * HZ; // 可能出现整数溢出
...
```

**【正确代码示例】**(乘法)

一种改进方案是将opt的类型修改为unsigned long类型，这种方案适用于修改了变量类型更符合业务逻辑的场景。

```
unsigned long opt = ... // 将类型重构为 unsigned long 类型。
if (opt > (ULONG_MAX / (60 * HZ))) {
	return -EINVAL;
}
... = opt * 60 * HZ;
...
```

另一种改进方案是将数值上限修改为INT_MAX。

```
int opt = ... // 来自用户态
if ((opt < 0) || (opt > (INT_MAX / (60 * HZ)))) { // 修改使用INT_MAX作为上限值
	return -EINVAL;
}
... = opt * 60 * HZ;
```

4)除法

**【错误代码示例】**(除法)

如下代码示例中，做除法运算前只检查了是否出现被零除的问题，缺少对数值范围的校验，可能出现整数溢出。

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

if (num_b == 0) {
	... // 对除数为0的错误处理
}

result = num_a / num_b; // 可能出现整数溢出
...
```

**【正确代码示例】**(除法)

如下代码示例中，按照最大允许值进行校验，防止整数溢出，在编程时可根据具体业务场景做更严格的值域校验。

```
int num_a = ... // 来自外部数据

int num_b = ... // 来自外部数据

int result = 0;

// 检查除数为0及除法溢出错误

if ((num_b == 0) || ((num_a == INT_MIN) && (num_b == -1))) {

 ... // 错误处理

}

result = num_a / num_b;
...
```

5)求余数

**【错误代码示例】**(求余数)

```
int num_a = ... // 来自外部数据
int num_b = ... // 来自外部数据
int result = 0;
if (num_b == 0) {
	... // 对除数为0的错误处理
}

result = num_a % num_b; // 可能出现整数溢出
...
}
```

**【正确代码示例】**(求余数)

如下代码示例中，按照最大允许值进行校验，防止整数溢出。在编程时可根据具体业务场景做更严格的值域校验。

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

// 检查除数为0及除法溢出错误
if ((num_b == 0)  || ((num_a == INT_MIN) && (num_b == -1))) {
	... // 错误处理
}

result = num_a % num_b;
...
}
```

6)一元减

当操作数等于有符号整数类型的最小值时，在二进制补码一元求反期间会发生溢出。

**【错误代码示例】**(一元减)

如下代码示例中，计算前未校验数值范围，可能出现整数溢出。

```
int num_a = ... // 来自外部数据
int result = -num_a; // 可能出现整数溢出
...
```

**【正确代码示例】**(一元减)

如下代码示例中，按照最大允许值进行校验，防止整数溢出。在编程时可根据具体业务场景做更严格的值域校验。

```
int num_a =  ... // 来自外部数据
int result = 0;

if (num_a == LNT_MIN) {
	... // 错误处理
}

result = -num_a;
...
```

## 确保无符号整数运算不回绕

**【描述】**

涉及无符号操作数的计算永远不会溢出，因为超出无符号整数类型表示范围的计算结果会按照（结果类型可表示的最大值 + 1）的数值取模。

这种行为更多时候被非正式地称为无符号整数回绕。

在精度低于int的整数类型上进行运算时，需要考虑整数提升。程序员还需要掌握整数转换规则，包括隐式转换规则，以便设计安全的算术运算。

出于安全考虑，对外部数据中的无符号整数值在如下场景中使用时，需要确保运算不会导致回绕：

-   指针运算的整数操作数(指针偏移值)

-   数组索引

-   变长数组的长度(及长度运算表达式)

-   内存拷贝的长度

-   内存分配函数的参数

-   循环判断条件

1)加法

**【错误代码示例】**(加法)

如下代码示例中，校验下一个子报文的长度加上已处理报文的长度是否超过了整体报文的最大长度，在校验条件中的加法运算可能会出现整数回绕，造成绕过该校验的问题。

```
size_t total_len =  ... // 报文的总长度
size_t read_len = 0 // 记录已经处理报文的长度
...
size_t pkt_len = parse_pkt_len(); // 从网络报文中解析出来的下一个子报文的长度
if (read_len + pkt_len > total_len) { // 可能出现整数回绕
	... // 错误处理
}

...
read_len += pkt_len;
...
```

**【正确代码示例】**(加法)

由于read_len变量记录的是已经处理报文的长度，必然会小于total_len，因此将代码中的加法运算修改为减法运算，导致条件绕过。

```
size_t total_len = ... // 报文的总长度
size_t read_len = 0; // 记录已经处理报文的长度
...

size_t pkt_len = parse_pkt_len(); // 来自网络报文
if (pkt_len > total_len - read_len) {
	... // 错误处理
}

...
read_len += pkt_len;
...
```

2)减法

**【错误代码示例】**(减法)

如下代码示例中，校验len合法范围的运算可能会出现整数回绕，导致条件绕过。

```
size_t len = ... // 来自用户态输入
if (SCTP_SIZE_MAX - len < sizeof(SctpAuthBytes)) { // 减法操作可能出现整数回绕
	... // 错误处理
}

... = kmalloc(sizeof(SctpAuthBytes) + len, gfp); // 可能出现整数回绕
...
```



**【正确代码示例】**(减法)

如下代码示例中，调整减法运算的位置（需要确保编译期间减法表达式的值不翻转），避免整数回绕问题。

```
size_t len = ... // 来自用户态输入
if (len > SCTP_SIZE_MAX - sizeof(SctpAuthBytes)) { // 确保编译期间减法表达式的值不翻转
	... // 错误处理
}

... = kmalloc(sizeof(SctpAuthBytes) + len, gfp);
...
```



3)乘法

**【错误代码示例】**（乘法）

如下代码示例中，使用外部数据计算申请内存长度时未校验，可能出现整数回绕。

```
size_t width =  ... // 来自外部数据
size_t hight =  ... // 来自外部数据
unsigned char  *buf = (unsigned char  *)malloc(width  * hight);
```

无符号整数回绕可能导致分配的内存不足。

**【正确代码示例】**（乘法）

如下代码是一种解决方案，校验参与乘法运算的整数数值范围，确保不会出现整数回绕。

```
size_t width =  ... // 来自外部数据
size_t hight =  ... // 来自外部数据
if (width == 0 || hight == 0) {
	... // 错误处理
}

if (width  > SIZE_MAX / hight) {
	... // 错误处理
}

unsigned char  *buf = (unsigned char  *)malloc(width  * hight);
```

**【例外】** 
为正确执行程序，必要时无符号整数可能表现出模态（回绕）。建议将变量声明明确注释为支持模数行为，并且对该整数的每个操作也应明确注释为支持模数行为。

**【相关软件CWE编号】** CWE-190

## 确保除法和余数运算不会导致除零错误(被零除)

**【描述】**

整数的除法和取余运算的第二个操作数值为0会导致程序产生未定义的行为，因此使用时要确保整数的除法和余数运算不会导致除零错误(被零除，下同)。

1)除法

**【错误代码示例】**(除法)

有符号整数类型的除法运算如果限制不当，会导致溢出。

如下示例对有符号整数进行的除法运算做了防止溢出限制，确保不会导致溢出，但不能防止有符号操作数num_a和num_b之间的除法过程中出现除零错误：

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

if ((num_a == INT_MIN) && (num_b == -1)) {
	... // 错误处理
}

result = num_a / num_b; // 可能出现除零错误
...
```

**【正确代码示例】**(除法)

如下代码示例中，添加num_b是否为0的校验，防止除零错误。

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

if ((num_b == 0)  | | ((num_a == INT_MIN) && (num_b == -1))) {
	... // 错误处理
}

result = num_a / num_b;
...
```

2)取余

**【错误代码示例】**(求余数)

如下代码，同除法的错误代码示例一样，可能出现除零错误，因为许多平台以相同的指令实现求余数和除法运算。

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

if ((num_a == INT_MIN) && (num_b == -1)) {
	... // 错误处理
}

result = num_a % num_b; // 可能出现除零错误
...
```

**【正确代码示例】**(求余数)

如下代码示例中，添加num_b是否为0的校验，防止除零错误。

```
int num_a =  ... // 来自外部数据
int num_b =  ... // 来自外部数据
int result = 0;

if ((num_b == 0)  | | ((num_a == INT_MIN) && (num_b == -1))) {
	... // 错误处理
}

result = num_a % num_b;
...
```

# 变量

## 禁止使用未经初始化的变量

**【描述】** 
这里的变量，指的是局部动态变量，并且还包括内存堆上申请的内存块。 
因为他们的初始值都是不可预料的，所以禁止未经有效初始化就直接读取其值。

```
void foo( ...)
{
	int data;
	bar(data); // 错误：未初始化就使用
	...
}
```

如果有不同分支，要确保所有分支都得到初始化后才能使用：

```
#define CUSTOMIZED_SIZE 100
void foo( ...)
{
	int data;
	if (condition > 0) {
		data = CUSTOMIZED_SIZE;
	}

	bar(data); // 错误：部分分支该值未初始化
	...
}
```

## 指向资源句柄或描述符的变量，在资源释放后立即赋予新值

**【描述】** 
指向资源句柄或描述符的变量包括指针、文件描述符、socket描述符以及其它指向资源的变量。 
以指针为例，当指针成功申请了一段内存之后，在这段内存释放以后，如果其指针未立即设置为NULL，也未分配一个新的对象，那这个指针就是一个悬空指针。 
如果再对悬空指针操作，可能会发生重复释放或访问已释放内存的问题，造成安全漏洞。 
消减该漏洞的有效方法是将释放后的指针立即设置为一个确定的新值，例如设置为NULL。对于全局性的资源句柄或描述符，在资源释放后，应该马上设置新值，以避免使用其已释放的无效值；对于只在单个函数内使用的资源句柄或描述符，应确保资源释放后其无效值不被再次使用。

**【错误代码示例】** 
如下代码示例中，根据消息类型处理消息，处理完后释放掉body或head指向的内存，但是释放后未将指针设置为NULL。如果还有其他函数再次处理该消息结构体时，可能出现重复释放内存或访问已释放内存的问题。

```
int foo(void)
{
	SomeStruct *msg = NULL;
	... // 初始化msg->type，分配 msg->body 的内存空间
	if (msg->type == MESSAGE_A) {
		...
		free(msg->body);
	}

	...
EXIT:
	...
	free(msg->body);
	return ret;
}
```

**【正确代码示例】** 
如下代码示例中，立即对释放后的指针设置为NULL，避免重复放指针。

```
int foo(void)
{
	SomeStruct  *msg = NULL;
	... // 初始化msg->type，分配 msg->body 的内存空间

	if (msg->type == MESSAGE_A) {
		...
		free(msg->body);
		msg->body = NULL;
	}

	...
EXIT:
	...
	free(msg->body);
	return ret;
}
```

当free()函数的入参为NULL时，函数不执行任何操作。

**【错误代码示例】** 
如下代码示例中文件描述符关闭后未赋新值。

```
SOCKET s = INVALID_SOCKET;
int fd = -1;
...
closesocket(s);
...
close(fd);
...
```

**【正确代码示例】** 
如下代码示例中，在资源释放后，对应的变量应该立即赋予新值。

```
SOCKET s = INVALID_SOCKET;
int fd = -1;
...
closesocket(s);
s = INVALID_SOCKET;
...
close(fd);
fd = -1;
...
```

# 指针和数组

## 外部数据作为数组索引时必须确保在数组大小范围内

**【描述】** 
外部数据作为数组索引对内存进行访问时，必须对数据的大小进行严格的校验，确保数组索引在有效范围内，否则会导致严重的错误。 
当一个指针指向数组元素时，可以指向数组最后一个元素的下一个元素的位置，但是不能读写该位置的内存。

**【错误代码示例】** 
如下代码示例中, set_dev_id()函数存在差一错误，当 index 等于 DEV_NUM 时，恰好越界写一个元素； 
同样get_dev()函数也存在差一错误，虽然函数执行过程中没有问题，但是当解引用这个函数返回的指针时，行为是未定义的。

```
#define DEV_NUM 10
#define MAX_NAME_LEN 128
typedef struct {
	int id;
	char name[MAX_NAME_LEN];
} Dev;

static Dev devs[DEV_NUM];
int set_dev_id(size_t index, int id)
{
	if (index > DEV_NUM) { // 错误：差一错误。
 		... // 错误处理
	}

	devs[index].id = id;
	return 0;
}

static Dev *get_dev(size_t index)
{
	if (index > DEV_NUM) { // 错误：差一错误。
 		... // 错误处理
	}

	return devs + index;
}
```



**【正确代码示例】** 
如下代码示例中，修改校验索引的条件，避免差一错误。

```
#define DEV_NUM 10
#define MAX_NAME_LEN 128
typedef struct {
	int id;
	char name[MAX_NAME_LEN];
} Dev;

static Dev devs[DEV_NUM];

int set_dev_Id (size_t index, int id)
{
	if (index >= DEV_NUM) {
		... // 错误处理
	}

	devs[index].id = id;
	return 0;
}

static Dev *get_dev(size_t index)
{
	if (index >= DEV_NUM) {
 		... // 错误处理
	}

	return devs + index;
}
```

**【相关软件CWE编号】** CWE-119，CWE-123，CWE-125

## 禁止通过对指针变量进行sizeof操作来获取数组大小

**【描述】** 
将指针当做数组进行sizeof操作时，会导致实际的执行结果与预期不符。例如：变量定义 char *p = array，其中array的定义为char array[LEN]，表达式sizeof(p) 得到的结果与 sizeof(char *)相同，并非array的长度。

**【错误代码示例】** 
如下代码示例中，buffer和path分别是指针和数组，程序员想对这2个内存进行清0操作，但由于程序员的疏忽，将内存大小误写成了sizeof(buffer)，与预期不符。

```
char path[MAX_PATH];
char *buffer = (char *)malloc(SIZE);
...
(void)memset(path, 0, sizeof(path));
// sizeof与预期不符，其结果为指针本身的大小而不是缓冲区大小
(void)memset(buffer, 0, sizeof(buffer));
```

**【正确代码示例】** 
如下代码示例中，将sizeof(buffer)修改为申请的缓冲区大小：

```
char path[MAX_PATH];
char *buffer = (char *)malloc(SIZE);
...
(void)memset(path, 0, sizeof(path));
(void)memset(buffer, 0, SIZE); // 使用申请的缓冲区大小
```

# 字符串

## 确保字符串存储有足够的空间容纳字符数据和null结束符

**【描述】** 
将数据复制到不足以容纳数据的缓冲区，会导致缓冲区溢出。缓冲区溢出经常发生在字符串操作中。为了避免这种错误，截断拷贝的数据以限制字符串的字节长度是一种防御方法，但是最好的措施是确保目标缓冲区的大小足以容纳复制数据和null结束符。当字符串存储在堆空间时， 确保分配内存时已分配了足够的空间。

部分字符串处理函数由于设计时安全考虑不足，或者存在一些隐含的目的缓冲区长度要求，容易被误用，导致缓冲区写溢出。此类典型函数包括不在C标准库函数中的itoa()，realpath()函数。

**【错误代码示例】**(itoa) 
有些函数如itoa(), realpath()需要在对传入的缓冲区指针位置进行写入操作，但函数并没有提供缓冲区长度。因此，在调用这些函数前，必须提供足够的缓冲区。 
如下代码示例中，试图将数字转为字符串，但是目标存储空间的预留长度不足：

```
int num = ...
char str[8];
itoa(num, str, 10); // 10进制整数的最大存储长度是12个字节
```

**【正确代码示例】** 
如下代码示例中，在对外部数据进行解析并将内容保存到name中，考虑了name的大小：

```
int num = ...
char str[13];
itoa(num, str, 10); // 10进制整数的最大存储长度是12个字节
```

【错误代码示例】**(realpath)**

如下代码示例中，试图将路径标准化，但是目标存储空间的长度不足：

```
#define MAX_PATH_LEN 100
char resolved_path[MAX_PATH_LEN];
/ *
- realpath函数的存储缓冲区长度是由PATH_MAX常量定义，
- 或是由_PC_PATH_MAX系统值配置的，通常都大于100字节
*/

char  *res = realpath(path, resolved_path);
...
```

**【正确代码示例】**

可以将realpath的第二个参数传入NULL, 以让系统自动分配合适的内存。

```
char *resolved_path = NULL;
resolved_path = realpath(path, NULL);
if (resolved_path == NULL) {
	... // 处理错误
}

...
if (resolved_path != NULL) {
	free(resolved_path);
	resolved_path = NULL;
}
...
```

## 对字符串进行存储操作，确保字符串有null结束符

**【描述】** 
部分字符串处理函数操作字符串时，将截断超出指定长度的字符串，如strncpy()函数最多复制n个字符到目的缓冲区，如果源字符串长度大于n，则不会写入null结束符到目的缓冲区，目的缓冲区的内容为n个被复制的字符。使用这类函数时，可能会无意截断导致数据丢失，并在某些情况下会导致软件漏洞。 
因此，对字符串进行存储操作，必须确保字符串有null结束符，否则在后续的调用strlen等操作中，可能会导致内存越界访问漏洞。

**【错误代码示例】** 
在如下代码示例中，使用strncpy函数复制字符串时可能会发生截断（发生条件为：strlen(name) > sizeof(file_name) - 1）。当发生截断时，file_name的内容是不完整的，并且缺少 '  0 '结束符，后续对file_name的操作可能会导致软件漏洞：

```
#define FILE_NAME_LEN 128
char file_name [FILE_NAME_LEN ];
(void)strncpy(file_name, name, sizeof(file_name) - 1);
...
```

**【正确代码示例】**

```
#define FILE_NAME_LEN 128
char file_name[FILE_NAME_LEN ];

if (strlen(name)  > FILE_NAME_LEN - 1) {
	... // 处理错误
}

(void)strcpy(file_name, name);
...
```

**【例外】**

程序员的目的是故意截断字符串。

**【相关软件CWE编号】** CWE-170，CWE-464

# 断言

## 禁止用断言检测程序在运行期间可能导致的错误，可能发生的错误要用错误处理代码来处理

**【描述】** 
断言主要用于调试期间，在编译Release版本时将其关闭。因此，断言应该用于防止不正确的程序员假设，而不能用在Release版本上检查程序运行过程中发生的错误。

断言永远不应用于验证是否存在运行时（与逻辑相对）错误，例如

-   无效的用户输入（包括命令行参数和环境变量）

-   文件错误（例如，打开，读取或写入文件时出错）

-   网络错误（包括网络协议错误）

-   内存不足的情况（例如，malloc()类似的故障）

-   系统资源耗尽（例如，文件描述符，进程，线程）

-   系统调用错误（例如，执行文件，锁定或解锁互斥锁时出错）

-   无效的权限（例如，文件，内存，用户）

例如，防止缓冲区溢出的代码不能使用断言实现，因为该代码必须编译到Release版本的可执行文件中。 
如果服务器程序在网运行时由恶意用户触发断言失败，会导致拒绝服务攻击。在这种情况下，更适合使用软故障模式，例如写入日志文件和拒绝请求。

**【错误代码示例】** 
以下代码的所有ASSERT的用法是错误的。例如，错误的使用ASSERT宏来验证内存分配是否成功，因为内存的可用性取决于系统的整体状态，并且在程序运行的任何时候都可能耗尽，所以必须以具有韧性的方式来妥善处理并将程序从内存耗尽中恢复。因此，使用ASSERT宏来验证内存分配是否成功将是不合适的，因为这样做可能导致进程突然终止，从而开启了拒绝服务攻击的可能性。

```
FILE  *fp = fopen(path,  "r");
ASSERT(fp != NULL); // 错误用法：文件有可能打开失败
char  *str = (char *)malloc(MAX_LINE);
ASSERT(str != NULL); // 错误用法：内存有可能分配失败
ReadLine(fp, str);
char  *p = strstr(str, "age=");
ASSERT(p != NULL); // 错误用法：文件中不一定存在该字符串
char  *end = NULL;
long age = strtol(p + 4, &end, 10);
ASSERT(age > 0); // 错误用法：文件内容不一定符合预期
```

**【正确代码示例】** 
下面代码演示了如何重构上面的错误代码:

```
FILE  *fp = fopen(path,  "r");
if (fp == NULL) {
	... // 错误处理
}

char  *str = (char *)malloc(MAX_LINE);
if (str == NULL) {
	... // 错误处理
}

read_line(fp, str);
char  *p = strstr(str,  "age=");
if (p == NULL) {
	... // 错误处理
}

char *end = NULL;

long age = strtol(p + 4, &end, 10);
if (age <= 0) {
	... // 错误处理
}
```

## 禁止在断言内改变运行环境

**【描述】** 
在程序正式发布阶段，断言不会被编译进去，为了确保调试版和正式版的功能一致性，严禁在断言中使用任何赋值、修改变量、资源操作、内存申请等操作。

例如，以下的断言方式是错误的：

```
ASSERT(p1 = p2); // p1被修改
ASSERT(i++  > 1000); // i被修改
ASSERT(close(fd) == 0); // fd被关闭
```

# 函数设计

## 数组作为函数参数时，必须同时将其长度作为函数的参数

**【描述】** 
通过函数参数传递数组函数参数必须同时传递数组可容纳元素的个数，而不是以字节为单位的数组最大大小；同样，通过函数参数传递一块内存进行读写操作时，必须同时传递内存块大小，否则函数在访问内存偏移时，无法判断偏移的合法范围，产生越界访问的漏洞。在本规则中所说的"数组"不仅局限为数组类型变量，还包括字符串或指向连续内存块的指针。

**【错误代码示例】** 
如下代码示例中，函数pars_msg不知道msg的范围，容易产生内存越界访问漏洞。

```
int parse_msg(unsigned char *msg)
{
	...
}

void foo(void)
{
	size_t len = get_msg_len();
	...
	unsigned char *msg = (unsigned char  *)malloc(len);
	...
	parse_msg(msg);
	...
}
```

**【正确代码示例】** 
正确的做法是将msg的大小作为参数传递到parse_msg中，如下代码：

```
int parse_msg(unsigned char *msg, size_t msg_len)
{
	ASSERT(msg != NULL);
	ASSERT(msg_len != 0);
	...
}

void foo(void)
{
	size_t len = get_msg_len();
 	...
	unsigned char *msg = (unsigned char *)malloc(len);

 	...
	parse_msg(msg, len);
 	...
}
```

## 函数的指针参数如果不是用于修改所指向的对象就应该声明为指向const的指针

**【描述】** 
const 指针参数，将限制函数通过该指针修改所指向对象，使代码更牢固、更安全。

示例：如strncmp 的例子，指向的对象不变化的指针参数声明为const。

```
// 正确：不变参数声明为const
int strncmp(const char *s1, const char *s2, size_t n);
```

注意：

指针参数要不要加const取决于函数设计，而不是看函数实体内有没有发生"修改对象"的动作。

## 调用格式化输入/输出函数时，禁止format参数受外部数据控制

**【描述】** 
调用格式化函数时，如果format参数由外部数据提供，或由外部数据拼接而来，会造成字符串格式化漏洞。

攻击者如果能够完全或者部分控制格式字符串内容，可以使被攻击的进程崩溃、查看栈内容、查看内存内容或者在任意内存位置写入数据。结果是，攻击者能够以被攻击进程的权限执行任意代码。

格式化输出函数特别危险，这是因为许多程序员没有意识到它们是具有攻击能力的。比如：格式化输出函数可以使用%n转换符，向指定地址写入一个整数值。

这些格式化函数有： 
格式化输出函数: xxxprintf; 
格式化输入函数: xxxscanf; 
格式化错误消息函数: err(), verr(), errx(), verrx(), warn(), vwarn(), warnx(), vwarnx(), error(), error_at_line(); 
格式化日志函数: syslog(), vsyslog().

**【错误代码示例】** 
如下代码示例中的incorrect_password()函数的功能是在身份验证无效时（指定用户没有找到或者密码不正确），显示一条错误信息。 
该函数接受一个源自用户的字符串数据user，而user是未验证的，是外部可控的。 
该函数将user构造一条错误信息，然后用C语言标准函数fprintf打印到stderr。

```
// 调用者需保证入参user的长度被限制为256个字节或者更少
void incorrect_password(const char *user)
{
	int ret = -1;
	static const char msg_format[] = "%s cannot be authenticated.\n";
	size_t len = strlen(user) + 1 + sizeof(msg_format);

	char *msg = (char *)malloc(len);
	if (msg == NULL) {
 		... // 错误处理
	}

	ret = snprintf(msg, msg_format, user);
	if (ret == -1) {
 		... // 错误处理
	} else {
		fprintf(stderr, msg); // msg中有来自未验证的外部数据，存在格式化漏洞
	}

	free(msg);
}
```

示例代码中首先计算了消息的长度，然后分配内存，接着利用snprintf()函数拼接了消息内容。因此消息内容中包含了msg_format的内容和用户的内容。 
当入参user中含有用户输入的格式符（如%s,%p,%n等后，fprintf()在执行时，会将msg作为一个格式化字符串来进行解析，而不是直接输出消息内容， 
也就是说此时msg中的内容不会被直接打印到stderr中，反而会将一些未知的数据打印到stderr，引发程序未定义的行为。这是一个非常严重的格式化漏洞。

**【正确代码示例】** 
下面是第一种推荐做法，代码中使用fputs()来代替fprintf()函数，fputs()会直接将msg的内容输出到stderr中，而不会去解析它。

```
// 入参user的长度被限制为256个字节或者更少
void incorrect_password(const char *user)
{
	int ret = -1;
	static const char msg_format[] = "%s cannot be authenticated.\n";

	// 这里加法运算不会整数溢出，因为user有限制
	size_t len = strlen(user) + 1 + sizeof(msg_format);
	char *msg = (char *)malloc(len);
	if (msg == NULL) {
 		... // 错误处理
	}

	ret = snprintf(msg, msg_format, user);
	if (ret == -1) {
 		... // 错误处理
	} else {
		fputs(stderr, msg); // 使用fputs函数代替fprintf函数
	}

	free(msg);
}
```

**【正确代码示例】** 
下面是第二种推荐做法，代码中将不受信任的用户输入user作为fprintf()的可选参数之一，用"%s"将user以字符串的形式固定下来，然后输出到stderr中，而不作为格式字符串的一部分，这样就消除了格式字符串漏洞出现的可能性。

```
void incorrect_password(const char  *user)
{
	static const char msg_format[] = "%s cannot be authenticated.\n";
	fprintf(stderr, msg_format, user);
}
```

**【错误代码示例】** 
如下代码示例中，使用了POSIX函数syslog()，但是syslog()函数也可能出现格式字符串漏洞。

```
void foo(void)
{
	char  *msg = get_msg();
	...

	syslog(LOG_INFO, msg); // 存在格式化漏洞
}
```

**【正确代码示例】** 
下面是推荐做法，代码中将不受信任的用户输入msg作为syslog()的可选参数之一，用"%s"将msg以字符串的形式固定下来，然后输出到系统日志中，而不作为格式字符串的一部分，这样就消除了格式字符串漏洞出现的可能性。

```
void foo(void)
{
	static const char msg_format[] =  "%s cannot be authenticated.\n";
	char  *msg = get_msg();
	...
	syslog(LOG_INFO, msg_format, msg); // 这里没有格式化漏洞
}
```



# 函数使用

## 调用格式化输入/输出函数时，使用有效的格式字符串

**【描述】**

格式化输入/输出函数（如fscanf()/fprintf()及相关函数）在format字符串控制下进行转换、格式化、打印其实参。

在创建格式化字符串时的常见错误包括：

-   format中参数个数与实参个数不一致；

-   使用无效的转换指示符；

-   使用与转换指示符不兼容的标志字符；

-   使用与转换指示符不兼容的长度修饰符；

-   format中转换指示符与实参类型不匹配；

-   使用int以外类型的实参指定宽度或者精度；

不要为格式化输入/输出函数提供未知的或者无效的转换规格，以及标志字符、精度、长度修饰符、转换指示符的无效组合。同样，不要提供与格式化字符串中的转换指示符类型不匹配的实参。这可能会使程序产生未定义行为。

**【错误代码示例】**

如下代码示例中，printf()的实参infoLevel类型与对应的转换指示符 's '不匹配，正确的转换指示符要使用 'd '。同样，实参infoMsg类型与对应的转换指示符 'd '不匹配，正确的转换指示符要使用 's '。 
这些用法会使程序产生未定义行为，比如：printf()将把infoLevel实参解释为指针，试图从infoLevel包含的地址中读取一个字符串，从而发生非法访问。

```
void foo(void)
{
	const char *info_msg = "Information seed to user.";
	int info_level = 3;
	...
	printf("infoLevel: %s, infoMsg: %d\n", info_level, info_msg);
	...
}
```

**【正确代码示例】**

正确的做法是确保printf()函数的实参匹配format的转换指示符。

```
void foo(void)
{
	const char *info_msg = "Information seed to user.";
	int info_level = 3;
	...

	printf("infoLevel: %d, infoMsg: %s\n", info_level, info_msg);
	...
}
```

**【影响】**

错误的格式串可能造成内存破坏或者程序异常终止。

## 禁止使用alloca()函数申请栈上内存

**【描述】** 
POSIX和C99均未定义alloca()的行为，在有些平台下不支持该函数，使用alloca会降低程序的兼容性和可移植性，该函数在栈帧里申请内存，申请的大小很可能超过栈的边界，影响后续的代码执行。

请使用malloc从堆中动态分配内存。

【影响】

程序栈的大小非常有限，如果分配导致栈溢出，则程序产生未定义行

## 禁止使用realloc()函数

**【描述】** 
realloc()是一个非常特殊的函数，原型如下：

void *realloc(void *ptr, size_t size);

随着参数的不同，其行为也是不同：

-   当ptr不为NULL，且size不为0时，该函数会重新调整内存大小，并将新的内存指针返回，并保证最小的size的内容不变；

-   参数ptr为NULL，但size不为0，那么其行为等同于malloc(size)；

-   参数size为0，则realloc的行为等同于free(ptr)。

由此可见，一个简单的C函数，却被赋予了3种行为，这不是一个设计良好的函数。虽然在编程中提供了一些便利性，如果认识不足，使用不当，是却极易引发各种bug。

**【错误代码示例】** 
如下代码示例中，使用realloc不当导致内存泄漏。 
代码中希望对ptr的空间进行扩充，当realloc()分配失败的时候，会返回NULL。但是参数中的ptr的内存是没有被释放的，如果直接将realloc()的返回值赋给ptr，那么ptr原来指向的内存就会丢失，造成内存泄漏。

```
// 当realloc()分配内存失败时会返回NULL，导致内存泄漏
char *ptr = (char  *)realloc(ptr, NEW_SIZE);
if (ptr == NULL) {
	...// 错误处理
}
```

**【正确代码示例】** 
使用malloc()函数代替realloc()函数。

```
// 使用malloc()函数代替realloc()函数
char *new_ptr = (char *)malloc(NEW_SIZE);
if (new_ptr == NULL) {
	... // 错误处理
}

(void)memcpy(new_ptr, old_ptr, old_size);
... // 返回前，释放old_Ptr
```

**【影响】**

使用不当容易造成内存泄漏和双重释放问题。不正确的内存对齐可能导致对象访问异常。

## 禁止外部可控数据作为进程启动函数的参数

**【描述】** 
本条款中进程启动函数包括system、popen、execl、execlp、execle、execv、execvp等。

system()、popen()等函数会创建一个新的进程，如果外部可控数据作为这些函数的参数，会导致注入漏洞。

使用execl()、execlp()等函数执行新进程时，如果使用shell启动的新进程，则同样存在命令注入风险。

因此，总是优先考虑使用C标准函数实现需要的功能。如果确实需要使用这些函数，请使用白名单机制确保这些函数的参数不受任何外来数据的影响。

**【错误代码示例】** 
如下代码示例中，使用 system() 函数执行 cmd 命令串来自外部，攻击者可以执行任意命令：

```
char  *cmd = get_cmd_from_remote();
if (cmd == NULL) {
	... // 处理错误
}

system(cmd);
```

如下代码示例中，使用 system() 函数执行 cmd 命令串的一部分来自外部，攻击者可能输入  'some dir;useradd xxx '字符串，创建一个xxx的用户：

```
char cmd[MAX_LEN ];
int ret;

char  *name = get_dir_name_from_remote();
if (name == NULL) {
	... // 处理错误
}

ret = sprintf(cmd, "ls %s", name);
...
system(cmd);
```

使用exec系列函数来避免命令注入时，注意exec系列函数中的path、file参数禁止使用命令解析器(如/bin/sh)。

```
int execl(const char *path, const char *arg,  ...);
int execlp(const char *file, const char *arg,  ...);
int execle(const char *path, const char *arg,  ..., char * const envp[]);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
```

例如，禁止如下使用方式：

```
char *cmd = get_dir_name_from_remote();
execl("/bin/sh", "sh",  "-c", cmd, NULL);
```

**【正确代码示例】** (使用库函数)

在Linux下实现对当前目录下文件名的打印，可以使用opendir(), readdir(), stat()等函数直接实现ls-l命令的功能，不必使用system()函数。下面是一个简化的ls -l示例版本，列出一个由程序内部指定的文件的信息，该函数仅考虑了不需要重入的情况。

```
static int OutputFileInfo(const char *file_name)
{
	const char priv[] = {'x', 'w', 'r'};
	ASSERT(file_name != NULL);

	struct stat st;
	int ret = stat(file_name, &st);
	if (ret == -1) {
		return -1;
	}

	const struct passwd *pw = getpwuid(st.st_uid);
	if (pw == NULL) {
		return -1;
	}

	const struct group *gp = getgrgid(st.st_gid);
	if (gp == NULL) {
		return -1;
	}

	if (S_ISREG(st.st_mode)) {
		printf("-");
	} else if (S_ISDIR(st.st_mode)) {
		printf("d");
	}

	for (int i = 8; i >= 0; i --) {
		if ((st.st_mode & (1 < < i)) != 0) {
			printf("%c", priv[i % 3]);
		} else {
			printf("-");
		}
	}

	printf("%s %s %ld %s\n",
	pw->pw_name,
	gp->gr_name,
	st.st_size,
	file_name);
	return 0;
}
```

**【正确代码示例】** (使用exec系列函数)

可以通过库函数简单实现的功能（如上例），需要避免调用命令处理器来执行外部命令。如果确实需要调用单个命令，应使用exec *函数来实现参数化调用，并对调用的命令实施白名单管理。

```
pid_t pid;
char * const envp[] = { NULL };
...
char *file_name = get_dir_name_from_remote();
if (file_name == NULL) {
	... // 处理错误
}
...
if ((pid = fork()) < 0) {
	...
} else if (pid == 0) {
	// 使用some_tool对指定文件进行加工
	execle( "/bin/some_tool", "some_tool", file_name, NULL, envp);
	_Exit(-1);
}

...
int status;
waitpid(pid, &status, 0);
FILE *fp = fopen(file_name, "r");
...
```

此时，外部输入的file_name仅作为some_tool命令的参数，没有命令注入的风险。

**【正确代码示例】** (使用白名单)

对输入的文件名基于合理的白名单检查，避免命令注入。

```
char *cmd = get_cmd_from_remote();
if (cmd == NULL) {
	... // 处理错误
}

// 使用白名单检查命令是否合法，仅允许 "some_tool_a", "some_tool_b"命令，外部无法随意控制
if (!is_valid_cmd(cmd)) {
	... // 处理错误
}

system(cmd);
...
```

**【相关软件CWE编号】** CWE-676，CWE-88

## 禁止在信号处理例程中调用非异步安全函数

**【描述】** 
在信号处理程序中只调用异步安全函数。

除了C语言标准函数以外，其他系统函数也提供了一些的异步安全函数，在信号处理程序中使用这些函数之前，应确保调用的函数在所有可能的执行环境下均是异步安全的。

**【错误代码示例】** 
如下代码示例中，信号处理函数中调用了非异步安全函数printf()：

```
void handler(int num)
{
	printf("receive signal = %d \n", SIGINT);
}

int main(int argc, char **argv)
{
	if (signal(SIGINT, handler) == SIG_ERR) {
 		... // 错误处理
	}

	while (true) {
 		... // 程序主循环代码
	}

	return 0;
}
```

**【正确代码示例】** 
如下代码示例中，尽量不在信号处理函数中调用其他函数，仅在信号处理程序中修改volatile sig_atomic_t类型的变量：

```
static volatile sig_atomic_t g_flag = 0;
void handler(int num)
{
	g_flag = 1;
}

int main(int argc, char **argv)
{
	if (signal(SIGINT, handler) == SIG_ERR) {
	... // 错误处理
	}

	while (true) {
		if (g_flag != 0) {
			printf("receive signal = %d\n", SIGINT);
		}

		... // 程序主循环代码
	}

	...
	return 0;
}
```

**【相关软件CWE编号】** CWE-479

# 内存

## 内存分配后必须判断是否成功

**【描述】** 
内存分配一旦失败，那么后续的操作会存在未定义的行为风险。比如malloc申请失败返回了空指针，对空指针的解引用是一种未定义行为。

**【错误代码示例】** 
如下代码示例中，调用malloc分配内存之后，没有判断是否成功，直接引用了p。如果malloc失败，它将返回一个空指针并传递给p。当如下代码在内存拷贝中解引用了该空指针p时，程序会出现未定义行为。

```
struct tm *make_tm(int year, int mon, int day, int hour, int min, int sec)
{
	struct tm *tmb = (struct tm*)malloc(sizeof(*tmb));
	tmb->year = year;
	...

	return tmb;
}
```

**【正确代码示例】** 
如下代码示例中，在malloc分配内存之后，立即判断其是否成功，消除了上述的风险。

```
struct tm  *make_tm(int year, int mon, int day, int hour, int min, int sec)
{
	struct tm  *tmb = (struct tm *)malloc(sizeof(*tmb));
	if (tmb == NULL) {
		... // 错误处理
	}

	tmb->year = year;
	...
	return tmb;
}
```

## 外部输入作为内存操作相关函数的复制长度时，需要校验其合法性

**【描述】** 
将数据复制到容量不足以容纳该数据的内存中会导致缓冲区溢出。为了防止此类错误，必须根据目标容量的大小限制被复制的数据大小，或者必须确保目标容量足够大以容纳要复制的数据。

**【错误代码示例】** 
外部输入的数据不一定会直接作为内存复制长度使用，还可能会间接参与内存复制操作。 
如下代码示例中，inputTable->count来自外部报文，虽然没有直接作为内存复制长度使用，而是作为for循环体的上限使用，间接参与了内存复制操作。由于没有校验其大小，可造成缓冲区溢出：

```
typedef struct {
	size_t count;
	int val[MAX_num_bERS];
} ValueTable;

ValueTable *value_table_dup(const ValueTable *input_table)
{
	ValueTable *output_table = ... // 分配内存
	...
	for (size_t i = 0; i  < input_table->count; i++) {
		output_table->val[i] = input_table->val[i];
	}
 	...
}
```

**【正确代码示例】** 
如下代码示例中，对input_table->count做了校验。

```
typedef struct {
size_t count;
int val[MAX_num_bERS];
}ValueTable;

ValueTable *value_table_dup(const ValueTable *input_table)
{
	ValueTable *output_table = ... // 分配内存
	...

	/ *
	- 根据应用场景，对来自外部报文的循环长度input_table->count
	- 与output_table->val数组大小做校验，避免造成缓冲区溢出
	*/
	if (input_table->count  > sizeof(output_table->val) / sizeof(output_table->val[0]){
		return NULL;
	}

	for (size_t i = 0; i  < input_table->count; i++) {
		output_table->val[i] = input_table->val[i];
	}

	...
}
```

## 内存中的敏感信息使用完毕后立即清0

**【描述】** 
内存中的口令、密钥等敏感信息使用完毕后立即清0，避免被攻击者获取或者无意间泄漏给低权限用户。这里所说的内存包括但不限于：

-   动态分配的内存

-   静态分配的内存

-   自动分配（堆栈）内存

-   内存缓存

-   磁盘缓存

**【错误代码示例】** 
通常内存在释放前不需要清除内存数据，因为这样在运行时会增加额外开销，所以在这段内存被释放之后，之前的数据还是会保留在其中。如果这段内存中的数据包含敏感信息，则可能会意外泄漏敏感信息。为了防止敏感信息泄漏，必须先清除内存中的敏感信息，然后再释放。 
在如下代码示例中，存储在所引用的动态内存中的敏感信息secret被复制到新动态分配的缓冲区newSecret，最终通过free()释放。因为释放前未清除这块内存数据，这块内存可能被重新分配到程序的另一部分，之前存储在newSecret中的敏感信息可能会无意中被泄露。

```
char *secret = NULL;

/ *
- 假设 secret 指向敏感信息，敏感信息的内容是长度小于SIZE_MAX个字符，
- 并且以null终止的字节字符串
*/
size_t size = strlen(secret);
char *new_secret = NULL;
new_secret = (char *)malloc(size + 1);
if (new_secret == NULL) {
	... // 错误处理
} else {
	strcpy(new_secret, secret);
	... // 处理 new_secret ...
	free(new_secret);
	new_secret = NULL;
}

...
```

**【正确代码示例】** 
如下代码示例中，为了防止信息泄漏，应先清除包含敏感信息的动态内存（用 '  0 '字符填充空间），然后再释放它。

```
char *secret = NULL;
/ *
- 假设 secret 指向敏感信息，敏感信息的内容是长度小于SIZE_MAX个字符，
- 并且以null终止的字节字符串
*/
size_t size = strlen(secret);
char *new_secret = NULL;
new_secret = (char *)malloc(size + 1);
if (new_secret == NULL) {
	... // 错误处理
} else {
	strcpy(new_secret, secret);
	... // 处理 new_secret ...
	(void)memset(new_secret, 0, size + 1);
	free(new_secret);
	new_secret = NULL;
}
...
```

**【正确代码示例】** 
下面是另外一个涉及敏感信息清理的场景，在代码获取到密码后，将密码保存到password中，进行密码验证，使用完毕后，通过memset对password清0。

```
int foo(void)
{
	char password [MAX_PWD_LEN ] = {0};
	if (!get_password(password, sizeof(password))) {
		...  // 处理错误 
	}

	if (!verify_password(password)) {
		... // 处理错误
	}

	...
	(void)memset(password, 0, sizeof(password));
	...
}
```



# 文件

## 创建文件时必须显式指定合适的文件访问权限

**【描述】** 
创建文件时，如果不显式指定合适访问权限，可能会让未经授权的用户访问该文件，造成信息泄露，文件数据被篡改，文件中被注入恶意代码等风险。

虽然文件的访问权限也依赖于文件系统，但是当前许多文件创建函数（例如POSIX open函数）都具有设置（或影响）文件访问权限的功能，所以当使用这些函数创建文件时，必须显式指定合适的文件访问权限，以防止意外访问。

**【错误代码示例】** 
使用POSIX open()函数创建文件但未显示指定该文件的访问权限，可能会导致文件创建时具有过高的访问权限，这可能会导致漏洞。

```
void foo(void)
{
	int fd = -1;
	char *file_name = NULL;
	... // 初始化 file_name
	fd = open(file_name, O_CREAT | O_WRONLY); // 没有显式指定访问权限
	if (fd == -1) {
		... // 错误处理
	}
	...
}
```

**【正确代码示例】** 
应该在open的第三个参数中显式指定新创建文件的访问权限。可以根据文件实际的应用情况设置何种访问权限。

```
void foo(void)
{
	int fd = -1;
	char *file_name = NULL;
	... // 初始化 file_name 和指定其访问权限
	// 此处根据文件实际需要，显式指定其访问权限
	int fd = open(file_name, O_CREAT | O_WRONLY, S_IRUSR | S_IWUSR);
	if (fd == -1) {
		... // 错误处理
	}
	...
}
```

## 必须对文件路径进行规范化后再使用

**【描述】** 
当文件路径来自外部数据时，必须对其做合法性校验，如果不校验，可能造成系统文件的被任意访问。但是禁止直接对其进行校验，正确做法是在校验之前必须对其进行路径规范化处理，因为： 
同一个文件可以通过多种形式的路径来描述和引用，例如既可以是绝对路径，也可以是相对路径；而且路径名、目录名和文件名可能包含使校验变得困难和不准确的字符（如："."、".."）。此外，文件还可以是符号链接，这进一步模糊了文件的实际位置或标识，增加了校验的难度和校验准确性。所以必须先将文件路径规范化，从而更容易校验其路径、目录或文件名，增加校验准确性，如使用realpath函数。

一个简单的案例说明如下：

当文件路径来自外部数据时，需要先将文件路径规范化，如果没有作规范化处理，攻击者就有机会通过恶意构造文件路径进行文件的越权访问。

例如，攻击者可以构造"../../../etc/passwd"的方式进行任意文件访问。

**【错误代码示例】** 
在此错误的示例中，argv[1]包含一个源于受污染源的文件名，并且该文件名已打开以进行写入。在使用此文件名操作之前，应该对其进行验证，以确保它引用的是预期的有效文件。 
不幸的是，argv[1]引用的文件名可能包含特殊字符，例如目录字符，这使验证变得困难，甚至不可能。而且，argv[1]中可能包含可以指向任意文件路径的符号链接，即使该文件名通过了验证，也会导致该文件名是无效的。 
这种场景下，对文件名的直接验证即使被执行也是得不到预期的结果，对fopen()的调用可能会导致访问一个意外的文件。

```
...
if (!verify_file(input_file_name) { // 没有对input_file_name做规范化，直接做校验
	... // 错误处理
}

if (fopen(input_file_name, "w") == NULL) {
	... // 错误处理
}
...
```

**【正确代码示例】** 
规范化文件名是具有一定难度的，因为这需要了解底层文件系统。 
POSIX realpath()函数可以帮助将路径名转换为规范形式。

对上面的错误代码示例，我们采用如下解决方案：

```
char *real_path_res = NULL;
...
// 在校验之前，先对input_file_name做规范化处理
real_path_res = realpath(input_file_name, NULL);
if (real_path_res == NULL) {
	... // 规范化的错误处理
}

// 规范化以后对路径进行校验
if (!verify_file(real_path_res) {
	... // 校验的错误处理
}

// 使用
if (fopen(real_path_res, "w") == NULL) {
	... // 实际操作的错误处理
}

...
free(real_path_res);
real_path_res = NULL;
...
```

**【正确代码示例】** 
根据我们的实际场景，我们还可以采用的第二套解决方案，说明如下：

如果PATH_MAX被定义为中的一个常量，那么使用非空的resolved_path调用realpath()也是安全的。 
在本例中realpath()函数期望resolved_path引用一个字符数组，该字符数组足够大，可以容纳规范化的路径。 
如果定义了PATH_MAX，则分配一个大小为PATH_MAX的缓冲区来保存realpath()的结果。正确代码示例如下：

```
char *real_path_res = NULL;
char *canonical_file_name = NULL;
size_t path_size = 0;
...
path_size = (size_t)PATH_MAX;
if (verify_path_size(path_size) == TRUE) {
	canonical_file_name = (char *)malloc(path_size);
	if (canonical_file_name == NULL) {
		... // 错误处理
	}

	real_path_res = realpath(inputFilename, canonical_file_name);
}

if (real_path_res == NULL) {
	... // 错误处理
}

if (verify_file(real_path_res) == FALSE) {
	... // 错误处理
}

if (fopen(real_path_res, "w") == NULL ) {
	... // 错误处理
}

...
free(canonical_file_name);
canonical_file_name = NULL;
...
```

**【错误代码示例】** 
下面的代码场景是从外部获取到文件名称，拼接成文件路径后，直接对文件内容进行读取，导致攻击者可以读取到任意文件的内容：

```
char *file_name = get_msg_from_remote();
...
sprintf(untrust_path, "/tmp/%s", file_name);
char *text = read_file_content(untrust_path);
```

**【正确代码示例】** 
正确的做法是，对路径进行规范化后，再判断路径是否是本程序所认为的合法的路径：

```
char *file_name = get_msg_from_remote();
...
sprintf(untrust_path, "/tmp/%s", file_name);
char path[PATH_MAX] = {0};
if (realpath(untrust_path, path) == NULL) {
	... // 处理错误
}

if (!is_valid_path(path)) { // 检查文件的位置是否正确
	... // 处理错误
}

char *text = read_file_content(path);
```

**【例外】**

运行于控制台的命令行程序，通过控制台手工输入文件路径，可以作为本条款例外。

```
int main(int argc, char **argv)
{
	int fd = -1;
	if (argc == 2) {
		fd = open(argv[1], O_RDONLY);
		...
	}

	...
}
```

## 不要在共享目录中创建临时文件

**【描述】** 
程序的临时文件应当是程序自身独享的，任何将自身临时文件置于共享目录的做法，将导致其他共享用户获得该程序的额外信息，产生信息泄露。因此，不要在任何共享目录创建仅由程序自身使用的临时文件。

程序员通常会在共享目录中(例如在/tmp和/var/tmp创建临时文件，并且还有可能会定期清除这些临时文件(例如，每晚或重新启动期间)，但也可能不注意清理。

临时文件通常用于辅助保存不能驻留在内存中的数据或存储临时的数据，也可用作进程间通信的一种手段（通过文件系统传输数据）。例如，一个进程在共享目录中创建一个临时文件，该文件名可能使用了众所周知的名称或者一个临时的名称，然后就可以通过该文件在进程间共享信息。这种通过在共享目录中创建临时文件的方法实现进程间共享的做法很危险，因为共享目录中的这些文件很容易被攻击者劫持或操纵。这里有几种缓解策略：

-   1.使用其他低级IPC（进程间通信）机制，例如套接字或共享内存。

-   2.使用更高级别的IPC机制，例如远程过程调用。

-   3.使用仅能由程序本身访问的安全目录(多线程/进程下注意防止条件竞争)。

同时，下面列出了几项临时文件创建使用的方法，产品根据具体场景执行以下一项或者几项，同时产品也可以自定义合适的方法。

-   1.文件必须具有合适的权限，只有符合权限的用户才能访问

-   2.创建的文件名是唯一的、或不可预测的

-   3.仅当文件不存在时才创建打开(原子创建打开)

-   4.使用独占访问打开，避免竞争条件

-   5.在程序退出之前移除

同时也需要注意到，当某个目录被开放读/写权限给多个用户或者一组用户时，该共享目录潜在的安全风险远远大于访问该目录中临时文件这个功能的本身。

如果想安全地在共享目录中创建临时文件，而不受威胁是不容易的。例如，用于本地挂载的文件系统的代码在与远程挂载的文件系统一起共享使用时可能会受到攻击。而且上面的函数安全版本还受限于所使用的C运行时库、操作系统和文件系统的版本。唯一安全的解决方案是不要在共享目录中创建临时文件。

**【错误代码示例】** 
如下代码示例，程序在Linux系统的共享目录/tmp下创建临时文件来保存临时数据，且文件名是硬编码的。 
由于文件名是硬编码的，因此是可预测的，攻击者只需用符号链接替换文件，然后链接所引用的目标文件就会被打开并写入新内容。

```
void proc_data(const char  *file_name)
{
	FILE  *fp = fopen(file_name, "wb+");
	if (fp == NULL) {
		... // 错误处理
	}

	... // 写文件
	fclose(fp);
}

int main(void)
{
	// 不合规：1.在系统共享目录中创建临时文件；2.临时文件名硬编码
	char  *real_file = "/tmp/data";
 	...
	proc_data(real_file);
 	...
	return 0;
}
```

**【正确案例】**

Linux下的/tmp目录是一个所有用户都可以访问的共享目录，不应在该目录下创建仅由程序自身使用的临时文件。

【业界典型漏洞】CVE-2004-2502

# 其它

## 不要在信号处理函数中访问共享对象

**【描述】** 
在信号处理程序中访问和修改共享对象可能会造成竞争条件，使数据处于不确定的状态。 
这条规则有两个不适用的场景（C11标准第5.1.2.3章节第5段）是：

1） 读写不需要加锁的原子对象;

2）读写volatile sig_atomic_t类型的对象，因为具有volatile sig_atomic_t类型的对象即使在出现异步中断的时候也可以作为一个原子实体访问，是异步安全的。

此外，在信号处理程序中，如果要调用函数，请仅调用异步信号安全函数。

**【错误代码示例】** 
在这个信号处理过程中，程序打算将p_msg作为共享对象，当产生SIGINT信号时更新共享对象的内容，但是该p_msg变量类型不是volatile sig_atomic_t，所以不是异步安全的。

```
#define MAX_MSG_SIZE 32
static char g_msg_buf[MAX_MSG_SIZE] = {0};
static char *g_msg = g_msg_buf;
void signal_handler(int signum)
{
	// 下面代码操作g_msg不合规，因为不是异步安全的
	(void)memset(g_msg, 0, MAX_MSG_SIZE);
	strcpy(g_msg,  "signal SIGINT received.");
	... //
}

int main(void)
{
	strcpy(g_msg,  "No msg yet."); // 初始化消息内容
	signal(SIGINT, signal_handler); // 设置SIGINT信号对应的处理函数
	... // 程序主循环代码
	return 0;
}
```

**【正确代码示例】** 
如下代码示例中，在信号处理函数中仅将volatile sig_atomic_t类型作为共享对象使用。

```
#define MAX_MSG_SIZE 32
volatile sig_atomic_t g_sig_flag = 0;
void signal_handler(int signum)
{
	g_sig_flag = 1; // 合规
}

int main(void)
{
	signal(SIGINT, signal_handler);
	char msg_buf[MAX_MSG_SIZE];
	strcpy(msg_buf, "No msg yet."); // 初始化消息内容
	... // 程序主循环代码
	if (g_sig_flag == 1) { // 在退出主循环之后，根据sigFlag状态再刷新消息内容
		strcpy(msgBuf, "signal SIGINT received.");
	}

	return 0;
}
```

**【相关软件CWE编号】** CWE-662，CWE-828

## 禁用rand函数产生用于安全用途的伪随机数

**【描述】** 
C语言标准库rand()函数生成的是伪随机数，所以不能保证其产生的随机数序列质量。所以禁止使用rand()函数产生的随机数用于安全用途，必须使用安全的随机数产生方式，如： /dev/random文件。

典型的安全用途场景包括(但不限于)以下几种：

-   会话标识SessionID的生成；

-   挑战算法中的随机数生成；

-   验证码的随机数生成；

-   用于密码算法用途（例如用于生成IV、盐值、密钥等）的随机数生成。

**【错误代码示例】** 
程序员期望生成一个唯一的不可被猜测的HTTP会话ID，但该ID是通过调用rand()函数产生的数字随机数，它的ID是可猜测的，并且随机性有限。

**【正确代码示例】(POSIX)** 
可以使用/dev/random文件得到随机数。

**【影响】**

使用rand()函数可能造成可预测的随机数。

# 内核操作

## 内核mmap接口实现中，确保对映射起始地址和大小进行合法性校验

**【描述】**

**说明：**Linux内核 mmap接口中，经常使用remap_pfn_range()函数将设备物理内存映射到用户进程空间。如果映射起始地址等参数由用户态控制并缺少合法性校验，将导致用户态可通过映射读写任意内核地址。如果攻击者精心构造传入参数，甚至可在内核中执行任意代码。

**【错误代码示例】**

如下代码在使用remap_pfn_range()进行内存映射时，未对用户可控的映射起始地址和空间大小进行合法性校验，可导致内核崩溃或任意代码执行。

```
static int incorrect_mmap(struct file *file, struct vm_area_struct *vma)
{
	unsigned long size;
	size = vma->vm_end - vma->vm_start;
	vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
	//错误：未对映射起始地址、空间大小做合法性校验
	if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff, size, vma->vm_page_prot)) { 
		err_log("%s, remap_pfn_range fail", __func__);
		return EFAULT;
	} else {
		vma->vm_flags &=  ~VM_IO;
	}

	return EOK;
}
```

**【正确代码示例】**

增加对映射起始地址等参数的合法性校验。

```
static int correct_mmap(struct file *file, struct vm_area_struct *vma)
{
	unsigned long size;
	size = vma->vm_end - vma->vm_start;
	//修改：添加校验函数，验证映射起始地址、空间大小是否合法
	if (!valid_mmap_phys_addr_range(vma->vm_pgoff, size)) { 
		return EINVAL;
	}

	vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
	if (remap_pfn_range(vma, vma->vm_start, vma->vm_pgoff, size, vma->vm_page_prot)) {
		err_log( "%s, remap_pfn_range fail ", __func__);
		return EFAULT;
	} else {
		vma->vm_flags &=  ~VM_IO;
	}

	return EOK;
}
```

## 内核程序中必须使用内核专用函数读写用户态缓冲区

**【描述】**

用户态与内核态之间进行数据交换时，如果在内核中不加任何校验（如校验地址范围、空指针）而直接引用用户态传入指针，当用户态传入非法指针时，可导致内核崩溃、任意地址读写等问题。因此，应当禁止使用memcpy()、sprintf()等危险函数，而是使用内核提供的专用函数：copy_from_user()、copy_to_user()、put_user()和get_user()来读写用户态缓冲区，这些函数内部添加了入参校验功能。

所有禁用函数列表为：memcpy()、bcopy()、memmove()、strcpy()、strncpy()、strcat()、strncat()、sprintf()、vsprintf()、snprintf()、vsnprintf()、sscanf()、vsscanf()。

**【错误代码示例】**

内核态直接使用用户态传入的buf指针作为snprintf()的参数，当buf为NULL时，可导致内核崩溃。

```
ssize_t incorrect_show(struct file *file, char__user *buf, size_t size, loff_t *data)
{
	// 错误：直接引用用户态传入指针，如果buf为NULL，则空指针异常导致内核崩溃
	return snprintf(buf, size, "%ld\n", debug_level); 
}
```

**【正确代码示例】**

使用copy_to_user()函数代替snprintf()。

```
ssize_t correct_show(struct file *file, char __user *buf, size_t size, loff_t *data)
{
	int ret = 0;
	char level_str[MAX_STR_LEN] = {0};
	snprintf(level_str, MAX_STR_LEN, "%ld \n", debug_level);
	if(strlen(level_str) >= size) {
		return EFAULT;
	}
	
	// 修改：使用专用函数copy_to_user()将数据写入到用户态buf，并注意防止缓冲区溢出
	ret = copy_to_user(buf, level_str, strlen(level_str)+1); 
	return ret;
}
```

**【错误代码示例】**

内核态直接使用用户态传入的指针user_buf作为数据源进行memcpy()操作，当user_buf为NULL时，可导致内核崩溃。

```
size_t incorrect_write(struct file  *file, const char __user  *user_buf, size_t count, loff_t  *ppos)
{
	...
	char buf [128] = {0};
	int buf_size = 0;
	buf_size = min(count, (sizeof(buf)-1));
	// 错误：直接引用用户态传入指针，如果user_buf为NULL，则可导致内核崩溃
	(void)memcpy(buf, user_buf, buf_size); 
	...
}
```

**【正确代码示例】**

使用copy_from_user()函数代替memcpy()。

```
ssize_t correct_write(struct file *file, const char __user *user_buf, size_t count, loff_t *ppos)
{
	...
	char buf[128] = {0};
	int buf_size = 0;

	buf_size = min(count, (sizeof(buf)-1));
	// 修改：使用专用函数copy_from_user()将数据写入到内核态buf，并注意防止缓冲区溢出
	if (copy_from_user(buf, user_buf, buf_size)) { 
		return EFAULT;
	}

	...
}
```

## 必须对copy_from_user()拷贝长度进行校验，防止缓冲区溢出

**说明：**内核态从用户态拷贝数据时通常使用copy_from_user()函数，如果未对拷贝长度做校验或者校验不当，会造成内核缓冲区溢出，导致内核panic或提权。

**【错误代码示例】**

未校验拷贝长度。

```
static long gser_ioctl(struct file  *fp, unsigned cmd, unsigned long arg)
{
	char smd_write_buf[GSERIAL_BUF_LEN];
	switch (cmd)
	{
		case GSERIAL_SMD_WRITE:
			if (copy_from_user(&smd_write_arg, argp, sizeof(smd_write_arg))) {...}
			// 错误：拷贝长度参数smd_write_arg.size由用户输入，未校验
			copy_from_user(smd_write_buf, smd_write_arg.buf, smd_write_arg.size); 
			...
	}
}
```

**【正确代码示例】**

添加长度校验。

```
static long gser_ioctl(struct file *fp, unsigned cmd, unsigned long arg)
{
	char smd_write_buf[GSERIAL_BUF_LEN];
	switch (cmd)
	{
		case GSERIAL_SMD_WRITE:
			if (copy_from_user(&smd_write_arg, argp, sizeof(smd_write_arg))){...}
			// 修改：添加校验
			if (smd_write_arg.size  >= GSERIAL_BUF_LEN) {......} 
			copy_from_user(smd_write_buf, smd_write_arg.buf, smd_write_arg.size);
 			...
	}
}
```

## 必须对copy_to_user()拷贝的数据进行初始化，防止信息泄漏

**【描述】**

**说明：**内核态使用copy_to_user()向用户态拷贝数据时，当数据未完全初始化（如结构体成员未赋值、字节对齐引起的内存空洞等），会导致栈上指针等敏感信息泄漏。攻击者可利用绕过kaslr等安全机制。

**【错误代码示例】**

未完全初始化数据结构成员。

```
static long rmnet_ctrl_ioctl(struct file *fp, unsigned cmd, unsigned long arg)
{
	struct ep_info info;
	switch (cmd) {
		case FRMNET_CTRL_EP_LOOKUP:
			info.ph_ep_info.ep_type = DATA_EP_TYPE_HSUSB;
			info.ipa_ep_pair.cons_pipe_num = port->ipa_cons_idx;
			info.ipa_ep_pair.prod_pipe_num = port->ipa_prod_idx;
			// 错误: info结构体有4个成员，未全部赋值
			ret = copy_to_user((void __user *)arg, &info, sizeof(info)); 
			...
	}
}
```

**【正确代码示例】**

全部进行初始化。

```
static long rmnet_ctrl_ioctl(struct file *fp, unsigned cmd, unsigned long arg)
{
	struct ep_info info;
	// 修改：使用memset初始化缓冲区，保证不存在因字节对齐或未赋值导致的内存空洞
	(void)memset(&info, '0', sizeof(ep_info)); 
	switch (cmd) {
		case FRMNET_CTRL_EP_LOOKUP:
			info.ph_ep_info.ep_type = DATA_EP_TYPE_HSUSB;
			info.ipa_ep_pair.cons_pipe_num = port->ipa_cons_idx;
			info.ipa_ep_pair.prod_pipe_num = port->ipa_prod_idx;
			ret = copy_to_user((void __user *)arg, &info, sizeof(info));
			...
	}
}
```

## 禁止在异常处理中使用BUG_ON宏，避免造成内核panic

**【描述】**

BUG_ON宏会调用内核的panic()函数，打印错误信息并主动崩溃系统，在正常逻辑处理中（如ioctl接口的cmd参数不识别）不应当使系统崩溃，禁止在此类异常处理场景中使用BUG_ON宏，推荐使用WARN_ON宏。

**【错误代码示例】**

正常流程中使用了BUG_ON宏

```
/ * 判断Q6侧设置定时器是否繁忙，1-忙，0-不忙 */
static unsigned int is_modem_set_timer_busy(special_timer *smem_ptr)
{
	int i = 0;
	if (smem_ptr == NULL) {
		printk(KERN_EMERG"%s:smem_ptr NULL!\n", __FUNCTION__);
		// 错误：系统BUG_ON宏打印调用栈后调用panic()，导致内核拒绝服务，不应在正常流程中使用
		BUG_ON(1); 
		return 1;
	}

	...
}
```

**【正确代码示例】**

去掉BUG_ON宏。

```
/ * 判断Q6侧设置定时器是否繁忙，1-忙，0-不忙  */
static unsigned int is_modem_set_timer_busy(special_timer *smem_ptr)
{
	int i = 0;
	if (smem_ptr == NULL) {
		printk(KERN_EMERG"%s:smem_ptr NULL!\n",  __FUNCTION__);
		// 修改：去掉BUG_ON调用，或使用WARN_ON
		return 1;
	}

	...
}
```

## 在中断处理程序或持有自旋锁的进程上下文代码中，禁止使用会引起进程休眠的函数

**【描述】**

Linux以进程为调度单位，在Linux中断上下文中，只有更高优先级的中断才能将其打断，系统在中断处理的时候不能进行进程调度。如果中断处理程序处于休眠状态，就会导致内核无法唤醒，从而使得内核处于瘫痪。

自旋锁在使用时，抢占是失效的。若自旋锁在锁住以后进入睡眠，由于不能进行处理器抢占，其它进程都将因为不能获得CPU（单核CPU）而停止运行，对外表现为系统将不作任何响应，出现挂死。

因此，在中断处理程序或持有自旋锁的进程上下文代码中，应该禁止使用可能会引起休眠（如vmalloc()、msleep()等）、阻塞（如copy_from_user(),copy_to_user()等）或者耗费大量时间（如printk()等）的函数。

## 合理使用内核栈，防止内核栈溢出

**【描述】**

Linux的内核栈大小是固定的（一般32位系统为8K，64位系统为16K，因此资源非常宝贵。不合理的使用内核栈，可能会导致栈溢出，造成系统挂死。因此需要做到以下几点：

-   在栈上申请内存空间不要超过内核栈大小；

-   注意函数的嵌套使用次数；

-   不要定义过多的变量。

**【错误代码示例】**

以下代码中定义的变量过大，导致栈溢出。

```
...
struct result
{
	char name[4];
	unsigned int a;
	unsigned int b;
	unsigned int c;
	unsigned int d;
}; // 结构体result的大小为20字节

int foo()
{
	struct result temp[512];
	// 错误: temp数组含有512个元素，总大小为10K，远超内核栈大小
	(void)memset(temp, 0, sizeof(result) * 512); 
	... // use temp do something
	return 0;
}

...
```

代码中数组temp有512个元素，总共10K大小，远超内核的8K，明显的栈溢出。

**【正确代码示例】**

使用kmalloc()代替之。

```
...
struct result
{
	char name[4];
	unsigned int a;
	unsigned int b;
	unsigned int c;
	unsigned int d;
}; // 结构体result的大小为20字节

int foo()
{
	struct result  *temp = NULL;
	temp = (result *)kmalloc(sizeof(result) * 512, GFP_KERNEL); //修改：使用kmalloc()申请内存
	... // check temp is not NULL
	(void)memset(temp, 0, sizeof(result)  * 512);
	... // use temp do something
	... // free temp
	return 0;
}
...
```

## 临时关闭地址校验机制后，在操作完成后必须及时恢复

**【描述】**

SMEP安全机制是指禁止内核执行用户空间的代码（PXN是ARM版本的SMEP）。系统调用（如open()，write()等）本来是提供给用户空间程序访问的。默认情况下，这些函数会对传入的参数地址进行校验，如果入参是非用户空间地址则报错。因此，要在内核程序中使用这些系统调用，就必须使参数地址校验功能失效。set_fs()/get_fs()就用来解决该问题。详细说明见如下代码：

```
...
mmegment_t old_fs;
printk("Hello, I'm the module that intends to write message to file.\n");
if (file == NULL) {
	file = filp_open(MY_FILE, O_RDWR | O_APPEND | O_CREAT, 0664);
}

if (IS_ERR(file)) {
	printk("Error occured while opening file %s, exiting ...\n", MY_FILE);
	return 0;
}

sprintf(buf, "%s", "The Message.");
old_fs = get_fs(); // get_fs()的作用是获取用户空间地址上限值  
                   // #define get_fs() (current->addr_limit
set_fs(KERNEL_DS); // set_fs的作用是将地址空间上限扩大到KERNEL_DS，这样内核代码可以调用系统函数
file->f_op->write(file, (char *)buf, sizeof(buf), &file->f_pos); // 内核代码可以调用write()函数
set_fs(old_fs); // 使用完后及时恢复原来用户空间地址限制值
...
```

通过上述代码，可以了解到最为关键的就是操作完成后，要及时恢复地址校验功能。否则SMEP/PXN安全机制就会失效，使得许多漏洞的利用变得很容易。

**【错误代码示例】**

在程序错误处理分支，未通过set_fs()恢复地址校验功能。

```
...
oldfs = get_fs();
set_fs(KERNEL_DS);
/* 在时间戳目录下面创建done文件 */
fd = sys_open(path, O_CREAT | O_WRONLY, FILE_LIMIT);
if (fd < 0) {
	BB_PRINT_ERR("sys_mkdir[%s] error, fd is[%d]\n", path, fd);
	return; // 错误：在错误处理程序分支未恢复地址校验机制
}

sys_close(fd);
set_fs(oldfs);
...
```

**【正确代码示例】**

在错误处理程序中恢复地址校验功能。

```
...
oldfs = get_fs();
set_fs(KERNEL_DS);

/* 在时间戳目录下面创建done文件 */
fd = sys_open(path, O_CREAT | O_WRONLY, FILE_LIMIT);
if (fd < 0) {
	BB_PRINT_ERR("sys_mkdir[%s] error, fd is[%d] \n", path, fd);
	set_fs(oldfs); // 修改：在错误处理程序分支中恢复地址校验机制
	return;
}

sys_close(fd);
set_fs(oldfs);
...
```
