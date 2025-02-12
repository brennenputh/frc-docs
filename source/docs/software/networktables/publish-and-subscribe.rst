Publishing and Subscribing to a Topic
=====================================

Publishing to a Topic
---------------------

In order to create a :term:`topic` and publish values to it, it's necessary to create a :term:`publisher`.

NetworkTable publishers are represented as type-specific Publisher objects (e.g. ``BooleanPublisher``: `Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/BooleanPublisher.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_boolean_publisher.html>`__). Publishers are only active as long as the Publisher object exists. Typically you want to keep publishing longer than the local scope of a function, so it's necessary to store the Publisher object somewhere longer term, e.g. in an instance variable. In Java, the ``close()`` method needs be called to stop publishing; in C++ this is handled by the destructor. C++ publishers are moveable and non-copyable.

In the handle-based APIs, there is only the non-type-specific ``NT_Publisher`` handle; the user is responsible for keeping track of the type of the publisher and using the correct type-specific set methods.

Publishing values is done via a ``set()`` operation. By default, this operation uses the current time, but a timestamp may optionally be specified. Specifying a timestamp can be useful when multiple values should have the same update timestamp. The timestamp units are integer microseconds (see example code for how to get a current timestamp that is consistent with the library).

.. tabs::

    .. group-tab:: Java

        .. code-block:: java

            public class Example {
              // the publisher is an instance variable so its lifetime matches that of the class
              final DoublePublisher dblPub;

              public Example(DoubleTopic dblTopic) {
                // start publishing; the return value must be retained (in this case, via
                // an instance variable)
                dblPub = dblTopic.publish();

                // publish options may be specified using PubSubOption
                dblPub = dblTopic.publish(PubSubOption.keepDuplicates(true));

                // publishEx provides additional options such as setting initial
                // properties and using a custom type string. Using a custom type string for
                // types other than raw and string is not recommended. The properties string
                // must be a JSON map.
                dblPub = dblTopic.publishEx("double", "{\"myprop\": 5}");
              }

              public void periodic() {
                // publish a default value
                dblPub.setDefault(0.0);

                // publish a value with current timestamp
                dblPub.set(1.0);
                dblPub.set(2.0, 0);  // 0 = use current time

                // publish a value with a specific timestamp; NetworkTablesJNI.now() can
                // be used to get the current time. On the roboRIO, this is the same as
                // the FPGA timestamp (e.g. RobotController.getFPGATime())
                long time = NetworkTablesJNI.now();
                dblPub.set(3.0, time);

                // publishers also implement the appropriate Consumer functional interface;
                // this example assumes void myFunc(DoubleConsumer func) exists
                myFunc(dblPub);
              }

              // often not required in robot code, unless this class doesn't exist for
              // the lifetime of the entire robot program, in which case close() needs to be
              // called to stop publishing
              public void close() {
                // stop publishing
                dblPub.close();
              }
            }

    .. group-tab:: C++

        .. code-block:: cpp

            class Example {
              // the publisher is an instance variable so its lifetime matches that of the class
              // publishing is automatically stopped when dblPub is destroyed by the class destructor
              nt::DoublePublisher dblPub;

             public:
              explicit Example(nt::DoubleTopic dblTopic) {
                // start publishing; the return value must be retained (in this case, via
                // an instance variable)
                dblPub = dblTopic.Publish();

                // publish options may be specified using PubSubOption
                dblPub = dblTopic.Publish({{nt::PubSubOption::KeepDuplicates(true)}});

                // PublishEx provides additional options such as setting initial
                // properties and using a custom type string. Using a custom type string for
                // types other than raw and string is not recommended. The properties must
                // be a JSON map.
                dblPub = dblTopic.PublishEx("double", {{"myprop", 5}});
              }

              void Periodic() {
                // publish a default value
                dblPub.SetDefault(0.0);

                // publish a value with current timestamp
                dblPub.Set(1.0);
                dblPub.Set(2.0, 0);  // 0 = use current time

                // publish a value with a specific timestamp; nt::Now() can
                // be used to get the current time.
                int64_t time = nt::Now();
                dblPub.Set(3.0, time);
              }
            };

    .. group-tab:: C++ (handle-based)

        .. code-block:: cpp

            class Example {
              // the publisher is an instance variable, but since it's a handle, it's
              // not automatically released, so we need a destructor
              NT_Publisher dblPub;

             public:
              explicit Example(NT_Topic dblTopic) {
                // start publishing. It's recommended that the type string be standard
                // for all types except string and raw.
                dblPub = nt::Publish(dblTopic, NT_DOUBLE, "double");

                // publish options may be specified using PubSubOption
                dblPub = nt::Publish(dblTopic, NT_DOUBLE, "double",
                    {{nt::PubSubOption::KeepDuplicates(true)}});

                // PublishEx allows setting initial properties. The
                // properties must be a JSON map.
                dblPub = nt::PublishEx(dblTopic, NT_DOUBLE, "double", {{"myprop", 5}});
              }

              void Periodic() {
                // publish a default value
                nt::SetDefaultDouble(dblPub, 0.0);

                // publish a value with current timestamp
                nt::SetDouble(dblPub, 1.0);
                nt::SetDouble(dblPub, 2.0, 0);  // 0 = use current time

                // publish a value with a specific timestamp; nt::Now() can
                // be used to get the current time.
                int64_t time = nt::Now();
                nt::SetDouble(dblPub, 3.0, time);
              }

              ~Example() {
                // stop publishing
                nt::Unpublish(dblPub);
              }
            };

    .. group-tab:: C

        .. code-block:: c

            // This code assumes that a NT_Topic dblTopic variable already exists

            // start publishing. It's recommended that the type string be standard
            // for all types except string and raw.
            NT_Publisher dblPub = NT_Publish(dblTopic, NT_DOUBLE, "double", NULL, 0);

            // publish options may be specified
            struct NT_PubSubOption options[1];
            options[0].type = NT_PUBSUB_KEEPDUPLICATES;
            options[0].value = 1;  // true
            NT_Publisher dblPub = NT_Publish(dblTopic, NT_DOUBLE, "double", options, 1);

            // PublishEx allows setting initial properties. The properties string must
            // be a JSON map.
            NT_Publisher dblPub =
                NT_PublishEx(dblTopic, NT_DOUBLE, "double", "{\"myprop\", 5}", NULL, 0);

            // publish a default value
            NT_SetDefaultDouble(dblPub, 0.0);

            // publish a value with current timestamp
            NT_SetDouble(dblPub, 1.0);
            NT_SetDouble(dblPub, 2.0, 0);  // 0 = use current time

            // publish a value with a specific timestamp; NT_Now() can
            // be used to get the current time.
            int64_t time = NT_Now();
            NT_SetDouble(dblPub, 3.0, time);

            // stop publishing
            NT_Unpublish(dblPub);


