Writing Testable JavaScript
============================================================

We’ve all been there: that bit of JavaScript functionality that started out as
just a handful of lines grows to a dozen, then two dozen, then more. Along the 
way, a function picks up a few more arguments; a conditional picks up a few more
conditions. And then one day, the bug report comes in: something’s broken, and 
it’s up to us to untangle the mess.

As we ask our client-side code to take on more and more responsibilities—
indeed, whole applications are living largely in the browser these days—two 
things are becoming clear. One, we can’t just point and click our way through 
testing that things are working as we expect; automated tests are key to having 
confidence in our code. Two, we’re probably going to have to change how we write
our code in order to make it possible to write tests.

Really, we need to change how we code? Yes—because even if we know that
automated tests are a good thing, most of us are probably only able to write 
integration tests right now. Integration tests are valuable because they focus 
on how the pieces of an application work together, but what they don’t do is 
tell us whether individual*units of functionality* are behaving as expected.

That’s where unit testing comes in. And we’ll have a very hard time *writing
unit tests* until we start *writing testable JavaScript*.


## Unit vs. integration: what’s the difference?

Writing integration tests is usually fairly straightforward: we simply write
code that describes how a user interacts with our app, and what the user should 
expect to see as she does.[Selenium][1] is a popular tool for automating
browsers.[Capybara][2] for Ruby makes it easy to talk to Selenium, and there
are plenty of tools for other languages, too.

Here’s an integration test for a portion of a search app:

    def test_search
      fill_in('q', :with => 'cat')
      find('.btn').click
      assert( find('#results li').has_content?('cat'), 'Search results are shown' )
      assert( page.has_no_selector?('#results li.no-results'), 'No results is not shown' )
    end
    

Whereas an integration test is interested in a user’s interaction with an app,
a unit test is narrowly focused on a small piece of code:


> When I call a function with a certain input, do I receive the expected output?

Apps that are written in a traditional procedural style can be very difficult
to unit test—and difficult to maintain, debug, and extend, too. But if we write 
our code with our future unit testing needs in mind, we will not only find that 
writing the tests becomes more straightforward than we might have expected, but 
also that we’ll simply*write better code*, too.

To see what I’m talking about, let’s take a look at a simple search app:

![Image][Srchr]

When a user enters a search term, the app sends an XHR to the server for the
corresponding data. When the server responds with the data, formatted as JSON, 
the app takes that data and displays it on the page, using client-side 
templating. A user can click on a search result to indicate that he “likes” it; 
when this happens, the name of the person he liked is added to the “Liked” list 
on the right-hand side.

A “traditional” JavaScript implementation of this app might look like this:

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
          html : 'Searching &hellip;'
        }).appendTo( resultsList.empty() );
      });

      resultsList.on('click', '.like', function (e) {
        e.preventDefault();
        var name = $(this).closest('li').find('h2').text();
        liked.find('.no-results').remove();
        $('<li>', { text: name }).appendTo(liked);
      });

    });

My friend [Adam Sontag][4] calls this Choose Your Own Adventure code—on any
given line, we might be dealing with presentation, or data, or user interaction,
or application state. Who knows! It’s easy enough to write integration tests for
this kind of code, but it’s hard to test individual*units of functionality*.

What makes it hard? Four things:

* A general lack of structure; almost everything happens in a
`$(document).ready()` callback, and then in anonymous functions that can’ t be
tested because they aren’t exposed.
* Complex functions; if a function is more than 10 lines, like the submit handler,
it’s highly likely that it’s doing too much.
* Hidden or shared state; for example, since `pending` is in a closure, there’s no
way to test whether the pending state is set correctly.
* Tight coupling; for example, a `$.ajax` success handler shouldn’t need direct
access to the DOM.


## Organizing our code

The first step toward solving this is to take a less tangled approach to our
code, breaking it up into a few different areas of responsibility:

* Presentation and interaction
* Data management and persistence
* Overall application state
* Setup and glue code to make the pieces work together

In the “traditional” implementation shown above, these four categories are
intermingled—on one line we’re dealing with presentation, and two lines later we
might be communicating with the server.

![Image][code-lines]

While we can absolutely write integration tests for this code—and we should
!—writing unit tests for it is pretty difficult. In our functional tests, we can
make assertions such as “when a user searches for something, she should see the 
appropriate results,” but we can’t get much more specific. If something goes 
wrong, we’ll have to track down exactly where it went wrong, and our functional 
tests won’t help much with that.

If we rethink how we write our code, though, we can write unit tests that will
give us better insight into where things went wrong, and also help us end up 
with code that’s easier to reuse, maintain, and extend.

Our new code will follow a few guiding principles:

* Represent each distinct piece of behavior as a separate object that falls into
one of the four areas of responsibility and doesn’t need to know about  other
objects. This will help us avoid creating tangled code.
* Support configurability, rather than hard-coding things. This will prevent us
from replicating our entire HTML environment in order to write our tests.
* Keep our objects’ methods simple and brief. This will help us keep our tests
simple and our code easy to read.
* Use constructor functions to create instances of objects. This will make it
possible to create “clean” copies of each piece of code for the sake of testing.

To start with, we need to figure out how we’ll break our application into
different pieces. We’ll have three pieces dedicated to presentation and 
interaction: the Search Form, the Search Results, and the Likes Box.

![Image][application-views]

We’ll also have a piece dedicated to fetching data from the server and a
piece dedicated to gluing everything together.

Let’s start by looking at one of the simplest pieces of our application: the
Likes Box. In the original version of the app, this code was responsible for 
updating the Likes Box:

    var liked = $('#liked');
    var resultsList = $('#results');

    // …

    resultsList.on('click', '.like', function (e) {
      e.preventDefault();
      var name = $(this).closest('li').find('h2').text();
      liked.find( '.no-results' ).remove();
      $('<li>', { text: name }).appendTo(liked);
    });

The Search Results piece is completely intertwined with the Likes Box piece and
needs to know a lot about its markup. A much better and more testable approach 
would be to create a Likes Box object that’s responsible for manipulating the 
DOM related to the Likes Box:

    var Likes = function (el) {
      this.el = $(el);
      return this;
    };

    Likes.prototype.add = function (name) {
      this.el.find('.no-results').remove();
      $('<li>', { text: name }).appendTo(this.el);
    };

This code provides a constructor function that creates a new instance of a
Likes Box. The instance that’s created has an`.add()` method, which we can use
to add new results. We can write a couple of tests to prove that it works:

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
      l.add('Brendan Eich');

      assert.equal(ul.find('li').length, 1);
      assert.equal(ul.find('li').first().html(), 'Brendan Eich');
      assert.equal(ul.find('li.no-results').length, 0);
    });


Not so hard, is it? Here we’re using [Mocha][7] as the test *framework*, and 
[Chai][8] as the *assertion library*. Mocha provides the `test` and `setup`
functions; Chai provides`assert`. There are plenty of other test frameworks and
assertion libraries to choose from, but for the sake of an introduction, I find 
these two work well. You should find the one that works best for you and your 
project—aside from Mocha,[QUnit][9] is popular, and [Intern][10] is a new
framework that shows a lot of promise.

Our test code starts out by creating an element that we’ll use as the
container for our Likes Box. Then, it runs two tests: one is a sanity check to 
make sure we can make a Likes Box; the other is a test to ensure that our
`.add()` method has the desired effect. With these tests in place, we can
safely refactor the code for our Likes Box, and be confident that we’ll know if 
we break anything.

Our new application code can now look like this:

    var liked = new Likes('#liked');
    var resultsList = $('#results');
    
    
    // …
    
    
    resultsList.on('click', '.like', function (e) {
      e.preventDefault();
      
      var name = $(this).closest('li').find('h2').text();
      
      liked.add(name);
    });

The Search Results piece is more complex than the Likes Box, but let’s take a
stab at refactoring that, too. Just as we created an`.add()` method on the
Likes Box, we also want to create methods for interacting with the Search 
Results. We’ll want a way to add new results, as well as a way to “broadcast” to
the rest of the app when things happen within the Search Results—for example, 
when someone likes a result.

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
    

Now, our old app code for managing the interaction between Search Results and
the Likes Box could look like this:

    var liked = new Likes('#liked');
    var resultsList = new SearchResults('#results'); 
     

    // …
      

    $(document).on('like', function (evt, name) {
      liked.add(name);
    })

It’s much simpler and less entangled, because we’re using the `document` as a
global message bus, and passing messages through it so individual components don
’t need to know about each other. (Note that in a real app, we’d use something 
like[Backbone][11] or the [RSVP][12] library to manage events. We’re just
triggering on`document` to keep things simple here.) We’re also hiding all
the dirty work—such as finding the name of the person who was liked—inside the 
Search Results object, rather than having it muddy up our application code. The 
best part: we can now write tests to prove that our Search Results object works 
as we expect:

    var ul;
    var data = [ /* fake data here */ ];

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
      assert.equal(flag[1], data[0].name, 'event handler receives data' );
    });

The interaction with the server is another interesting piece to consider. The
original code included a direct`$.ajax()` request, and the callback interacted
directly with the DOM:

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

Again, this is difficult to write a unit test for, because so many different
things are happening in just a few lines of code. We can restructure the data 
portion of our application as an object of its own:

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

Now, we can change our code for getting the results onto the page:

    var resultsList = new SearchResults('#results'); 
    var searchData = new SearchData();
    
    // …
    
    searchData.fetch(query).then(resultsList.setResults);

Again, we’ve dramatically simplified our application code, and isolated the
complexity within the Search Data object, rather than having it live in our main
application code. We’ve also made our search interface testable, though there 
are a couple caveats to bear in mind when testing code that interacts with the 
server.

The first is that we don’t want to *actually* interact with the server—to do
so would be to reenter the world of integration tests, and because we’re 
responsible developers, we already have tests that ensure the server does the 
right thing, right? Instead, we want to “mock” the interaction with the server, 
which we can do using the[Sinon][13] library. The second caveat is that we
should also test non-ideal paths, such as an empty query.

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
    
      test('returns a promise', function () {
        var sd = new SearchData();
        var req = sd.fetch('cat');
    
        assert.isFunction(req.then);
      });
    
      test('no request if no query', function () {
        var sd = new SearchData();
        var req = sd.fetch();
        assert.equal(requests.length, 0);
      });
    
      test('return a promise even if no query', function () {
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

For the sake of brevity, I’ve left out the refactoring of the Search Form,
and also simplified some of the other refactorings and tests, but you can see a 
finished version of the app[here][14] if you’re interested.

When we’re done rewriting our application using testable JavaScript patterns,
we end up with something much cleaner than what we started with:

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

Even more important than our much cleaner application code, though, is the fact
that we end up with a codebase that is thoroughly tested. That means we can 
safely refactor it and add to it without the fear of breaking things. We can 
even write new tests as we find new issues, and then write the code that makes 
those tests pass.


## Testing makes life easier in the long run

It’s easy to look at all of this and say, “Wait, you want me to write more
code to do the same job?”

The thing is, there are a few inescapable facts of life about Making Things On
The Internet. You will spend time designing an approach to a problem. You will 
test your solution, whether by clicking around in a browser, writing automated 
tests, or
—*shudder*—letting your users do your testing for you in production. You will
make changes to your code, and other people will use your code. Finally: there 
will be bugs, no matter how many tests you write.

The thing about testing is that while it might require a bit more time at the
outset, it really does save time in the long run. You’ll be patting yourself on 
the back the first time a test you wrote catches a bug before it finds its way 
into production. You’ll be grateful, too, when you have a system in place that 
can prove that your bug fix really does fix a bug that slips through.


## Additional resources

This article just scratches the surface of JavaScript testing, but if you’d
like to learn more, check out:

* [My presentation][15] from the 2012 Full Frontal conference in Brighton, UK.
* [Grunt][16], a tool that helps automate the testing process and lots of other
things.
* [Test-Driven JavaScript Development][17] by Christian Johansen, the creator
of the Sinon library. It is a dense but informative examination of the practice
of testing JavaScript.








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

[Srchr]: http://d.alistapart.com/375/app.png "Srchr"
[code-lines]: http://d.alistapart.com/375/code-lines.png "Code lines"
[application-views]: http://d.alistapart.com/375/app-views.png "Application Views"