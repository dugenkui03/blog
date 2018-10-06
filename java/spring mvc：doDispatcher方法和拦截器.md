```

	/**
	  *	处理处理器实际的派遣工作
	 */
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		
				//第一步：调用处理器映射器返回HandlerChainExecution，没有则404
				// 检测当前求情对应的处理器，调用HandlerMapping.
				//并返回包含多个拦截器(数组)和一个处理器的
				//HandlerExecutionChain handler = HandlerMapping.getHandler(request);
				HandlerExecutionChain mappedHandler = getHandler(processedRequest, false);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					//如果返回信息没有为null或者没有对应的处理器，404
					noHandlerFound(processedRequest, response);
					return;
				}

				// 返回处理器对应的处理器适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				//第二步：顺序调用每个拦截器的preHandler方法
				//如果某个拦截器中断返回，则逆序运行他之前的所有拦截器的afterCompletion方法
				//拦截器的preHandler前置处理是顺序的，postHandler和afterCompletion都是逆序的
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}
				
				// 第三步：实际调用拦截器的地方
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				//第四步：处理器执行完毕，逆序调用拦截器的后置处理方法，后置处理方法可以修改视图
				applyDefaultViewName(request, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			//第五步：处理直接结果：请求对象、response对象、HandlerExecutionChain、视图对象和异常对象
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
			//第六步：方法中也执行了拦截器最后一个方法afterCompletion
		}
	}
```

```
	//拦截器的三个方法：
	public interface HandlerInterceptor {

	//前置处理（顺序：中断，则前边执行此方法的处理器的afterCompletion逆序执行）：请求、相应、处理器：返回boolean值：布尔值返回false都会中断流程
	boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
	    throws Exception;


	//后置处理（逆序）：请求、响应、处理器、视图
	void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception;

	//返回视图前处理（逆序）请求、相应、处理器、异常
	void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception;

}
```