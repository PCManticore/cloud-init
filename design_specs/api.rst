======================================
Ideas for a new cloudinit architecture
======================================

API convention
--------------

Before dwelving into the details of the proposed API, some conventions should be
established, so that the API could be pythonic, easy to comprehend and extend.
We have the following examples of how an object should look,
depending on its state and behaviour:

 - Use ``.attribute`` if it the attribute is not changeable
   throughout the life of the object.
   For instance, the name of a name of a device.

 - Use ``.method()`` for obtaining a variant attribute, which can be
   different throughout the execution time and not modifiable through our
   object. This is the case for ``device.size()``, we can't set a new size
   and it can vary throughout the life of the device.

 - For attributes which are modifiable by us and which aren't changing
   throughout the life of the object, we could use
   two approaches, a property-based approach or a method based approach.
   The first one is more Pythonic, but can
   hide from the user of that object the fact that the property does
   more than it says, the second one is more in-your-face,
   but feels too much like Java's setters and getters and so on:

       >>> device.mtu
       1500
       >>> device.mtu = 1400 # actually changing the value, not the cached value.
       1400

       vs

       >>> device.mtu()
       1500
       >>> device.set_mtu(1400)
       >>>

 - similar to the previous point, we could have
   ``is_something`` or ``is_something()``.
   We must choose one of this variants and use it consistently
   accross the project.


Distro hierarchy
================

Both frameworks have a concept of Distro, each different in its way:

    - cloudinit has a ``distros`` location. There is a ``Distro`` base class,
      with abstract methods implemented by particular distros.

        Problems:

        * not DRY: many implementations have duplicate code with the base class
        * not properly encapsulated: distro-specific code executed outside the ``distros`` namespace.
        * lots of utilities, with low coherence between them.

    - cloudbaseinit has a ``osutils`` location. There is a ``BaseOSUtils``
      base class, with a WindowsUtils implementation.

       Problems:

           * it's a pure utilities class, leading to low coherence between functions.
           * it is not a namespace of OS specific functionality.
             For this, there is also ``utils.windows``.

As seen, both projects lack a namespaced location for all the OS related code.
The following arch proposal tries to solve this issue, by having one
namespace for both general utilies for the OS,
as well as OS specific code, which doesn't have a counterpart on other OS-es.
Also, it tries to avoid leaky abstractions and it's more OOP centered than the
current alternatives. The distros location is proposed, with the following
structure and attributes:

    - The base class for a distro is found in ``distros.base``.

    - There are sub-namespaces for specific interaction with the OS,
      such as network, users.

    - More namespaces can be added, if we identify a group of interactions that can
      be categorized in a namespace.

    - There is a ``general`` namespace, which contains general utilities that can't be moved
      in another namespace.

    - Each subnamespace has its own abstract base class, which must be implemented
      by different distros. Code reuse between distros is recommended.

    - Each subnamespace exposes a way to obtain an instance of that namespace for
      the underlying distro. This can be exposed in namespace.factory.
      Thus, the following are equivalent:

        >>> from cloudinit import distro
        >>> network = distro.get_distro().network

        >>> from cloudinit.distro.network import factory
        >>> network = factory.get_network()

    - Each subnamespace can expose additional behaviour that might not exist in
      the base class, if that behaviour does not make sense or if there is no
      equivalent on other platforms. But don't expose leaky abstraction, this
      specific code must be written in an abstract way, so that possible alternatives
      can be found for other distros in the mean time. This means that no ctypes
      interaction with the Windows should be exposed,
      but an object with a guaranteed interface.


      cloudinit/distros/__init__.py
                        base.py
                        factory.py

                        network/
                          factory.py
                          base.py
                          windows.py
                          freebsd.py
                          debian.py
                          ....
                          __init__.py

                        users
                          factory.py
                          base.py
                          freebsd.py
                          debian.py
                          ....
                          __init__.py

                        filesystem
                          factory.py
                          base.py
                          freebsd.py
                          ...
                          __init__.py

                        packaging
                          factory.py
                          base.py
                          freebsd.py
                          debian.py
                          ....
                          __init__.py

                        general
                          base.py
                          windows.py
                          debian.py
                          ....
                          __init__.py


