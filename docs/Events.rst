Developer's Guide To Events
###########################

.. contents::


Introduction
============


There are several points in the event processing pipeline in which a developer 
may interact with events. This guide outlines all of the ways that ZenPack 
developers can take advantage of these integration points to provide custom 
functionality and behavior.


Event Details
=============

Events may have arbitrary data attached to them. This data can be accessed in 
``zeneventd`` plugins, transforms and ``zeneventserver`` plugins. This data is 
also returned with events whenever ``zeneventserver`` is queried. By default, 
this data is not indexed and cannot be used to create an event filter.

When choosing custom detail keys, the naming convention is to use lowercased 
zenpack dotted notation: "ZenPacks.example.Techniques" becomes 
"zenpacks.example.techniques" with whatever key or keys you would like: 
"zenpacks.example.techniques.my_custom_detail".


Indexed Event Details
---------------------

Custom details must be indexed in order to be used in event filters. To specify
details to be indexed, a ZenPack must provide a ``zep.json`` file. This file 
should reside in the ``zep`` directory in the root of your ZenPack (i.e.: 
``ZenPacks/example/Techniques/zep/zep.json``). This file specifies details to 
be indexed using the detail key, type and human-readable name. Details must be
provided under the 'EventDetailItem' key which corresponds to the 
EventDetailItem protobuf class 
(see: zenoss.protocols.protobufs.zep_pb2.EventDetailItem).

An example file:

::

    {
        "EventDetailItem": [
            {
                "key": "zenpacks.example.techniques.my_custom_detail",
                "type": "1",
                "name": "My Custom Detail"
            },
            {
                "key": "zenpacks.example.techniques.my_event_color",
                "type": "2",
                "name": "Event Color"
            }
        ]
    }

Upon installation, ``zeneventserver`` will begin indexing events by the details
described in this JSON file. After removal of this ZenPack, these details will
no longer be able to be queried against.


Custom Detail Renderers
^^^^^^^^^^^^^^^^^^^^^^^

Along with custom details, ZenPacks may provide custom column definitions and 
renderers for the custom details in the event console. This is done by adding
custom column definitions in javascript provided by the ZenPack.

**@TODO : More info.**

**@TODO : Examples.**


Add Details From The Command Line
---------------------------------

To add details from the command line, specify the ``-o`` or ``--other`` 
arguments. More detail can be seen in the ``zensendevent --help`` output.
An example::

    $ zensendevent -d localhost -s Critical --other="zenpacks.example.techniques.my_custom_detail='my value'"

The value for a detail by default has the type of ``str``.


Add Details From Python
-----------------------


Using ZenEventManager
^^^^^^^^^^^^^^^^^^^^^

When using ZenEventManager, any additional keys passed into the ``sendEvent``
method that do not correspond to a known event attribute are considered 
details. This example can be run from within zendmd::

    event = {
        'device' : "localhost",
        'severity' : "Critical",
        'message' : "This is a test event.",
        'zenpacks.example.techniques.example_detail' : 'A custom detail example.'
    }
    dmd.ZenEventManager.sendEvent(event)


Using Protobufs
^^^^^^^^^^^^^^^

Developers may choose to construct event protobufs and manually publish them
for processing. Custom details must be added explicitly to protobuf Event
objects. The following shows two methods for manually constructing and 
publishing events.

