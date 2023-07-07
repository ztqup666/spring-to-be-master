# 基本功能

@ControllerAdvice，是Spring3.2提供的新注解,它是一个Controller增强器,可对controller中被 @RequestMapping注解的方法加一些逻辑处理。主要作用有一下三种

- 通过@ControllerAdvice注解可以将对于控制器的全局配置放在同一个位置。
- 注解了@ControllerAdvice的类的方法可以使用@ExceptionHandler、@InitBinder、@ModelAttribute注解到方法上。
  - @ExceptionHandler：用于全局处理控制器里的异常，进行全局异常处理
  - @InitBinder：用来设置WebDataBinder，用于请求中注册自定义参数的解析，从而达到自定义请求参数格式的目的。
  - @ModelAttribute：本来作用是绑定键值对到Model中，此处让全局的@RequestMapping都能获得在此处设置的键值对 ，全局数据绑定。
- @ControllerAdvice注解将作用在所有注解了@RequestMapping的控制器的方法上。

## 具体用法

```java
// 这里@RestControllerAdvice等同于@ControllerAdvice + @ResponseBody
// annotations关联相应的Controller
@RestControllerAdvice(annotations = {Controller.class, RestController.class})
public class GlobalHandler {
    private final Logger logger = LoggerFactory.getLogger(GlobalHandler.class);
    
    @ModelAttribute(name = "md")
    public Map<String,Object> getGlobalData(){
        HashMap<String, Object> map = new HashMap<>();
        map.put("age", 99);
        map.put("gender", "男");
        return map;
    }
    
  // @InitBinder标注的initBinder()方法表示注册一个Date类型的类型转换器，用于将类似这样的2019-06-10
  // 日期格式的字符串转换成Date对象
  @InitBinder
  protected void initBinder(WebDataBinder binder) {
    binder.registerCustomEditor(Date.class,
      new CustomDateEditor(new SimpleDateFormat("yyyyMMdd"), false));
  }
    
    
    private static final String DEFAULT_ERROR_VIEW = "error";

    /**
     * 统一 json 异常处理
     *
     * @param exception JsonException
     * @return 统一返回 json 格式
     */
    @ExceptionHandler(value = JsonException.class)
    @ResponseBody
    public ApiResponse jsonErrorHandler(JsonException exception) {
        log.error("【JsonException】:{}", exception.getMessage());
        return ApiResponse.ofException(exception);
    }

      @ExceptionHandler(Exception.class)
      public ApiResponse exceptionHandler(Exception e) {
        return ApiResponse.ofMessage(e.getMessage());
      }

}
```

```java
@RestController
@Slf4j
public class TestController {

    @GetMapping("/json")
    @ResponseBody
    public ApiResponse jsonException() {
        throw new JsonException(Status.UNKNOWN_ERROR);
    }

    @GetMapping("/page")
    public ModelAndView pageException() {
        throw new PageException(Status.UNKNOWN_ERROR);
    }
	
    // @ModelAttribute(name = "md")标志了对应的md
    // 这里Date请求：GET http://localhost:8080/demo/getExamListByOpInfo?   date=20190610 会通过InitBinder直接转化为Date，或者可以通过添加@DateTimeFormat(pattern = "yyyy-MM-dd")来解这个问题
  @GetMapping("/getExamListByOpInfo")
  public ApiResponse getExamListByOpInfo(@RequestParam(name = "date") Date examOpDate, @ModelAttribute(name = "md") Map<String,Object> md) {
    log.info("date: " + examOpDate.toString());
    log.info("md: " + md.toString());
    return ApiResponse.ofSuccess("success");
  }
}
```

# 原理简析

下面我们看看Spring是怎么实现的，首先前端控制器DispatcherServlet对象在创建时会初始化一系列的对象：

```java
public class DispatcherServlet extends FrameworkServlet {
    // ......
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
    // ......
}
```

对于@ControllerAdvice 注解，我们重点关注initHandlerAdapters(context)和initHandlerExceptionResolvers(context)这两个方法。

