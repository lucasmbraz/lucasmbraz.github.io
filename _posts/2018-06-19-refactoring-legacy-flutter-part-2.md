---
layout: post
title: "Refactoring a Legacy Flutter App Part 2"
---

This is the second post in a series where we’ll attempt to safely and gradually refactor a legacy app written in Flutter.

## Previously…
In the [previous episode](https://lucasmbraz.com/2018-06-04/refactoring-legacy-flutter-part-1) of *“Refactoring a Legacy Flutter App”* we:

* Used Michael Feathers definition of legacy code as being code without tests instead of code that is old.
* Covered our entire app with widget tests so we can catch mistakes we make during refactoring.
* Set up continuous integration and code coverage.

## Today’s Goal
Here’s what we’ll do this time:

1. Define what Refactoring is.
2. Apply the MVP pattern, making sure to run our tests along the way.
3. Discuss when to use unit tests.

If you want to jump straight to the solution, check out the [part-2 branch](https://github.com/lucasmbraz/planets/tree/part-2).

## To change, or not to change
Our app has one issue: the data we’re displaying is hardcoded. To make it production-ready, we’d have to fetch the data from some Rest API and display error messages and a progress indicator to the user.

We’re already changing the structure of the whole app in this refactoring. It makes sense that we take this opportunity to include those features, right?

[Martin Fowler](https://www.amazon.com.br/Refactoring-Improving-Design-Existing-Code/dp/0201485672) defines refactoring as follows:

> “A change made to the internal structure of software to make it easier to understand and cheaper to modify **without changing its observable behavior.**”

So, even though it’s tempting, the answer is no. **We don’t create features or fix bugs during a refactoring because we don’t want the behavior of the system to change.**

That’s why part-1 is so important. When you have automated tests in place, you can have much higher confidence that you’re preserving the behavior of your app.

## MVP
I won’t go over the details of MVP because there are a ton of resources out there covering the pattern. Also, we’ll be following *Chema Rubio’s* implementation. Check out [his post](https://medium.com/@develodroid/flutter-iv-mvp-architecture-e4a979d9f47e) for more details.

In the next few sections, we’ll refactor the home page.

### The Contract
Let’s start by specifying the responsibilities of our view. We’ll define a contract that the view adheres to and the presenter uses to talk to the view.

Go ahead and create the file `/lib/ui/home/home_page_contract.dart` with the content below:

{% highlight dart %}
import 'package:flutter_planets_tutorial/model/planets.dart';

abstract class HomePageView {
  showPlanets(List<Planet> planets);
}
{% endhighlight %}

Simple stuff. The home page has only one responsibility: to show planets.

### The Presenter
For the presenter, create the file `/lib/ui/home/home_page_presenter.dart` like so:

{% highlight dart %}
import 'package:flutter_planets_tutorial/model/planets.dart';
import 'package:flutter_planets_tutorial/ui/home/home_page_contract.dart';

class HomePagePresenter {
  final HomePageView _view;

  HomePagePresenter(this._view);

  loadPlanets() {
    _view.showPlanets(planets);
  }
}
{% endhighlight %}

Again, this is very simple. We receive an instance of `HomePageView`, and when `loadPlanets` is called, we tell the view to show the planets. Note that `planets` is the same hardcoded array our `HomePage` currently uses.

Next, we’ll unit test this presenter. First, create two folders inside `/test` named `widget` and `unit`. Then, move the existing tests to the `widget` folder. Your tree structure should look like this:

![Tree structure](/assets/refactoring-legacy-flutter-part-2/tree-structure.png)

After that, create the file `/unit/ui/home/home_page_presenter_test.dart`:

{% highlight dart %}
import 'package:flutter_planets_tutorial/model/planets.dart';
import 'package:flutter_planets_tutorial/ui/home/home_page_contract.dart';
import 'package:flutter_planets_tutorial/ui/home/home_page_presenter.dart';
import 'package:mockito/mockito.dart';  //1
import 'package:test/test.dart';

void main() {
  group('HomePagePresenter ', () {
    test('loads all the planets', () {
      final view = MockView(); //2
      final presenter = HomePagePresenter(view); //3

      presenter.loadPlanets(); //4

      verify(view.showPlanets(planets)); //5
    });
  });
}

class MockView extends Mock implements HomePageView {} //6
{% endhighlight %}

Here’s what this code does:

1. Import *mockito*. (Remember we added it as a dependency in the previous post.)
2. Create a mock of `HomePageView`.
3. Instantiate the presenter we want to test, passing the mock view.
4. Execute the method to be tested.
5. Verify that the mock view was called with the right argument.
6. This is how you can create mocks. You define an empty class that extends `Mock` and implements the class you want to mock.

Run all the tests and they obviously still pass. We created the presenter and the contract but haven’t connected them to the view yet. We’ll do it next.

### The View
Change the file `/lib/ui/home/home_page_body.dart` as below:

{% highlight dart %}
import 'package:flutter/material.dart';
import 'package:flutter_planets_tutorial/model/planets.dart';
import 'package:flutter_planets_tutorial/ui/common/plannet_summary.dart';
import 'package:flutter_planets_tutorial/ui/home/home_page_contract.dart';
import 'package:flutter_planets_tutorial/ui/home/home_page_presenter.dart';

class HomePageBody extends StatefulWidget { //1
  @override
  _HomePageBodyState createState() {
    return new _HomePageBodyState();
  }
}

class _HomePageBodyState extends State<HomePageBody> implements HomePageView {
  HomePagePresenter _presenter; //2
  List<Planet> _planets; //3

  _HomePageBodyState() {
    _presenter = HomePagePresenter(this);
    _planets = const [];
  }

  @override
  void initState() {
    super.initState();
    _presenter.loadPlanets(); //4
  }

  @override
  showPlanets(List<Planet> planets) {
    setState(() {
      _planets = planets;
    });
  }

  @override
  Widget build(BuildContext context) {
    return new Expanded(
      child: new Container(
        color: new Color(0xFF736AB7),
        child: new CustomScrollView(
          scrollDirection: Axis.vertical,
          shrinkWrap: false,
          slivers: <Widget>[
            new SliverPadding(
              padding: const EdgeInsets.symmetric(vertical: 24.0),
              sliver: new SliverList(
                delegate: new SliverChildBuilderDelegate(
                    (context, index) => new PlanetSummary(_planets[index]), //5
                  childCount: _planets.length,
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
{% endhighlight %}

Here are the main changes we’ve made to this file:

1. Turn `HomePageBody` into a `StatefulWidget`.
2. The `presenter` this view interacts with.
3. The data this view displays. This starts as an empty array and will be populated once the presenter calls `showPlanets`.
4. Ask the presenter to load the data as soon as this widget is inserted into the tree.
5. Replace the hardcoded `planets` array with the class property `_planets` that’s populated by the presenter.

Run all the tests again and enjoy the green lights of success. We’ve managed to implement the MVP architectural pattern without introducing bugs 💪.

### PlanetSummary
Another nice refactoring you can do is to extract the `onTap` function out of `PlanetSummary`. Right now, `PlanetSummary` is a bit inflexible. If you were to use it in some new feature, you’d have to modify its code to make sure `onTap` handles the new scenario.

Instead, if `PlanetSummary` receives the `onTap` function via a constructor or a setter, it could be used anywhere without the need for any change.

Give it a go. And remember to run the tests often.

### The Detail Page
For now, I chose to leave the detail page as it is. The reason is that it receives all the data it needs through its constructor and there’s no logic or interactions in this page. If and when requirements for this page change, that’s the moment to refactor (before doing any new features, of course).

### To unit test, or to not unit test
You can see how easy it is to unit test presenters. The question you might ask yourself is: *is it worth it?* Like everything in software, the answer is: it depends.

Consider the `HomePagePresenter` above. This might be controversial, but in my opinion, there’s no need for unit testing it. There are two reasons why I think this way:

1. This presenter has no business logic. It’s just the glue that connects the view to the data source (in this case, the hardcoded `planets` array).
2. If there’s a bug in this presenter, it’ll break the view, and the widget tests would catch that.

So, if a presenter, or any other class for that matter, has no logic or has simple logic that’s directly linked to the UI, I’d say it’s not worth of unit tests. Widget tests get the job done.

However, if a class has complex logic or logic that’s not user-facing (e.g., caching mechanism) than unit tests are a must.

In general, mobile apps have very little business logic. Typically, the heavy logic is in the backends and the apps fetch and display data (although that doesn’t mean mobile apps are simple, far from it). In those cases, I’d argue that very few unit tests are needed, and you should focus your efforts in widget tests.

That’s all folks. Thanks for making it this far and I hope you’ve found it useful. Reach out if you disagree or have suggestions on what you want to see next. I’ll probably do something reactive like [@ThomasBurkhartB](https://twitter.com/ThomasBurkhartB) suggested on Twitter. We’ll see. Time will tell.
