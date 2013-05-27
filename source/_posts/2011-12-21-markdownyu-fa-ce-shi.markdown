---
layout: post
title: "Markdown语法测试"
date: 2011-12-21 15:17
comments: true
categories: Others
---

<http://ytliu.info>

\[mctrain](http://ytliu.info "mctrain's blog")    
[mctrain](http://ytliu.info "mctrain's blog")    
[mctrain-2][link-mctrain-2]   
[link-mctrain-2]:http://ytliu.info "mctrain's blog"   

![pic](http://ytliu.info/images/search.png "pic")

        #title1
#title1
        ##title2
##title2
        ###title3
###title3
        ####title4
####title4
        #####title5
#####title5
        ######title6
######title6

        *italic*
*italic*

        **bold**
**bold**

        ***italic&bold***
***italic&bold***

        >highlight
>highlight

        <html> *a* </html>
<html> *a* </html>

* 1
* 2
* 3

16. first
* second
* third
18. fourth

hr-high
- - - - - -
hr-below

>highlight
>>highlight again

{% codeblock %}
Awesome code snippet
{% endcodeblock %}

{% codeblock lang:objc %}
[rectangle setX: 10 y: 10 width: 20 height: 20];
{% endcodeblock %}

{% codeblock Javascript Array Syntax lang:js http://j.mp/pPUUmW MDN Documentation %}
var arr1 = new Array(arrayLength);
var arr2 = new Array(element0, element1, ..., elementN);
{% endcodeblock %}
