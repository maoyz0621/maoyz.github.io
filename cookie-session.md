###### 一、request
```
    // 获得发出请求字符串的客户端地址
    String requestURI = request.getRequestURI();
    String requestURL = request.getRequestURL().toString();
    // 获得客户端向服务器端传送数据的方法有GET、POST、PUT等类型
    String method = request.getMethod();
    String remoteUser = request.getRemoteUser();
    // 获得客户端的IP地址
    // String remoteAddr = request.getRemoteAddr();
    String ip = IpUtils.getIpAddress(request);
    String remoteHost = request.getRemoteHost();
    // 返回发出请求的客户机的端口号
    int remotePort = request.getRemotePort();
    String localAddr = request.getLocalAddr();
    String localName = request.getLocalName();
    // 浏览器
    String userAgent = request.getHeader("User-Agent");

    String property = System.getProperty("os.name");
    String property1 = System.getProperty("os.version");
    String property2 = System.getProperty("os.arch");
    // 该方法返回请求中的参数部分（参数名+值）
    String queryString = request.getQueryString();
```

###### 二、IpUtils
```
public class IpUtils {

    private static final String UNKNOWN = "unknown";

    private IpUtils() {
    }

    /**
     * 获取用户真实IP地址，不使用request.getRemoteAddr()的原因是有可能用户使用了代理软件方式避免真实IP地址,
     * 可是，如果通过了多级反向代理的话，X-Forwarded-For的值并不止一个，而是一串IP值，究竟哪个才是真正的用户端的真实IP呢？
     * 答案是取X-Forwarded-For中第一个非unknown的有效IP字符串。
     */
    public static String getIpAddress(HttpServletRequest request) {
        String ip = null;

        String ipAddresses = request.getHeader("X-Forwarded-For");
        if (ipAddresses == null || ipAddresses.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddresses)) {
            //Proxy-Client-IP：apache 服务代理
            ipAddresses = request.getHeader("Proxy-Client-IP");
        }

        if (ipAddresses == null || ipAddresses.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddresses)) {
            //WL-Proxy-Client-IP：weblogic 服务代理
            ipAddresses = request.getHeader("WL-Proxy-Client-IP");
        }

        if (ipAddresses == null || ipAddresses.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddresses)) {
            //HTTP_CLIENT_IP：有些代理服务器
            ipAddresses = request.getHeader("HTTP_CLIENT_IP");
        }

        if (ipAddresses == null || ipAddresses.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddresses)) {
            //X-Real-IP：nginx服务代理
            ipAddresses = request.getHeader("X-Real-IP");
        }

        //有些网络通过多层代理，那么获取到的ip就会有多个，一般都是通过逗号（,）分割开来，并且第一个ip为客户端的真实IP
        if (ipAddresses != null && ipAddresses.length() != 0) {
            ip = ipAddresses.split(",")[0];
        }

        //还是不能获取到，最后再通过request.getRemoteAddr();获取
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddresses)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}
```

###### 三、cookie
```
    Cookie cookie = new Cookie("mao", "123");
    cookie.setDomain(request.getRequestURI());
    cookie.setPath(request.getContextPath());
    response.addCookie(cookie);

    Cookie[] cookies = request.getCookies();
    String name = cookie.getName();
    String domain = cookie.getDomain();
    int maxAge = cookie.getMaxAge();
    String path = cookie.getPath();
    int version = cookie.getVersion();
    String value = cookie.getValue();
    String comment = cookie.getComment();
```

###### 四、session
```
    HttpSession session = request.getSession();
    String id = session.getId();
    long creationTime = session.getCreationTime();
    session.getAttribute("SESSIONID");

    session.setAttribute("SESSIONID", sessionid);
    session.setMaxInactiveInterval(60 * 30);
```