The base class for the Distro specific implementation must provide an accessor member
for each namespace, so that it will be sufficient to obtain the distro in order to have each namespace.

>>> from cloudinit.distros import factory
>>> distro = factory.get_distro()
>>> distro.network # the actual object, not the subpackage
<WindowsNetwork:/distro/network/windows>
>>> distro.users
<WindowsUsers:/distro/users/windows>
>>> distro.general
<WindowsGeneral:/distro/general/windows>


In the following, I'll try to emphasize some possible APIs for each namespace.


Network subnamespace
----------------------

    The abstract class can look like this:

        class NetworkBase(ABCMeta):

           def routes(self):
             """Get the available routes, this can be the output of
             `netstat` on Posix and ``GetIpForwardTable`` on Windows.
             Each route should be an object encapsulating the inner workings
             of each variant.

             So :meth:`routes` returns ``RouteContainer([Route(...), Route(...), Route(...))``
             See the description for :class:`RouteContainer` for more details,
             as well as the description of :class:`Route` for the API of the route object.

             Using ``route in network.routes()`` and ``network.routes().add(route)
             removes the need for ``cloudbaseinit.osutils.check_static_route_exists``
             and ``cloudbaseinit.osutils.add_static_route``.
             """

          def default_gateway(self):
             """Get the default gateway.

             Can be implemented in the terms of :meth:`routes`.
             """

         def interfaces(self):
             """Get the network interfaces

             This can be implemented in the same vein as :meth:`routes`, e.g.
             ``InterfaceContainer([Interface(...), Interface(...), Interface(...)])``
             """

         def firewall_rules(self):
             """Get a wrapper over the existing firewall rules.

             Since this seems to be only used in Windows, it can be provided only in the Windows utils.
             The same behaviour as for :meth:`routes` can be used, that is:

                 >>> rules = distro.network.firewall_rules()
                 >>> rule = distro.network.FirewallRule(name=..., port=..., protocol=...)
                 >>> rules.add(rule)
                 >>> rules.delete(rule)
                 >>> rule in rules
                 >>> for rule in rules: print(rules)
                 >>> del rules[i]
                 >>> rule = rules[0]
                 >>> rule.name, rule.port, rule.protocol, rule.allow

             This gets rid of ``cloudbaseinit.osutils.firewall_add_rule`` and
             ``cloudbaseinit.osutils.firewall_remove_rule``.
             """

         def set_static_network_config(self, adapter_name, address, netmask,
                                       broadcast, gateway, dnsnameservers):
             """Configure a new static network.

             The :meth:``cloudinit.distros.Distro.apply_network`` should be
             removed in the favour of this method,
             which will be called by each network plugin.
             The method can be a template method, providing
             hooks for setting static DNS servers, setting static gateways or
             setting static IP addresses, which will be implemented by specific
             implementations of Distros.
             """

        def hosts(self):
             """Get the content of /etc/hosts file in a more OO approach.


             >>> hosts = distro.network.hosts()
             >>> list(hosts) # support iteration and index access
             >>> hosts.add(ipaddress, hostname, alias)
             >>> hosts.remove(ipaddress, hostname, alias)

             This gets rid of ``cloudinit.distros.Distro.update_etc_hosts``
             and can provide support for adding a new hostname for Windows, as well.
             """

        class Route(object):
             """
             Encapsulate behaviour and state of a route.
             Something similar to Posix can be adopted, with the following API:

                  route.destination
                  route.gateway
                  route.flags
                  route.refs
                  route.use
                  route.netif -> instance of :class:`Interface` object
                  route.expire
                  route.static -> 'S' in self.flags
                  route.usable -> 'U' in self.flag
             """

          @classmethod
          def from_route_item(self, item):
              """
              Build a Route from a routing entry, either from
              the output of `netstat` or what will be used on Posix or
              from `GetIpForwardTable`.
              """

        class RouteContainer(object):
            """A wrapper over the result from :meth:`NetworkBase.routes()`,
            which provides some OO interaction with the underlying routes.


            >>> routes = network.routes() # a RouteContainer
            >>> route_object in routes
            True
            >>> '192.168.70.14' in routes
            False
            >>> route = Route.from_route_entry(
                       "0.0.0.0         192.168.60.2    "
                       "0.0.0.0         UG        0 0          "
                       "0 eth0")
            >>> routes.add(route)
            >>> routes.delete(route)
            """

            def __iter__(self):
               """Support iteration."""

            def __contains__(self, item):
                """Support containment."""

            def __getitem__(self, item):
                """Support element access"""

            def __delitem__(self, item):
                """Delete a route, equivalent to :meth:`delete_route`."""

            def __add__(self, item):
                """Add route, equivalent to :meth:`add_route``."""

            def add(self, route):
                """Add a new route."""

            def delete(self, destination, mask, metric, ...):
                """Delete a route."""

      class InterfaceContainer(object):
            """Container for interfaces, with similar API as for RouteContainer."""

            def __iter__(self):
               """Support iteration."""

            def __contains__(self, item):
                """Support containment."""

            def __getitem__(self, item):
                """Support element access"""

      class Interface(object):
            """Encapsulation for the state and behaviour of an interface.

            This method gets rid of ``cloudbaseinit.osutils.get_network_adapters``
            and with the following behaviour
            it gets rid of ``cloudinit.distros._bring_up_interface``:

                >>> interfaces = distro.network.interfaces()
                >>> interface = interfaces[0]
                >>> interface.up()
                >>> interface.down()
                >>> interface.is_up()
                # Change mtu for this interface
                >>> interface.mtu = 1400
                # Get interface mtu
                >>> interface.mtu
                1400

            If we have only the name of an interface, we should be able to
            obtain a :class:`Interface` instance from it.

            >>> interface = distro.network.Interface.from_name('eth0')
            >>> nterface = distro.network.Interface.from_mac( u'00:50:56:C0:00:01')

            Each Distro specific implementation of :class:`Interface` should
            be exported in the `network` namespace as the `Interface` attribute,
            so that the underlying OS is completely hidden from an API point-of-view.
            """

            # alternative constructors

            @classmethod
            def from_name(cls, name):
                # return a new Interface

            @classmethod
            def from_mac(self, mac):
                # return a new Interface

            # Actual methods for behaviour

            def up(self):
                """Activate the interface."""

            def down(self):
                """Deactivate the interface."""

            def is_up(self):
                """Check if the interface is activated."""

            # Other getters and setter for what can be changed for an
            # interface, such as the mtu.

            @property
            def mtu(self):
                pass

            @mtu.setter
            def mtu(self, value):
                pass

            # Other read only attributes, such as ``.name``, ``.mac`` etc.

   .. note::

       TODO: finish this section with APis for set_hostname, _read_hostname, update_hostname