::

    from Products.ZenMessaging.queuemessaging.interfaces import IEventPublisher
    
    publisher = getUtility(IEventPublisher)
    
    # When interacting with enum types, the values are imported at the module
    # level. This import just shows an example of how to properly import values
    # from enums.
    from zenoss.protocols.protobufs.zep_pb2 import Event, SEVERITY_CRITICAL


    # Create a protobuf directly and publish it.
    protobuf_event = Event()
    protobuf_event.actor.element_identifier = 'localhost'
    protobuf_event.severity = SEVERITY_CRITICAL
    protobuf_event.event_class = "/App/Info"
    detail = protobuf_event.details.add()
    detail.name = 'zenpacks.example.techniques.my_custom_detail'
    detail.value.append("My custom detail value")
    publisher.publish(protobuf_event)
    
    
    # Use a utility to quickly go from protobuf -> dict -> protobuf
    from zenoss.protocols.jsonformat import to_dict, from_dict
    
    # Create a dictionary (slower due to conversion)
    event_dict = {
        'actor' : {
            'element_identifier' : 'localhost'
        },
        'details' : [
            {
                'name' : 'zenpacks.example.techniques.my_other_detail',
                'value' : [
                    'My Other Detail Value.'
                ]
            }
        ],
        'event_class' : '/App/Info',
        'severity' : SEVERITY_CRITICAL
    }
    second_event = from_dict(Event, event_dict)
    publisher.publish(second_event)

.. Note:: When manually constructing events with custom details, all details 
    are considered multi-valued.


Manipulating Details In Transforms
----------------------------------

Custom event details can be accessed and manipulated in transforms as well. If
using the recommended detail key dot notation, ``getattr`` and ``setattr`` will
need to be used. An example transform::
    
    my_detail_key = 'zenpacks.example.techniques.my_transform_detail'
    if getattr(evt, my_detail_key) == 'super special value':
        setattr(evt, my_detail_key, 'another special value')
        evt.device = 'localhost.localdomain'


Manipulating Details In Plugins
-------------------------------

The ``zeneventd`` pre- and post-plugins implement the same interface:
``Products.ZenEvents.interfaces.IEventPlugin``. This interface specifies a
single method. An example of an implementation::

    def apply(self, event, dmd):
        my_detail_key = 'zenpacks.example.techniques.my_transform_detail'
        if getattr(event, my_detail_key) == 'super special value':
            setattr(event, my_detail_key, 'another special value, from plugins.')


There is an additional plugin provided by the interface 
``IEventIdentifierPlugin``. This plugin specifies a different method, but the 
implementation is similar::

    def resolveIdentifiers(self, eventContext, eventProcessorManager):
        my_special_key = 'zenpacks.example.techniques.special_id'
        if not getattr(eventContext.eventProxy, my_special_key, False):
            eventContext.eventProxy.device = 'localhost.localdomain'



``zeneventd`` Plugins
=====================


Check the event plugin directives:
Products.ZenModel.ZenPackTemplate.CONTENT.configure.zcml

- Pre/Post plugins
- Identifier plugin


From here:

    from Products.ZenEvents.interfaces import IPreEventPlugin, IPostEventPlugin, IEventIdentifierPlugin

General pipeline workings:

0. **IPreEventPlugin** - this plugin is called before anything else happens.
    see ZenPacks.zenoss.MultiRealmIP/ZenPacks/zenoss/MultiRealmIP/__init__.py
    This just sets a detail.

1. Input Check - verify required fields are present (actor, summary, severity)

2. Identified - Run all **IEventIdentifierPlugin** - the Zenoss BaseEventIdentifierPlugin is run last.
    see ZenPacks.zenoss.MultiRealmIP/ZenPacks/zenoss/MultiRealmIP/__init__.py
    This zenpack uses some custom logic to identify an element.
    
3. Device Context Added, Tags added

4. Transformed - If the device or component changes, step 2 and 3 are re-run.

5. Event Class Context Added, Tags added

6. Fingerprint calculated

7. **IPostEventPlugin** - Post-processing plugins are run.


If you want to drop an event, raise DropEvent from here:

    from Products.ZenEvents.events2.processing import DropEvent

signature is: (message, event)

If the event makes it all the way through the pipeline, it gets published to 
the zep exchanged to be consumed by zep.


``zeneventserver`` Plugins
==========================

- EventPreCreatePlugin
- EventPostCreatePlugin
- EventPostIndexPlugin
- EventUpdatePlugin


The Fanout Queue
================

Events are sent to this fanout queue through an ``EventPostIndexPlugin`` 
(``EventFanOutPlugin``). Triggers are also run as an ``EventPostIndexPlugin``.

