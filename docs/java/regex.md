##java正则

### 1. 合法字符
x
\0mn
\xhh
\uhhhh
\t
\n
\r
\f
\a
\e
\cx

### 2. 特殊字符
$
^
()
[]
{}
*
+
?
.
\
|

### 3. 预定义字符
.
\d 0-9所有数字
\D 非数字
\s 所有空白字符，包括空格、制表符、回车符、换行符、换页符等
\S 所有非空白字符
\w 所有单词字符，包括0-9所有数字，26个英文字母和下划线(_)
\W 所有非单子字符

### 4. 方括号表达式
表示枚举       [abc]表示a、b、c中任意一个字符
表示范围:-     [a-f]表示a-f范围内任意字符
表示求否       [^abc]表示非a、b、c的任意字符
表示与运算:&&   [a-z&&[def]]求a-z和[def]的交集，表示d、e、f
表示并运算     [a-d[m-p]]表示[a-dm-p]

### 5. 圆括号表达式
"((public)|(protected)|(private))" 匹配java中3个控制符之一

### 6.边界匹配符
^  行开头
$  行结尾
\b 单词边界
\B 非单词边界
\A 输入开头
\G 前一个匹配的结尾
\Z 输入的结尾，仅用于最后的结束符
\z 输入的结尾

### 大括号表达式
贪婪模式：一直匹配，直到无法匹配
勉强模式：只匹配最少的字符
贪婪    勉强
X?     X??
X*     X*?
X+     X+?
X{n}   X{n}?
X{n,}  X{n}?
X{n,m} X{n,m}?

### Java正则类使用
```
    @Test
    public void test4() {
        Matcher m = Pattern.compile("\\w+").matcher("Java is very easy");
        while (m.find()) {
            System.out.println(m.group() + " begin at:" + m.start() + " end at:" + m.end());
        }
    }
    
    @Test
    public void test5() {
        String[] msgs = {"Java has regular expression in 1.4",
                "regular expression now expressing in Java",
                "Java represses oracular expressions"};
        Pattern p = Pattern.compile("re\\w*");
        Matcher m = null;
        for (int i = 0; i < msgs.length; i++) {
            if (m == null) {
                m = p.matcher(msgs[i]);
            } else {
                m.reset(msgs[i]);
            }
            System.out.println(m.replaceAll("哈哈:)"));
        }
    }
```