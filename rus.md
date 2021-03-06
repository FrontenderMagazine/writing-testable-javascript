Болеутоляющее или статья о том, что можно писать тестируемый JavaScript
============================================================

У всех наступал момент, когда ваше JavaScript-приложение, начавшееся с нескольких
полезных строчек разрасталось на тысячу строк, затем на две, дальше — больше.
Постепенно функция начинает принимать чуть больше параметров; ветки условий
получают ещё немного условий. И в один прекрасный день появляется баг: что-то
сломано. И нам предстоит распутать весь этот бардак в коде.

Сейчас код на фронтенде берёт на себя всё больше и больше ответственности — на
самом деле, уже существует целый пласт приложений, существующих полностью на
клиентской стороне — в связи со всем этим становятся очевидными две идеи. Первая — мы
просто не можем прокликать все возможные варианты с помощью текущего метода
тестирования. Вторая — возможно, нам потребуется изменить подход к тому, как 
мы привыкли писать код, в угоду возможности писать тесты.

Правда ли, что нам необходимо поменять то, как мы пишем код? Абсолютное да — так
как мы осознаём пользу автоматического тестирования, но большинство из нас,
возможно, сумеют прямо сейчас написать только интеграционные тесты. Интеграционные
тесты важны тем, что отслеживают, насколько хорошо работают между собой
отдельные части приложения, но в то же время они не несут никакой информации о
том, *работают ли отдельные части так, как от них ожидают*.

В этот момент на сцену действия выходят модульные тесты (прим. переводчика: также
известны как юнит-тесты и функциональные тесты). Нам придётся серьёзно
потрудиться над *написанием модульных тестов*, пока мы не начнём писать
*тестируемый джаваскрипт*.


## Модульные тесты и интеграционные: в чём разница? ##

Обычно способ создания интеграционных тестов достаточно прямолинеен: мы просто
пишем код, описывающий, как пользователь взаимодействует с нашим приложением, и
то, что пользователь ожидает увидеть. Популярным средством автоматического
тестирования в браузере является [Selenium][1]. [Capybara][2] для Ruby облегчает
взаимодействие с Selenium, более того, существуют тысячи подобных инструментов на
других языках.

Ниже представлен интеграционный тест для небольшой части поискового приложения:

    def test_search
      fill_in('q', :with => 'cat')
      find('.btn').click
      assert( find('#results li').has_content?('cat'), 'Результаты поиска отображены' )
      assert( page.has_no_selector?('#results li.no-results'), 'Результаты поиска отсутствуют' )
    end

В то время как интеграционный тест заинтересован в проверке взаимодействия
пользователя с приложением, внимание модульного теста лежит на небольших
участках кода.

> Если я вызову функцию с зафиксированными параметрами, то получу ли я ожидаемый
результат?

Приложения, написанные в традиционном процедурном стиле, чаще всего трудно
модульно тестировать — так же, как трудно поддерживать, отлаживать и расширять. Но
если мы будем писать код, держа в уме необходимость модульного тестирования, то мы
обнаружим не только то, что тестировать становится проще, но также то, что мы
просто пишем аккуратный и *более качественный код*.

В качестве иллюстрации к тому, о чём я говорю, давайте взглянем на обычное
поисковое приложение:

![Изображение][Srchr]

Когда пользователь начинает что-то искать, приложение отправляет XHR-запрос на
сервер. Когда сервер отвечает данными в формате JSON, приложение принимает эти
данные и отображает их на странице при помощи клиентской шаблонизации.
Пользователь может кликнуть на элементе поисковой выдачи, чтобы показать, что
ему понравился этот пункт; когда это происходит, имя человека, к которому
пользователь проявил интерес, добавляется в список «Понравившиеся» в правой
колонке приложения.

«Обычное» JavaScript приложение может выглядеть так:

    var tmplCache = {};

    function loadTemplate (name) {
      if (!tmplCache[name]) {
        tmplCache[name] = $.get('/templates/' + name);
      }
      return tmplCache[name];
    }

    $(function () {

      var resultsList = $('#results');
      var liked = $('#liked');
      var pending = false;

      $('#searchForm').on('submit', function (e) {
        e.preventDefault();

        if (pending) { return; }

        var form = $(this);
        var query = $.trim( form.find('input[name="q"]').val() );

        if (!query) { return; }

        pending = true;

        $.ajax('/data/search.json', {
          data : { q: query },
          dataType : 'json',
          success : function (data) {
            loadTemplate('people-detailed.tmpl').then(function (t) {
              var tmpl = _.template(t);
              resultsList.html( tmpl({ people : data.results }) );
              pending = false;
            });
          }
        });

        $('<li>', {
          'class' : 'pending',
          html : 'Идёт поиск…'
        }).appendTo( resultsList.empty() );
      });

      resultsList.on('click', '.like', function (e) {
        e.preventDefault();
        var name = $(this).closest('li').find('h2').text();
        liked.find('.no-results').remove();
        $('<li>', { text: name }).appendTo(liked);
      });

    });

Мой друг [Адама Сонтега][4] (Adam Sontag) называет это
«Выбери себе приключение сам» — в каждой строчке мы с равной вероятностью можем 
иметь дело как с представлением, так и с информацией, обслуживанием логики
пользовательского взаимодействия или проверкой состояния приложения. Остаётся
только догадываться! Достаточно просто написать интеграционные тесты для этого
кода, и в тоже время очень сложно написать тесты для тестирования отдельных
*функциональных частей* приложения.

Почему это сложно? На это существуют четыре причины:

* общий недостаток структурированности; чаще всего всё происходит в коллбэке
`$(document).ready()` — и, так как это анонимная функция, её невозможно
протестировать ввиду невозможного к ней обращения;
* слишком сложные функции; Если в функции больше 10 строк кода, как в обработчике
отправки формы, то эта функция, вероятно, выполняет и несёт отвественность за
слишком много вещей;
* скрытые состояния; так как состояние `pending` помещено в замыкание, то нет
никакой возможности проверить, правильно ли установлено это состояние;
* избыточная связанность функционала; к примеру, обработчику `$.ajax` совершенно
нет необходимости иметь доступ к DOM;


## Организация кода ##

Первое, что небходимо сделать — выбрать менее запутанный метод организации кода,
разбить его на несколько зон ответственности:

* представление и пользовательское взаимодействие;
* управление данными;
* общее состояние приложения;
* настройка и код-прослойка, чтобы все части работали вместе;

В «традиционной» реализации, показанной выше, эти четыре категории перемешаны —
на одной строчке мы работаем с представлением, двумя строчками ниже мы общаемся
с сервером.

![Изображение][строчки кода]

Несмотря на то, что мы можем без проблем писать интеграционные тесты для этого кода (и
мы обязаны это делать!), писать модульные тесты действительно сложно. В
наших функциональных тестах мы можем утверждать: «когда пользователь ищет что-то,
он должен видеть соответствующие результаты», но мы не можем быть точнее. Если
что-то пойдёт не так, нам следует определить, что именно пошло не так, и
наши функциональные тесты не смогут помочь в этом.

Если мы переосмыслим то, как мы пишем код, то мы можем не только написать юнит-
тесты, которые дадут нам лучшее представление о том, откуда все пошло не по плану,
но и, в конечном итоге, писать более удобный код — поддерживаемый, расширяемый.

Каждая новая строчка кода будет следовать этому небольшому списку правил:

* каждый отдельный фрагмент кода должен быть отдельным объектом, попадающим в одну
из четырёх зон ответственности, и не должен иметь ни малейшего представления о
других объектах. Это поможет избегать запутанного кода;
* поддерживайте возможность настройки вместо того, чтобы задавать конкретные
значения определённым параметрам. Это предотвратит необходимость повторения всего
HTML-окружения, чтобы написать тесты;
* каждый метод любого объекта должен быть простым и коротким. Этот пункт
позволит иметь простые и читаемые тесты;
* для создания объектов следует использовать конструкторы. Это поможет сделать
возможным создание «чистых» копий объектов, необходимых для тестирования.

Для начала надо определиться, на какие части мы разобьём наше приложение. У нас
есть три части, относящиеся к представлению и взаимодействию: поисковая форма,
поисковые результаты и сектор для понравившегося.

![Изображение][application-views]

Также существует кусок кода, относящийся к запросу данных с сервера, и часть,
обеспечивающая возможность совместной работы остальных частей.

Начнём с наиболее простой части приложения — сектор для понравившегося. В
оригинальном приложении следующий код отвечал за обновление этого сектора:

    var liked = $('#liked');
    var resultsList = $('#results');

    // …

    resultsList.on('click', '.like', function (e) {
      e.preventDefault();
      var name = $(this).closest('li').find('h2').text();
      liked.find( '.no-results' ).remove();
      $('<li>', { text: name }).appendTo(liked);
    });

Код поисковой формы сильно переплетен с сектором понравившегося и также требует
информации о том, как устроена разметка. Гораздо лучшим подходом (и
для тестируемости тоже) будет создание объекта сектора понравившегося,
ответственного за манипуляции с DOM:

    var Likes = function (el) {
      this.el = $(el);
      return this;
    };

    Likes.prototype.add = function (name) {
      this.el.find('.no-results').remove();
      $('<li>', { text: name }).appendTo(this.el);
    };

В этом коде приведён конструктор, создающий новую копию объекта Likes Box.
Созданная копия имеет метод `.add()`, используемый для добавления новых
результатов. Мы можем написать немного тестов, чтобы проверить, работает ли этот
метод:

    var ul;

    setup(function(){
      ul = $('<ul><li class="no-results"></li></ul>');
    });

    test('constructor', function () {
      var l = new Likes(ul);
      assert(l);
    });

    test('adding a name', function () {
      var l = new Likes(ul);
      l.add('Дмитрий Менделеев');

      assert.equal(ul.find('li').length, 1);
      assert.equal(ul.find('li').first().html(), 'Дмитрий Менделеев');
      assert.equal(ul.find('li.no-results').length, 0);
    });

Не так сложно, правда? Здесь используется [Mocha][7] в качестве *тестирующего*
фреймворка и [Chai][8] как дополнительная библиотека. Mocha обеспечивает
функции `test` и `setup`; Chai — функцию `assert`. Существует бездна других
фреймворков для тестирования, но я нахожу эти два достаточными для введения в
предметную область. Вам же следует найти свой фреймворк по своим предпочтениям —
[Qunit][9] популярен, а новый [Intern][10] подаёт большие надежды.

Приведённый код начинается с создания элемента, который будет использован как
контейнер для сектора понравившегося. Затем запускается два теста: первая
проверка на вменяемость — можем ли мы создать Like Box; вторая, чтобы
удостовериться, что метод `.add()` имеет желаемый эффект. При наличии этих
тестов, у нас появляется возможность безопасно рефакторить и быть уверенными, 
что мы сразу узнаем баге.

Код нашего приложения теперь выглядит так:

    var liked = new Likes('#liked');
    var resultsList = $('#results');
    
    
    // …
    
    
    resultsList.on('click', '.like', function (e) {
      e.preventDefault();
      
      var name = $(this).closest('li').find('h2').text();
      
      liked.add(name);
    });

Код, отвечающий за поисковые результаты, сложнее, чем Like Box, но давайте
попробуем свои силы в рефакторинге. Точно так же, как мы создали метод `.add()` у
Likes Box, мы хотим создать методы для общения с поисковыми результатами. Мы
хотим добавлять новые результаты и удобные способы оповещения других частей
приложения о событиях внутри поисковых результатов — например, когда кому-то
понравился пункт поисковой выдачи.

    var SearchResults = function (el) {
      this.el = $(el);
      this.el.on( 'click', '.btn.like', _.bind(this._handleClick, this) );
    };
    
    SearchResults.prototype.setResults = function (results) {
      var templateRequest = $.get('people-detailed.tmpl');
      templateRequest.then( _.bind(this._populate, this, results) );
    };
    
    SearchResults.prototype._handleClick = function (evt) {
      var name = $(evt.target).closest('li.result').attr('data-name');
      $(document).trigger('like', [ name ]);
    };
    
    SearchResults.prototype._populate = function (results, tmpl) {
      var html = _.template(tmpl, { people: results });
      this.el.html(html);
    };

Теперь код нашего старого приложения, отвечающий за взаимодействие между
поисковыми результатами и Likes Box, выглядит так:

    var liked = new Likes('#liked');
    var resultsList = new SearchResults('#results'); 
     

    // …
      

    $(document).on('like', function (evt, name) {
      liked.add(name);
    })

Такой код намного более простой и менее запутанный, потому что мы используем
`document` как глобальный транспорт для сообщений, и, передавая данные через
него, мы избавляем отдельные части приложения от необходимости знать друг о
друге. (В реальной жизни мы использовали бы [backbone][11] или [RSVP][12] для
управления событиями. В текущем демонстрационном приложении мы запускаем события
в `document` для упрощения кода). Мы также спрячем всю рутинную работу — поиск
имени понравившегося человека из поисковой выдачи — внутри объекта поисковых
результатов, чтобы не загрязнять им код приложения. Наконец, хорошие новости —
теперь мы можем писать тесты, чтобы доказать, что работа поисковых результатов
соотствует нашим ожиданиям:

    var ul;
    var data = [ /* ненастоящие данные */ ];

    setup(function () {
      ul = $('<ul><li class="no-results"></li></ul>');
    });

    test('constructor', function () {
      var sr = new SearchResults(ul);
      assert(sr);
    });

    test('display received results', function () {
      var sr = new SearchResults(ul);
      sr.setResults(data);

      assert.equal(ul.find('.no-results').length, 0);
      assert.equal(ul.find('li.result').length, data.length);
      assert.equal(
        ul.find('li.result').first().attr('data-name'),
        data[0].name
      );
    });

    test('announce likes', function() {
      var sr = new SearchResults(ul);
      var flag;
      var spy = function () {
        flag = [].slice.call(arguments);
      };

      sr.setResults(data);
      $(document).on('like', spy);

      ul.find('li').first().find('.like.btn').click();

      assert(flag, 'event handler called');
      assert.equal(flag[1], data[0].name, 'обработчик события получил данные' );
    });

Взаимодействие с сервером — другая часть для обсуждения. Оригинальный код
содержит в себе прямой вызов `$.ajax()`, и обработчик этого вызова работает
напрямую с DOM:

    $.ajax('/data/search.json', {
      data : { q: query },
      dataType : 'json',
      success : function( data ) {
        loadTemplate('people-detailed.tmpl').then(function(t) {
          var tmpl = _.template( t );
          resultsList.html( tmpl({ people : data.results }) );
          pending = false;
        });
      }
    });

Повторюсь, что очень трудно писать модульные тесты для такого кода из-за того,
что слишком много вещей происходит всего в нескольких строчках кода. Мы можем
переделать объект с информацией в самостоятельный объект:

    var SearchData = function () { };
    
    SearchData.prototype.fetch = function (query) {
      var dfd;
    
      if (!query) {
        dfd = $.Deferred();
        dfd.resolve([]);
        return dfd.promise();
      }
    
      return $.ajax( '/data/search.json', {
        data : { q: query },
        dataType : 'json'
      }).pipe(function( resp ) {
        return resp.results;
      });
    };

Сейчас мы можем изменить код, чтобы получить результаты на странице:

    var resultsList = new SearchResults('#results'); 
    var searchData = new SearchData();
    
    // …
    
    searchData.fetch(query).then(resultsList.setResults);

В который раз замечу, что мы невообразимо упростили код нашего приложениня, и
спрятали всю сложность кода в объект Search Data вместо того, чтобы хранить его
в общем коде. Также мы сделали наш поисковой интерфейс тестируемым, в тоже время
надо помнить о некоторых оссобенностях при тестировании кода, взаимодействующего
с сервером.

Первое — это то, что нам не нужно взаимодействовать с сервером *по-настоящему*
— это проникновение в мир интеграционных тестов, а так как мы ответственные
разработчики, то у нас уже есть тесты, сообщающие, что сервер работает
правильно; всё верно? Вместо этого мы хотим создать заглушку для серверного
взаимодействия, и это мы можем сделать с помощью библиотеки [Sinion][13]. Второй
важный момент — нам также необходимо тестировать неидеальные случаи, например,
пустой запрос.

    test('constructor', function () {
      var sd = new SearchData();
      assert(sd);
    });
    
    suite('fetch', function () {
      var xhr, requests;
    
      setup(function () {
        requests = [];
        xhr = sinon.useFakeXMLHttpRequest();
        xhr.onCreate = function (req) {
          requests.push(req);
        };
      });
    
      teardown(function () {
        xhr.restore();
      });
    
      test('fetches from correct URL', function () {
        var sd = new SearchData();
        sd.fetch('cat');
    
        assert.equal(requests[0].url, '/data/search.json?q=cat');
      });
    
      test('вернуть promise', function () {
        var sd = new SearchData();
        var req = sd.fetch('cat');
    
        assert.isFunction(req.then);
      });
    
      test('нет ответа, если нет запроса', function () {
        var sd = new SearchData();
        var req = sd.fetch();
        assert.equal(requests.length, 0);
      });
    
      test('вернуть promise, даже если нет запроса', function () {
        var sd = new SearchData();
        var req = sd.fetch();
    
        assert.isFunction( req.then );
      });
    
      test('no query promise resolves with empty array', function () {
        var sd = new SearchData();
        var req = sd.fetch();
        var spy = sinon.spy();
    
        req.then(spy);
    
        assert.deepEqual(spy.args[0][0], []);
      });
    
      test('returns contents of results property of the response', function () {
        var sd = new SearchData();
        var req = sd.fetch('cat');
        var spy = sinon.spy();
    
        requests[0].respond(
          200, { 'Content-type': 'text/json' },
          JSON.stringify({ results: [ 1, 2, 3 ] })
        );
    
        req.then(spy);
    
        assert.deepEqual(spy.args[0][0], [ 1, 2, 3 ]);
      });
    });

