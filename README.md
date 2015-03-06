# Maxwell
Maxwell is an application architecture designed for building client-side web applications

## Overview

Maxwell is an application architecture designed for building client-side web applications. It is described by a set of design patterns and conventions and does not require implementation as a formal framework.

Maxwell is a variation of Model-View-Controller (MVC) which favors unidirectional data flow by replacing the traditional Controller with Actions - simple encapsulations of logic for coordinating async web service calls and Model updates. When a user interacts with a View this triggers execution of a corresponding Action, which coordinates service calls and Model updates, which in turn notify relevant Views which update as required to show the new Model state.


## Data Flow

As the high-level block diagram shows, data flows in a single direction:

![Overview](/images/overview.png)

1. When the user interacts with a View, this triggers execution of a corresponding Action, passing relevant data to the Action.

2. The Action accomplishes the task requested by the View by coordinating API requests and Model updates.

3. The Model simply stores application data and notifies subscribers (Views) if the data is modified.


### Example
This may be explained through a simple example:

![Example](/images/example.png)

In this example UI the ItemCountView displays the number of items in the ItemListModel and the ItemTableView shows a table of items from the ItemListModel, with a "Remove" link next to each item.

1. If the user clicks a "Remove" link the ItemTableView will execute the DeleteItemAction, passing it the ID of the item to be removed.

2. The DeleteItemAction will call the APIService to request deletion of the item. In case of error, the Action will update the AppErrorsModel, providing details of the error. In case of success, the Action will update the ItemListModel, providing the ID of the item to remove.

3. When the ItemListModel is updated by the DeleteItemAction, it notifies its subscribers that it has changed.

4. The ItemCountView subscribes to the ItemListModel, and updates to show the new count of items.

5. The ItemTableView subscribes to the ItemListModel, and updates to show the current list of items (no longer including the removed item).


## Architecture Details

The detailed block diagram shows the specific relationships and multiplicities between the layers:

![Details](/images/details.png)


### Models

Models store application state. They provide an interface to query the state, update the state, and subscribe for notifications when the state is modified.

Models do not execute Actions, call API services, directly update Views, or interact with each other. All coordination is handled by Actions.

Models are not necessarily mapped directly to API services or data structures. They should store data in the most effective structure for use by Views throughout the application.

A single Model may provide access to several elements of state, but these should be related such that a single change notification is appropriate if any of them is modified.

All modifications to data should be through methods exposed on the Model. In other words, Views and Actions should not directly modify objects or arrays that are queried from the Model - otherwise, Model subscribers would not be notified of the change.

In order to enable view rendering optimizations, Models should use immutable data structures, either by making use of a formal library or simply by cloning objects when their properties are modified via Model methods.


### Actions

Actions encapsulate logic for performing a request, typically triggered by user interaction with Views. An Action accomplishes its task by use of external resources (API services, router queries/changes, etc) and modification of state stored in Models.

Actions do not update Views directly. Actions may query or modify Models, call into API services, use other utilities, and even delegate to other Actions.

Actions do not directly modify data structures queried from Models. They use Model methods to accomplish the required changes.


### Views

Views display data queried from Models and respond to user interaction by updating their internal state (for localized changes) or executing Actions (for changes affecting the application state).

Views do not modify Models (or data structures queried from Models) or call API services, instead they execute Actions to accomplish tasks.

Views are often structured in a hierarchy, with parent Views composed of multiple children. In these cases, parents do not pass application data directly to children - the children get relevant data directly from Models.

Views typically maintain an internal View-Model which is a data structure mapped directly to the rendered UI and derived from Model data. When the View is notified that Models it subscribes to have changed, it queries those Models, transforms the data as needed, and stores relevant parts in the View-Model.

This View-Model may be updated by user interaction in which case it is holding intermediate state which may be made permanent by executing an Action. This would be a typical approach for implementing a Form: as the user edits the Form the changes update the View-Model. When the user submits the Form a SubmitFormAction is executed and the View-Model data is passed to the Action.

View renders may be optimized by using immutable state for the View-Model and using strict equality checks to determine if the View needs to re-render. However, in cases where the View-Model is mapped to Form elements and changes frequently it is usually better to use mutable state.


## Comparing Maxwell with Facebook Flux

Maxwell and [Facebook Flux](http://facebook.github.io/flux/docs/overview.html) have several similarities:
* unidirectional data flow
* views, actions, stores (called models in Maxwell)
* design patterns rather than formal framework

The most significant difference is that Maxwell does not invert the Action to Store/Model relationship.

Per the Flux Overview:
> "Control is inverted with stores: the stores accept updates and reconcile them as appropriate, rather than depending on something external to update its data in a consistent way. Nothing outside the store has any insight into how it manages the data for its domain, helping to keep a clear separation of concerns. Stores have no direct setter methods like setAsRead(), but instead have only a single way of getting new data into their self-contained world — the callback they register with the dispatcher."

In contrast, Maxwell Actions encapsulate the information about which Models must be updated and in which order.     Instead of going through a Dispatcher, Maxwell Actions directly update Models. There are several reasons for taking this approach:

1. The logic for accomplishing a task is found in one place (the Action) rather than spread around multiple Stores. This makes it easier to reason about the flow of the Action. To understand the Action flow in a Flux application requires searching out all of the Stores that subscribe to the Action and imagining how they would interact if notified in various orders.

2. It is no longer necessary to couple Stores/Models to each other (Flux Dispatcher.waitFor) in order to coordinate their updates. The Action provides a clear coordination of Model updates in a single place.

3. Stores/Models are simplified to observable data stores with methods to get data, modify data, and subscribe/unsubscribe. They are no longer directly coupled to Actions

With this approach we are able to simplify the architecture while preserving the unidirectional data flow.