Subscribing to a Topic
----------------------

A :term:`subscriber` receives value updates made to a topic. Similar to publishers, NetworkTable subscribers are represented as type-specific Subscriber classes (e.g. ``BooleanSubscriber``: `Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/BooleanSubscriber.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_boolean_subscriber.html>`__) that must be stored somewhere to continue subscribing.

Subscribers have a range of different ways to read received values. It's possible to just read the most recent value using ``get()``, read the most recent value, along with its timestamp, using ``getAtomic()``, or get an array of all value changes since the last call using ``readQueue()`` or ``readQueueValues()``.

.. tabs::

    .. group-tab:: Java

        .. code-block:: java

            public class Example {
              // the subscriber is an instance variable so its lifetime matches that of the class
              final DoubleSubscriber dblSub;

              public Example(DoubleTopic dblTopic) {
                // start subscribing; the return value must be retained.
                // the parameter is the default value if no value is available when get() is called
                dblSub = dblTopic.subscribe(0.0);

                // subscribe options may be specified using PubSubOption
                dblSub =
                    dblTopic.subscribe(0.0, PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));

                // subscribeEx provides the options of using a custom type string.
                // Using a custom type string for types other than raw and string is not recommended.
                dblSub = dblTopic.subscribeEx("double", 0.0);
              }

              public void periodic() {
                // simple get of most recent value; if no value has been published,
                // returns the default value passed to the subscribe() function
                double val = dblSub.get();

                // get the most recent value; if no value has been published, returns
                // the passed-in default value
                double val = dblSub.get(-1.0);

                // subscribers also implement the appropriate Supplier interface, e.g. DoubleSupplier
                double val = dblSub.getAsDouble();

                // get the most recent value, along with its timestamp
                TimestampedDouble tsVal = dblSub.getAtomic();

                // read all value changes since the last call to readQueue/readQueueValues
                // readQueue() returns timestamps; readQueueValues() does not.
                TimestampedDouble[] tsUpdates = dblSub.readQueue();
                double[] valUpdates = dblSub.readQueueValues();
              }

              // often not required in robot code, unless this class doesn't exist for
              // the lifetime of the entire robot program, in which case close() needs to be
              // called to stop subscribing
              public void close() {
                // stop subscribing
                dblSub.close();
              }
            }

    .. group-tab:: C++

        .. code-block:: cpp

            class Example {
              // the subscriber is an instance variable so its lifetime matches that of the class
              // subscribing is automatically stopped when dblSub is destroyed by the class destructor
              nt::DoubleSubscriber dblSub;

             public:
              explicit Example(nt::DoubleTopic dblTopic) {
                // start subscribing; the return value must be retained.
                // the parameter is the default value if no value is available when get() is called
                dblSub = dblTopic.Subscribe(0.0);

                // subscribe options may be specified using PubSubOption
                dblSub =
                    dblTopic.subscribe(0.0,
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});

                // SubscribeEx provides the options of using a custom type string.
                // Using a custom type string for types other than raw and string is not recommended.
                dblSub = dblTopic.SubscribeEx("double", 0.0);
              }

              void Periodic() {
                // simple get of most recent value; if no value has been published,
                // returns the default value passed to the Subscribe() function
                double val = dblSub.Get();

                // get the most recent value; if no value has been published, returns
                // the passed-in default value
                double val = dblSub.Get(-1.0);

                // get the most recent value, along with its timestamp
                nt::TimestampedDouble tsVal = dblSub.GetAtomic();

                // read all value changes since the last call to ReadQueue/ReadQueueValues
                // ReadQueue() returns timestamps; ReadQueueValues() does not.
                std::vector<nt::TimestampedDouble> tsUpdates = dblSub.ReadQueue();
                std::vector<double> valUpdates = dblSub.ReadQueueValues();
              }
            };

    .. group-tab:: C++ (handle-based)

        .. code-block:: cpp

            class Example {
              // the subscriber is an instance variable, but since it's a handle, it's
              // not automatically released, so we need a destructor
              NT_Subscriber dblSub;

             public:
              explicit Example(NT_Topic dblTopic) {
                // start subscribing
                // Using a custom type string for types other than raw and string is not recommended.
                dblSub = nt::Subscribe(dblTopic, NT_DOUBLE, "double");

                // subscribe options may be specified using PubSubOption
                dblSub =
                    nt::Subscribe(dblTopic, NT_DOUBLE, "double",
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});
              }

              void Periodic() {
                // get the most recent value; if no value has been published, returns
                // the passed-in default value
                double val = nt::GetDouble(dblSub, 0.0);

                // get the most recent value, along with its timestamp
                nt::TimestampedDouble tsVal = nt::GetAtomic(dblSub, 0.0);

                // read all value changes since the last call to ReadQueue/ReadQueueValues
                // ReadQueue() returns timestamps; ReadQueueValues() does not.
                std::vector<nt::TimestampedDouble> tsUpdates = nt::ReadQueueDouble(dblSub);
                std::vector<double> valUpdates = nt::ReadQueueValuesDouble(dblSub);
              }

              ~Example() {
                // stop subscribing
                nt::Unsubscribe(dblSub);
              }

    .. group-tab:: C

        .. code-block:: c

            // This code assumes that a NT_Topic dblTopic variable already exists

            // start subscribing
            // Using a custom type string for types other than raw and string is not recommended.
            NT_Subscriber dblSub = NT_Subscribe(dblTopic, NT_DOUBLE, "double", NULL, 0);

            // subscribe options may be specified using NT_PubSubOption
            struct NT_PubSubOption options[2];
            options[0].type = NT_PUBSUB_KEEPDUPLICATES;
            options[0].value = 1;  // true
            options[1].type = NT_PUBSUB_POLLSTORAGE;
            options[1].value = 10;
            NT_Subscriber dblSub = NT_Subscribe(dblTopic, NT_DOUBLE, "double", options, 2);

            // get the most recent value; if no value has been published, returns
            // the passed-in default value
            double val = NT_GetDouble(dblSub, 0.0);

            // get the most recent value, along with its timestamp
            struct NT_TimestampedDouble tsVal;
            NT_GetAtomic(dblSub, 0.0, &tsVal);
            NT_DisposeTimestamped(&tsVal);

            // read all value changes since the last call to ReadQueue/ReadQueueValues
            // ReadQueue() returns timestamps; ReadQueueValues() does not.
            size_t tsUpdatesLen;
            struct NT_TimestampedDouble* tsUpdates = NT_ReadQueueDouble(dblSub, &tsUpdatesLen);
            NT_FreeQueueDouble(tsUpdates, tsUpdatesLen);

            size_t valUpdatesLen;
            double* valUpdates = NT_ReadQueueValuesDouble(dblSub, &valUpdatesLen);
            NT_FreeDoubleArray(valUpdates, valUpdatesLen);

            // stop subscribing
            NT_Unsubscribe(dblSub);


