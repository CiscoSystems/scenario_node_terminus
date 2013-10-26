# scenario based override terminus

####Table of Contents

1. [Overview - What is the scenario node terminus?](#overview)
2. [Hiera Primer - Understanding the model](#hiera-primer)
    * [Hierarchies and Global Variables](#hierarchies-and-global-variables)
    * [Variable interpolation](#variable-interpolation)
3. [Module Description - What does it do?](#module-description)
    * [Scenario Selection](#scenario-selection)
    * [Global Parameters](#global-parameters)
    * [Scenarios](#scenarios)
    * [Class Groups](#class-groups)
    * [Role Mappings](#role-mappings)
    * [Data Mappings](#data-mappings)
    * [Hiera Data](#hiera-data)
4. [Installation - The basics of getting started](#installation)
5. [Command Line Debugging Tools](#command-line-debugging-tools)
6. [Getting Required User Configuration](#getting-required-user-configuration)
7. [Implementation - An under-the-hood peek at what the module is doing](#implementation)

## Overview

This module allow users to specify a data layer that lives on top of Puppet
and allows an additional level of flexibility to deployments.

Is was designed to compose a set of core Puppet modules into multiple deployment
permutations (or reference architectures) called scenarios while requiring as
little duplicate code as possible.

Each scenario can also be configured in many ways.

Although it was originally designed with the Openstack modules in mind,
it has a great potential to be leveraged outside of that project.

## Hiera Primer

Understanding hiera as well the Puppet's data binding system available in 3.x
are crucial for understanding this model.

This model borrows heavily from both hiera concepts as well as APIs.

Hiera allows users to supply data sets to supply data to your Puppet classes
in a way that can be configured based on a set of provided global variables.

### Hierarchies and Global Variables

Hiera uses a set of know global variables (for typical hiera use cases,
these globals are synonymous with facts) to determine the value that should
be provided for each piece of data that it needs to lookup.

The way in which those variables effect the final value of an individual piece
of data is driven by the hierarchy specified in hiera's configuration file.

Given the following hierarchy from hiera.yaml:

     ---
    :hierarchy:
      - "hostname/%{hostname}"
      - user
      - "osfamily/%{osfamily}"
      - "scenario/%{scenario}"
      - common

Hiera will looks in a set of files for data values that match a specific key. The data
files that it searches are determined by using a set of global data specified
to check data in the associated file.

Assume the following:

  * data will be searched from : /etc/puppet/data/hiera\_data
  * the hierarchy key in our hiera.yaml file is the same as above
  * The following global variables are available to hiera

    osfamily: redhat
    scenario: all_in_one
    hostname: my_host.domain.name

When hiera searches for that key, it will do the following:

1. use the global variables to build out the list of files to search for our key.

Using the provided hierarchy and set of global variables, this would be

+ /etc/puppet/data/hiera\_data/hostname/my\_host.domain.name.yaml
+ /etc/puppet/data/hiera\_data/user.yaml
+ /etc/puppet/data/hiera\_data/osfamily/redhat.yaml
+ /etc/puppet/data/hiera\_data/scenario/all\_in\_one.yaml
+ /etc/puppet/data/hiera\_data/common.yaml

2. Search those files in the same order as they are provided in hiera.yaml.
This order also implies their lookup precedence. The first value that hiera
finds is the value that is provided for that variable.

for example, if we were lookup up the key *variable1*

+ if my\_host.domain.name.yaml has the key *variable1*, this values will
be used.

    variable1: value_from_hostname

### Variable interpolation

Sometimes the value returned form a hiera value lookup is not a static string,
or array, but a variable that should be interpolated from the current set
of global variables.

Using the same globals set in the previous section, if the value for *variable2*
is derived as *"%{scenario}"*

The value for that key would be *all_in_one*

### How this applies to the data model

The data model is based on hiera, but has several sets of hiera data that are processed.

This data is processed via the hierarchy in hiera.yaml to determine:

+ what global data should be used to process data values
+ what roles exist as a part of the current deployment model
+ what classes should be applied as a part of a role
+ what hiera data should be set for a class

In fact, the data model is composed of the following parts that are processed in this order:

* global hiera data
* scenario
* data mappings
* hiera data

The following image shows both the processing order of each data type and the
global variables used to determine how it's hierarchy is processed.

![](https://raw.github.com/bodepd/scenario_node_terminus/hiera_docs/docs/images/data_model_scopes.png)

### Further reading

Here is the upstream documentation for [hiera](http://docs.puppetlabs.com/hiera/1/) and
[data bindings](http://docs.puppetlabs.com/hiera/1/puppet.html#automatic-parameter-lookup)


## Module Description

This model providers a data layer that sits above your Puppet manifests.

This data layer can be used instead of composition manifests and
roles/profiles.

The data layer is processed into a list of classes as well as data\_bindings
that should be used to configure a node.

### Scenario Selection

Config.yaml is used to store the deployment scenario currently in use.

It contains a single configuration:

    scenario: scenario_name

+  *scenario* is used to select the specific references architecture
   that you wish to deploy. Its value is used to select the roles for
   that specific deployment model from the file: scenarios/<scenario>.yaml.
   If you are using this project for CD, scenario is also used to select
   the set of nodes that will be provisioned for your deployment.
   Scenario is also passed to Puppet as a global variable and used to drive
   both interpolation as well as category selection in hiera.

### Global Parameters

This directory is used to specify the global variables that can be used
to effect the hierarchical overrides that will be used to determine both
the classes contained in a scenario roles as well as the hiera overrides
for both data mappings and the regular yaml hierarchy.

The selection of the global\_hiera\_params is driven by hiera using the following
hierarchy:

  - global\_hiera\_params/user.yaml - users can provide their own global
  overrides in this file.
  - global\_hiera\_params/scenario/%{scenario}.yaml - Default values specific to a
  scenario are loaded from here (they override values from common.yaml)
  - global\_hiera\_params/common.yaml - Default values for globals are located here.

These variables are used by hiera to determine both what classes are included as a
part of the role lookup, and are also used to drive the hierarchical lookups of
data both by effecting the configuration files that are consulted (like the scenario
specific config file from above)).

### Scenarios

Scenarios are used to describe the roles that should be deployed as a part of
a reference architecture as well as the classes that are used to configure those
roles.

The following config snippet shows how a scenario specifies multiple roles and
assigns them classes:

    scenarios:
      roles:
        role1:
          classes:
            - class_one
        role2:
          classes
            - class_two

Scenarios are constructed by compiling hierarchies in your
*scenarios* data directory.

Each of the roles for a specific scenario is specified in its scenario
files:


  - scenarios/user.yaml - users can provide their own global
  overrides in this file.
  - scenarios/%{scenario\_name}.yaml - Default values for a specific
  scenario are loaded from here.
  - scenarios/common.yaml - Default roles that can be applied to all scenarios
  can be found here.

You can also insert custom hierarchies based on hiera\_global\_params to customize
the way that roles can be overridden.

### Class Groups

Class groups are a set of classes that can be referenced by a
single identifier. Class groups are used to store combinations
of classes for reuse.

For example, if several classes are required to build out a role
called nova compute, then it's class group might look like this:

  data/class\_groups/nova\_compute.yaml

    classes:
      - nova
      - nova::compute
      - "nova::compute::%{compute_type}"
      - "nova::network::%{network_service}"
      - "nova::compute::%{network_service}"
      - "%{network_service}"
      - "%{network_service}::agents::%{network_plugin}"
    class_groups:
      - base

Two things to note here:

1. It contains a list of classes that comprise nova compute
2. Some of the classes use the hiera syntax for variable interpolation to
   set the names of classes used to the values provided from the
   hiera\_global\_params.
3. class groups can themselves can contain class groups.

### Role Mappings

role\_mappings are used to map a Puppet certificate name to a specific roles
from your selected scenario.

The following example shows how to map a certname of controller-server to
a role of controller:

    controller-server: controller

The certificate name in Puppet defaults to a systems hostname, but can be
overridden from the command line using the --certname option. The following
command could be used to convert a node into a controller.

    puppet agent --certname controller-server

It the provided certificate, contains a domain name, it will try to match it's
role against the shortened version of that name.

For example:

    foo.bar.com

would try to match

* foo.bar.com
* foo.bar
* foo

**TODO: the role mappings do not currently support regex, but probably need to**

### Data Mappings

Data mappings are used to express the way in which
global variables from map to individual class parameters.

Previous, this was done with parameter forwarding in parameterized
classes. In fact, this style of parameter forwarding is one of the main
functions of the previous openstack module.

The example below, shows how parameterized class forwarding could be used
to indicate that a single value called verbose should be used to set
the verbose setting of multiple classes.

    class openstack::controller(
      $verbose = false
    ) {

      class { 'nova': verbose => $verbose }
      class { 'glance': verbose => $verbose }
      class { 'keystone': verbose => $verbose }
      class { 'cinder': verbose => $verbose }
      class { 'quantum': verbose => $verbose }

    }

This is pretty concise way to express how a single data value assigns
multiple class parameters. The problem is, that it uses the parameterized
class declaration syntax to forward this data, meaning that it is hard to
reuse this code if you want to provider different settings. Any attempt to
specify any of these composed classes will result in a class duplication
error in Puppet.

The same configuration above can be expressed with the data\_mappings as
follows:

    verbose:
      - nova::verbose
      - glance::verbose
      - keystone::verbose
      - cinder::verbose
      - quantum::verbose

For each of those variables, the data-binding will call out to hiera when
the classes are processed (if they are included)

### Hiera Data

hiera data is used to express what values are going to be used to
configure the roles of your scenarios.

Hiera data is used to either express global keys (that were mapped to
class parameters in the data mappings), or fully qualified class parameter
namespaces.

NOTE: at the moment, fully qualified variables are ignored from hiera\_data
if they were defined in the data\_mappings. This is probably a bug (b/c they should
probably override), but this is how it works at the moment.

## Installation

1. install this module
2. synchronize it's plug-ins
3. configure puppet.conf

/etc/puppet/puppet.conf

    [master]
      node_terminus=scenario

4. configure your hiera backend

/etc/puppet/hiera.yaml

    ---
    :backends:
      - data_mapper
    :hierarchy:
      - "hostname/%{hostname}"
      - "client/%{clientcert}"
      - user
      - user.%{scenario}
      - user.common
      - global/${global_hiera_param}
      - common

## Command Line Debugging Tools

This module also comes with a collection of tools that can be used for debugging:

All of these utilities are implemented as a Puppet Face. To see the available commands, run:

    puppet help scenario

or to learn about an individual command:

    puppet help scenario get_classes

The data model exists outside of Puppet and is forwarded to
Puppet using a node terminus interface.

It currently supports several commands that can be used to pre-generate
parts of the data model for debugging purposes::

### get scenario

Returns the currently configured scenario:

    puppet scenario get_scenario

### get roles

Returns the list of roles for the current scenario along with their classes.

### get classes

To retrieve the list of classes that are associated with a role:

    puppet scenario get_classes <ROLE_NAME> --render-as yaml

### get class group

Retrieves the set of classes currently included in a class group.

    puppet scenario get_class_group <class_group_name>

### get all data

To retrieve the list of classes together with their specified data:

    puppet scenario compile_role <ROLE_NAME> --render-as yaml

This command is very similar to how Puppet interacts with the scenario
based data model.

### Getting Required User Configuration

Another use case of the data model is to generate data that can be compiled
to a list of configuration options available to an end user.

In this model, the data\_mappings are used to not only express the way that
a single key can populate multiple class parameters, it also is used to
signify all of the keys that should be exposed to an end user as a part of the
basic configuraiton.

### getting all current user data

The following command can be used to provide a list of all data that a user
may want to configure in their hiera\_data/user.yaml

    puppet scenario get_user_inputs

This command takes the current scenario and global settings into
account and produces a list of the configuration settings a user
may want to adjust along with their default values.

It also accepts --role, if you only want to get the settings that
are applicable to a specific role.

    puppet scenario get_user_inputs --role=build-server

### allow users to interactively specify data

  NOTE: this is still a prototype and is not fully functional

The following command can be used to supply user configuration data
to build out an example model:

    puppet scenario setup_scenario_data

It will prompt users for tons of questions related to how to configure
their specified deployment scenario.

## Implementation

This folder contains data that is used to express openstack deployment as data.

Technically, it is implemented as a custom hiera backend, and an external node classifier
(more specfically, as a custom node terminus, but for our purposes here, they can be
considered as the same thing)

It is critical to understand the following in order to understand this model:
