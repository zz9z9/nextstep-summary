## Chapter4 : HTTP 웹 서버 구현을 통해 HTTP 이해하기 

### 구현 (의식의 흐름..)
> ### 요구사항1 : http://localhost:8080/index.html로 접속했을 때 webapp 디렉토리의 index.html 파일을 읽어 클라이언트에 응답한다
  - index.html 파일 읽어서 byte 형태로 만들어준 뒤 DataOutputStream을 통해 response 넘기면 되지 않을까.. ?   
  - ### <b> http://localhost:8080/index.html 라는 요청왔을 때 '/' 이후 부분에 파일명을 어떻게 가져오지 ?? </b>
    - connection.getInputStream() 디버거로 찍어보니 여기엔 없는 것 같다. (connection은 Socket 객체)
    - 값이 없던게 아니라 InputStream은 바이트 단위로 나타내기 때문에 디버거로 찍어도 원하는 값 볼 수 없었던 것. <br>
   (https://docs.oracle.com/javase/7/docs/api/java/io/InputStream.html 참고)
    - 따라서, 클라이언트가 요청한 html 파일이 무엇인지 알기 위해 InputStreamReader 활용하여 byte array인 InputStream을 디코딩하고 BufferdReader 이용해서 라인 단위로 읽는다.
  - 최종적으로 아래와 같은 메서드로 구현
```java
 // 클라이언트가 요청한 html 파일 이름 가져온다
    private String getRequestHtmlName(InputStream in) throws IOException {
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(in));
        String requestInfo = bufferedReader.readLine();

        if(!requestInfo.contains(".html")) {
            return null;
        }

        String htmlPage = requestInfo.split(" ")[1];

        return htmlPage.substring(1); // 맨 앞에 '/' 지우기 위함
    }
    // http response를 위해 파일을 byte로 변환(절대경로 ver)
    private byte[] convertHtmlToByte(String fileName) throws IOException {
        String rootPath = System.getProperty("user.dir");
        String filePath = rootPath + "/webapp/" + fileName;
        File file = new File(filePath);

        return Files.readAllBytes(file.toPath());
    }  
    // http response를 위해 파일을 byte로 변환(상대경로 ver - 책 참고)
    private byte[] convertHtmlToByte(String fileName) throws IOException {
        String filePath = "./webapp/" + fileName;
        File file = new File(filePath);

        return Files.readAllBytes(file.toPath());
    }  
```
<br>

> ### 요구사항2 : GET 방식으로 회원가입하기 
- 요구사항1 에서 구현한 메서드를 개선해야 할 것 같다. 
- 일단 js, css 파일에 대한 요청도 response로 넘겨줘야 하기 때문에 getRequestHtmlName, convertHtmlToByte 이라는 메서드명을 좀 더 추상적으로 변경하고 html 파일이 아닌 경우에도 처리가 가능하도록 만들어야한다. 
- client 관련 파일 디렉토리 최상위 폴더인 "webapp"도 변수에 추가하지 말고 클래스 변수로 선언하고 final로 상수화 하는게 나을 것 같다.
- 본격적으로 회원가입에 요청에 대해 생각해보면, 지금까지 했던 페이지를 응답과는 달리 회원가입 이라는 '비즈니스 로직'을 수행해야 한다. 
  - 따라서, 비즈니스 로직 수행을 담당하는 '무언가'를 만들어야 할 것 같다. 
  - 또한, RequestHandler에서 현재는 페이지 응답에 관련된 것만 수행하는데 다양한 request를 적절하게 처리할 수 있도록 변경해야 할 것 같다. 
- ### <b>페이지 요청, 비즈니스 로직 요청 등 각기 다른 요청을 어떻게 구분해야 할까 ? </b>
  - 일단 아래처럼 구현해봤는데 이렇게 하드코딩으로 분기시키면 변화에 유연하게 대응하기도 어렵고 생각지 못한 케이스도 있을 것 같다.
  - 어떻게 하는게 좋을지 고민해보고 추후에 책도 참고해보자
  
<details>
   <summary>구분하는 메서드</summary>

```java
 private RequestType getRequestType(HttpRequest httpRequest) {
        String httpMethod = httpRequest.getHttpMethod();
        String requestUrl = httpRequest.getRequestUrl();

        switch (HttpMethod.valueOf(httpMethod)) {
            case GET:
                if(requestUrl.contains("?")) {
                    return RequestType.REQUEST_BUSINESS_LOGIC;
                } else if(requestUrl.equals("/") || requestUrl.contains(".")) {
                    return RequestType.REQUEST_FILE;
                } else {
                    return RequestType.REQUEST_BUSINESS_LOGIC;
                }

            case POST:
                break;
        }

        return null;
    }         
```
</details>
  
- ### <b> 비즈니스 로직 호출하는 요청은 해당 요청을 처리할 로직에 어떻게 맵핑시키면 좋을까 ? </b>
  - 사용자 요청과 해당 요청을 처리하는데 필요한 메서드를 맵핑시켜주는 역할을 하는 LogicMapper 생성. 
  - RequestHandler 내에 코드를 추가해서 할 수도 있겠지만, 그렇게 되면 책임이 많아지기 때문에 코드가 점점 복잡해질 것이다. <br>
   따라서, RequestHandler의 책임을 '클라이언트의 요청을 받아들이고 어떤 요청인지 판단 하는 것'으로 한정하는게 좋을 것 같다고 생각했다.
  - 일단 아래와 같이 구현했는데, 리플렉션을 마구 썼다. 이 방법 말고는 없을까 ? 
  - 사실, executeMethodWithParamsForGetRequest에서 파라미터 맵핑하는 부분도 파라미터 객체에 기본 생성자가 없으면 paramClass.getDeclaredConstructor().newInstance() 이 부분에서 에러 난다.
  - 그리고 LogicMapper가 최초 한 번만 생성되는 것도 보장되어야 하는 등 아직 미흡한 부분이 많다.

<details>
  <summary>LogicMapper</summary>
  
  ```java
  package webserver;
  
  import logic.UserLogic;
  import model.User;
  import util.HttpRequestUtils;
  
  import java.lang.reflect.InvocationTargetException;
  import java.util.HashMap;
  import java.util.Map;
  import java.util.Optional;
  
  public class LogicMapper {
      static class Execution <T> {
          T targetInstance;
          Class logicClass;
          String methodName;
          Class paramClass;
  
          public Execution(T targetInstance, Class logicClass, String methodName) {
              this.targetInstance = targetInstance;
              this.logicClass = logicClass;
              this.methodName = methodName;
          }
  
          public Execution(T targetInstance, Class logicClass, String methodName, Class paramClass) {
              this.targetInstance = targetInstance;
              this.logicClass = logicClass;
              this.methodName = methodName;
              this.paramClass = paramClass;
          }
  
          public T getTargetInstance() {
              return targetInstance;
          }
  
          public Class getLogicClass() {
              return logicClass;
          }
  
          public String getMethodName() {
              return methodName;
          }
  
          public Class getParamClass() {
              return paramClass;
          }
      }
  
      static Map<String, Execution> getMappingUrl = new HashMap<>();
  
      public LogicMapper() {
          initGetRequest();
      }
  
      private void initGetRequest() {
          getMappingUrl.put("/user/create", new Execution(UserLogic.getInstance(), UserLogic.class, "signup", User.class));
      }
  
      private void initPostRequest() {
  
      }
  
      public byte[] doRequestLogic(HttpRequest httpRequest) throws Exception {
          HttpMethod httpMethod = HttpMethod.valueOf(httpRequest.getHttpMethod());
          String requestUrl = httpRequest.getRequestUrl();
          byte[] response = {};
  
          switch (httpMethod) {
              case GET:
                  response = requestUrl.contains("?") ? executeMethodWithParamsForGetRequest(requestUrl) : executeMethodWithoutParamsForGetRequest(requestUrl);
                  executeMethodWithParamsForGetRequest(requestUrl);
                  break;
          }
  
          return response;
      }
  
      public byte[] executeMethodWithParamsForGetRequest(String requestUrl) throws Exception {
          String[] info = requestUrl.split("\\?");
          String url = info[0];
          Map<String, String> params = HttpRequestUtils.parseQueryString(info[1]);
          Execution execution = Optional.ofNullable(getMappingUrl.get(url)).orElseThrow(NoSuchMethodError::new);
          Class paramClass = execution.getParamClass();
          Object instance = paramClass.getDeclaredConstructor().newInstance();
  
          for(String key : params.keySet()) {
              Optional.ofNullable(paramClass.getDeclaredField(key)).ifPresent((field) -> {
                  field.setAccessible(true);
                  try {
                      field.set(instance, params.get(key));
                  } catch (IllegalAccessException e) {
                      e.printStackTrace();
                  }
              });
          }
  
          execution.getLogicClass().getMethod(execution.getMethodName(), execution.getParamClass()).invoke(execution.getTargetInstance(), instance);
  
          return "SUCCESS".getBytes();
      }
  
      public byte[] executeMethodWithoutParamsForGetRequest(String requestUrl) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
          Execution execution = Optional.ofNullable(getMappingUrl.get(requestUrl)).orElseThrow(NoSuchMethodError::new);
          execution.getLogicClass().getMethod(execution.getMethodName(), execution.getParamClass()).invoke(execution.getTargetInstance());
  
          return "SUCCESS".getBytes();
      }
  }
  ```  
</details>

<br>

> ### 요구사항3 : POST 방식으로 회원가입하기 
- ### <b>POST 방식으로 넘어오는(body에 담겨있는) 파라미터는 어떻게 받지? </b>
  - POST 방식의 경우 bufferedReader의 readLine() 사용하게 되면 http header는 읽지만 body는 읽어오지 못한다.  
  - 원인을 찾아보니 header의 마지막 부분("") readLine() 하는 부분에서 계속 hanging 되어있다. 
  - hanging 되는 이유는 request header 마지막 라인이 공백인데 readLine()의 경우 line의 끝에 개행 문자가 없는 경우, 값이 올 때 까지 계속 기다린다고 한다. 
  - 따라서, 공백인 경우를 체크하는 로직이 필요하고, Content-Length를 구해서 그 크기만큼 나머지를 read() 하면 된다. 
  - readLine() 대신 read()를 사용하는 이유는, body로 넘어오는 파라미터 또한 끝에 개행문자가 없기 때문에 readLine() 사용시 대기상태에 빠진다. 
  
```java
        if(httpMethod.equals("POST")) {
            int contentLen = 0;
            for(String line = bufferedReader.readLine(); (!line.isEmpty() && line!=null); line=bufferedReader.readLine()) {
                if(line.contains("Content-Length")) {
                    String[] info = line.split(":");
                    contentLen = Integer.parseInt(info[1].trim());
                    break;
                }
            }

            if (contentLen > 0) {
                char[] body = new char[contentLen];
                bufferedReader.read(body);
                String params = new String(body);
            }
        }
```

<br>

> ### 요구사항4 : 302 status code 적용
- ### <b> 회원가입 완료 후 response header에 하드코딩 없이 어떻게하면 Location : localhost:8080/index.html 을 세팅해줄 수 있을까 ?</b>

<br>

> ### 요구사항5 : 로그인하기
- ### <b> 비즈니스 로직 수행하는 메서드의 다양한 타입의 리턴값을 LogicMapper에 추상화 시킨 두 개의 메서드 executeMethodWithParams, executeMethodWithoutParams에서 어떻게 받아서 처리해야할까? </b>
  - 먼저 웹 서버의 관점에서 생각해봤을때, 웹 서버에서 클라이언트로 응답하는 형태는 크게 'html 페이지 / 데이터(json 등)' 이라고 생각했다. 
  - 따라서, 클라이언트의 요청에 대해 적절한 응답을 하기 위해 필요한 것은 위에서 정의한 '응답의 형태'와 '실제 데이터' 두 가지라고 생각하여 이를 나타내는 ExecutionResult 클래스를 만들었다.
  - 이에 따라 executeMethodWithParams, executeMethodWithoutParams 메서드의 리턴값을 ExecutionResult로 바꾸고 다양한 타입의 리턴값을 해당 클래스의 속성 중 '실제 데이터'(Object) 부분에 세팅했다. 
```java
public class ExecutionResult {
    private ResponseType responseType;
    private Object returnData;

    public ExecutionResult(ResponseType responseType, Object returnData) {
        this.responseType = responseType;
        this.returnData = returnData;
    }

    public ResponseType getResponseType() {
        return responseType;
    }

    public Object getReturnData() {
        return returnData;
    }
}
```     
     
- ### <b> 비즈니스 로직 수행하는 메서드들의 파라미터는 개수도 다양하고 타입도 다양한데 LogicMapper에 추상화 시킨 두 개의 메서드 executeMethodWithParams, executeMethodWithoutParams에서 어떻게 받아서 처리해야할까? </b>
  - RequestHandler에서 LogicMapper로 넘어오는 파라미터의 형태는 Map인데, LogicMapper에서 맵핑시켜줄 다양한 비즈니스 로직 메서드의 파라미터를 일일이 맞추려면 추상화된 두 개의 메서드만 갖고는 불가능하다고 생각했다. 
  - 왜 이렇게 됐을까를 생각해보니 LogicMapper 입장에서 실제로 실행되는 비즈니스 로직 클래스, 파라미터 타입, 개수 등 알아야할게 너무 많은 것 같았다. 위 요구사항2에 보면 LogicMapper의 미흡함에 대해 적어놨는데, 그 때 더 깊게 생각하지 않았던게 결국 발목을 잡은 것 같다.
  - 따라서, LogicMapper에서 실제 비즈니스 로직을 바로 실행하는게 아니라 중간에 LogicExecutor라는 객체를 두고 LogicMapper에서는 들어온 요청에 따라 실제 로직 메서드가 아닌 LogicExecutor의 특정 메서드를 실행하고 파라미터도 Map 형태로만 넘기도록 수정.
  - LogicExcecutor에서는 Map형태로 받은 파라미터를 가공하여 실제 로직 메서드에 필요한 파라미터 형태로 만들고 해당 메서드를 호출한다.
  - 이렇게 하고 보니 LogicExecutor가 MVC 모델에서 Controller와 약간 비슷한 역할이지 않나 하는 생각이 든다. 
  
<details>
   <summary>LogicMapper 개선해본 버전(RequestLogicMapper)</summary>
    
```java
    public class RequestLogicMapper {
        static class Execution {
            private String methodName;
            private ResponseType responseType;
    
            public Execution(String methodName, ResponseType responseType) {
                this.methodName = methodName;
                this.responseType = responseType;
            }
    
            public String getMethodName() {
                return methodName;
            }
    
            public ResponseType getResponseType() {
                return responseType;
            }
        }
    
        private LogicExecutor logicExecutor = LogicExecutor.getInstance();
        private Map<String, Execution> getMappingUrl = new HashMap<>();
        private Map<String, Execution> postMappingUrl = new HashMap<>();
    
        public RequestLogicMapper() {
            initGetRequest();
            initPostRequest();
        }
    
        private void initGetRequest() {
            getMappingUrl.put("/user/create", new Execution("signup", ResponseType.HTML_PAGE));
        }
    
        private void initPostRequest() {
            postMappingUrl.put("/user/create", new Execution("signup", ResponseType.HTML_PAGE));
            postMappingUrl.put("/user/login", new Execution("login", ResponseType.HTML_PAGE));
        }
    
        public ExecutionResult doRequestLogic(HttpRequest httpRequest) throws Exception {
            HttpMethod httpMethod = httpRequest.getHttpMethod();
            String requestUrl = httpRequest.getRequestUrl();
            Map<String,String> params = httpRequest.getParams();
            Execution execution = null;
    
            switch (httpMethod) {
                case GET:
                    execution = Optional.ofNullable(getMappingUrl.get(requestUrl)).orElseThrow(NoSuchMethodError::new);
                    break;
                case POST:
                    execution = Optional.ofNullable(postMappingUrl.get(requestUrl)).orElseThrow(NoSuchMethodError::new);
                    break;
            }
    
            ExecutionResult result = (params!=null) ? executeMethodWithParams(execution, params) : executeMethodWithoutParams(execution);
    
            return result;
        }
    
        public ExecutionResult executeMethodWithParams(Execution execution, Map<String,String> params) throws Exception {
            Method logic = logicExecutor.getClass().getMethod(execution.getMethodName(), Map.class);
            Object returnObj = logic.invoke(logicExecutor, params);
    
            return new ExecutionResult(execution.getResponseType(), returnObj);
        }
    
        public ExecutionResult executeMethodWithoutParams(Execution execution) throws Exception {
            Method logic = logicExecutor.getClass().getMethod(execution.getMethodName());
            Object returnObj = logic.invoke(logicExecutor);
    
            return new ExecutionResult(execution.getResponseType(), returnObj);
        }
    
```
</details>

<details>
   <summary>LogicExecutor</summary>
   
```java
public class LogicExecutor {

    private static final LogicExecutor logicExecutor = new LogicExecutor();

    private LogicExecutor(){}

    public static LogicExecutor getInstance() {
        return logicExecutor;
    }

    private UserLogic userLogic = UserLogic.getInstance();

    public String signup(Map<String, String> params) {
        String id = params.get("userId");
        String pw = params.get("password");
        String name = params.get("name");
        String email = params.get("email");
        User newUser = new User(id, pw, name, email);

        return userLogic.signup(newUser);
    }

    public String login(Map<String, String> params) {
        String id = params.get("userId");
        String pw = params.get("password");

        return userLogic.login(id,pw);
    }

}
```
</details>

- ### <b> LogicExecutor에 선언한 메서드에서 Map 형태의 파라미터 뿐 아니라 setCookie 등의 처리를 위해 HttpResponse 같은 파라미터도 받아야하는 경우 RequestLogicMapper에서 어떻게 처리해줘야할까? </b>
  - 일단은 아래 코드처럼 했는데 너무 어거지로 한거같다. 실제 Spring MVC에서는 Controller 단에서 HttpServletRequest, HttpServletRespone 등을 자유롭게 받을 수 있던데, 어떻게 가능한건지 공부해보자 
```java
 try {
    logic = logicExecutor.getClass().getMethod(execution.getMethodName(), Map.class);
 } catch (NoSuchMethodException e) {
    logic = logicExecutor.getClass().getMethod(execution.getMethodName(), Map.class, HttpResponse.class);
 }
```

<br>

> ### 요구사항6 : 사용자 목록 출력
- <h3><b> cookie에 세팅된 login 여부 값을 가져와서 로그인 한 경우 목록 출력해야하는데, 그러려면 LogicExecutor에 HttpRequest를 받아서 실제 로직에서 getCookie로 해당 값이 있는지 여부를 알아야하는데
HttpRequest까지 메서드 파라미터로 받으려면 어떻게 해야할까?? 요구사항5 마지막 고민이 여기서도 발목을 잡는다</b></h3>
  
<br>  
  
> ### 요구사항7 : CSS 지원하기 
- ### 왜 css 파일이 적용이 안된 화면이 렌더링될까 ?? 
  - 현재 내가 만든 애플리케이션의 response header에 세팅되는 Content-type은 모두 "text/html;charset=utf-8" 이다.
  - 브라우저가 css 파일을 읽고 적용하기 위해선 "text/css"로 세팅해줘야 한다
  - js파일은 "application/javascript"로 세팅해줘야한다.
  
### 배운 것
- Java I/O (InputStream, InputStreamReader, BufferedReader, FileReader 등 - [참고 링크](https://st-lab.tistory.com/41)) 
- 상대경로를 사용할 경우, ./ (현재 위치)는 bin, src 폴더를 포함하는 해당 자바 프로젝트 폴더의 위치이다. 
- BufferedReader readLine() 사용시 잘못하면 계속 대기 상태에 있을 수 있다. 
- equals() 오버라이드시 hashcode() 오버라이드 하지 않으면 HashMap에서 값 가져올 때 원하지 않는 결과를 얻게된다. 
- 쿠키의 domain과 path

### 인상 깊었던 말