Using Entry to Both Subscribe and Publish
-----------------------------------------

An :term:`entry` is a combined publisher and subscriber. The subscriber is always active, but the publisher is not created until a publish operation is performed (e.g. a value is "set", aka published, on the entry). This may be more convenient than maintaining a separate publisher and subscriber. Similar to publishers and subscribers, NetworkTable entries are represented as type-specific Entry classes (e.g. ``BooleanEntry``: `Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/BooleanEntry.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_boolean_entry.html>`__) that must be retained to continue subscribing (and publishing).

.. tabs::

    .. group-tab:: Java

        .. code-block:: java

            public class Example {
              // the entry is an instance variable so its lifetime matches that of the class
              final DoubleEntry dblEntry;

              public Example(DoubleTopic dblTopic) {
                // start subscribing; the return value must be retained.
                // the parameter is the default value if no value is available when get() is called
                dblEntry = dblTopic.getEntry(0.0);

                // publish and subscribe options may be specified using PubSubOption
                dblEntry =
                    dblTopic.getEntry(0.0, PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));

                // getEntryEx provides the options of using a custom type string.
                // Using a custom type string for types other than raw and string is not recommended.
                dblEntry = dblTopic.getEntryEx("double", 0.0);
              }

              public void periodic() {
                // entries support all the same methods as subscribers:
                double val = dblEntry.get();
                double val = dblEntry.get(-1.0);
                double val = dblEntry.getAsDouble();
                TimestampedDouble tsVal = dblEntry.getAtomic();
                TimestampedDouble[] tsUpdates = dblEntry.readQueue();
                double[] valUpdates = dblEntry.readQueueValues();

                // entries also support all the same methods as publishers; the first time
                // one of these is called, an internal publisher is automatically created
                dblEntry.setDefault(0.0);
                dblEntry.set(1.0);
                dblEntry.set(2.0, 0);  // 0 = use current time
                long time = NetworkTablesJNI.now();
                dblEntry.set(3.0, time);
                myFunc(dblEntry);
              }

              public void unpublish() {
                // you can stop publishing while keeping the subscriber alive
                dblEntry.unpublish();
              }

              // often not required in robot code, unless this class doesn't exist for
              // the lifetime of the entire robot program, in which case close() needs to be
              // called to stop subscribing
              public void close() {
                // stop subscribing/publishing
                dblEntry.close();
              }
            }

    .. group-tab:: C++

        .. code-block:: cpp

            class Example {
              // the entry is an instance variable so its lifetime matches that of the class
              // subscribing/publishing is automatically stopped when dblEntry is destroyed by
              // the class destructor
              nt::DoubleEntry dblEntry;

             public:
              explicit Example(nt::DoubleTopic dblTopic) {
                // start subscribing; the return value must be retained.
                // the parameter is the default value if no value is available when get() is called
                dblEntry = dblTopic.GetEntry(0.0);

                // publish and subscribe options may be specified using PubSubOption
                dblEntry =
                    dblTopic.GetEntry(0.0,
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});

                // GetEntryEx provides the options of using a custom type string.
                // Using a custom type string for types other than raw and string is not recommended.
                dblEntry = dblTopic.GetEntryEx("double", 0.0);
              }

              void Periodic() {
                // entries support all the same methods as subscribers:
                double val = dblEntry.Get();
                double val = dblEntry.Get(-1.0);
                nt::TimestampedDouble tsVal = dblEntry.GetAtomic();
                std::vector<nt::TimestampedDouble> tsUpdates = dblEntry.ReadQueue();
                std::vector<double> valUpdates = dblEntry.ReadQueueValues();

                // entries also support all the same methods as publishers; the first time
                // one of these is called, an internal publisher is automatically created
                dblEntry.SetDefault(0.0);
                dblEntry.Set(1.0);
                dblEntry.Set(2.0, 0);  // 0 = use current time
                int64_t time = nt::Now();
                dblEntry.Set(3.0, time);
              }

              void Unpublish() {
                // you can stop publishing while keeping the subscriber alive
                dblEntry.Unpublish();
              }
            };

    .. group-tab:: C++ (handle-based)

        .. code-block:: cpp

            class Example {
              // the entry is an instance variable, but since it's a handle, it's
              // not automatically released, so we need a destructor
              NT_Entry dblEntry;

             public:
              explicit Example(NT_Topic dblTopic) {
                // start subscribing
                // Using a custom type string for types other than raw and string is not recommended.
                dblEntry = nt::GetEntry(dblTopic, NT_DOUBLE, "double");

                // publish and subscribe options may be specified using PubSubOption
                dblEntry =
                    nt::GetEntry(dblTopic, NT_DOUBLE, "double",
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});
              }

              void Periodic() {
                // entries support all the same methods as subscribers:
                double val = nt::GetDouble(dblEntry, 0.0);
                nt::TimestampedDouble tsVal = nt::GetAtomic(dblEntry, 0.0);
                std::vector<nt::TimestampedDouble> tsUpdates = nt::ReadQueueDouble(dblEntry);
                std::vector<double> valUpdates = nt::ReadQueueValuesDouble(dblEntry);

                // entries also support all the same methods as publishers; the first time
                // one of these is called, an internal publisher is automatically created
                nt::SetDefaultDouble(dblPub, 0.0);
                nt::SetDouble(dblPub, 1.0);
                nt::SetDouble(dblPub, 2.0, 0);  // 0 = use current time
                int64_t time = nt::Now();
                nt::SetDouble(dblPub, 3.0, time);
              }

              void Unpublish() {
                // you can stop publishing while keeping the subscriber alive
                nt::Unpublish(dblEntry);
              }

              ~Example() {
                // stop publishing and subscribing
                nt::ReleaseEntry(dblEntry);
              }

    .. group-tab:: C

        .. code-block:: c

            // This code assumes that a NT_Topic dblTopic variable already exists

            // start subscribing
            // Using a custom type string for types other than raw and string is not recommended.
            NT_Entry dblEntry = NT_GetEntryEx(dblTopic, NT_DOUBLE, "double", NULL, 0);

            // publish and subscribe options may be specified using NT_PubSubOption
            struct NT_PubSubOption options[2];
            options[0].type = NT_PUBSUB_KEEPDUPLICATES;
            options[0].value = 1;  // true
            options[1].type = NT_PUBSUB_POLLSTORAGE;
            options[1].value = 10;
            NT_Entry dblEntry = NT_GetEntryEx(dblTopic, NT_DOUBLE, "double", options, 2);

            // entries support all the same methods as subscribers:
            double val = NT_GetDouble(dblEntry, 0.0);

            struct NT_TimestampedDouble tsVal;
            NT_GetAtomic(dblEntry, 0.0, &tsVal);
            NT_DisposeTimestamped(&tsVal);

            size_t tsUpdatesLen;
            struct NT_TimestampedDouble* tsUpdates = NT_ReadQueueDouble(dblEntry, &tsUpdatesLen);
            NT_FreeQueueDouble(tsUpdates, tsUpdatesLen);

            size_t valUpdatesLen;
            double* valUpdates = NT_ReadQueueValuesDouble(dblEntry, &valUpdatesLen);
            NT_FreeDoubleArray(valUpdates, valUpdatesLen);

            // entries also support all the same methods as publishers; the first time
            // one of these is called, an internal publisher is automatically created
            NT_SetDefaultDouble(dblPub, 0.0);
            NT_SetDouble(dblPub, 1.0);
            NT_SetDouble(dblPub, 2.0, 0);  // 0 = use current time
            int64_t time = NT_Now();
            NT_SetDouble(dblPub, 3.0, time);

            // you can stop publishing while keeping the subscriber alive
            // it's not necessary to call this before NT_ReleaseEntry()
            NT_Unpublish(dblEntry);

            // stop subscribing
            NT_ReleaseEntry(dblEntry);


Using GenericEntry, GenericPublisher, and GenericSubscriber
-----------------------------------------------------------

For the most robust code, using the type-specific Publisher, Subscriber, and Entry classes is recommended, but in some cases it may be easier to write code that uses type-specific get and set function calls instead of having the NetworkTables type be exposed via the class (object) type. The ``GenericPublisher`` (`Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/GenericPublisher.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_generic_publisher.html>`__), ``GenericSubscriber`` (`Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/GenericSubscriber.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_generic_subscriber.html>`__), and ``GenericEntry`` (`Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/GenericEntry.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_generic_entry.html>`__) classes enable this approach.

.. tabs::

    .. group-tab:: Java

        .. code-block:: java

            public class Example {
              // the entry is an instance variable so its lifetime matches that of the class
              final GenericPublisher pub;
              final GenericSubscriber sub;
              final GenericEntry entry;

              public Example(Topic topic) {
                // start subscribing; the return value must be retained.
                // when publishing, a type string must be provided
                pub = topic.genericPublish("double");

                // subscribing can optionally include a type string
                // unlike type-specific subscribers, no default value is provided
                sub = topic.genericSubscribe();
                sub = topic.genericSubscribe("double");

                // when getting an entry, the type string is also optional; if not provided
                // the publisher data type will be determined by the first publisher-creating call
                entry = topic.getGenericEntry();
                entry = topic.getGenericEntry("double");

                // publish and subscribe options may be specified using PubSubOption
                pub = topic.genericPublish("double",
                    PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));
                sub =
                    topic.genericSubscribe(PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));
                entry =
                    topic.getGenericEntry(PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));

                // genericPublishEx provides the option of setting initial properties.
                pub = topic.genericPublishEx("double", "{\"retained\": true}",
                    PubSubOption.keepDuplicates(true), PubSubOption.pollStorage(10));
              }

              public void periodic() {
                // generic subscribers and entries have typed get operations; a default must be provided
                double val = sub.getDouble(-1.0);
                double val = entry.getDouble(-1.0);

                // they also support an untyped get (also meets Supplier<NetworkTableValue> interface)
                NetworkTableValue val = sub.get();
                NetworkTableValue val = entry.get();

                // they also support readQueue
                NetworkTableValue[] updates = sub.readQueue();
                NetworkTableValue[] updates = entry.readQueue();

                // publishers and entries have typed set operations; these return false if the
                // topic already exists with a mismatched type
                boolean success = pub.setDefaultDouble(1.0);
                boolean success = pub.setBoolean(true);

                // they also implement a generic set and Consumer<NetworkTableValue> interface
                boolean success = entry.set(NetworkTableValue.makeDouble(...));
                boolean success = entry.accept(NetworkTableValue.makeDouble(...));
              }

              public void unpublish() {
                // you can stop publishing an entry while keeping the subscriber alive
                entry.unpublish();
              }

              // often not required in robot code, unless this class doesn't exist for
              // the lifetime of the entire robot program, in which case close() needs to be
              // called to stop subscribing/publishing
              public void close() {
                pub.close();
                sub.close();
                entry.close();
              }
            }

    .. group-tab:: C++

        .. code-block:: cpp

            class Example {
              // the entry is an instance variable so its lifetime matches that of the class
              // subscribing/publishing is automatically stopped when dblEntry is destroyed by
              // the class destructor
              nt::GenericPublisher pub;
              nt::GenericSubscriber sub;
              nt::GenericEntry entry;

             public:
              Example(nt::Topic topic) {
                // start subscribing; the return value must be retained.
                // when publishing, a type string must be provided
                pub = topic.GenericPublish("double");

                // subscribing can optionally include a type string
                // unlike type-specific subscribers, no default value is provided
                sub = topic.GenericSubscribe();
                sub = topic.GenericSubscribe("double");

                // when getting an entry, the type string is also optional; if not provided
                // the publisher data type will be determined by the first publisher-creating call
                entry = topic.GetEntry();
                entry = topic.GetEntry("double");

                // publish and subscribe options may be specified using PubSubOption
                pub = topic.GenericPublish("double",
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});
                sub = topic.GenericSubscribe(
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});
                entry = topic.GetGenericEntry(
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});

                // genericPublishEx provides the option of setting initial properties.
                pub = topic.genericPublishEx("double", {{"myprop", 5}},
                    {{nt::PubSubOption::KeepDuplicates(true), nt::PubSubOption::PollStorage(10)}});
              }

              void Periodic() {
                // generic subscribers and entries have typed get operations; a default must be provided
                double val = sub.GetDouble(-1.0);
                double val = entry.GetDouble(-1.0);

                // they also support an untyped get
                nt::NetworkTableValue val = sub.Get();
                nt::NetworkTableValue val = entry.Get();

                // they also support readQueue
                std::vector<nt::NetworkTableValue> updates = sub.ReadQueue();
                std::vector<nt::NetworkTableValue> updates = entry.ReadQueue();

                // publishers and entries have typed set operations; these return false if the
                // topic already exists with a mismatched type
                bool success = pub.SetDefaultDouble(1.0);
                bool success = pub.SetBoolean(true);

                // they also implement a generic set and Consumer<NetworkTableValue> interface
                bool success = entry.Set(nt::NetworkTableValue::MakeDouble(...));
              }

              void Unpublish() {
                // you can stop publishing an entry while keeping the subscriber alive
                entry.Unpublish();
              }
            };