Users subnamespace
------------------

The base class for this namespace can look like this


     class UserBase(ABCMeta):

         def groups(self):
             """Get all the user groups from the instance.

             Similar with network.routes() et al, that is

             >>> groups = distro.users.groups()
             GroupContainer(Group(...), Group(....), ...)
             # create a new group
             >>> group = distro.users.Group.create(name)
             # Add new members to a group
             >>> group.add(member)
             # Add a new group
             >>> groups.add(group)
             # Remove a group
             >>> groups.remove(group)
             # Iterate groups
             >>> list(groups)

             This gets rid of ``cloudinit.distros.Distro.create_group``,
             which creates a group and adds member to it as well and it get rids of
             ``cloudbaseinit.osutils.add_user_to_local``.
             """

       def users(self):
             """Get all the users from the instance.

             Using the same idion as for :meth:`routes` and :meth:`groups`.

             >>> users = distro.users.users()
             # containment (cloudbaseinit.osutils.user_exists)
             >>> user in users
             # Iteration
             >>> for i in user: print(user)
             # Add a new user
             >>> user = users.create(username, password, password_expires=False)
             """

     class User:
         """ Abstracts away user interaction.

         # get the home dir of an user
         >>> user.home()
         # Get the password (?)
         >>> user.password
         # Set the password
         >>> user.password = ....
         # Get an instance of an User from a name
         >>> user = distros.users.User.from_name('user')
         # Disable login password
         >>> user.disable_login_password()
         # Get ssh keys
         >>> keys = user.ssh_keys()

         Posix specific implementations might provide some method
         to operate with '/etc/sudoers' file.
         """

