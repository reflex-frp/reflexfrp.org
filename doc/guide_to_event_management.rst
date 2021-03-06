.. _guide_to_event_management:

A Guide to Event Management
===========================

The reflex package provides many APIs to create the control logic of reflex app
which is independent of the DOM.

See `Quick Ref <https://github.com/reflex-frp/reflex/blob/develop/Quickref.md>`_

In order to leverage the full power of reflex, one has to effectively use
ability to create Event propagation graphs. This guide gives an overview of
various useful techniques.

Overview
--------

In reactive programming you have various sources of events
which have to be utilised for providing responses. For example when user clicks a
button, this event can have various different reponses depending
upon the context or more specifically the state of the application.

The response to an event in most cases will do some change in DOM, AJAX request or
change the internal state of application.

In Reflex this response can be expressed or implemented by

1. Firing another ``Event``.
2. Modification of a ``Dynamic`` Value.

Note that there is no explicit callbacks or function calls in response to the
incoming events. Instead there is generation of new Events and modification of
Dynamic values. These Event and Dynamic values are then propagated to widgets
which provide the appropriate response to the event.

Since this propagation of ``Event``/``Dynamic`` values can be cyclic, it can be thought
as an Event propagation graph.

The following sections covers details of constructing this graph.

Event
-----

Creation
~~~~~~~~

The following are the primary sources of events

#. DOM

   #. Input fields like button, text-box, etc.

      See the type of input fields (``TextInput RangeInput`` etc)
      to see what events are available.

   #. User interaction events like mouse click, mouse over, etc.

      ``domEvent`` API can be used to create ``Event`` on DOM elements::

        (e,_) <- el' "span" $ text "Click Here"

        clickEv :: Event t ()
        clickEv <- domEvent Click e

      For a complete list of events accepted by ``domEvent`` see ``EventName``

      .. todo:: Add a link to haddock

#. Response from AJAX request or WebSocket connection
   see :ref:`guide_to_ajax`

#. ``Dynamic`` values - By calling ``updated`` on a ``Dynamic`` value one can obtain the event
   when its value changes.

Manipulation
~~~~~~~~~~~~

Using these primary ``Event``\s you can create secondary / derived events by

#. Manipulated the value using fmap::

    -- inputValueEv :: Event t Int

    doubledInputValueEv = ffor inputValue (* 2)

#. Filter the value::

    -- inputValueEv :: Event t Int

    -- This Event will fire only if input value is even
    evenOnlyEv = ffilter even inputValueEv

   Use ``fmapMaybe fforMaybe`` for similar filtering

#. Tagging value of ``Dynamic`` or ``Behavior``.

   Use these APIs, see 
   `Quick Ref <https://github.com/reflex-frp/reflex/blob/develop/Quickref.md#functions-producing-event>`_
   ::

    tagPromptlyDyn, tag, attachDyn, attachDynWith, attachPromptlyDynWithMaybe

.. todo:: Explain the sampling of Dynamic: Promptly vs delayed?

          May be an example which showcase the correct and wrong usage

Behavior
--------

The sink of a behavior is ``sample``, so it is only useful when a widget is created
dynamically by sampling some behavior, but remain static after its creation.


Dynamic
-------

Creation
~~~~~~~~

The following are the primary sources of ``Dynamic`` values

#. DOM

   #. Input fields like text-box, range input etc.

      See the type of input fields (``TextInput RangeInput`` etc)

.. Any other places where we can get Dynamic??

Event to Dynamic
~~~~~~~~~~~~~~~~

Create a ``Dynamic`` which changes value when ``Event`` occurs::

  holdDyn :: a -> Event t a -> m (Dynamic t a)
  foldDyn :: (a -> b -> b) -> b -> Event t a -> m (Dynamic t b)

These can be utilised to maintain a state in application.
For more see :ref:`maintain_state`

Manipulation
~~~~~~~~~~~~

Using these primary ``Dynamic`` values you can create secondary / derived values by

#. ``fmap``

#. ``zipDyn zipDynWith``

   Zipping is useful when multiple ``Dynamic`` values have a common point of influence
   in the application.

   For example if I have two variable parameters like color and font of text.
   Then I can construct the dynamic attributes from these parameters by simply
   zipping them together.::

    -- textFont :: Dynamic t Text
    -- textColor :: Dynamic t Text

    getAttr (f,c) = ("style" =: ("font-family: " <> f "; color: " <> c)) 

    elDynAttr "div" (getAttr <$> (zipDyn textFont textColor)) $ text "Text"

Simple Event Propagation Graph
--------------------------------

.. Its probably better to just give some example here?

Simple
~~~~~~

Simply pass the Event/Dynamic to input of function

In monadic code create simple event propagation tree

Recursive Do
~~~~~~~~~~~~

In Monadic code - create a cyclic graph of event propagation

Problems in cyclic dependency

#. Deadlock - Runtime deadlock due to block on an MVar operation
   This can occur if a widget depends on an Event which is created
   in a ``let`` clause after the widget creation.
   To fix this simply move the ``let`` clause before the widget creation

#. Loop - Output of holdDyn feeds back can cause this??


.. _maintain_state:

Maintaining State via fold
--------------------------

In order to store a state/data for your app (ie create a state machine) simply
use ``foldDyn``

::

  -- State can be any arbitrary haskell data
  -- stateDynVal :: Dynamic t MyState

  -- ev can a collection of all events on which the state depends
  -- For example all input events
  -- ev :: Event t Inputs

  -- This is a pure API which can process the input events and current state
  -- to generate a new state.
  -- eventHandler :: (Inputs -> MyState -> MyState)

  -- foldDyn :: (a -> b -> b) -> b -> Event t a -> Dynamic t b
  stateDynVal <- foldDyn eventHandler initState ev

Even nested state machines can be designed if your have a state with nested ``Dynamic`` value
by using ``foldDynM``

see nested_dynamic.hs

Use ``foldDynMaybe``, ``foldDynMaybeM`` in cases where you want to filter input
events, such that they don't modify the state of application.

For example in a shopping cart if the user has not selected any items, the "add
to cart" button should do nothing. This kind of behavior can be implemented by
returning ``Nothing`` from the eventHandler.


Using Collections in Event propagation graph
--------------------------------------------

In order to model complex flows of events or dynamically changing data
collection, we need to use higher order containers like lists (``[]``) or Maps
(``Data.Map``)

.. todo:: This section is relevant with appropriate examples

          So add examples here

Use of Dynamic t [], Dynamic t (Map k v), etc

User data model design : separate guide?

Fanning
~~~~~~~

Split or distribute the event

.. todo:: How to effectively use fan? EventSelector?

Merging/Switching
~~~~~~~~~~~~~~~~~

``Dynamic`` values can be merged simply by ``zipDyn``, ``mconcat``, etc.

``Events``

  Given some events you can choose either to keep them all by using ``align``
  align - If two events can possibly happen together (because of a common driver
  perhaps), then use this to capture them in a single event.

  or select just one from the list using ``leftmost``

  or use one of these to merge
  mergewith, mergeList - returns a NonEmpty list


Higher order FRP
----------------

Nested Values and flattening
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When you model real world ``Dynamic`` values many times you end up with nested
structures.

For example, if the value of items in a shopping cart depends on the shipping
method chosen, then you can end up with a value ``total' :: Dynamic t [Dynamic t Int]``::

  selectedItems :: Dynamic t [Item]
  isExpeditedShipping :: Dynamic t Bool

  total' = Dynamic t [Dynamic t Int]
  total' = ffor selectedItems
            (map getItemPrice)

  getItemPrice :: Item -> Dynamic t Int
  getItemPrice itm = ffor isExpeditedShipping
                      (\case
                        True -> (itemPrice itm) + (shippingCharges itm)
                        False -> itemPrice itm)

In such cases in order to get a total value ``Dynamic t Int``, you need to use
flattening APIs. In case of ``Dynamic`` it is simply ``join`` from
``Control.Monad`` (since ``Dynamic`` has an instance of ``Monad``)::

  total'' :: Dynamic t (Dynamic t Int)
  total'' = foldr1 (\a b -> (+) <$> a <*> b) <$> total'

  total :: Dynamic t Int
  total = join total''

See `QuickRef <https://github.com/reflex-frp/reflex/blob/develop/Quickref.md#flattening-functions>`_
for details on other flattening APIs.



.. Push/Pull APIs?

.. Note from Divam - The ``Reflex`` typeclass provides functions which I think
  are not important discussing here?
  Similarly MonadSample, MonadHold are not relevant in introduction
  They are relevant in QuickRef which lists the API and their constraints



.. just some other content, may be relevant here

  https://www.reddit.com/r/reflexfrp/comments/3bocn9/how_to_extract_the_current_value_from_a_text_box/

  Event is probably as you understand it, discrete events. Behavior's are values which change over time (but you don't know when they changed)
  and a Dynamic is Event + Behavior, values which change over time, and you're notified when they change, too.
  The problem with your example, is that omg is not an Event, Behavior or Dynamic but just a String (so it will never change).
  What you might want to do is tag the event with the value from the text box like this:
  omg <- mapDyn (\t -> "myUrl/" ++ t ++ "/me") value questionBox
  dyn <- mkAsyncDyn "default" $ tag (current omg) insertEvent
  This way omg is a Dynamic, so it can change over time. Then we tag the event with the value of the behavior current omg.
  (Note that if we used directly tagDyn omg insertEvent the event would fire both when omg changed as well as when the button was clicked, which is not what we want)
  mkAsyncDyn :: MonadWidget t m => T.Text -> Event t String -> m (Dynamic t (Maybe T.Text))
  mkAsyncDyn defaultValue event = do
    ev <- performRequestAsync $ fmap (\url -> xhrRequest "GET" url def) event
    holdDyn (Just defaultValue) $ fmap _xhrResponse_body ev
  So the takeaway here is that for values to update they need to be reactive type (Event, Behavior, Dynamic), sample is almost never what you want to do.


  https://www.reddit.com/r/reflexfrp/comments/4nyteu/joindyn_and_eboth/
  http://anderspapitto.com/posts/2016-11-09-efficient-updates-of-sum-types-in-reflex.html

