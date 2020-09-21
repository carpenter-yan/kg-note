JavaScript 是一种专为与网页交互而设计的脚本语言，由下列三个不同的部分组成：
- ECMAScript，由ECMA-262 定义，提供核心语言功能；
- 文档对象模型（DOM），提供访问和操作网页内容的方法和接口；
- 浏览器对象模型（BOM），提供与浏览器交互的方法和接口。

把JavaScript 插入到HTML 页面中要使用\<script\>元素。使用这个元素可以把JavaScript 嵌入到
HTML 页面中，让脚本与标记混合在一起；也可以包含外部的JavaScript 文件。而我们需要注意的地方有：
- 在包含外部JavaScript 文件时，必须将src 属性设置为指向相应文件的URL。而这个文件既可
以是与包含它的页面位于同一个服务器上的文件，也可以是其他任何域中的文件。
- 所有\<script\>元素都会按照它们在页面中出现的先后顺序依次被解析。在不使用defer 和
async 属性的情况下，只有在解析完前面\<script\>元素中的代码之后，才会开始解析后面
\<script\>元素中的代码。
- 由于浏览器会先解析完不使用defer 属性的\<script\>元素中的代码，然后再解析后面的内容，
所以一般应该把\<script\>元素放在页面最后，即主要内容后面，\</body\>标签前面。
- 使用defer 属性可以让脚本在文档完全呈现之后再执行。延迟脚本总是按照指定它们的顺序执行。
- 使用async 属性可以表示当前脚本不必等待其他脚本，也不必阻塞文档呈现。不能保证异步脚
本按照它们在页面中出现的顺序执行。
另外，使用\<noscript\>元素可以指定在不支持脚本的浏览器中显示的替代内容。但在启用了脚本
的情况下，浏览器不会显示\<noscript\>元素中的任何内容。

for-in语句
for-in 语句是一种精准的迭代语句，可以用来枚举对象的属性。以下是for-in 语句的语法：
for (property in expression) statement
下面是一个示例：
for (var propName in window) {
document.write(propName);
}

label语句
使用label 语句可以在代码中添加标签，以便将来使用。以下是label 语句的语法：
label: statement
下面是一个示例：
start: for (var i=0; i < count; i++) {
alert(i);
}

with 语句的作用是将代码的作用域设置到一个特定的对象中。with 语句的语法如下：
with (expression) statement;
定义with 语句的目的主要是为了简化多次编写同一个对象的工作，如下面的例子所示：
var qs = location.search.substring(1);
var hostName = location.hostname;
var url = location.href;
上面几行代码都包含location 对象。如果使用with 语句，可以把上面的代码改写成如下所示：
with(location){
var qs = search.substring(1);
var hostName = hostname;
var url = href;
}

函数
函数对任何语言来说都是一个核心的概念。通过函数可以封装任意多条语句，而且可以在任何地方、
任何时候调用执行。ECMAScript 中的函数使用function 关键字来声明，后跟一组参数以及函数体。
函数的基本语法如下所示：
function functionName(arg0, arg1,...,argN) {
statements
}
以下是一个函数示例：
function sayHi(name, message) {
alert("Hello " + name + "," + message);
}