.. note::

   TODO: what is cloudinit.distros.get_default_user?

Packaging namespace
-------------------

This object is a thin layer over Distro specific packaging utilities,
used in cloudinit through ``distro.Distro.package_command``.
Instead of passing strings with arguments, as it currently does,
we could have a more OO approach:

      >>> distro.packaging.install(...)

      # cloudinit provides a ``package_command`` and an ``update_package_sources`` method,
      # which is:
      #          self._runner.run("update-sources", self.package_command,
      #                   ["update"], freq=PER_INSTANCE)
      #  distro.packaging.update() can be a noop operation if it was already called
      >>> distro.packaging.update(...)

 On Windows side, this can be implemented with OneGet.


Filesystem namespace
--------------------

Layer over filesystem interaction specific for each OS.
Most of the uses I encountered are related to the concept of devices and partitions.


class FilesystemBase(ABC):

     def devices(self):
         """Get a list of devices for this instance.

         As usual, this is abstracted through a container
         DevicesContainer([Device(...), Device(...), Device(...)])

         Where the container has the following API:

         >>> devices = distro.filesystem.devices()
         >>> devices.name, devices.type, devices.label
         >>> devices.size()
         # TODO: geometry on Windows? Define the concept better.
         >>> devices.layout()
         >>> device in devices
         >>> for device in devices: print(device)
         >>> devices.partitions()
         [DevicePartition('sda1'), DevicePartition('sda2'), ...]
         # TODO: FreeBSD has slices, which translates to partitions on
         # Windows and partitions of slices, how
         # does this translate with the current arch?


         Each DevicePartition shares a couple of methods / attributes with the Device,
         such as ``name``, ``type``, ``label``, ``size``. They have extra methods:

           >>> partition.resize()
           >>> partition.recover()
           >>> partition.mount()
           >>> with partition.mount(): # This can be noop on Windows.
                     ....

         Obtaining either a device or a partition from a string, should be done
         in the following way:

           >>> device = Device.from_name('sda')
           >>> partition = DevicePartition.from_name('sda', 1)
           >>> partition = DevicePartition.from_name('sda1')
         """

General namespace
-----------------

Here we could have other general OS utilities: terminate, apply_locale,
set_timezone, execute_process etc.


Plugins hierarchy
=================

.. note::

   This text talks about plugins in the cloudbaseinit's terminology.
   In cloudinit they are called ``configs``.


* One important thing for the new project is the backward
  compatibility with the user's format. Currently, cloud-init uses a cloud-config
  format to control the execution of the plugins, as well as obtaining additional
  data from the user, while cloudbaseinit uses a .conf file, where passwords and
  others can be provided. Since we should support either of them in the new project
  in order to gain user base traction, what is needed is a common layer over both
  formats, which will be called whenever a plugin requires information for either side.

* The plugins for both projects operate differently, one on values provided by
  a .conf file, another with values provided in a cloud-config file.
  This should be normalized in a way that makes a plugin work with values either
  from the cloud-config plugin or from the conf or default values.

