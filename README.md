# ID生成器

生成带校验码的卡号、19位的Long ID、不大于22位的短UUID、短卡号、激活码、数字加密、付款码。生成器是分布式，无需数据库、redis或者zk作为ID分配的key。这样的好处是ID分配无需RPC调用，都是内存计算，每秒可以分配几十万的ID。

为保证兼容性，代码基于Java7，只依赖slf4j-api。

# Features

* [安全激活码](#安全激活码)
* [带店铺编号的卡号](#带店铺编号的卡号)
* [店铺编号卡号激活码](#店铺编号卡号激活码)
* [带系统编号的卡号](#带系统编号的卡号)
* [短卡号](#短卡号)
* [Long类型的ID](#Long类型的ID)
* [短UUID](#Short UUID)
* [数字加密](#数字加密)
* [混入时间信息的数字加密](#混入时间信息的数字加密)

# 安全激活码

该激活码无需密码，凭码就可以直接激活消费。

## 说明

1. 激活码固定16位，全大写字母和数字，排除掉易混字符0O、1I，一共32个字符。

2. 激活码本质上是一个16*5=80bit的正整数，通过一定的编码规则转换成全大写字符和数字。

3. 为了安全，使用者在创建生成器的时候，需要提供32套随机编码规则，以字符A来说，可能在“KMLVAPPGRABH”激活码中代表数字4，在"MONXCRRIUNVA"激活码中代表数字23。即每个字符都可以代表0-31的任一数字。

4. 具体使用何种编码规则，是通过卡号进行ChaCha20加密后的随机数hash决定的。

激活码的正整数由80bit组成

+========================================================  
| 5bit编码号 | 30bit序号明文 | 45bit序号、店铺编号生成的密文  |   
+========================================================  

## 激活码生成流程

1. 输入参数为店铺编号、卡号、序号

2. 用ChaCha20算法对序号加密，得到一个512字节的随机数

3. 将步骤2生成的随机数取前256字节作为HMAC算法的密钥

4. 将序号、店铺编号、步骤2生成的随机数的后256字节拼成字节数组

5. 用步骤3生成的HMAC对步骤4生成的字节数组进行加密

6. 将店铺编号编码为27bit，步骤5生成的字节数组取前18bit，拼成45bit报文

7. 步骤4生成的字节数组取前45bit报文M1，步骤6生成的45bit报文M2，将M1和M2进行异或运算

8. 根据序号得到30bit的明文，步骤7得到45bit密文，将明文和密文拼接成75bit的激活码主体

9. 用ChaCha20算法对卡号进行加密，得到的随机数按字节求和，然后对32取模

10. 根据步骤9的结果，得到一套base32的编码方式，对步骤8产生的75bit激活码主体进行编码，得到15位的32进制数（大写字母和数字，排除掉0O1I）

11. 步骤9得到的结果进行base32编码得到一位32进制数

12. 将步骤11和步骤10得到的结果拼在一起，得到16位的激活码

## 激活码验证流程

和生成流程相反。

## 校验

如果对激活码进行暴力破解，校验通过的概率为0。

即使激活码生成算法暴露了，要破解一个激活码需要进行2的45次方次尝试，如果是一半概率的话也要2的44次方次。

# 带店铺编号的卡号

## 说明

卡号固定为16位，为53bit。

+======================================================================  
| 4bit店铺编号校验位 | 30bit时间戳 | 3bit机器编号  | 9bit序号 | 7bit卡号校验位 |      
+======================================================================

* 30 bit的秒时间戳支持34年
* 9 bit序号支持512个序号
* 3 bit机器编号支持8台负载

即卡号生成最大支持8台负载，每台负载每秒钟可以生成512个卡号。

时间戳、机器编号、序号和校验位的bit位数支持业务自定义，方便业务定制自己的生成器。

## 校验

如果对卡号进行暴力破解，卡号校验通过的概率基本为0。

测试代码随机生成1000万16位的卡号，多次测试校验均没有任何卡号通过校验。

## 示例

```java

private ShopCardIdGenerator cardIdGenerator = new ShopCardIdGenerator(1);
Long id = cardIdGenerator.generate("A00001");
Assert.assertTrue(cardIdGenerator.validate("A00001", id));

```

# 店铺编号卡号激活码

## 说明

激活码有如下特点：

1. 激活码固定12位，全大写字母。
2. 激活码生成时植入关联的卡号的Hash，但是不可逆；即无法从激活码解析出卡号，也无法从卡号解析出激活码。
3. 激活码本质上是一个正整数，通过一定的编码规则转换成全大写字符。为了安全，生成器使用26套编码规则，以字符A来说，
可能在“KMLVAPPGRABH”激活码中代表数字4，在"MONXCRRIUNVA"激活码中代表数字23。即每个大写字符都可以代表0-25的任一数字。
4. 具体使用何种编码规则，是通过时间戳+店铺编号Hash决定的。
5. 校验激活码分为两个步骤。（1）、 首先校验激活码的合法性 （2）1校验通过后，从数据库查询出关联的卡号，对卡号和激活码的关系做二次校验

激活码的正整数由51bit组成

+==============================================================================================   
| 4bit店铺编号校验位 | 29bit时间戳 | 3bit机器编号  | 7bit序号 | 4bit激活码校验位 | 4bit卡号校验位 |    
+==============================================================================================  

* 29 bit的秒时间戳支持17年，激活码生成器计时从2017年开始，可以使用到2034年
* 7 bit序号支持128个序号
* 3 bit机器编号支持8台负载

即激活码生成最大支持8台负载，每台负载每秒钟可以生成128个激活码，整个系统1秒钟可以生成1024个激活码

时间戳、机器编号、序号和校验位的bit位数支持业务自定义，方便业务定制自己的生成器。

## 校验

如果对激活码进行暴力破解，激活码校验通过的概率基本为0。

测试代码随机生成1000万激活码，多次测试校验均没有任何激活码通过校验。

## 示例

### 生成激活码：

```java

ActivationCodeGenerator codeGenerator = new ActivationCodeGenerator();
Long cardId = 2285812209233540L;
String code = codeGenerator.generate("A00001", cardId);
Assert.assertTrue(cardIdGenerator.validate("A00001", id));

```

### 校验激活码

```java

ActivationCodeGenerator codeGenerator = new ActivationCodeGenerator();
String code = "IMJPVQREBMCO";
//首先校验激活码是否正确
Assert.assertTrue(codeGenerator.validate("A00001", code));

//校验通过后从数据库查询出关联的卡号
Long cardId = 2285812507541518;
//对卡号做二次校验
Assert.assertTrue(codeGenerator.validateCardId(code, cardId));

```

# 带系统编号的卡号

## 说明

卡号固定为16位，53bit。 目的是支持不同卡的类型。

+=======================================================================  
| 3bit卡类型 | 31bit时间戳 | 3bit机器编号  | 9bit序号 | 7bit卡号校验位 |      
+=======================================================================

* 31 bit的秒时间戳支持68年
* 9 bit序号支持512个序号
* 3 bit机器编号支持8台负载

即卡号生成最大支持8台负载，每台负载每秒钟可以生成512个卡号。

时间戳、机器编号、序号和校验位的bit位数支持业务自定义，方便业务定制自己的生成器。

## 校验

如果对卡号进行暴力破解，卡号校验通过的概率基本为0。

测试代码随机生成1000万16位的卡号，多次测试校验均没有任何卡号通过校验。

## 示例

```java

private CardIdGenerator cardIdGenerator = new CardIdGenerator(1, 2);
Long id = cardIdGenerator.generate();
Assert.assertTrue(cardIdGenerator.validate(id));

```

# 短卡号

## 说明

卡号固定为13位，43bit。

+=======================================================  
| 3bit机器编号 | 29bit时间戳  | 8bit序号 | 3bit卡号校验位 |      
+=======================================================

* 29 bit的秒时间戳支持17年
* 8 bit序号支持256个序号（起始序号是20以内的随机数）
* 3 bit机器编号支持7台负载（负载编号从1-7）

即卡号生成最大支持7台负载；每台负载每秒钟可以生成最少236，最多256个卡号。


## 校验

如果对卡号进行暴力破解，卡号校验通过的概率基本为0。

测试代码随机生成1000万13位的卡号，多次测试校验均没有任何卡号通过校验。

## 示例

```java

private ShortCardIdGenerator cardIdGenerator = new ShortCardIdGenerator(2);
Long id = cardIdGenerator.generate();
Assert.assertTrue(cardIdGenerator.validate(id));

```


# Long类型的ID

## 说明

ID固定为19位，64bit。 可用于各种业务系统的ID生成.

+=======================================================================  
| 42bit 毫秒时间戳 | 10bit机器编号  | 12bit序号  |      
+=======================================================================

* 42 bit的毫秒时间戳支持68年
* 12 bit序号支持4096个序号
* 10 bit机器编号支持1024台负载

即ID生成最大支持1024台负载，每台负载每毫秒可以生成4096个ID，这样每台负载每秒可以产生40万ID。

## 示例

```java

private LongIdGenerator generator = new LongIdGenerator(1L);
Long id = generator.generate();
Assert.assertEquals(19, String.valueOf(id).length());

```

# 短UUID

## 说明

UUID最长22位。 排除掉1、l和I，0和o易混字符。

## 示例

```java

private ShortUuidGenerator generator = new ShortUuidGenerator();
String id = generator.generate();

```

# 数字加密

很多场景下为了信息隐蔽需要对数字进行加密，比如用户的手机号码；并且需要支持解密。

本算法支持对不大于12位的正整数（即1000,000,000,000）进行加密，输出固定长度为18位的数字字符串；支持解密。

## 说明

1. 加密字符串固定18位数字，原始待加密正整数不大于12位

2. 加密字符串本质上是一个56bit的正整数，通过一定的编码规则转换而来。

3. 为了安全，使用者在创建生成器的时候，需要提供10套随机编码规则，以数字1来说，可能在“5032478619”编码规则中代表数字8，在"2704168539"编码规则中代表数字4。即每个字符都可以代表0-9的任一数字。

4. 具体使用何种编码规则，是通过原始正整数进行ChaCha20加密后的随机数hash决定的。

5. 为了方便开发者使用，提供了随机生成编码的静态方法。

加密后的数字字符串由编码规则+密文报文体组成，密文由56bit组成，可转化为17位数，编码规则为一位数字:

+====================================================  
| 1位编码规则 | 37bit原始数字 |  19bit原始数字生成的密文  |   
+====================================================

## 加密流程

1. 输入参数为原始正整数

2. 用ChaCha20算法对原始正整数加密，得到一个512字节的随机数

3. 将步骤2生成的随机数取前256字节作为HMAC算法的密钥

4. 将步骤2生成的随机数的后256字节、原始正整数拼成字节数组

5. 用步骤3生成的HMAC对步骤4生成的字节数组进行加密

6. 将原始正整数编码为37bit，步骤5生成的字节数组取前19bit，拼成56bit报文

7. 步骤2得到的随机数按字节求和，然后对9取模加1

8. 根据步骤7的结果，得到一套base10的编码方式，对步骤6产生的56bit激活码主体进行编码，得到17位的10进制数

9. 步骤7得到的结果进行base10编码得到一位10进制数

10. 将步骤9和步骤8得到的结果拼在一起，得到18位的加密字符串

## 解密流程

和加密流程相反。

如果为非法字符串，解密方法则返回null。

## 安全

如果对加密数字进行暴力破解，校验通过的概率为0。

内置10套编码方式，

## 使用方式

### 加密

```java

String alphabetsStr = "5032478619,9108736245,8152079436,2704168539,6240951738,0984127356,6501984723,4670182359,1325048769,0517968243";
NumberHidingGenerator generator = new NumberHidingGenerator("abcdefj11p23710837e]q222rqrqweqe",
                "!@#$&123frwq", 10, alphabetsStr);
generator.generate(1L);
generator.generate(99999999999L);
generator.generate(13816750988L);

```

### 解密

```java

String alphabetsStr = "5032478619,9108736245,8152079436,2704168539,6240951738,0984127356,6501984723,4670182359,1325048769,0517968243";
NumberHidingGenerator generator = new NumberHidingGenerator("abcdefj11p23710837e]q222rqrqweqe",
                "!@#$&123frwq", 10, alphabetsStr);
generator.parse("470652367255781114");
generator.parse("003164009537339359");
generator.parse("741077206095296438");

```

# 混入时间信息的数字加密

很多场景下为了信息隐蔽需要对数字进行加密，比如用户的付款码；并且需要支持解密。

加密结果混入了时间信息，有效时间为1分钟，超过有效期加密结果会失效。

本算法支持对不大于12位的正整数（即1000,000,000,000）混合时间信息进行加密，输出固定长度为20位的数字字符串；支持解密。

## 说明

1. 加密字符串固定20位数字，原始待加密正整数不大于12位

2. 加密字符串本质上是一个63bit的正整数，通过一定的编码规则转换而来。

3. 为了安全，使用者在创建生成器的时候，需要提供10套随机编码规则，以数字1来说，可能在“5032478619”编码规则中代表数字8，在"2704168539"编码规则中代表数字4。即每个字符都可以代表0-9的任一数字。

4. 具体使用何种编码规则，是通过原始正整数进行ChaCha20加密后的随机数hash决定的。

5. 为了方便开发者使用，提供了随机生成编码的静态方法。

加密后的数字字符串由编码规则+密文报文体组成，密文由63bit组成，可转化为19位数，编码规则为一位数字:

+===========================================================================================   
| 1位编码规则 | 37bit原始数字 |  15bit原始数字加当前时间加密生成的密文 |  11bit当天时间分钟信息    |    
+===========================================================================================  

## 加密流程

1. 输入参数为原始正整数

2. 用ChaCha20算法对原始正整数加密，得到一个512字节的随机数

3. 将步骤2生成的随机数取前256字节作为HMAC算法的密钥

4. 将步骤2生成的随机数的后256字节、原始正整数、当前时间序列拼成字节数组

5. 用步骤3生成的HMAC对步骤4生成的字节数组进行加密

6. 将原始正整数编码为37bit，步骤5生成的字节数组取前15bit，当天时间分钟信息编码为11bit，拼成63bit报文

7. 步骤2得到的随机数按字节在求和，然后对9取模加1

8. 根据步骤7的结果，得到一套base10的编码方式，对步骤6产生的56bit激活码主体进行编码，得到19位的10进制数

9. 步骤7得到的结果进行base10编码得到一位10进制数

10. 将步骤9和步骤8得到的结果拼在一起，得到20位的加密字符串

## 解密流程

和加密流程相反。

如果为非法字符串或者已经过期，解密方法则返回null。

## 安全

如果对加密数字进行暴力破解，校验通过的概率为0。

内置10套编码方式，

## 使用方式

### 加密

```java

String alphabetsStr = "5032478619,9108736245,8152079436,2704168539,6240951738,0984127356,6501984723,4670182359,1325048769,0517968243";
TimeNumberHidingGenerator generator = new TimeNumberHidingGenerator("abcdefj11p23710837e]q222rqrqweqe",
                "!@#$&123frwq", 10, alphabetsStr);
generator.generate(1L);
generator.generate(99999999999L);
generator.generate(13816750988L);

```

### 解密

```java

String alphabetsStr = "0381592647,1270856349,4685109372,3904682157,7316492805,3645927810,1803756249,6153940728,2905437861,7968012435";
TimeNumberHidingGenerator generator = new TimeNumberHidingGenerator("abcdefj11p23710837e]q222rqrqweqe",
                "!@#$&123frwq", 10, alphabetsStr);
generator.parse("66889894210875624948");
generator.parse("31998985426988503427");
generator.parse("93009092167094930096");

```