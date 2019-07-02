---
title: "如何优雅的抛出异常|How to throws exception properly in Spring mvc"
published: true
---

### Answer: throws new ResponseStatusException(...);

### How it works:

程序在运行期间会抛出各种各样的异常, 其中一部分会被 Spring 按照约定的方式处理, 并返回相关的状态码以及消息给客户端或者将浏览器导航到特定的页面,
另一部分则统一返回 `500 Internal server error`. 那么这一功能是怎么实现的呢?

#### HandlerExceptionResolver:

经过一些搜索和翻阅源码后找的了相关的 `HandlerExceptionResolver` 接口:

```java
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
	 * returning a ModelAndView that represents a specific error page if appropriate.
	 * <p>The returned {@code ModelAndView} may be empty
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
	ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
``` 


#### DefaultHandlerExceptionResolver:

`DefaultHandlerExceptionResolver` 处理了一部分异常, 并将其转化成了相应的 `ModelAndView`.
其中应用层比较常用的 400(SC_BAD_REQUEST) 状态以及 404(SC_NOT_FOUND) 状态对应的异常包括:

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

然而这几种异常的构造器从 Controller 或者 Service 中调用起来都不太方便, 让我们再看看其他的异常处理方式.


#### ExceptionHandlerExceptionResolver:

这个类负责处理 `@ExceptionHandler` 注解过的方法, 常用于对某种类型的异常做统一处理.


#### SimpleMappingExceptionResolver:

当抛出异常时, 这个类负责按照特定的映射关系返回对应的视图.


#### ResponseStatusExceptionResolver:

当应用抛出 `ResponseStatusException` 或者抛出 `@ResponseStatus` 注解过的异常时, 都会被这个类捕获. 并且这个类内部有
`MessageSource` 成员变量, 因此它也可以对异常信息做国际化处理, 美中不足的一点是暂时无法对异常信息进参数绑定:

```java
public class ResponseStatusExceptionResolver extends AbstractHandlerExceptionResolver implements MessageSourceAware {
    
    protected ModelAndView applyStatusAndReason(int statusCode, @Nullable String reason, HttpServletResponse response)
    		throws IOException {
    
    	if (!StringUtils.hasLength(reason)) {
    		response.sendError(statusCode);
    	}
    	else {
    		String resolvedReason = (this.messageSource != null ?
    				this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) :
    				reason);
    		response.sendError(statusCode, resolvedReason);
    	}
    	return new ModelAndView();
    }
}
// spring-webmvc-5.1.8.RELEASE
```


### Summary:

对于 Spring 提供的这几种异常处理方式进行研究后不难发现, 对于应用想要以最简单的方式抛出异常并让浏览器导航到特定错误页的话, 可以针对
`SimpleMappingExceptionResolver` 进行配置, 而对于返回 Json 消息的应用, 则可以考虑抛出 `ResponseStatusException`.