* cloudbaseinit also has the concept of a default plugin, it has a list of
   hardcoded plugins, each plugin having default values for its options.

* cloudinit, AFAICT, can decide if a plugin runs or not according to some criteria
  (it has a key in the cloudconfig or not). At the same time,
  some plugins are default, they don't look for switch options, they just do their job.


Taking  these things into account, we can create a model of
interaction between the plugins that uses both data formats:

    1) discover all the plugin classes. See the discovery section for more
       thoughts about htis.

    2) the plugin base class should have a method which says that a plugin can run at that time or not.
       ``can_run()`` or other alternatives. ``can_run`` should be implemented by plugins.
       Default plugins can return True, other plugins can inspect the configurations: cloud-config,
       cloudbaseinit's conf or the default values, if any.
       cloudbaseinit's ``get_os_requirements`` can be merged here.

    3) obtain the list of plugins that can run at that time, using the provided information.

    4) cloudbaseinit's can have its plugins customized through the cloudbase conf,
       meaning that other plugins can be executed rather than the default ones.
       The plugin manager needs to take this into account: if the plugin
       order is rewritten, just use those plugins instead of the full loaded list.
       Also, it should check, even these user-defined plugins, that they can run
       using the current context.

    5) sort the plugins according to their dependencies.
       For the dependencies part, we could use something like TaskFlow: https://wiki.openstack.org/wiki/TaskFlow
       to mark a plugin that it needs some other plugins to run before.

    6) run the plugins.


When a plugin is running, it should look through the common layer for its options,
either in cloud-config, or in cloudbaseinit's conf or default values, if any.
``can_run`` could call another method, ``.data()``, which returns plugin required data,
in order to check if it can run or not. When a plugin is running, it could call the same method
to obtain the required information to operate on.

These changes have the following implications:

    - UserData plugin needs to be splitted in two, where the execution part is a new plugin
      by itself and the decoding part is used to retrieve each plugin's data.

    - There will be no concept of cloud-config plugins and normal plugins, as it is now in cloudbaseinit.
      All plugins will be the same, the only differing element
      will be the data they will operate on. The retrieval of data will be intrisic to each plugin,
      not from outside.


Plugin discovery
----------------

Currently, cloudbaseinit uses static plugins, knowing each plugin's location beforehand. On
the other side, cloudinit is more dynamic, looking up in the ``config`` location for all
the plugins which have exported a ``handle`` method. This approach is similar to
known protocols, such as ``load_tests`` protocol for the unittest.

Certainly, there are multiple ways to do plugins in the Python world, but we should
stick with what it works and what's easy to extend.

Some proposals:

  * use stevedore: https://pypi.python.org/pypi/stevedore
    One problem i that the plugins are known beforehand as well,
    they are included in setup.py's metadata.

  * use a protocol similar with what cloudinit has, but with a couple
    of enhancements:

       - have a plugin manager, an object which knows all the plugins
         and which can sort them according to their priorities,
         so on and so forth.

       - provide a ``register_plugin`` protocol, which will be a module
         level function, which receives one argument, a plugin manager.

       - each plugin module defines a ``register_plugin`` function,
         where they'll call register themselves with the manager,
         as in ``manager.register_plugin(PluginInstance())``.

       - the manager loads all the modules of a known location,
         looking for a ``register_plugin`` function. If it founds
         one, calls the function with itself.

         .. note::

             the loading of modules needs more debate.


       This can look like (pseudocode):

           ::

           plugin.py

               def register_plugin(manager):
                   manager.register_plugin(NetworkConfig())
                   manager.register_plugin(OtherPluginKnownByThisModule())

           manager.py

               for module in modules():
                   if hasattr(module, 'register_plugin'):
                       module.register_plugin(self)


Metadata provider
=============
TODO - parallel discovery
     - capabilities advertisment
     - choosing a metadata according to its capabilities
