---
layout: post
title:  "Pontoon with Mithril"
date:   2014-04-22 21:40:00
categories: javascript
---

Mithril (http://lhorie.github.io/mithril/) is a new, lightweight and fast MVC framework for front-end javascript.

It looks like it could be a good alternative to something like Angular and it certainly seems easier to work with.

To try it out, I've been building a simple version of the card game "pontoon" and I've managed to put together a passable version within a couple of hours (though it is missing a few things).

I'm sure lots of this could be written better as this is my first stab at using the framework, but it's definitely very easy to get something up and running.

Anyway, onto the code.

First, we create our game module:

{% highlight javascript %}

    var game = {};

{% endhighlight %}

Now we'll deal with our 'model' layer. In this instance our base entity is the card:

{% highlight javascript %}

    game.Card = function(data) {
        this.name = m.prop(data.name);
        this.suit = m.prop(data.suit);
        this.value = m.prop(data.value);
    }

{% endhighlight %}

m.prop() is a Mithril helper function which means "create a property from this". It automatically creates a getter/setter method allowing us to do something like this:

{% highlight javascript %}
    var diamondKing = new game.Card( { name: 'king', 'suit': 'diamonds'});

    console.log(diamondKing.name() + " of " + diamondKing.suit());  // "king of diamonds"

    diamondKing.name("queen");

    console.log(diamondKing.name() + " of " + diamondKing.suit());  // "queen of diamonds"

{% endhighlight %}

Now we'll define a couple of methods to build and shuffle a deck of cards:

{% highlight javascript %}

    game.Deck = function() {
        deck = [];
        var suits = ["Hearts","Clubs","Spades","Diamonds"];
        var names = ["2","3","4","5","6","7","8","9","10","Jack","Queen","King","Ace"];
        var values = [2,3,4,5,6,7,8,9,10,10,10,10,1];

        // get a full deck of cards
        for(i=0;i<4;i++) {
            for(j=0;j<13;j++) {
                cardSpec = {
                    suit: suits[i],
                    name: names[j],
                    value: values[j]
                }
                card = new game.Card(cardSpec);
                deck.push(card);

            }
        }

        return deck;
    }

    game.shuffle = function(deck) {
        shuffled = [];
        while(deck.length >0) {
            pos = Math.floor(Math.random() * deck.length);
            taken = deck.splice(pos,1);
            card = taken[0];
            console.log("Added: " + card.name() + " of " + card.suit());
            shuffled.push(taken[0]);
        }

        return shuffled;
    }

{% endhighlight %}

These are just simple methods to build a deck of every possible card and to shuffle them. No doubt both could be done more efficiently but that's not the object of this exercise.

Finally for the model, we add a quick method to draw a card from the deck:

{% highlight javascript %}
game.draw = function(deck) {
    return deck.pop();
}
{% endhighlight %}

Now we'll look at the controller. We'll define three methods, which will be attached to three buttons.

First, 'twist', which will allow the user to draw a card:

{% highlight javascript %}
this.twist = function() {
    drawn = this.deck.pop();
    this.hand.push(drawn);
    this.total += drawn.value();

    if(this.total == 21) {
        alert("Winner!");
    }

    if(this.total > 21) {
        alert("Loser!");
    }
}
{% endhighlight %}

We draw a card from the deck and add it to their hand, adding its value to their score. Then we check to see if they scored 21, in which case we'll tell them they won, or if they went over 21, in which case they automatically lose.

The other option they have is to stick, so we'll add a method for that:

{% highlight javascript %}
    while(this.dealerTotal < 16) {
        // dealer twists on anything below 16
        drawn = this.deck.pop();
        this.dealer.push(drawn);
        this.dealerTotal += drawn.value();
    }

    if(this.dealerTotal > 21 || this.dealerTotal < this.total) {
        alert("Winner!");
    } else {
        alert("Loser!");
    }
{% endhighlight %}

When the player chooses to stick, it is the dealer's go, so we'll simulate that. Our dealer follows a simple rule: so long as he is below 16 he'll keep twisting. So we take cards from the deck into his hand until the score is above 16. Then we check to see if his score is above 21 or below the player's, in which case the player wins. If the dealer scored above the player but below 22 then the dealer wins.

We'll also add a quick controller method to reset the game by clearing the hands and re-shuffling the deck:

{% highlight javascript %}
        this.reset = function() {
            this.deck = game.shuffle(game.Deck());
            this.hand = [];
            this.total = 0;
            this.dealer = [];
            this.dealerTotal =0;
        }
{% endhighlight %}

Finally, we come to the V in MVC: the view.
Views in Mithril are built up using a helper method to generate elements. Pretty much all the elements are constructed on the fly here and it's this that allows the framework to be so fast: by maintaining its own representation of the DOM it can calculate which elements have been changed at any point and only redraw those.

Typical usage of the helper method is as follows:
{% highlight javascript %}
    m(<element type>, {element options}, [element contents]);
{% endhighlight %}

element type is the type of element to be created e.g. "div","p","input" etc.
element options allows us to specify options to be passed to the element. This could be a class:
{% highlight javascript %}
    m("button", {class:"btn btn-primary"}, "Press me");
{% endhighlight %}

It could also be an event handler (as we will use below):
{% highlight javascript %}
    m("button", {onclick: ctrl.twist.bind(ctrl)}, "Twist"),
{% endhighlight %}

The contents of the element could be simple text, as in the examples above, or it could be an array of child elements:
{% highlight javascript %}
    m("ul",[
        m("li","option one"),
        m("li","option two"),
        m("li","option three"),
    ]);
{% endhighlight %}

For our view, we have three parts of the view to draw: the control buttons, the player's hand and score, and the dealer's hand and score:

{% highlight javascript %}
  game.view = function(ctrl) {

        return m("html", [
            m("body", [

                // control buttons
                m("button", {onclick: ctrl.twist.bind(ctrl)}, "Twist"),
                m("button", {onclick: ctrl.stick.bind(ctrl)}, "Stick"),
                m("button", {onclick: ctrl.reset.bind(ctrl)}, "Reset"),

                // player's hand
                m("h2","Hand"),
                m("p",[
                    m("strong","Total: "),
                    ctrl.total
                ]),
                m("table", [
                     ctrl.hand.map(function(card,index) {

                        return m("tr", [
                            m("td", card.suit()),
                            m("td", card.name()),
                        ])
                    })
                ]),

                // dealer's hand
                m("h2","Dealer"),
                m("p",[
                    m("strong","Total: "),
                    ctrl.dealerTotal
                ]),
                m("table", [

                    ctrl.dealer.map(function(card,index) {

                        return m("tr", [
                            m("td", card.suit()),
                            m("td", card.name()),
                        ])
                    })
                ])
            ])
        ]);
{% endhighlight %}

The view itself takes the form of a function, to which we pass the controller object. We're going to be working directly on the document object of the DOM, so we start by adding the <html> and <body> elements.
Next, we add the three buttons for the controls, with their associated event handlers. To set up the event handler, we simply do something like this:

{% highlight javascript %}
m("button", {onclick: ctrl.reset.bind(ctrl)}, "Reset")
{% endhighlight %}

Here we tell it that we want the reset button's onclick event bound to the ctrl.reset() method (see above). The ctrl object will be an instance of the controller defined above.

Next, we display the two players' hands. First we show them the total:
{% highlight javascript %}
    m("p",[
        m("strong","Total: "),
        ctrl.total
    ]),
{% endhighlight %}

We add a <p> element containing a <strong> tag for the label and then the total variable on the controller.

Now we'll show them their hand. For this, we take advantage of javascript's Array.map function to allow us to iterate over the array, adding rows to a table for each element:
{% highlight javascript %}
    m("table", [
        ctrl.hand.map(function(card,index) {

            return m("tr", [
                m("td", card.suit()),
                m("td", card.name()),
            ])
        })
    ]),
{% endhighlight %}

So, we add a <table> element, within which we loop over the array of cards which represents the player's hand. Each card is a game.Card instance, so we can use the prop() methods (see above) to retrieve the data e.g. card.name()

We repeat the above steps to show the dealer's hand and the view is complete.

Finally, we just need to put the module together and set it running:
{% highlight javascript %}
     game.init = function() {
        m.module(document,game);
    }

    // start the app
    game.init();
{% endhighlight %}

We put the init code into a game.init() function for neatness, but all it's going to do here is call Mithril's module() function, giving it the DOM element in which to render views (document) and the module to use (game).
We call game.init() and the game begins!

This was just a simple experiment in getting something running. It's neither the best-written code ever, nor even a complete game (for one thing I've considered the Ace card to always be worth 1, whereas in reality it can be 1 or 11). Nevertheless, it demonstrates how quickly we can get something running with this framework and how lightweight and easy to understand it is.
So far I'm a fan of Mithril. In the future I hope to explore it a bit more to really see what it can do.

You can view the full code here: https://github.com/dan-d-martin/mithril-pontoon