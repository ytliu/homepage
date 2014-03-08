---
layout: page
title: "Ruby学习笔记"
date: 2013-05-26 19:50
comments: true
sharing: true
footer: true
---

想记录下学习Ruby过程中的一些笔记和想法，也就给自己看看，因为感觉写在这里比较能迫使自己坚持下去吧~

------

#### 关于数据类型

##### String

* \`ls\`和%x[ls]是同样的效果，都是执行`ls`命令
* 在Ruby1.8, ?A = 65，而在Ruby1.9，?A = 'A'
* String Interpolation: `"Hello " + i.to_s`和`"Helle #{i}"`表示同样的效果
* 在这个场景中：
{% codeblock lang:ruby %}
a = 0
"#{a=a+1} * 3"
{% endcodeblock %}
得到的结果是“1 1 1”，因为在这里`a=a+1`只执行一次

###### 关于Iterating Strings：

* 在Ruby1.8中，可以用each方法来遍历String，而且用each_byte和用[]来访问速度不会差太多，因为在1.8中，随机访问和顺序访问基本是一样快的；
* 在Ruby1.9中，each由三个方法替代了：each_byte, each_char, each_line，而且顺序访问效率大大优于随机访问，所以如果要一个个访问String的每个字节，用each_char会比[]明显快很多；
* 另外在Ruby1.9中提供了multibyte的支持，String里面的char不仅仅是指ASCII character了，方法length/size返回的是char的数量，而bytesize才是返回byte的数量：
{% codeblock lang:ruby %}
#-*-coding:utf-8-*- #Specify Unicode UTF-8 characters
# This is a string literal containing a multibyte multiplication character 
s = "2×2=4"
# The string contains 6 bytes which encode 5 characters 
s.length #=>5:Characters: '2' '×' '2' '=' '4' 
s.bytesize # => 6: Bytes (hex): 32 c3 97 32 3d 34
{% endcodeblock %}

##### Array

* %w[this is a test] => ['this', 'is', 'a', 'test']，类似与String的`%q`和`%Q`
* Array.new(3) => [nil, nil, nil]
* Array.new(4, 0) => [0, 0, 0, 0]
* Array.new(3) {|i| i+1} => [1, 2, 3]

##### Object

* instance_of? vs. is_a?(kind_of?)，和前者相比，后者还包括判断该obj是否是某个类的子类；
* respond_to?，该方法判断一个obj是否有某种方法，缺点在于它只关注方法名，而并不在意该方法的参数：
{% codeblock lang:ruby %}
o.respond_to?:"<<" #true if o has an << operator
{% endcodeblock %}

###### Object Equality

* The **equal?** method：判断两个obj是否是同一个obj（即同一个reference），类似于`a.object_id == b.object_id`;
* The **==** operator：类似于`equal?`，但是大部分类都会重新定义该方法，判断两个obj是否具有相同的值（比如Array, Hash，Numeric，String...）；
* The **eql?** method：同样类似于`==`，但是大部分类重新定义它来表示一个严格意义上的(strict version of)`==`，没有自动的类型转换：
{% codeblock lang:ruby %}
1 == 1.0 # true: Fixnum and Float objects can be == 
1.eql?(1.0) # false: but they are never eql!
{% endcodeblock %}
* The **===** operator：“case equality"操作符，主要用在case语句中，当然也有一些不是那么常见的用法：
{% codeblock lang:ruby %}
(1..10) === 5 # true: 5 is in the range 1..10
/\d+/ === "123" #true:the string matches the regular expression 
String === "s" #true:"s" is an instance of the class String 
:s === "s" # true in Ruby 1.9
{% endcodeblock %}

###### Copying，Marshaling，Freezing and Tainting Object

* clone和dup：返回一个shallow copy，即如果一个object里面有指向其它object的reference的话，那么只拷贝那个reference。
* 可以用Marshal.dump和Marshal.load来做深度拷贝：
{% codeblock lang:ruby %}
def deepcopy(o) 
	Marshal.load(Marshal.dump(o))
end
{% endcodeblock %}
* freeze一个object意味着这个obj之后都不可以被修改，调用它的任何一个mutator方法都会失败，而且不存在一个unfreeze的操作（即不能解除对一个obj的freeze）；另外需要注意的是，freeze的操作对象是object reference，而不是变量，也就是说任何一个可以产生新的object的操作都是可行的：
{% codeblock lang:ruby %}
str = 'Original string - '
str.freeze
str += 'attachment'
puts str # Output is - Original string - attachment 
{% endcodeblock %}
在这里表达式`str + 'attachment'`会产生一个新的对象，并被赋值给`str`，此时`str`已经不再是frozen了，而原来`str`指向的对象在不久后就会被垃圾回收掉。（可以参见[这里](http://rubylearning.com/satishtalim/mutable_and_immutable_objects.html))

* taint一个object意味着如果其它obj是从一个tainted的obj来的话，那么它也会被taint；
* clone和dup的区别：clone会把taint和frozen的状态都传递，而dup只会传递taint的状态，frozen的状态不会被传递。

------

#### 关于API的使用

##### String.gsub

* APIs: 
{% codeblock lang:ruby %}
gsub(pattern, replacement) → new_str click to toggle source
gsub(pattern, hash) → new_str
gsub(pattern) {|match| block } → new_str
gsub(pattern) → enumerator
{% endcodeblock %}	
* `pattern`一般是一个正则表达式，如果是一个String的话，那么会将`\\d`译成`\d`
* 如果`replacement`是一个String，那么匹配的文本会被替代为它，同时它也可以通过`\d`来表示group number，通过`\k<n>`里的`n`来表示group name：
{% codeblock lang:ruby %}
"hello".gsub(/[aeiou]/, '*')                  #=> "h*ll*"
"hello".gsub(/([aeiou])/, '<\1>')             #=> "h<e>ll<o>"
"hello".gsub(/(?<foo>[aeiou])/, '{\k<foo>}')  #=> "h{e}ll{o}"
{% endcodeblock %}
这里，`<\1>`表示的是第一个通过`()`括起来（captures）的变量，叫做capture group number，当然也有`<\2>`、`<\3>`等等，而`?<foo>`则表示把匹配这个括号中的正则表达式的文本作为一个局部变量，供后面使用，叫做capture group name;

* 如果`replacement`是一个Hash，则可以将匹配它的key的文本替换成相对应的value：
{% codeblock lang:ruby %}
'hello'.gsub(/[eo]/, 'e' => 3, 'o' => '*')    #=> "h3ll*"
{% endcodeblock %}	
* 对于第三、第四种block的情况，所有被匹配到的文本会作为block的参数，而block的返回值则会替换当前作为参数的被匹配的文本：
{% codeblock lang:ruby %}
"hello".gsub(/./) {|s| s.ord.to_s + ' '}      #=> "104 101 108 108 111 "
{% endcodeblock %}	

##### Array.uniq & uniq!

* APIs:
{% codeblock lang:ruby %}
uniq → new_ary click to toggle source
uniq { |item| ... } → new_ary
uniq! → ary or nil click to toggle source
uniq! { |item| ... } → ary or nil
{% endcodeblock %}
* uniq返回的是消除重复后的array，而uniq!返回的值比较不一样，如果有重复的话，则和uniq返回同样的array，如果没有重复，则返回nil
{% codeblock lang:ruby %}
a = [ "a", "a", "b", "b", "c" ]
a.uniq   # => ["a", "b", "c"]

b = [["student","sam"], ["student","george"], ["teacher","matz"]]
b.uniq { |s| s.first } # => [["student", "sam"], ["teacher", "matz"]]

a = [ "a", "a", "b", "b", "c" ]
a.uniq!   # => ["a", "b", "c"]

b = [ "a", "b", "c" ]
b.uniq!   # => nil
{% endcodeblock %}

##### String.chomp & chop & strip

* APIs
{% codeblock lang:ruby %}
chomp(separator=$/) → new_str
chop → new_str
strip → new_str
{% endcodeblock %}

其中，如果没有定义`$/`的话，**chomp**是把如`\r`、`\n`、`\r\n`等去除，**chop**就是去除最后一个字母，**strip**是把前后的空格都去掉，还有相关的**lstrip**和**rstrip**

* Examples
{% codeblock lang:ruby %}
"hello".chomp            #=> "hello"
"hello\n".chomp          #=> "hello"
"hello\r\n".chomp        #=> "hello"
"hello\n\r".chomp        #=> "hello\n"
"hello\r".chomp          #=> "hello"
"hello \n there".chomp   #=> "hello \n there"
"hello".chomp("llo")     #=> "he"

"string\r\n".chop   #=> "string"
"string\n\r".chop   #=> "string\n"
"string\n".chop     #=> "string"
"string".chop       #=> "strin"
"x".chop.chop       #=> ""

"    hello    ".strip   #=> "hello"
"\tgoodbye\r\n".strip   #=> "goodbye"
{% endcodeblock %}
------

#### 关于语法和语句

* 在一个循环中有6种方法可以打断当前正在执行的语句：break，next，redo，return，retry，throw/catch

* rescue with Method, Class and Module definition:

{% codeblock lang:ruby %}
def method_name(x)
# The body of the method goes here.
# Usually, the method body runs to completion without exceptions # and returns to its caller normally.
rescue
# Exception-handling code goes here.
# If an exception is raised within the body of the method, or if # one of the methods it calls raises an exception, then control # jumps to this block.
else
# If no exceptions occur in the body of the method # then the code in this clause is executed.
ensure
# The code in this clause is executed no matter what happens in the
# body of the method. It is run if the method runs to completion, if
# it throws an exception, or if it executes a return statement.
end
{% endcodeblock %}

##### 关于method

Ruby是一个纯OO的语言，定义的所有的方法都是属于某个类的。对于如下的场景，我们在整个文件的最上方定义一个function，包含在所有定义的class之外：
{% codeblock lang:ruby %}
def square(n)
	n * n
end
{% endcodeblock %}	
似乎这个就是一个传统意义上的function，并不像我们所说的method属于哪一个类，但是其实在ruby的实现中，这个function其实会被定义为类Object的一个private method，也就是说我们可以在任何一个Object类的子类中调用这个方法，但是不能在某个对象上调用它，因为它只属于这个类的私有方法：
{% codeblock lang:ruby %}
class Foo
    def fourth_power_of(x)
      square(x) * square(x)
    end
end
{% endcodeblock %}	

	ruby> "fish".square(5)
	ERR: (eval):1: private method `square' called for "fish":String

###### Singleton Method

对一个单独的对象定义的方法叫做单例方法：

	ruby> class SingletonTest
	    |   def size
	    |     25
	    |   end
	    | end
	   nil
	ruby> test1 = SingletonTest.new
	   #<SingletonTest:0xbc468>
	ruby> test2 = SingletonTest.new
	   #<SingletonTest:0xbae20>
	ruby> def test2.size
	    |    10
	    | end
	   nil
	ruby> test1.size
	   25
	ruby> test2.size
	   10

##### 关于module

* module和class的区别：

	module没有实例
	module没有子类
	module由`module ... end`括起来

##### 关于procedure object

* trap - 为系统信号定义handler：

	ruby> inthandler = proc{ puts "ctrl-C was pressed." }
	ruby> trap "SIGINT", inthandler

另外，对于`trap`来说，可以更简略地定义这个handler：

	ruby> trap "SIGINT", proc{ puts "^C was pressed." }

甚至：

	ruby> trap "SIGINT", 'puts "^C was pressed."'

##### 关于variable

###### 4种variable

* @开头的是instance variable
* @@开头的是class variable
* $开头的是global variable
* 小写字母或者`_`开头的是local variable

###### trace_var

可以对某个变量进行trace，在它发送改变的时候进行一些操作：

	ruby> trace_var :$x, proc{puts "$x is now #{$x}"}
	   nil
	ruby> $x = 5
	$x is now 5
	   5

