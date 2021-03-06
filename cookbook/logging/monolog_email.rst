.. index::
   single: Logging; Emailing errors

How to Configure Monolog to Email Errors
========================================

Monolog_ can be configured to send an email when an error occurs with an
application. The configuration for this requires a few nested handlers
in order to avoid receiving too many emails. This configuration looks
complicated at first but each handler is fairly straightforward when
it is broken down.

.. configuration-block::

    .. code-block:: yaml

        # app/config/config_prod.yml
        monolog:
            handlers:
                mail:
                    type:         fingers_crossed
                    # 500 errors are logged at the critical level
                    action_level: critical
                    # to also log 400 level errors (but not 404's):
                    # action_level: error
                    # excluded_404s:
                    #     - ^/
                    handler:      buffered
                buffered:
                    type:    buffer
                    handler: swift
                swift:
                    type:       swift_mailer
                    from_email: 'error@example.com'
                    to_email:   'error@example.com'
                    # or list of recipients
                    # to_email:   ['dev1@example.com', 'dev2@example.com', ...]
                    subject:    An Error Occurred!
                    level:      debug

    .. code-block:: xml

        <!-- app/config/config_prod.xml -->
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:monolog="http://symfony.com/schema/dic/monolog"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd
                                http://symfony.com/schema/dic/monolog http://symfony.com/schema/dic/monolog/monolog-1.0.xsd">

            <monolog:config>
                <monolog:handler
                    name="mail"
                    type="fingers_crossed"
                    action-level="critical"
                    handler="buffered"
                    <!--
                    To also log 400 level errors (but not 404's):
                    action-level="error"
                    And add this child inside this monolog:handler
                    <monolog:excluded-404>^/</monolog:excluded-404>
                    -->
                />
                <monolog:handler
                    name="buffered"
                    type="buffer"
                    handler="swift"
                />
                <monolog:handler
                    name="swift"
                    type="swift_mailer"
                    from-email="error@example.com"
                    subject="An Error Occurred!"
                    level="debug">

                    <monolog:to-email>error@example.com</monolog:to-email>

                    <!-- or multiple to-email elements -->
                    <!--
                    <monolog:to-email>dev1@example.com</monolog:to-email>
                    <monolog:to-email>dev2@example.com</monolog:to-email>
                    ...
                    -->
                </monolog:handler>
            </monolog:config>
        </container>

    .. code-block:: php

        // app/config/config_prod.php
        $container->loadFromExtension('monolog', array(
            'handlers' => array(
                'mail' => array(
                    'type'         => 'fingers_crossed',
                    'action_level' => 'critical',
                    // to also log 400 level errors (but not 404's):
                    // 'action_level' => 'error',
                    // 'excluded_404s' => array(
                    //     '^/',
                    // ),
                    'handler'      => 'buffered',
                ),
                'buffered' => array(
                    'type'    => 'buffer',
                    'handler' => 'swift',
                ),
                'swift' => array(
                    'type'       => 'swift_mailer',
                    'from_email' => 'error@example.com',
                    'to_email'   => 'error@example.com',
                    // or a list of recipients
                    // 'to_email'   => array('dev1@example.com', 'dev2@example.com', ...),
                    'subject'    => 'An Error Occurred!',
                    'level'      => 'debug',
                ),
            ),
        ));

The ``mail`` handler is a ``fingers_crossed`` handler which means that
it is only triggered when the action level, in this case ``critical`` is reached.
The ``critical`` level is only triggered for 5xx HTTP code errors. If this level
is reached once, the ``fingers_crossed`` handler will log all messages
regardless of their level. The ``handler`` setting means that the output
is then passed onto the ``buffered`` handler.

.. tip::

    If you want both 400 level and 500 level errors to trigger an email,
    set the ``action_level`` to ``error`` instead of ``critical``. See the
    code above for an example.

The ``buffered`` handler simply keeps all the messages for a request and
then passes them onto the nested handler in one go. If you do not use this
handler then each message will be emailed separately. This is then passed
to the ``swift`` handler. This is the handler that actually deals with
emailing you the error. The settings for this are straightforward, the
to and from addresses and the subject.

You can combine these handlers with other handlers so that the errors still
get logged on the server as well as the emails being sent:

.. configuration-block::

    .. code-block:: yaml

        # app/config/config_prod.yml
        monolog:
            handlers:
                main:
                    type:         fingers_crossed
                    action_level: critical
                    handler:      grouped
                grouped:
                    type:    group
                    members: [streamed, buffered]
                streamed:
                    type:  stream
                    path:  "%kernel.logs_dir%/%kernel.environment%.log"
                    level: debug
                buffered:
                    type:    buffer
                    handler: swift
                swift:
                    type:       swift_mailer
                    from_email: 'error@example.com'
                    to_email:   'error@example.com'
                    subject:    An Error Occurred!
                    level:      debug

    .. code-block:: xml

        <!-- app/config/config_prod.xml -->
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:monolog="http://symfony.com/schema/dic/monolog"
            xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd
                                http://symfony.com/schema/dic/monolog http://symfony.com/schema/dic/monolog/monolog-1.0.xsd">

            <monolog:config>
                <monolog:handler
                    name="main"
                    type="fingers_crossed"
                    action_level="critical"
                    handler="grouped"
                />
                <monolog:handler
                    name="grouped"
                    type="group"
                >
                    <member type="stream"/>
                    <member type="buffered"/>
                </monolog:handler>
                <monolog:handler
                    name="stream"
                    path="%kernel.logs_dir%/%kernel.environment%.log"
                    level="debug"
                />
                <monolog:handler
                    name="buffered"
                    type="buffer"
                    handler="swift"
                />
                <monolog:handler
                    name="swift"
                    from-email="error@example.com"
                    to-email="error@example.com"
                    subject="An Error Occurred!"
                    level="debug"
                />
            </monolog:config>
        </container>

    .. code-block:: php

        // app/config/config_prod.php
        $container->loadFromExtension('monolog', array(
            'handlers' => array(
                'main' => array(
                    'type'         => 'fingers_crossed',
                    'action_level' => 'critical',
                    'handler'      => 'grouped',
                ),
                'grouped' => array(
                    'type'    => 'group',
                    'members' => array('streamed', 'buffered'),
                ),
                'streamed'  => array(
                    'type'  => 'stream',
                    'path'  => '%kernel.logs_dir%/%kernel.environment%.log',
                    'level' => 'debug',
                ),
                'buffered'    => array(
                    'type'    => 'buffer',
                    'handler' => 'swift',
                ),
                'swift' => array(
                    'type'       => 'swift_mailer',
                    'from_email' => 'error@example.com',
                    'to_email'   => 'error@example.com',
                    'subject'    => 'An Error Occurred!',
                    'level'      => 'debug',
                ),
            ),
        ));

This uses the ``group`` handler to send the messages to the two
group members, the ``buffered`` and the ``stream`` handlers. The messages will
now be both written to the log file and emailed.

.. _Monolog: https://github.com/Seldaek/monolog