Оставив это за пределами статьи, я провела рефакторинг объекта Search Form и
упростила несколько участков кода и тестов, но если вам интересно, вы можете
взглянуть на законченную версию приложения в моём [репозитории][14] на гитхабе.

По окончании переписывания приложения с использованием шаблонов тестируемого
JavaScript, мы пришли к более ясному, чистому и поддерживаемому, в
отличие от стартового, коду:

    $(function() {
      var pending = false;
    
      var searchForm = new SearchForm('#searchForm');
      var searchResults = new SearchResults('#results');
      var likes = new Likes('#liked');
      var searchData = new SearchData();
    
      $(document).on('search', function (event, query) {
        if (pending) { return; }
    
        pending = true;
    
        searchData.fetch(query).then(function (results) {
          searchResults.setResults(results);
          pending = false;
        });
    
        searchResults.pending();
      });
    
      $(document).on('like', function (evt, name) {
        likes.add(name);
      });
    });

Важнее того, что мы получили более аккуратный код, может быть только то, что он
превосходно покрыт модульными тестами. Это означает, что мы можем смело
рефакторить приложение без страха поломки чего-либо. Мы даже можем написать дополнительные
тесты, если появится такая необходимость, и потом мы напишем код, который с
успехом пройдёт все тесты.


## Тестирование повышает качество жизни в долгосрочной перспективе

Несложно посмотреть на всё, что тут написано и спросить: «Подождите, вы хотите,
чтобы я писал больше кода, который бы делал ту же самую работу?»

Причина в некоторых непререкаемых фактах относительно создания вещей
в интернете. Вы потратите время, разрабатывая решение проблемы. Вы проверите своё
решение, прокликав интерфейс в браузере и написав автоматические модульные тесты,
или *вероломно* позволите пользователю тестировать ваше приложение в продакшене.
И тем не менее, сколько бы вы не написали тестов, баги будут всегда. 

Понимание тестирования заключается в том, что оно, возможно, и занимает чуть
больше времени на старте, но в итоге оно экономит вам время в будущем. Вы будете
прыгать от радости, когда тест, написанный вами, впервые обнаружит баг до того,
как он попадёт в продакшн. Также вы будете счастливы, когда система тестов
сможет подтвердить, что ваши правки действительно фиксят баг, для которого они
предназначались.


## Дополнительные источники ##

Эта статья только поверхностно описывает JavaScript-тестирование, но если вы
заинтересовались и хотите изучить тему глубже, то обязательно проверьте
следующие ссылки:

* [Моя презентация][15] с конференции Full Frontall (Брайтон, Великобритания, 2012);
* [Grunt][16] — инструмент, который поможет автоматизировать процесс тестирования;
* Книга [Test-Driven JavaScript Development][17] Кристиана Джохансона, создателя
библиотеки Sinion. Это краткая, но очень информативная проверка по основным
постулатам тестируемого JavaScript.


[1]: http://docs.seleniumhq.org/
[2]: https://github.com/jnicklas/capybara
[4]: https://twitter.com/ajpiano
[7]: http://visionmedia.github.io/mocha/
[8]: http://chaijs.com/
[9]: http://qunitjs.com/
[10]: http://theintern.io/
[11]: http://backbonejs.org
[12]: https://github.com/tildeio/rsvp.js
[13]: http://sinonjs.org/
[14]: https://github.com/rmurphey/testable-javascript
[15]: http://lanyrd.com/2012/full-frontal/sztqh/
[16]: http://gruntjs.com/
[17]: http://www.amazon.com/Test-Driven-JavaScript-Development-Developers-Library/dp/0321683919/ref=sr_1_1?ie=UTF8&qid=1366506174&sr=8-1&keywords=test+driven+javascript+development

[Srchr]: ./img/app.png "Srchr"
[строчки кода]: ./img/code-lines.png "Строчки кода"
[application-views]: ./img/app-views.png "Application Views"