Subscribing to Multiple Topics
------------------------------

While in most cases it's only necessary to subscribe to individual topics, it is sometimes useful (e.g. in dashboard applications) to subscribe and get value updates for changes to multiple topics. Listeners (see :ref:`docs/software/networktables/listening-for-change:listening for changes`) can be used directly, but creating a ``MultiSubscriber`` (`Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/MultiSubscriber.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_multi_subscriber.html>`__) allows specifying subscription options and reusing the same subscriber for multiple listeners.

.. tabs::

    .. group-tab:: Java

        .. code-block:: java

            public class Example {
              // the subscriber is an instance variable so its lifetime matches that of the class
              final MultiSubscriber multiSub;
              final NetworkTableListenerPoller poller;

              public Example(NetworkTableInstance inst) {
                // start subscribing; the return value must be retained.
                // provide an array of topic name prefixes
                multiSub = new MultiSubscriber(inst, new String[] {"/table1/", "/table2/"});

                // subscribe options may be specified using PubSubOption
                multiSub = new MultiSubscriber(inst, new String[] {"/table1/", "/table2/"},
                    PubSubOption.keepDuplicates(true));

                // to get value updates from a MultiSubscriber, it's necessary to create a listener
                // (see the listener documentation for more details)
                poller = new NetworkTableListenerPoller(inst);
                poller.addListener(multiSub, EnumSet.of(NetworkTableEvent.Kind.kValueAll));
              }

              public void periodic() {
                // read value events
                NetworkTableEvent[] events = poller.readQueue();

                for (NetworkTableEvent event : events) {
                  NetworkTableValue value = event.valueData.value;
                }
              }

              // often not required in robot code, unless this class doesn't exist for
              // the lifetime of the entire robot program, in which case close() needs to be
              // called to stop subscribing
              public void close() {
                // close listener
                poller.close();
                // stop subscribing
                multiSub.close();
              }
            }

    .. group-tab:: C++

        .. code-block:: cpp

            class Example {
              // the subscriber is an instance variable so its lifetime matches that of the class
              // subscribing is automatically stopped when multiSub is destroyed by the class destructor
              nt::MultiSubscriber multiSub;
              nt::NetworkTableListenerPoller poller;

             public:
              explicit Example(nt::NetworkTableInstance inst) {
                // start subscribing; the return value must be retained.
                // provide an array of topic name prefixes
                multiSub = nt::MultiSubscriber{inst, {{"/table1/", "/table2/"}}};

                // subscribe options may be specified using PubSubOption
                multiSub = nt::MultiSubscriber{inst, {{"/table1/", "/table2/"}},
                    {{nt::PubSubOption::KeepDuplicates(true)}}};

                // to get value updates from a MultiSubscriber, it's necessary to create a listener
                // (see the listener documentation for more details)
                poller = nt::NetworkTableListenerPoller{inst};
                poller.AddListener(multiSub, nt::EventFlags::kValueAll);
              }

              void Periodic() {
                // read value events
                std::vector<nt::Event> events = poller.ReadQueue();

                for (auto&& event : events) {
                  nt::NetworkTableValue value = event.GetValueEventData()->value;
                }
              }
            };

    .. group-tab:: C++ (handle-based)

        .. code-block:: cpp

            class Example {
              // the subscriber is an instance variable, but since it's a handle, it's
              // not automatically released, so we need a destructor
              NT_MultiSubscriber multiSub;
              NT_ListenerPoller poller;

             public:
              explicit Example(NT_Inst inst) {
                // start subscribing; the return value must be retained.
                // provide an array of topic name prefixes
                multiSub = nt::SubscribeMultiple(inst, {{"/table1/", "/table2/"}});

                // subscribe options may be specified using PubSubOption
                multiSub = nt::SubscribeMultiple(inst, {{"/table1/", "/table2/"}},
                    {{nt::PubSubOption::KeepDuplicates(true)}});

                // to get value updates from a MultiSubscriber, it's necessary to create a listener
                // (see the listener documentation for more details)
                poller = nt::CreateListenerPoller(inst);
                nt::AddPolledListener(poller, multiSub, nt::EventFlags::kValueAll);
              }

              void Periodic() {
                // read value events
                std::vector<nt::Event> events = nt::ReadListenerQueue(poller);

                for (auto&& event : events) {
                  nt::NetworkTableValue value = event.GetValueEventData()->value;
                }
              }

              ~Example() {
                // close listener
                nt::DestroyListenerPoller(poller);
                // stop subscribing
                nt::UnsubscribeMultiple(multiSub);
              }

    .. group-tab:: C

        .. code-block:: c

            // This code assumes that a NT_Inst inst variable already exists

            // start subscribing
            // provide an array of topic name prefixes
            struct NT_String prefixes[2];
            prefixes[0].str = "/table1/";
            prefixes[0].len = 8;
            prefixes[1].str = "/table2/";
            prefixes[1].len = 8;
            NT_MultiSubscriber multiSub = NT_SubscribeMultiple(inst, prefixes, 2, NULL, 0);

            // subscribe options may be specified using NT_PubSubOption
            struct NT_PubSubOption option;
            option.type = NT_PUBSUB_KEEPDUPLICATES;
            option.value = 1;  // true
            NT_MultiSubscriber multiSub = NT_SubscribeMultiple(inst, prefixes, 2, &option, 1);

            // to get value updates from a MultiSubscriber, it's necessary to create a listener
            // (see the listener documentation for more details)
            NT_ListenerPoller poller = NT_CreateListenerPoller(inst);
            NT_AddPolledListener(poller, multiSub, NT_EVENT_VALUE_ALL);

            // read value events
            size_t eventsLen;
            struct NT_Event* events = NT_ReadListenerQueue(poller, &eventsLen);

            for (size_t i = 0; i < eventsLen; i++) {
              NT_Value* value = &events[i].data.valueData.value;
            }

            NT_DisposeEventArray(events, eventsLen);

            // close listener
            NT_DestroyListenerPoller(poller);
            // stop subscribing
            NT_UnsubscribeMultiple(multiSub);


NetworkTableEntry
-----------------

``NetworkTableEntry`` (`Java <https://github.wpilib.org/allwpilib/docs/beta/java/edu/wpi/first/networktables/NetworkTableEntry.html>`__, `C++ <https://github.wpilib.org/allwpilib/docs/beta/cpp/classnt_1_1_network_table_entry.html>`__) is a class that exists for backwards compatibility. New code should prefer using type-specific Publisher and Subscriber classes, or GenericEntry if non-type-specific access is needed.

It is similar to ``GenericEntry`` in that it supports both publishing and subscribing in a single object. However, unlike ``GenericEntry``, ``NetworkTableEntry`` is not released (e.g. unsubscribes/unpublishes) if ``close()`` is called (in Java) or the object is destroyed (in C++); instead, it operates similar to ``Topic``, in that only a single ``NetworkTableEntry`` exists for each topic and it lasts for the lifetime of the instance.
