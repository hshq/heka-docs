.. _configuration:

=================
Configuring hekad
=================

`hekad` is the primary daemon application deployed to collect node data
and route messages to the back-ends. It's a multi-purpose agent as it
can act in several different roles depending on how its configured.

A simple example configuration file:

.. code-block:: javascript

    {
        "inputs": [
            {
                "name": "udp:29330",
                "type": "UdpInput",
                "address": "127.0.0.1:29330"
            }
        ],
        "decoders": [
            {"name": "json", type": "JsonDecoder", "default": true}
        ],
        "outputs": [
            {
                "name": "default_log",
                "type": "FileOutput",
                "path": "/var/log/hekad.log",
                "prefix_ts": true
            },
            {
                "name": "json_log",
                "type": "FileOutput",
                "path": "/var/log/hekad.json-log",
                "format": "json"
            }
        ],
        "chains": {
            "default": {
                "outputs": ["default_log"]
            },
            "json": {
                "message_type": ["SomeMessageType"],
                "outputs": ["json_log"]
            }
        }
    }

This rather contrived example will accept UDP input on the specified address,
decode messages that arrive serialized as JSON, and then write the message to
either a text or a JSON log file, depending on the message type.

Inputs, decoders, filters, and outputs are all hekad plug-ins and have
some configuration keys in common. Individual plug-ins may have
additional optional or required parameters as well.

Each plug-in declaration results in an 'instance' of the plug-in being
created when `hekad` starts up.

Common Roles
============

- **Agent** - Single default chain that passes all messages directly to
  another `hekad` daemon on a separate machine configured as an
  Router.
- **Aggregator** - Runs filters that can roll-up statistics (similar to
  statsd), and handles aggregating similar messages before saving them
  to a back-end directly or possibly forwarding them to a `hekad`
  router.
- **Router** - Collects input messages from multiple sources (including
  other `hekad` daemons acting as Agents), rolls up stats, and routes
  messages to appropriate back-ends.

Command Line Options
====================

--config
--------

Specifies a configuration file to load. Defaults to `agent.conf` in the
current directory.

--maxprocs
----------

Specifies how many processors hekad should use. Best performance is
usually attained by setting this to 2 x (number of cores). This assumes
each core is hyper-threaded.

--poolSize
----------

Size of the message pipeline. This value determines how many messages
can be 'in-flight' at once in `hekad`. It's default value of `1000` is
usually sufficient and performs optimally.

Configuration File Format
=========================

hekad's configuration file is a plain JSON text file that designates
several keys to configure the various hekad plug-ins:

- inputs
- decoders
- filters
- outputs
- chains

Individual sections of the configuration file will need to reference
other sections, as the `chains` section does.

Inputs, decoders, filters, and output plug-ins must each be specified
by `type` and may optionally supply a `name` to be used for referring
to it later. Plug-ins may be specified multiple times as needed. For
example, if `hekad` should listen on multiple UDP sockets then it can
be added twice with the appropriate `address` for each IP/port to
listen on. They then must be named to avoid plug-in name conflicts.

Inputs
======

MessageGeneratorInput
---------------------

Parameters: **None**

Allows other plug-ins to generate messages. This input plug-in makes a
channel available for other plug-ins that need to create messages at
different points in time. Plug-ins requiring this input will indicate
it as a prerequisite.

Multiple plug-ins may use a single instance of the
MessageGeneratorInput.

UdpInput
--------

Parameters:

    - Address (string): An IP address:port.

Example:

.. code-block:: javascript

    {
        "type": "UdpInput",
        "address": "127.0.0.1:4880"
    }

Listens on a specific UDP address and port for messages.

Decoders
========

One of the decoders specified must include the key/value of:

.. code-block:: javascript

    "default": true

so that unknown messages are passed through a default decoder if a
decoder cannot be determined.

JsonDecoder
-----------

Parameters: **None**

Decodes binary messages that were JSON serialized into a hekad message.
Metlog clients frequently encode their messages as JSON.


MsgPackDecoder
--------------

Parameters: **None**

Decodes binary messsages that were msgpack encoded into a hekad
message.

.. seealso:: `Msgpack website <http://msgpack.org/>`_

Filters
=======

StatRollupFilter
----------------

Prerequisites:

    - MessageGeneratorInput must be configured.
    - Message must be of type `counter`, `gauge`, or `timer`.

Parameters:

    - FlushInterval (int): How often the stats should be rolled up and
      flushed. Defaults to ``10``.
    - PercentThreshold (int): Threshold value for timer outliers to
      ignore. Defaults to ``90``.

A rollup occurs every `FlushInterval` seconds, which then causes
MessageGeneratorInput to emit a new message of type `statmetric`.

Outputs
=======

CounterOutput
-------------

Parameters: **None**

Prints to stdout a count every second of how many messages were seen.
Every 10 seconds an aggregate count with an average per second is
printed to stdout.

FileOutput
----------

Parameters:

    - Path (string): Path to the file to write.
    - Format (string): Output format for the message to be written.
      Can be either `json` or `text`. Defaults to ``text``.
    - Prefix_ts (bool): Whether a timestamp should be prefixed to each
      message line in the file. Defaults to ``false``.
    - Perm (int): File permission for writing. Defaults to ``0666``.

Writes a message to the designated file in the format given (including
a prefixed timestamp if configured).

LogOutput
---------

Parameters: **None**

Logs the message to stdout.

Chains
======

A chain describes the set of filters and outputs to apply to a specific
message. A default chain must be declared which will be used if no
other chain matches the message.

The message will be passed to every filter and output named in the
configuration. Some filters may alter the remaining output list used or
consume a message entirely which will prevent later filters and outputs
from seeing it.

.. note::

    At the moment chains can only match a message based on the message
    type.

Example
-------

.. code-block:: javascript

    "chains": {
        "stats": {
            "message_type": ["Counter", "Timer", "Gauge"],
            "filters": ["StatRollupFilter"]
        },
        "stat_dump": {
            "message_type": ["StatMetric"],
            "outputs": ["GraphiteOutput"]
        },
        "default": {
            "outputs": ["LogFileOutput"]
        }
    }

The chain named ``default`` will be used in the event a message does
not match ``StatMetric``, ``Counter``, ``Timer``, or ``Gauge``. Each
chain may contain two keyed sections: ``filters`` and / or ``outputs``.
They must be lists that indicate the configured plugin to use, and
refer to it either by the plugins configured `name` or if the `name`
for the plugin is omitted, its full `plugin type` (As the above example
refers to them).

The chain must include the key ``message_type`` to differentiate what
message types will trigger it unless its the `default` chain.