###### $开头的变量

Variable   		| Description
:-----------: 	| :-----------
$!		  		| latest error message
$@	      		| location of error 
$_	 			| string last read by gets
$.	 			| line number last read by interpreter
$&	 			| string last matched by regexp
$~	 			| the last regexp match, as an array of subexpressionsmark
$n	 			| the nth subexpression in the last match (same as $~[n])
$=	 			| case-insensitivity flag
$/	 			| input record separator
$\	 			| output record separator
$0	 			| the name of the ruby script file
$*	 			| the command line arguments
$$	 			| interpreter's process ID
$?	 			| exit status of last executed child process

其中`$_`和`$~`是局部变量

###### 跨文件访问变量的方法

有两种：

* 通过$var
* 通过class的静态变量@@var，然后在class中采用静态方法对@@var进行赋值和取值


------

#### 特定类或模块笔记

##### Logger
* Logger初始化 (If your log is being stored in a file, you can have Logger rotate or replace the log file when it gets too big, or once a certain amount of time has elapsed)
{% codeblock lang:ruby %}
require 'logger'
# Keep data for the current month only
Logger.new('this_month.log', 'monthly')
# Keep data for today and the past 20 days.
Logger.new('application.log', 20, 'daily')
# Start the log over whenever the log exceeds 100 megabytes in size.
Logger.new('application.log', 0, 100 * 1024 * 1024)
{% endcodeblock %}	
* Logger的级别有以下五种：Logger.debug, Logger.info, Logger.warn, Logger.error, Logger.fatal，可以通过修改level来修改Logger的级别，只有在`>=`当前Logger级别的log才会被记录下来：
{% codeblock lang:ruby %}
$LOG.level = Logger::ERROR
{% endcodeblock %}

使用Logger：
{% codeblock lang:ruby %}
$LOG.debug("Some log")
{% endcodeblock %}

##### Time
{% codeblock lang:ruby %}
t = Time.now
# to get day, month and year with century  
# also hour, minute and second  
puts t.strftime("%d/%m/%Y %H:%M:%S")
  
# You can use the upper case A and B to get the full  
# name of the weekday and month, respectively  
puts t.strftime("%A")
puts t.strftime("%B")
  
# You can use the lower case a and b to get the abbreviated  
# name of the weekday and month, respectively  
puts t.strftime("%a")
puts t.strftime("%b")
  
# 24 hour clock and Time zone name  
puts t.strftime("at %H:%M %Z")
{% endcodeblock %}