initHandlerAdapters(context)方法会取得所有实现了HandlerAdapter接口的bean并保存起来，其中就有一个类型为RequestMappingHandlerAdapter的bean，这个bean就是@RequestMapping注解能起作用的关键，这个bean在应用启动过程中会获取所有被@ControllerAdvice注解标注的bean对象做进一步处理，关键代码在这里：

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
    // ......
    private void initControllerAdviceCache() {
                // ......
		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		AnnotationAwareOrderComparator.sort(adviceBeans);

		List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

		for (ControllerAdviceBean adviceBean : adviceBeans) {
			Class<?> beanType = adviceBean.getBeanType();
			if (beanType == null) {
				throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
			}
                        // 找到所有ModelAttribute标注的方法并缓存起来
			Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
			if (!attrMethods.isEmpty()) {
				this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @ModelAttribute methods in " + adviceBean);
				}
			}
                        // 找到所有InitBinder标注的方法并缓存起来
			Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
			if (!binderMethods.isEmpty()) {
				this.initBinderAdviceCache.put(adviceBean, binderMethods);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @InitBinder methods in " + adviceBean);
				}
			}
                        // ......
		}
	}
    // ......
}
```

经过这里处理之后，@ModelAttribute和@InitBinder就能起作用了，至于DispatcherServlet和RequestMappingHandlerAdapter是如何交互的这就是另一个复杂的话题了，此处不赘述。

再来看DispatcherServlet的initHandlerExceptionResolvers(context)方法，方法会取得所有实现了HandlerExceptionResolver接口的bean并保存起来，其中就有一个类型为ExceptionHandlerExceptionResolver的bean，这个bean在应用启动过程中会获取所有被@ControllerAdvice注解标注的bean对象做进一步处理，关键代码在这里：

```java
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver
		implements ApplicationContextAware, InitializingBean {
    // ......
	private void initExceptionHandlerAdviceCache() {
		// ......
		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		AnnotationAwareOrderComparator.sort(adviceBeans);

		for (ControllerAdviceBean adviceBean : adviceBeans) {
			ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(adviceBean.getBeanType());
			if (resolver.hasExceptionMappings()) {
			    // 找到所有ExceptionHandler标注的方法并保存成一个ExceptionHandlerMethodResolver类型的对象缓存起来
				this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @ExceptionHandler methods in " + adviceBean);
				}
			}
			// ......
		}
	}
    // ......
}
```

当Controller抛出异常时，DispatcherServlet通过ExceptionHandlerExceptionResolver来解析异常，而ExceptionHandlerExceptionResolver又通过ExceptionHandlerMethodResolver 来解析异常， ExceptionHandlerMethodResolver 最终解析异常找到适用的@ExceptionHandler标注的方法是这里：

```java
public class ExceptionHandlerMethodResolver {
	// ......
	private Method getMappedMethod(Class<? extends Throwable> exceptionType) {
		List<Class<? extends Throwable>> matches = new ArrayList<Class<? extends Throwable>>();
		// 找到所有适用于Controller抛出异常的处理方法,例如Controller抛出的异常
		// 是BizException(继承自RuntimeException),那么@ExceptionHandler(BizException.class)和
		// @ExceptionHandler(Exception.class)标注的方法都适用此异常
		for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
			if (mappedException.isAssignableFrom(exceptionType)) {
				matches.add(mappedException);
			}
		}
		if (!matches.isEmpty()) {
		        // 这里通过排序找到最适用的方法,排序的规则依据抛出异常相对于声明异常的深度,例如
			// Controller抛出的异常是BizException(继承自RuntimeException),那么BizException
			// 相对于@ExceptionHandler(BizException.class)声明的BizException.class其深度是0,
			// 相对于@ExceptionHandler(Exception.class)声明的Exception.class其深度是2,所以
			// @ExceptionHandler(BizException.class)标注的方法会排在前面
			Collections.sort(matches, new ExceptionDepthComparator(exceptionType));
			return this.mappedMethods.get(matches.get(0));
		}
		else {
			return null;
		}
	}
    // ......
}
```

整个@ControllerAdvice处理的流程就是这样，这个设计还是非常灵活的。