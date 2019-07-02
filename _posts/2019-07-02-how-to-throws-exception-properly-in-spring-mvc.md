---
title: "如何优雅的抛出异常|How to throws exception properly in Spring mvc"
published: true
---

### Answer: throws new ResponseStatusException(...);

### How it works:

程序在运行期间会抛出各种各样的异常, 其中一部分会被 Spring 按照约定的方式处理, 并返回相关的状态码给客户端,
另一部分则统一返回 `500 Internal server error`.

##### HandlerExceptionResolver:

经过一些搜索和翻阅源码后找的了相关的 `HandlerExceptionResolver` 接口:

```java

package org.springframework.web.servlet;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.lang.Nullable;

/**
 * Interface to be implemented by objects that can resolve exceptions thrown during
 * handler mapping or execution, in the typical case to error views. Implementors are
 * typically registered as beans in the application context.
 *
 * <p>Error views are analogous to JSP error pages but can be used with any kind of
 * exception including any checked exception, with potentially fine-grained mappings for
 * specific handlers.
 *
 * @author Juergen Hoeller
 * @since 22.11.2003
 */
public interface HandlerExceptionResolver {

	/**
	 * Try to resolve the given exception that got thrown during handler execution,
	 * returning a {@link ModelAndView} that represents a specific error page if appropriate.
	 * <p>The returned {@code ModelAndView} may be {@linkplain ModelAndView#isEmpty() empty}
	 * to indicate that the exception has been resolved successfully but that no view
	 * should be rendered, for instance by setting a status code.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @param handler the executed handler, or {@code null} if none chosen at the
	 * time of the exception (for example, if multipart resolution failed)
	 * @param ex the exception that got thrown during handler execution
	 * @return a corresponding {@code ModelAndView} to forward to,
	 * or {@code null} for default processing in the resolution chain
	 */
	@Nullable
	ModelAndView resolveException(
			HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);

}
``` 

##### DefaultHandlerExceptionResolver:

查看它默认的几个实现类以后, 发现 `DefaultHandlerExceptionResolver` 处理了一部分异常, 并将其转化成了相应的状态码返回给了客户端.
其中应用层比较关心的 400(SC_BAD_REQUEST) 状态以及 404(SC_NOT_FOUND) 状态对应的异常包括:

<table>
<thead>
<tr>
<th class="colFirst">Exception</th>
<th class="colLast">HTTP 状态码</th>
<th class="colLast">应用场景</th>
</tr>
</thead>
<tbody>
<tr class="altColor">
<td><p>MissingServletRequestParameterException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口缺少必要的参数时抛出该异常. 与 @RequestParameter 相关.</p></td>
</tr>
<tr class="rowColor">
<td><p>ServletRequestBindingException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口缺少必要的属性时抛出该异常. 与 @RequestAttribute 相关.</p></td>
</tr>
<tr class="rowColor">
<td><p>TypeMismatchException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口无法将字符串解析成对应的类型时抛出该异常.</p></td>
</tr>
<tr class="altColor">
<td><p>HttpMessageNotReadableException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在 HttpMessageConverter 从 InputStream 中读消息失败时抛出该异常.</p></td>
</tr>
<tr class="altColor">
<td><p>MethodArgumentNotValidException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口校验失败时抛出该异常. 与 @Valid, @RequestPart, @RequestBody 相关.</p></td>
</tr>
<tr class="rowColor">
<td><p>MissingServletRequestPartException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口缺少必要的部分时抛出该异常. 与 @RequestPart 相关.</p></td>
</tr>
<tr class="altColor">
<td><p>BindException</p></td>
<td><p>400</p></td>
<td><p>Spring 会在请求的接口绑定参数失败时抛出该异常. 与 @Valid, BindingResult 相关.</p></td>
</tr>
<tr class="rowColor">
<td><p>NoHandlerFoundException</p></td>
<td><p>404</p></td>
<td><p>Spring 会在找不到请求的对应的接口, 并且 spring.mvc.throw-exception-if-no-handler-found=true 时抛出该异常.</p></td>
</tr>
</tbody>
</table>

然而这几种异常的构造器从 Controller 或者 Service 中调用起来都不太方便.

##### ExceptionHandlerExceptionResolver:

这个类负责处理 `@ExceptionHandler` 注解过的方法, 常用于对某种类型的异常做统一处理.