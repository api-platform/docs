# Upgrade path from v1.x to v2.x

## Known issues :


1. Resources Declaration

Changing the resources declaration from services to annotation :



2. PHP7


Since V2 is in PHP7 make sure that your codebase is compatible with PHP7


3. Filters and custom filters
Custom Filters should be registered like before
Alot of filters are now present by default, you have to [enable them like settled in the documentation](core/filters.md)

4. Event system

The event system has been totally rebuild. You will to implement `EventSubscriberInterface` and the contract of this class (function getSubscribedEvents) and declare which events you are subscribing to with the priority.

You have to register this in the service.yml file but without the tag
