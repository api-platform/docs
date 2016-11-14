# Migration from version 1 to version 2.

## Known issues :


1. Resources Declaration

Changing the resources declaration from services to annotation :



2. PHP7


Since V2 is in PHP7 make sure that your codebase is compatible with PHP7


3. Filters and custom filters
Custom Filters should be registered like before
Alot of filters are now present by default, you have to [enable them like settled in the documentation](core/filters.md) since the way to declare filters has changed

4. Event system

The event system has been totally rebuild. You will to implement `EventSubscriberInterface` and the contract of this class (function getSubscribedEvents) and declare which events you are subscribing to with the priority.

You don't have to register this in the service.yml file thanks to [`DunglasActionBundle`](https://github.com/dunglas/DunglasActionBundle)
