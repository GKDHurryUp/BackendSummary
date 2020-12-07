# SpringMVC  #

## Spring MVC的工作原理了解嘛？ ##
1. 客户端（浏览器）发送请求，直接请求到DispatcherServlet。
2. DispatcherServlet根据请求信息调用HandlerMapping，解析请求对应的Handler。
3. HandlerMapping会返回一个执行链
4. HandlerAdapter会根据Handler来调用真正的处理器来处理请求和执行相对应的业务逻辑。
5. 处理器处理完业务后，会返回一个ModelAndView对象，Model是返回的数据对象，View是逻辑上的View。
6. ViewResolver会根据逻辑View去查找实际的View。
7. DispatcherServlet把返回的Model传给View（视图渲染）。
8. 把View返回给请求者（浏览器）。



## 视图解析器是如何工作的？



## @Pathvarible和@RequestParam的区别

 @PathVariable和@RequestParam都能够完成url中的参数，只不过输入的部分不同，一个在**URL路径部分**，另一个在**参数部分**(通过?key=value的形式)

选用哪一种？

1. 当URL指向的是某一具体业务资源（或资源列表），例如博客，用户时，使用@PathVariable

2. 当URL需要对资源或者资源列表进行过滤，筛选时，用@RequestParam

例如我们会这样设计URL：

- /blogs/{blogId}
- /blogs?state=publish而不是/blogs/state/publish来表示处于发布状态的博客文章

