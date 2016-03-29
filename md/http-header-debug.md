# Http Header debug

偶然读到infoQ的一篇文章，[chrome调试一切](http://www.infoq.com/cn/news/2015/04/chrome-devtools-for-all)，颇有趣味，遂折腾一番。

clone了[Chrome Logger](https://github.com/ccampbell/chromelogger)的代码，小修改一番，挂到chrome上。

手里恰好在做java项目，找到java的server端实现[ChromeLogger4J](https://github.com/JamesPan/chromelogger4j/blob/master/src/name/yumaa/ChromeLogger4J.java)，替换了Base64实现为java8的新api，替换simpleJson为fastjson，翻了下源码，格式化部分读起来索然无味，于是略过，magic的部分其实只有一个方法，坑也在里头：

~~~ java
// 原方法
private void writeHeader(JSONObject data) {
        // String encodedData = Base64.encode(data.toJSONString(), "UTF-8").replaceAll("\\n", "");
		
		// 作者其实把坑挖好了，丢了个fixme
        //FIXME if there is long header it rises "header full: java.lang.ArrayIndexOutOfBoundsException: 16384" uncatchable exception
        // response.setHeader(HEADER_NAME, encodedData);
}


// 不严谨的小改了下
private void writeHeader(JSONObject data) {
        String encodedData = "";
        try {
            encodedData = Base64.getEncoder().encodeToString(data.toJSONString().getBytes("utf-8")).replaceAll("\\n", "");
        } catch (UnsupportedEncodingException e) {
			response.setHeader(HEADER_NAME, e.getMessage());
            e.printStackTrace();
        }

        try {
            int len = encodedData.getBytes("UTF-8").length;
            if(len <= 1024 * 7) {
                response.setHeader(HEADER_NAME, encodedData);
            } else {
                response.setHeader(HEADER_NAME, "header is to max!!!");
            }
        } catch (UnsupportedEncodingException e) {
            response.setHeader(HEADER_NAME, e.getMessage());
            e.printStackTrace();
        }
    }
~~~

js扩展核心部分：
~~~ javascript
    // btoa base-64 encode
    // atob base-64 decode
    
    function _process(details) {
        var headers = details.responseHeaders;

        for (var i = 0; i < headers.length; i++) {
            if (HEADER_NAMES.indexOf(headers[i].name.toLowerCase()) !== -1) {
                _logData(_decode(headers[i].value));
                return;
            }
        }
    }

	function _decode(header) {
        return JSON.parse(atob(header));
    }

	// 总体逻辑：
	// 后台脚本处理监听http请求，通过chrome消息发送给相应前台脚本，在控制台打印调试信息
~~~


一言蔽之：
~~~
// server side:
header(DEBUG_HEADER_NAME_KEY, base64_encode(json_encode(DEBUG_INFO_STR)))

// client side:
Object = json_decode(base64_decode(DEBUG_HEADER_NAME_VALUE))
~~~


调试的精巧之处可看似薛定谔的猫，观察本身影响结果。c与c++有条件编译可用，debug代码尽可以丢进去。道理是一样的。
利用Chrome调试一切的问题在于，如何不影响被调试的http请求。放请求体么？不处理好，很容易干扰业务，不甚可行，于是只剩下header；
善用header，将调试信息格式化之后塞到header里头，豁然开朗。

header一个很古老的业务应用是放授权信息，标准里头有authorize，非标准实现可以放token，或者说header的cookie段也是偏业务的属性。


~~~ java

// 原谅我还是喜欢c#的action与function的命名方式
@FunctionalInterface
public interface Act1<T1> {
    void call(T1 t1);
}

// typedef
public interface Console extends Act1<Act1<ChromeLogger4J>> {}

public class $ {
	
	// 封装了一个获取ChromeLog的静态方法
	public static Console getChromeConsole(
        final HttpServletRequest req,
        final HttpServletResponse res
    ) {
        Console ignoreAct = (ignore) -> {};
        if(res == null || req == null) {
            return ignoreAct;
        }

		// 通过header来判断是否开启chromedebug调试
		// chrome扩展开关控制
        try {
            if(req.getIntHeader("Debug") != 1) {
                return ignoreAct;
            }
        } catch (/*NumberFormat*/Exception e) {
            return ignoreAct;
        }

        ChromeLogger4J console = new ChromeLogger4J(res);
        console.stack = true;
        console.addnull = true;
        console.reflect = true;
        console.reflectmethods = true;

        return (act) -> {
            try {
                act.call(console);
            } catch (Exception ignore) {}
        };
    }
}

// 在不修改ChromeLogger4J基础上，屏蔽了debug的条件判断
// example
Console console = getChromeConsole(req, res);
console.call(c -> c.log("log"));
console.call(c -> c.info("info"));
console.call(c -> c.error("error"));

~~~

在springMVC注册了简单的异常处理handler：

~~~ java
@ControllerAdvice
public class ExceptionController {
    private static final Log logger = LogFactory.getLog(ScheduleController.class);

    @ExceptionHandler(Exception.class)
    @ResponseStatus(value = HttpStatus.OK) // 废弃http状态吗，错误已通过错误码封装（自行忽略）
    @ResponseBody
    public ServiceResult<Object> exceptionHandler(
            final Exception ex,
            final HttpServletRequest req,
            final HttpServletResponse res)
    {
        Console console = $$.getChromeConsole(req, res);
        try {
            String exMsg = $$.extostr(ex); // 拿到美化的stacktrace 字符串
            String errUrl = req.getPathInfo() + "?" + req.getQueryString();
            logger.error("exception in " + errUrl + ": " + ex.getMessage(), ex);
			
			// chrome debug
            console.call(c -> c.error(/*ex,*/ $$.extostr(ex)));

            boolean isAjax = "XMLHttpRequest".equals(req.getHeader("X-Requested-With"));
            if(isAjax) {
                return $$.retFalse("抱歉, %>_<% 有错误发生");
            } else {
                // FIXME remove
                // res.sendRedirect("/error");
                ex.printStackTrace();
                res.getWriter().write("抱歉, %>_<% 有错误发生");
            }
        }catch (Throwable t){
			// chrome debug
            console.call(c -> c.error(t));
            
			t.printStackTrace();
            logger.error(t.getMessage() , t);
        }
        return null;
    }
}
~~~

看起来很美好的一个玩具，比如线上代码报错，不方便调试，临时开一下chrome扩展，看下异常；
但是，但现实总是骨感；项目的架构是流行的nginx反向代理，负载均衡，后面挂着tomcat。
~~~
explorer -> nginx -> tomcat -> java 
java -> tomcat -> nginx -> explorer 
~~~
实际使用中发现，nginx不久就无情的报502错误，看日志显示，header太大了!!!

协议里头木有规定header大小，问题是web服务器实现不得不考虑这个问题，查了下资料,
万能的[stackoverflow](http://stackoverflow.com/questions/686217/maximum-on-http-header-values)
> Apache 2.0, 2.2: 8K
> nginx: 4K - 8K
> IIS: varies by version, 8K - 16K
> Tomcat: varies by version, 8K - 48K (?!)
   
tomcat默认 maxHttpHeaderSize 8kb，nginx估计默认4k,改了下nginx的配置，
~~~
proxy_buffers 16 16k;
proxy_buffer_size 32k;
~~~
瓶颈就不在nginx上了~

不过chrome调试一切还是有点像噱头，特么的不适合输出复杂的java对象~

想来如果php使用的话，要各种ob_*函数控制缓存了，set完header之后才能刷缓冲区~

**end**
     