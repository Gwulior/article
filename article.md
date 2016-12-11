Здравствуйте.
Это туториал о переводе небольшого учебного Java-проекта, с JSP на Angular 2.
В роли учебного проекта у нас **TopJava** ведущего Java Enterprise тренингов [Григория Кислина](https://habrahabr.ru/users/gkislin/).
В данном материале, мы рассмотрим переделку Back-End'а под нужды RESTful API и устройство Angular 2 приложения и его компонентов таких как:

- Components
- Services
- Pipes
- Router Guard
- RxJS (Observable, Subject)

Статья по большей части направлена на людей занимающихся back-end'ом, но которым время от времени приходится делать web

- Если вы не сильно любите JSP и JSF
- Голый JavaScript без родных типов и нормальной компонентной структуры вас совсем не радует
- Вы ищите хороший JS фреймворк, с большим сообществом и будущим, а не месиво из библиотек

Angular 2 может оказаться вполне хорошим выбором.

____________________________________________


**Оригинал** кода до переделки можно найти в [этом репозитории](https://github.com/12ozCode/topjava08-to-angular2) в ветке _master_
**Переделанный** проект лежит [здесь](https://github.com/12ozCode/topjava08-to-angular2/tree/angular2) в ветке _angular2_
**Демо** результата [здесь](topjava-angular2.herokuapp.com)

-картинка


Целью статьи не является рефактор проекта и демонстрация идеальной REST архитектуры, статья показывает как устроен **Angular 2** и как
с минимальными затратами перейти на SPA. В любом случае, за любые полезные поправки и замечания буду благодарен, и буду стараться оперативно вносить правки.
Итак начнем.

**Немного о проекте**: _приложение с регистрацией/авторизацией и интерфейсом на основе ролей (USER, ADMIN). Администратор может создавать/редактировать/удалять/пользователей,
а пользователь - управлять своим профилем и данными (день, еда, калории) через UI (по AJAX) и по REST интерфейсу с базовой авторизацией.
Возможна фильтрация данных по датам и времени, при этом цвет записи таблицы еды зависит от того, превышает ли сумма калорий за день норму
(редактируемый параметр в профиле пользователя)._

##Backend
Итак для удобного взаимодействий с сервером, нам придется этот сервер немного изменить.


[AdminAjaxController](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/user/AdminAjaxController.java) будет выпилиен за ненадобностью,
а его метод **enabled** (для активации\блокировки пользователя) заберем в [AdminRestController](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/user/AdminRestController.java)
оставив таким образом один контроллер для админа.
Также поступим и с [MealAjaxController](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/meal/MealAjaxController.java) оставив только [MealRestController](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/meal/MealRestController.java)

[ModelInterceptor](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/interceptor/ModelInterceptor.java) нам также не нужен так мы не используем JSP
[GlobalControllerExceptionHandler](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/GlobalControllerExceptionHandler.java) выпилываем у нас нету jsp-error страницы, в [RootController](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/java/ru/javawebinar/topjava/web/RootController.java) убираем все методы возвращающие jsp-view, нам понадобиться только регистрация и загрузка сообщений по языку о котором дальше.
Так как я не нашел нормального пути загрузки сразу всех сообщений по языку, мне пришлось создать [CustomReloadableResourceBundleMessageSource](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/resources/CustomReloadableResourceBundleMessageSource.java)

```java
public class CustomReloadableResourceBundleMessageSource extends ReloadableResourceBundleMessageSource {

    private static final Logger log = LoggerFactory.getLogger(CustomReloadableResourceBundleMessageSource.class);

    public Properties getAllMessages(Locale locale) {
        PropertiesHolder mergedProperties = getMergedProperties(locale);
        return mergedProperties.getProperties();
    }
}
```
и добавить метод вытягивающие все сообщения по запрошенному языку (при смене локали ва нашем вэб-приложении мы будем его вызывать).

Также нам понадобятся хендлеры для Spring Security что бы избежать ненужных нам в REST редиректов
и томкатовских ошибок и получать подходящие ответы для фронта в json.

### Spring Security Handlers
**[AuthenticationEntryPoint](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/handler/CustomRestAuthenticationEntryPoint.java)** -  При неаутентифицированном доступе к закрытому ресурсу мы будем возвращать статус с ошибкой JSON'е.
```Java
public class CustomRestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {

//        As we don't need Tomcat HTML Exception we override behavior
//        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");

        //new behavior produces JSON Exception
        ErrorInfo errorDTO = new ErrorInfo(authException.getClass().getSimpleName(), authException);
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.setCharacterEncoding("utf-8");
        response.getWriter().print(JsonUtil.writeValue(errorDTO));
    }
}
```

**[AccessDeniedHandler](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/handler/CustomAccessDeniedHandler.java)** - так же переопределяем поведении при выбрасывании исключения **AccessDeniedException**
```Java
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    protected static final Logger logger = LoggerFactory.getLogger(CustomAccessDeniedHandler.class);

    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException,
            ServletException {
//        We don't need tomcat html exception so we override behavior
//        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized");

//        New behavior

        PrintWriter writer = response.getWriter();
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.setCharacterEncoding("utf-8");
    }
}
```

**[CustomLogoutSuccessHandler](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/handler/CustomLogoutSuccessHandler.java)** - при логауте просто отдаем HTTP 200.
```Java
public class CustomLogoutSuccessHandler extends SimpleUrlLogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {

        response.setStatus(HttpServletResponse.SC_OK);

    }
}
```

**[CustomSavedRequestAwareAuthenticationSuccessHandler](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/handler/CustomSavedRequestAwareAuthenticationSuccessHandler.java)** - после успешного логина нам не нужны никакие редиректы.
```Java
public class CustomSavedRequestAwareAuthenticationSuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    private RequestCache requestCache = new HttpSessionRequestCache();


    @Override
    public void onAuthenticationSuccess(final HttpServletRequest request, final HttpServletResponse response, final Authentication authentication) throws ServletException, IOException {
        final SavedRequest savedRequest = requestCache.getRequest(request, response);

        if (savedRequest == null) {
            clearAuthenticationAttributes(request);
            return;
        }
        final String targetUrlParameter = getTargetUrlParameter();
        if (isAlwaysUseDefaultTargetUrl() || (targetUrlParameter != null && StringUtils.hasText(request.getParameter(targetUrlParameter)))) {
            requestCache.removeRequest(request, response);
            clearAuthenticationAttributes(request);
            return;
        }

        clearAuthenticationAttributes(request);

        //WE DO NOT NED REDIRECT
        // So disable the code below
        // final String targetUrl = savedRequest.getRedirectUrl();
        // logger.debug("Redirecting to DefaultSavedRequest Url: " + targetUrl);
        // getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }

    public void setRequestCache(final RequestCache requestCache) {
        this.requestCache = requestCache;
    }

}
```

**[CustomUrlAuthenticationFailureHandler](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/java/ru/javawebinar/topjava/web/handler/CustomUrlAuthenticationFailureHandler.java)** - после неуспешной авторизации, так же нам нужен только HTTP-ответ.

```Java
public class CustomUrlAuthenticationFailureHandler extends SimpleUrlAuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {

        PrintWriter writer = response.getWriter();
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType("application/json");
        response.setCharacterEncoding("utf-8");
        writer.write(JsonUtil.writeValue(new ErrorInfo(request.getRequestURI(), exception)));
    }
}
```

###[web.xml](https://github.com/12ozCode/topjava08-to-angular2/blob/master/src/main/webapp/WEB-INF/web.xml)
- Теперь перенесем наш REST с корня на **/topjava-rs/**, статические ресурсы сайта будут лежать в корне
```xml
    <!-- Spring MVC -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <servlet>
        <servlet-name>mvc-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/spring-mvc.xml</param-value>
        </init-param>
        <init-param>
            <param-name>throwExceptionIfNoHandlerFound</param-name>
            <param-value>true</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>mvc-dispatcher</servlet-name>
        <url-pattern>/topjava-rs/*</url-pattern>
    </servlet-mapping>
```

```xml
    <!-- Spring Security -->
    <filter>
        <filter-name>springSecurityFilterChain</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>springSecurityFilterChain</filter-name>
        <url-pattern>/topjava-rs/*</url-pattern>
    </filter-mapping>
```

Так, вроде, все.

## Angular 2
Можно переходить к ангулару.
Angular 2 это полностью переписанный с 0 фреймворк. Который по заверениям разработчиков и тестам стал намного быстрее.
Теперь здесь используется TypeScript который дает нам возможность использовать статическую типизацию, классы
и нормальный компонентный подход. Конечно, в Angular 2 можно еще писать на Dart или чистом JavaScript. Но мне не совсем понятен смысл, плюс весь материал который вы
сможете найти в сети, скорее всего будет написан на TypeScript. Кроме того Angular 2 это не только ценный мех но и полноценный фреймворк, это значит, что скорее всего
то что вам понадобиться уже будет в коробке. И вам не придется сшивать очередного франкенштейна с разных библиотек. Для пессимистов скажу, что TypeScript транспилируется во
вполне читаемый JavaScript, который при надобности реально подредактировать руками.

Все в Angular 2 является компонентом который описывается классом с метаданными.

**Component**
Например типичный компонент выглядит примерно так :
```typescript
@Component({
  selector: 'my-component',
  template: '<div>Hello my name is {{name}}. <button (click)="sayMyName()">Say my name</button></div>'
})
export class MyComponent {
  constructor() {
    this.name = 'Tom'
  }
  sayMyName() {
    console.log('My name is', this.name);
  }
}
```
Это класс с метаданными в функции-декораторе.
В декораторе **@Component** описаны
 1. **selector** - Селектор который используется для отрисовки данного компонента на странице (yна страницу помещается кастомный тег с этим именем внутри которого и будет отрисован шаблон компонента).
 2. **templateUrl\templateUrl** - относительная ссылка на html-шаблон компонента или шабло инлайн как в примере.
 Шаблон имеет доступ ко всем методам и переменным своего компонента.

**Service**
Есть еще другие виды компонентов например сервисы они также описываются класом, но не имеют html-шаблона.
Например
```typescript
@Injectable()
export class HeroService {
    doSmth() {
        console.log('I'm working');
    }
}
```
Может использоваться как зависимость для предоставления данных компонентам. @Injectable() нужен для инжектора Angular 2. Этот декоратор указывает, что этот сервис может требовать
в качестве зависимостей другие сервисы. Если у сервиса нету в зависимостях других сервисов, @Injectable() можно опускать, чего делать не рекомендуется.

**Attribute directives**
Может использоваться для изменения поведения или вида элемента шаблона.
Пример использования:
```html
<p [style.background]="'lime'">I am green with envy!</p>
```

```typescript
@Directive({ selector: '[myHighlight]' })
export class HighlightDirective {
    constructor(el: ElementRef) {
       el.nativeElement.style.backgroundColor = 'yellow';
    }
}
```
Имеет свой декоратор и селектор который потом используется в элементе шаблона, наша директива получает управление над элементом.

Мы рассмотрели основные "кубики" из которых строится приложение в Angular 2. Далее будет логичным рассмотреть рабочее приложение на примере нашего проекта.

Для этого нам понадобиться установленный [NodeJS](https://nodejs.org/). Если вы раньше не имели опыта общение с ним, ничего страшного там нет, качаем версию LTS и кликаем далее.

Все эти конфигурационные файлы находятся в корне папки **webapp** нашего проекта.

Создаем файл с зависимостями для NPM.
Тут можно провести параллель с .pom файлом в Java для Maven если вы знакомы с Java.
Так же можно заметить, наличие PrimgNG в зависимостях. Это набор старых, добрых компонентов из PrimeFaces, написанных на Angualar 2. Мы будем использовать эти готовые компоненты в нашем приложении такие как:

- **datatable** (вывод данных)
- **calendar** (выбор даты и времени)
- **growl** (вывод ошибок)

[**package.json**](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/webapp/package.json)
```JavaScript
{
  "name": "angular-quickstart",
  "version": "1.0.0",
  "scripts": {
    "start": "tsc && concurrently \"tsc -w\" \"lite-server\" ",
    "lite": "lite-server",
    "tsc": "tsc",
    "tsc:w": "tsc -w"
  },
  "licenses": [
    {
      "type": "MIT",
      "url": "https://github.com/angular/angular.io/blob/master/LICENSE"
    }
  ],
  "dependencies": {
    "@angular/common": "~2.2.0",
    "@angular/compiler": "~2.2.0",
    "@angular/core": "~2.2.0",
    "@angular/forms": "~2.2.0",
    "@angular/http": "~2.2.0",
    "@angular/platform-browser": "~2.2.0",
    "@angular/platform-browser-dynamic": "~2.2.0",
    "@angular/router": "~3.2.0",

    "typescript": "2.0.10",
    "bootstrap": "^3.3.7",
    "core-js": "^2.4.1",
    "primeng": "^1.0.0-rc.6",
    "reflect-metadata": "^0.1.8",
    "rxjs": "5.0.0-beta.12",
    "systemjs": "0.19.39",
    "zone.js": "^0.6.25"
  },
  "devDependencies": {
    "gulp": "^3.9.1",
    "@types/core-js": "^0.9.34",
    "@types/node": "^6.0.45",
    "concurrently": "^3.0.0",
    "lite-server": "^2.2.2",
    "typescript": "^2.0.3"
  }
}
```

Файл с настройками TypeScript, с того что важно отметить здесь указывается:

- в какую версию JavaScript транспилировать код, лучше оставить в es5 для сохранения совместимости
- какую систему модулей JavaScript использовать
- **"sourceMap": true** генерировать ли специальный маппинг файлы для транспилированного JS, с помощью их вы сможете дебажить сразу красивый и приятный TypeScript как буд-то он и выполняется.
- **outDir** папка куда складывать готвоый JavaScript

[**tsconfig.json**](https://github.com/12ozCode/topjava08-to-angular2/blob/angular2/src/main/webapp/tsconfig.json)
```javascript
{
  "compilerOptions": {
    "target": "es5",
    "module": "commonjs",
    "moduleResolution": "node",
    "sourceMap": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "removeComments": false,
    "noImplicitAny": false,
    "outDir": "./app"
  }
}
````

И конфигурация для нашего загрузчика модулей SystemJS, из того что стоит отметить:

- Здесь есть маппинг наших импортов на расположение этих зависимостей в **node_modules**, здесь есть то что мы используем то есть angular, rxjs, primeng
- Также мы указываем папку нашего приложения, ту которую указали в настройках TypeScript, а именно **app: 'app'**

_**systemjs.config.js**_
```javascript
/**
 * System configuration for Angular samples
 * Adjust as necessary for your application needs.
 */
(function (global) {
    System.config({
        transpiler: false,
        paths: {
            // paths serve as alias
            'npm:': 'node_modules/'
        },
        // map tells the System loader where to look for things
        map: {
            // our app is within the app folder
            app: 'app',
            // angular bundles
            '@angular/core': 'npm:@angular/core/bundles/core.umd.js',
            '@angular/common': 'npm:@angular/common/bundles/common.umd.js',
            '@angular/compiler': 'npm:@angular/compiler/bundles/compiler.umd.js',
            '@angular/platform-browser': 'npm:@angular/platform-browser/bundles/platform-browser.umd.js',
            '@angular/platform-browser-dynamic': 'npm:@angular/platform-browser-dynamic/bundles/platform-browser-dynamic.umd.js',
            '@angular/http': 'npm:@angular/http/bundles/http.umd.js',
            '@angular/router': 'npm:@angular/router/bundles/router.umd.js',
            '@angular/forms': 'npm:@angular/forms/bundles/forms.umd.js',
            // other libraries
            'rxjs':                      'npm:rxjs',
            'angular-in-memory-web-api': 'npm:angular-in-memory-web-api',
            'primeng':                    'node_modules/primeng'
        },
        // packages tells the System loader how to load when no filename and/or no extension
        packages: {
            app: {
                main: './boot.js',
                defaultExtension: 'js'
            },
            rxjs: {
                defaultExtension: 'js'
            },
            'angular-in-memory-web-api': {
                main: './index.js',
                defaultExtension: 'js'
            },
            'primeng': { defaultExtension: 'js' }
        }
    });
})(this);
```
Готово, теперь, не уходя с webapp вводим в консоль **_npm install_** и NPM (пакетный менеджер NodeJS) начнет скачивать наши зависимости описанные в **_package.json_**
создав папку node_modules, и аккуратно все сложив туда.

Теперь переходим к самому интересному. Начинаем трогать **Angular 2**.
Для этого нам понадобиться создать index.html на котором у нас и будет располагаться родительский компонент нашего приложения.

```html
<html>
<head>
    <title>Angular QuickStart</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- 1. Load libraries -->
    <!-- Polyfill(s) for older browsers -->
    <script src="node_modules/core-js/client/shim.min.js"></script>
    <script src="node_modules/zone.js/dist/zone.js"></script>
    <script src="node_modules/reflect-metadata/Reflect.js"></script>
    <script src="node_modules/systemjs/dist/system.src.js"></script>
    <!-- 2. Configure SystemJS -->
    <script src="systemjs.config.js"></script>
    <script>
        System.import('app').catch(function (err) {
            console.error(err);
        });
    </script>

    <link rel="stylesheet" href="resources/css/style.css">
    <link rel="stylesheet" href="resources/css/spinner.css">

    <!--primeng-->
    <link rel="stylesheet" type="text/css" href="node_modules/primeng/resources/themes/omega/theme.css"/>
    <link rel="stylesheet" type="text/css"
          href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css"/>
    <link rel="stylesheet" type="text/css" href="node_modules/primeng/resources/primeng.min.css"/>

    <!-- Latest compiled and minified CSS -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css"
          integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
    <!-- Optional theme -->
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css"
          integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" crossorigin="anonymous">

    <link rel="shortcut icon" href="resources/images/icon-meal.png">
</head>
<!-- 3. Display the application -->
<body>
<app>
    <div class="spinner">
        <div class="cube1"></div>
        <div class="cube2"></div>
    </div>
</app>
</body>
</html>
```

В **\<body>** страницы мы можем заметить кастомный тег **\<app>**. Это и есть селектор для ангулара. Контент которого будет заменен при загрузке приложения,
а пока оно грузиться - будет отображаться спиннер.

**В Angular 2** приложение состоит из модулей, так как у нас довольно простенькое приложение, оно будет состоять из одного модуля.
Итак создадим наше первый модуль:

```typescript
@NgModule({
    imports: [BrowserModule, FormsModule, ReactiveFormsModule, HttpModule, routing, CalendarModule, DataTableModule, GrowlModule],
    declarations: [AppComponent, MealListComponent, EntryComponent,
        EditMealComponent, HeaderComponent,
        ProfileComponent,
        RegisterComponent,
        UserListComponent,
        UserEditComponent,
        I18nPipe
    ],
    bootstrap: [AppComponent],
    providers: [AuthService, AuthActivateGuard, MealService, UserService, ProfileService,
        I18nService, DateTimeTransformer, DatePipe, ExceptionService,
        {
            provide: APP_INITIALIZER,
            // useFactory: (i18NService: I18nService) => () => i18NService.initMessages(I18Enum.ru),
            // or
            useFactory: initApp,
            deps: [I18nService],
            multi: true
        }
    ]
})
export class TopJavaModule {

}

export function initApp(i18nService: I18nService) {
    // Do initing of services that is required before app loads
    // NOTE: this factory needs to return a function (that then returns a promise)
    return () => i18nService.initMessages(I18Enum.ru);  // + any other services...
}
```

Здесь на декораторе **@NgModule**  нужно остановиться поподробнее.
1. В параметре **imports** мы перечисляем все импортируемые с зависимостей модули которые понадобятся нам в нашем приложении.
2. В **declarations** мы объявляем все наши компоненты, компонент в **Angular 2** это сущность которая имеет свой view (html template)
и несет в себе логику.
3. В **bootstrap** мы объявляем корневой компонент нашего приложения, который будет размещать на себе все остальные компоненты.
4. в **providers** мы объявляем наши сервисы, singleton сущности которые доступны в любом месте нашего приложения как зависимости.
Это настоящий DI (Dependency injection) в Angular 2 (кто работал с  IoC контейнером - Spring или другим) может провести параллели.
Для получения зависимости в компонент достаточно потребовать ее в конструкторе. Конечно DI в Angular 2 намного мощнее чем я здесь описываю, но это уже отдельная тема для разговора.
5. Как вы наверное заметили один из компонентов, а именно **i18NService** имеет свою стратегию создания, но об этом мы поговорим дальше.

Теперь создадим файл с загрузчиком нашего модуля

```javascript
platformBrowserDynamic().bootstrapModule(TopJavaModule);
```

Он и загружает наше приложения, вы могли заметить что в конфиге SystemJS (systemjs.config.js) есть кусочек
```javascript
app: {
  main: './boot.js',
  defaultExtension: 'js'
},
```
который ссылается на уже транспилированный (переведенный с TypeScript в JavaScript код) и показывает откуда стартовать наше приложение.

Перед тем как перейти к первому компоненту осталось ознакомиться с нашим конфигом роутера. Мы используем старую **#(хэш)** стратегию так как она не требует дополнительной настройки со стороны сервера.

app.routes.ts
```typescript
const appRoutes: Routes = [
    {
        path: "",
        pathMatch: "full",
        redirectTo: "/meal-list",
    },
    {
        path: "login",
        component: EntryComponent,
    },
    {
        path: "register",
        component: RegisterComponent
    },
    {
        path: "meal-list",
        component: MealListComponent,
        canActivate: [AuthActivateGuard],
    },
    {
        path: "profile",
        component: ProfileComponent,
        canActivate: [AuthActivateGuard]
    },
    {
        path: "user-list",
        component: UserListComponent,
        canActivate: [AuthActivateGuard]
    }
];

export const routing: ModuleWithProviders = RouterModule.forRoot(appRoutes, {useHash: true});
```
Тут компоненты замаплены на пути по которым они доступны, так же есть параметр **canActivate** который принимает реализацию интерфейса
**CanActivate** смысл в том, что при попытке доступа к таким защищенным компонентам, сначала выполнится логика в **CanActivate**
и только при позитивном ответе будет осуществлен переход в роутере.

Переходим к компонентам

AppComponent
Наш родительский компонент.

 Теперь интереснее, взглянув на шаблон нашего родительского компонента нужно понимать что:

 1. **<header-component></header-component>** является селектором дочернего компонента, который будет всегда отображаться в нашем приложении
 в демо приложения это наш хидер, который выпоняет функции логин формы и не только.
 2. **<router-outlet></router-outlet>** является специальной дериктивой, внутри этих тегов будет отображен тот контент который соответствует
 URL положении на сайте. То есть если мы находимся на /#/meal-list, то внутри этих тегов будет отрисован компонент meal-list, если
 /#/profile то компонент profile.
 3. **[innerHTML]** это специальная директива которая дает возможность нам отрисовать полученный например с сервера, готовый html-код
 внутри контейнера.

 Начнем с HeaderComponent

 ```TypeScript
 @Component({
     templateUrl: '../../../templates/auth/header.html',
     selector: 'header-component',
     styleUrls: ["../../../resources/css/i18n.css"]
 })
 export class HeaderComponent implements OnInit {

     private errors: ErrorModel[] = [];

     loginForm: FormGroup = this.formBuilder.group({
         "login": ["", Validators.required],
         "password": ["", Validators.required]
     });

     constructor(private authService: AuthService,
                 private router: Router,
                 private formBuilder: FormBuilder,
                 private i18Service: I18nService,
                 private exceptionService: ExceptionService) {
     }

     ngOnInit(): void {
         this.exceptionService.errorStream.subscribe(
             e => {
                 this.errors.push(e);
             }
         )
     }

     onLogout() {
         this.authService.logout();
         this.router.navigate(["login"]);
     }

     onSubmit() {
         this.authService.login(this.loginForm.value);
     }

     chooseEng() {
         this.i18Service.reloadLocale(I18Enum.en);
     }

     chooseRu() {
         this.i18Service.reloadLocale(I18Enum.ru);
     }
 }
 ```

 1. Мы уже рассматривали функцию-декоратор **@Component** добавился только параметр **styleUrls** который ссылается на стили для этого компонента
 2. В TypeScript создание класса выглядит следующим образом
 **export class HeaderComponent implements OnInit**
 ключевое слово **export** делает доступным класс в контексте нашего приложения, наш класс имплементирует один из интерфейсов Angular 2
 **OnInit** это один из так называемых [LIFECYCLE HOOKS](https://angular.io/docs/ts/latest/guide/lifecycle-hooks.html) это значит что при создании компонента
 Angular 2 вызовет метод **OnInit** сразу после конструктора, в котором мы может доинициализировать наш компонент, тем что требуется для его корректной работы.

 ```TypeScript
private errors: ErrorModel[] = [];
 ```

 Это массив с ошибками, когда происходит какое-то исключение в приложении в этот массив мы добавляем объект типа ErrorModel,
 при появлении новой ошибки в массиве компонент [growl](http://www.primefaces.org/primeng/#/growl) из [primeng](http://www.primefaces.org/primeng) который объявлен в шаблоне хидера
```TypeScript
<p-growl [value]="errors"></p-growl>
```
ее отображает на странице.
Синтаксис **[value]="errors"** значит что кроме объявления этого компонента в нашем хидере мы передаем ему наш массив с ошибками за которым он и будет следить выводя их на экран.

```TypeScript
     loginForm: FormGroup = this.formBuilder.group({
         "login": ["", Validators.required],
         "password": ["", Validators.required]
     });
```

Здесь мы используем model-driven forms из Angular 2
То есть мы создаем объект-отражение нашей формы в шаблоне. Отличие от template-driven forms в том что мы описываем логику формы в этом объекте,
а не на странице через атрибуты. Единственное что нужно сделать в шаблоне  - это связать форму с нашим объектом через специальную директиву
**[formGroup]="loginForm"** и ее составляющие, например **formControlName="login"**
подробнее можно почитать [здесь](http://blog.angular-university.io/introduction-to-angular-2-forms-template-driven-vs-model-driven/)

в билдер передается объект, в котором ключ - название поля формы, а значение должно быть массивом, в котором первый элемент это дефолтное значение поля после создания, а следующий валидатор валидаторы можно объеденять через Validators.compose(), который принимает массив валидаторов.

Конструктор компонента, во входящим компоненте мы указываем зависимости которые Angular'овский DI должен нам заинжектить в качестве параметров, на этапе создания, если какой-то из зависимостей не окажется созданной, мы получим ошибку.
**private** в параметрах конструктора это синтаксический сахар, который сразу объявляет поля класса с таким же названием и присваивает им соответствующие входящие параметры, делая их доступными на уровне класса.
синтаксис имя_переменной:Тип, типа можно опускать практически везде, здесь же он нам нужен что бы Angular 2 понимал какую зависимость мы хотим.
```TypeScript
    constructor(private authService: AuthService,
                private router: Router,
                private formBuilder: FormBuilder,
                private i18Service: I18nService,
                private exceptionService: ExceptionService) {
    }
```

Рекомендуется использовать конструктор для получения зависимостей а инициализацию компонента переносить в один из хуков например в
```TypeScript
    ngOnInit(): void {
        this.exceptionService.errorStream.subscribe(
            e => {
                this.errors.push(e);
            }
        )
    }
```
Здесь мы используем [RxJS](https://github.com/Reactive-Extensions/RxJS) который повсеместно используется в Angular 2. У нас есть сервис **ExceptionService** который грубо говоря содержит очередь
**errorStream**, при создании нашего компонента HeaderComponent мы подписываемся на эту очередь и когда какой-либо другой компонент бросает туда ошибку, мы перебрасываем ее в наш массив **errors** который как
я показывал ранее используется в компоненте [growl](http://www.primefaces.org/primeng/#/growl) из [primeng](http://www.primefaces.org/primeng/) для вывода ошибок, после показа ошибки она из массива удаляется.

Метод который еще стоит рассмотреть это
```TypeScript
    onLogout() {
        this.authService.logout();
        this.router.navigate(["login"]);
    }
```
При нажатии кнопки Logout в шаблоне компонента вызывается этот метод,
1. Вызывается AuthService он посылает запрос к Spring Security и инвалидирует сессию.
2. Далее вызывается роутер который мы получили через DI в конструкторе и переводит роутер на компонент который соответствует URL'у **login**.
А это как мы видели
```TypeScript
    {
        path: "login",
        component: EntryComponent,
    },
```
В свою очередь он отображается в **<header-component></header-component>** нашего родительского компонента
```TypeScript
<header-component></header-component>

<router-outlet></router-outlet>

<div class="footer">
    <div class="container" [innerHTML]="'app.footer' | i18nPipe">

    </div>
</div>
```

Теперь рассмотрим шаблон
```html
<div class="navbar navbar-inverse navbar-fixed-top" role="navigation">
    <div class="container">

        <a [routerLink]="['/meal-list']" class="navbar-brand">{{'app.title' | i18nPipe}}</a>


        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav navbar-right">
                <li *ngIf="authService.authenticatedAs">
                    <div class="navbar-form">
                        <a *ngIf="authService.hasAdminRole()" class="btn btn-info" role="button"
                           [routerLink]="['/user-list']">{{'users.title' | i18nPipe}}</a>
                        <a class="btn btn-info" role="button" [routerLink]="['/profile']">{{authService.authenticatedAs.name}}
                            {{'app.profile' | i18nPipe}}</a>
                        <button type="button" class="btn btn-primary" (click)="onLogout()">{{'app.logout' | i18nPipe}}
                        </button>
                    </div>
                </li>
                <li *ngIf="!authService.authenticatedAs">
                    <form [formGroup]="loginForm" class="navbar-form" role="form" (ngSubmit)="onSubmit()"
                          method="post">
                        <div class="form-group">
                            <input type="text" formControlName="login" placeholder="{{'users.email' | i18nPipe}}" class="form-control">
                        </div>
                        <div class="form-group">
                            <input type="password" formControlName="password" placeholder="{{'users.password' | i18nPipe}}"
                                   class="form-control">
                        </div>
                        <button type="submit" class="btn btn-success">{{'app.login' | i18nPipe}}</button>
                    </form>
                </li>

                <li class="dropdown">
                    <button class="dropbtn">{{i18Service.getCurrentLocale()}}</button>
                    <div class="dropdown-content">

                        <a (click)="chooseEng()">English</a>

                        <a (click)="chooseRu()">Русский</a>

                    </div>
                </li>
            </ul>
        </div>
    </div>
    <p-growl [value]="errors"></p-growl>
</div>
```
По порядку
1. У нас есть псевдоссылка которая переводит пользователя на список с едой
 ```html
<a [routerLink]="['/meal-list']" class="navbar-brand">{{'app.title' | i18nPipe}}/</a>
```
Для этого используется директива **[routerLink]** которая принимает параметром желаемую ссылку из объявленных в конфиге нашего роутера. Контент тега мы разберем дальше когда будем говорить о интернационализации.
2. Дальше мы видим *ngIf это структурная директива Angular 2, она принимает параметром boolean выражение при позитивном результате блок разметки на котором есть эта директива будет показан, в противном случае будет не спрятан, а полностью выпилен из DOM.
```html
<li *ngIf="authService.authenticatedAs">
```
Здесь мы обращаемся к полю нашего сервиса **authenticatedAs**, которое хранит профиль залогиненого пользователя. Который мы получаем при первом вызове AuthActivateGuard. То есть при наличии профиля в данной переменной
будет показана кнопка **logout**, в противном - форма входа.
3. Также нужно рассмотреть кнопку **logout**
```html
<button type="button" class="btn btn-primary" (click)="onLogout()">{{'app.logout' | i18nPipe}}</button>
```
Синтаксис **(click)="onLogout()"** означает что мы слушаем событие кнопки **click** и вызываем метод нашего компонента onLogout().
4. В шаблонах много {{}} это так называемая [Interpolation](https://angular.io/docs/ts/latest/guide/template-syntax.html) внутри скобок вычисляется выражение и возвращается его результат, или например переменная.

В нашем приложении основная функциональность состоит в управлении едой, своим профилем и если мы админ - управлении юзерами. По своей структуре они имеют идентичную реализацию
**сервис <- компонент <- шаблон**. Потому каждый по отдельности рассматривать нету смысла. Рассмотрим Angular 2 на примере реализации одного из них.

Итак MealListComponent

```TypeScript
@Component({
    templateUrl: '../../../templates/meal/meal-list.html'
})
export class MealListComponent implements OnInit {

    startDate: Date;
    endDate: Date;
    startTime: Date;
    endTime: Date;

    mealsHolder: Observable<UserMeal[]>;

    @ViewChild(EditMealComponent)
    private editMealChild: EditMealComponent;

    constructor(private mealService: MealService,
                private formBuilder: FormBuilder) {
    }

    ngOnInit() {
        this.mealsHolder = this.mealService.loadAllMeals();
    }

    updateList() {
        this.mealsHolder = this.mealService.loadAllMeals();
    }

    deleteItem(meal: UserMeal) {
        this.mealService.deleteMeal(meal).subscribe(res => {
            this.updateList();
        });
    }

    showCreateModal() {
        this.editMealChild.resetForm();
        this.editMealChild.showToggle = true;
    }

    selectMeal(meal) {
        this.editMealChild.showToggle = true;
        this.editMealChild.fillMyForm(meal.data);
    }

    save(userMeal: UserMeal) {
        this.editMealChild.showToggle = false;
        this.mealService.save(userMeal).subscribe(
            res => {
                this.updateList();
            }
        );
    }

    onFilter() {
        this.mealsHolder = this.mealService.getFilteredDataSet(
            this.startDate,
            this.endDate,
            this.startTime,
            this.endTime);
    }

    clearFilters() {
        this.startDate = null;
        this.endDate = null;
        this.startTime = null;
        this.endTime = null;
        this.updateList();
    }

    getRowClass(rowData: UserMeal, rowIndex): string {
        return rowData.exceed ? "exceeded" : null;
    }
}
```

Что интересного можно здесь увидеть ?
1. Селектор здесь нам не нужен, так как компонент показывается роутером.
2. Переменные с датами используется для передачи в datePicker, где они будут инициализированы при выборе даты пользователем дальше в шаблоне мы это увидим.
3. Так как компонент выводит грит еды, то у нас в классе компонента есть поле **mealsHolder: Observable<UserMeal[]>**.
 Тип [Observable](http://reactivex.io/documentation/observable.html) это снова отсылка к [RxJS](https://github.com/Reactive-Extensions/RxJS) тип который возвращает любой запрос сделанный http-клиентом Angular 2.
Когда мы делаем http-запрос, он возвращает нам [Observable](http://reactivex.io/documentation/observable.html), но сам запрос будет послан только
 после вызова на этом объекте метода **subscribe()** который в в свою очередь принимает 3 коллбека.
```JavaScript
 let subscription = this.data.subscribe(
           value => this.values.push(value), //если запрос прошел без ошибок вызовется этот, value - результат
           error => this.anyErrors = true, // при ошибке этот
           () => this.finished = true // после завершения запроса
       );
```

Мы могли бы сделать вот так:
```JavaScript
...
receivedData = null;

 let subscription = this.data.subscribe(
           value => {
                this.receivedData = value.json(); // метод json() пытается спарсить body ответа в объект
           },
           error => this.anyErrors = true,
           () => this.finished = true
       );
```
А receivedData использовать в шаблоне. Но Angular 2 позволяет нам использовать в шаблонах [Observable](http://reactivex.io/documentation/observable.html) с помощью пайпов
[Pipe](https://angular.io/docs/ts/latest/guide/pipes.html). А именно [async](https://angular.io/docs/ts/latest/api/common/index/AsyncPipe-pipe.html)
Например как мы увидим дальше в шаблоне, когда нам нужно передать датасет в грид, мы может сделать вот так.
```html
<p-dataTable [value]="(mealsHolder | async)" ...>
```
Синтаксис очень прозрачный, наша переменная [Observable](http://reactivex.io/documentation/observable.html) "|" и название пайпы. Смысл в том что наша переменная пройдет через **async pipe** и только потом будет передана в грид.
Дальше мы разберем создание пайпов, на примере интернационализации.

4. У нас есть потребность в модальном окне которое будет открываться при редактировании или добавлении новых записей. Сделать это можно по-разному.
В частном случае у нас есть компонент который представляет собой это модальное окно, хотя мы могли бы часть разметки хранить в этом же компоненте, я предпочел вынести его. Что бы показать как работают события в Angular 2.
Angular 2 позволяет нам получить ссылку на компонент который размещен в нашем шаблоне, через функцию-декоратор @ViewChild(EditMealComponent) здесь мы передаем ей класс компонента которого мы хотим получить.
```TypeScript
@ViewChild(EditMealComponent)
private editMealChild: EditMealComponent;
```
Мы имеем полный доступ к дочернему компоненту, что дает нам возможность вызывать его методы. А это нам нужно что бы
 a) Показать\спрятать компонент
 b) Заполнить форму компонента выбранной с грида записью, для редактирования.

Далее ничего нового в компоненте мы не увидим, переходим к шаблону.
```html
<div class="jumbotron">
    <div class="container">
        <div class="shadow">
            <h3>{{'meals.title' | i18nPipe}}</h3>
            <div class="view-box">

                <div class="form-horizontal" role="form">
                    <div class="form-group">
                        <label class="control-label col-sm-2" for="startDate">{{'meals.startDate' | i18nPipe}}</label>

                        <div class="col-sm-3" id="startDate">
                            <p-calendar [(ngModel)]="startDate" dateFormat="yy-mm-dd"
                                        [showIcon]="true"></p-calendar>
                        </div>


                        <label class="control-label col-sm-2" for="endDate">{{'meals.endDate' | i18nPipe}}</label>

                        <div class="col-sm-3" id="endDate">
                            <p-calendar [(ngModel)]="endDate" dateFormat="yy-mm-dd"
                                        [showIcon]="true"></p-calendar>
                        </div>

                        <button type="button" class="btn btn-primary pull-right" (click)="onFilter()">
                            {{'meals.filter' |
                            i18nPipe}}
                        </button>
                    </div>


                    <div class="form-group">
                        <label class="control-label col-sm-2" for="startTime">{{'meals.startTime' | i18nPipe}}</label>

                        <div class="col-sm-3" id="startTime">
                            <p-calendar [(ngModel)]="startTime" timeOnly="true" hourFormat="24"
                                        [showIcon]="true"></p-calendar>
                        </div>


                        <label class="control-label col-sm-2" for="endTime">{{'meals.endTime' | i18nPipe}}</label>

                        <div class="col-sm-3" id="endTime">
                            <p-calendar
                                    [(ngModel)]="endTime"
                                    timeOnly="true"
                                    showTime="showTime"
                                    hourFormat="24"
                                    [showIcon]="true">
                            </p-calendar>
                        </div>

                        <button type="button" class="btn btn-danger pull-right" (click)="clearFilters()">
                            {{'meals.clearFilter' | i18nPipe}}
                        </button>
                    </div>

                    <br/>

                    <p-dataTable [value]="(mealsHolder | async)" selectionMode="single" [rowStyleClass]="getRowClass"
                                 (onRowSelect)="selectMeal($event)" [paginator]="true" rows="10" [responsive]="true">

                        <p-column field="dateTime" [sortable]="true" header="{{'meals.dateTime' | i18nPipe}}">
                            <template let-meal="rowData" pTemplate type="body">
                                {{meal.dateTime | date:'y-MM-dd HH:mm'}}
                            </template>
                        </p-column>

                        <p-column field="description" header="{{'meals.description' | i18nPipe}}" [filter]="true"
                                  [sortable]="true"></p-column>
                        <p-column field="calories" header="{{'meals.calories' | i18nPipe}}"
                                  [sortable]="true"></p-column>

                        <p-column [style]="{width : '120px'}">
                            <template let-meal="rowData" pTemplate type="body">
                                <button type="button" pButton (click)="deleteItem(meal)"
                                        label="{{'common.delete' | i18nPipe}}"></button>
                            </template>
                        </p-column>
                        <footer>
                            <div class="ui-helper-clearfix" style="width:100%">
                                <button type="button" pButton icon="fa-plus" style="float:left"
                                        (click)="showCreateModal()" label="{{'meals.add' | i18nPipe}}"></button>
                            </div>
                        </footer>
                    </p-dataTable>
                </div>
            </div>
        </div>
    </div>
</div>
<edit-meal (onSaveEvent)="save($event)"></edit-meal>
```
В шаблоне мы используем [DataTable](http://www.primefaces.org/primeng/#/datatable) и [Сalendar](http://www.primefaces.org/primeng/#/calendar) от [PrimeNG](http://www.primefaces.org/primeng/) в шаблоне компонента
есть полный доступ ко всем его переменным и методам  мы передаем в календарь переменные для дат фильтрации из нашего компонента, когда пользователь выберет дату и время они будут инициализированы. На передаче датасета в грид мы уже останавливались,
единственное это **[rowStyleClass]="getRowClass"**  в этот параметр грида мы передаем функцию из нашего компонента, при отрисовке поля, функция вызывается и на основе переданных переданных в нее данных отдает **CSS** класс
для поля. Здесь мы используем это для подкрашивания строк  которые превышают норму калорий в день. На API PrimeNG смысла останавливаться нету, лучше и больше можно узнать на их сайте.
И последняя строка
```html
 <edit-meal (onSaveEvent)="save($event)"></edit-meal>
```
 это наш компонент модального окна. **(onSaveEvent)="save($event)"** значит что мы слушаем event **onSaveEvent** (имя EventEmitter'a) и при выбрасывания ивента передаем его в функцию компонента **save()**.

 Теперь давайте поближе взглянем на компонент модального окна

```TypeScript
 @Component({
     templateUrl: '../../templates/meal/meal-edit.html',
     selector: 'edit-meal'
 })
 export class EditMealComponent implements OnInit {

     showToggle: boolean = false;

     mealForm: FormGroup;

     @Output()
     onSaveEvent: EventEmitter<UserMeal> = new EventEmitter<UserMeal>();

     constructor(private formBuilder: FormBuilder) {
     }

     ngOnInit(): void {
         this.mealForm = this.formBuilder.group({
             id: [''],
             dateTime: [null, Validators.required],
             description: [``, Validators.required],
             calories: [``, Validators.compose([Validators.required, ValidateUtil.validateMealCalories])]
         });
     }

     fillMyForm(userMeal: UserMeal) {
         this.mealForm.patchValue(userMeal);
     }

     resetForm() {
         this.mealForm.reset();
     }

     onSave() {
         let value = this.mealForm.value;
         this.onSaveEvent.emit(value);
         this.mealForm.reset();
     }
 }
 ```

1. **showToggle** это переключатель видимости окна, к нему привязана *ngIf директива которую мы рассматривали раньше. Таким образом мы можем управлять видимостью компонента с родителя.
2. **mealForm** форма нашего модального окна
3. EventEmitter специальный объект который "излучает" events, типизируется типом который буду нести в себе евенты в качестве информации, в нашем случае объект добавленной или отредактированной UserMeal.
Что бы создать свой собственный event достаточно создать EventEmitter помеченный функцией-декоратором @Output(), и вызвать на этом объекте emit(). Если родительский компонент должен "слушать" данный event и обрабатывать.
4. При создании компонента мы заполняем форму пустыми значениями.
5. Добавляем валидаторы стандартный, делающий поле - обязательным и свой ValidateUtil.validateMealCalories.
6. При выборе записи в гриде в родительском компоненте мы заполняем форму через **fillMyForm(userMeal: UserMeal)** и показываем ее.
7. **resetForm()** сбрасывает поля на пустые.
8. При сохранении **onSave()** мы "излучаем" event который обрабатывает родительский компонент и отправляет в сервис.

Далее рассмотри **MealService**

```TypeScript
@Injectable()
export class MealService {

    constructor(private http: Http,
                private dateTimeTransformer: DateTimeTransformer) {
    }

    loadAllMeals(): Observable<UserMeal[]> {
        return this.http.get(basePath + mealPath, reqOptions)
            .map((res) => {
                return this.mapResponse(res);
            });
    }

    deleteMeal(meal: UserMeal): Observable<Response> {
        return this.http.delete(basePath + mealPath + '/' + meal.id, reqOptions);
    }

    mapResponse(resp) {
        let mealsList: UserMeal[] = resp.json();
        mealsList.forEach((meal: UserMeal) => {
            meal.dateTime = this.dateTimeTransformer.deserializeDateTime(meal.dateTime);
        });
        return mealsList;
    }

    save(userMeal: UserMeal): Observable<Response> {
        userMeal.dateTime = this.dateTimeTransformer.serializeDateTime(userMeal.dateTime);
        if (userMeal.id) {
            return this.update(userMeal);
        }
        else {
            return this.http.post(basePath + mealPath, JSON.stringify(userMeal), reqOptionsJson);
        }
    }

    private update(userMeal: UserMeal): Observable<Response> {
        console.log('saved! (actually not..)');
        return this.http.put(basePath + mealPath + '/' + userMeal.id, JSON.stringify(userMeal), reqOptionsJson);
    }

    getFilteredDataSet(startDate: Date, endDate: Date, startTime: Date, endTime: Date) {
        let getParams = new URLSearchParams();

        if (startDate != null) {
            getParams.set('startDate', this.dateTimeTransformer.serializeDate(startDate));
        }
        if (endDate != null) {
            getParams.set('endDate', this.dateTimeTransformer.serializeDate(endDate));
        }
        if (startTime != null) {
            getParams.set('startTime', this.dateTimeTransformer.serializeTime(startTime));
        }
        if (endTime != null) {
            getParams.set('endTime', this.dateTimeTransformer.serializeTime(endTime));
        }

        var clone: RequestOptionsArgs = Object.assign({}, reqOptions);
        clone.search = getParams;

        return this.http.get(basePath + mealPath + '/' + 'filter', clone).map((res) => {
            return this.mapResponse(res);
        });
    }
}
```
MealService используется для общения приложения с сервером по HTTP.
1. Первое что нужно понимать это - сервис. Он не имеет html-шаблона и декоратора @Component() он ничего не отображает. Его задача предоставлять функциональность компонентам.
2.
Но сами создатели рекомендуют всегда использовать этот декоратор.
3. Некоторые методы перед тем как вернуть Observable, вызывают на нем .map() этот метод позволяет изменить содержимое потока. Таким образом подписчик уже будет получать преобразованный результат. В нашем случае сразу готовый объект
а не сырой ответ сервера.
4. Для десериализации даты используется простенький класс DateTimeTransformer.
5. Все настройки и объекты RequestOptions которые нужны в любом завпросе вынесены в отдельный файл **config.ts**

```TypeScript
export const basePath: string = '/topjava-rs';
export const loginPath: string = "/spring_security_check";
export const mealPath: string = '/rest/profile/meals';
export const profilePath: string = '/rest/profile';
export const registerPath: string = '/register';
export const usersPath: string = '/rest/admin/users';
export const i18nPath: string = '/i18n';


export const headers: Headers = new Headers({
  'Content-Type': 'application/json'
});
export const reqOptions: RequestOptions = new RequestOptions({
  withCredentials: true
});
export const reqOptionsJson: RequestOptions = new RequestOptions({
  withCredentials: true,
  headers: this.headers
});
```
Для того что бы посылать вместе с запросами coockie, нужно в **RequestOptions** указать **withCredentials: true**

Далее **AuthService**
```TypeScript
@Injectable()
export class AuthService {

    private _authenticatedAs: UserModel = null;

    constructor(private http: Http,
                private router: Router,
                private profileService: ProfileService,
                private exceptionService: ExceptionService) {
    }

    login(token: Token): void {

        let headers: Headers = new Headers({'Content-Type': 'application/x-www-form-urlencoded;charset=utf-8'});
        let options = new RequestOptions({
            headers: headers,
            withCredentials: true
        });
        this.http.post(basePath + loginPath,
            "username=" + token.login +
            "&password=" + token.password,
            options).map(res => {
            return res;
        }).subscribe(
            res => {
                this.router.navigate(["meal-list"])
            },
            error => {
                this.exceptionService.onError(error);
            }
        );
    }

    get authenticatedAs(): UserModel {
        return this._authenticatedAs;
    }

    set authenticatedAs(value: UserModel) {
        this._authenticatedAs = value;
    }

    hasAdminRole(): boolean {
        return this._authenticatedAs.roles.includes("ROLE_ADMIN");
    }

    logout() {
        this.http.get(basePath + "/logout").subscribe();
        this._authenticatedAs = null;
    }

    isAuthenticated(): Observable<boolean> {
        if (this._authenticatedAs == null) {
            return this.profileService.getOwnProfile().map(res => {
                this._authenticatedAs = res.json();
                return true;
            }).catch((error: any) => {
                this._authenticatedAs = null;
                this.exceptionService.onError(error);
                return Observable.of(false);
            });
        } else {
            return Observable.of(true);
        }
    }
}
```
Простой, очень примитивный сервис для аутентификации.
1. При успешном логине происходит переход на meal-list. При ошибке, мы бросаем ошибку в общую очередь **ExceptionService.onError(error);** которая потом показывается в хидере.
2. Может показаться запутанным метод isAuthenticated() он используется в нашем ActivateGuard который защищает наши компоненты от
неавторизированного доступа. Если поле пустое (например заходж по ссылке с валидной сессией) мы запрашиваем свой профиль, при успехе сохраняем результат в поле и отдаем true (успех),
при ошибке, добавляем ошибку в очередь и отдаем false. Если же переменная инициализирована сразу отдаем true.

###RouteGuard's
**AuthActivateGuard** о котором мы вспоминали выше
```javascript
@Injectable()
export class AuthActivateGuard implements CanActivate {

    constructor(private authService: AuthService,
                private router: Router) {
    }

    canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): Observable<boolean> {
        return this.authService.isAuthenticated().map(res => {
            if (!res) {
                this.router.navigate(["login"]);
            }
            return res;
        });
    }
}
```
Является по сути тоже Singleton сервисом, который имплементирует интерфейс **CanActivate**, используется в нашем роутер-конфиге для проверки наличия прав доступа к определенному компоненту.
В нашем случае только на наличии авторизации, хотя эта часть может быть во много раз сложнее если в приложении много ролей и привилегий.
Здесь мы вызывает наш AuthService.isAuthenticated() и если он отдает false то переводим роутер на страницу логина. **CanActivate** может
возвращать как boolean значения так и Observable<boolean> (например для проверки которая требует запроса к серверу) и первое и второе
поддерживается. В частном случае для однообразностия использую везде Observable. Оборачивая даже обычные boolean.

###Pipes
В нашем приложении используется один **pipe** для интернационализации. Как вы виджели, там где нам нужно вывести сообщение мы используем код сообщения,
например _\<h2 class="modal-title">_**{{'meals.edit' | i18nPipe}}**_\</h2>_.
Наш pipe

```javascript
@Pipe({name: 'i18nPipe',
       pure: false})
export class I18nPipe implements PipeTransform {

    constructor(private i18Service: I18nService) {
    }

    transform(value: any, args: any): any {
        return this.i18Service.getMessage(value);
    }
}
```
получает ключ в параметре **value: any** и использует его для запроса нужного нам сообщения в **i18Service.getMessage(value)** в итоге
результатом нашей интерполяции является нужное нам сообщение. Pipes могу иметь также и параметры, например **{{birthday | date:'fullDate'}}**
и даже использоваться последовательно всего лишь добавив еще один **'|'** **{{birthday | date:'fullDate' | uppercase}}** так что это довольно мощный инструмент.
Но наш пайп не самый обычный, мы может видеть что в декораторе **@Pipe()** кроме указания имени еще используется параметр **pure**.
Все Pipes по умолчанию **stateless** то есть **(pure = true)** вызываются единоразово при рендеринге шаблона. Если же использовать **pure = false**
то наш пайп уже не будет **stateless** он будет следить за всеми значениями которые он через себя пропустил и при изменении например набора
сообщений в I18Service (при изменении языка) немедленно перерисовывать их. Это довольно опасная фича которую надо использовать с большой осторожностью,
так как это может серьезно ударить по производительности.

###Итого
Мне как человеку не имевшего дела с первым Angularo'ом (который на первый взгляд мне не нравился), нет возможности их сравнить,
 но второй очень нравится. Когда я решал какой из них начинать использовать, решил в пользу второго, прошел через все те изменения которые случились с ним
 с беты, переписанный роутер, формы и т.д.

Мне лично очень нравится: статическая типизация, красивый и понятный компонентный подход, удобное управление зависимостями.
Надеюсь этот туториал будет полезен, планирую дописать еще не большую часть про деплой всего этого добра на heroku.

