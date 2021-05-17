Resolvers
=========

Sceptre implements resolvers, which can be used to resolve a value of a
CloudFormation ``parameter`` or ``sceptre_user_data`` value at runtime. This is
most commonly used to chain the outputs of one Stack to the inputs of another.

If required, users can create their own resolvers, as described in the section
on `Custom Resolvers`_.

Syntax:

.. code-block:: yaml

   parameters:
     <parameter_name>: !<resolver_name> <resolver_value>
   sceptre_user_data:
     <name>: !<resolver_name> <resolver_value>

Available Resolvers
-------------------

environment_variable
~~~~~~~~~~~~~~~~~~~~

Fetches the value from an environment variable.

Syntax:

.. code-block:: yaml

   parameter|sceptre_user_data:
     <name>: !environment_variable ENVIRONMENT_VARIABLE_NAME

Example:

.. code-block:: yaml

   parameters:
       database_password: !environment_variable DATABASE_PASSWORD

file
~~~~

A Sceptre resolver to get file contents.  The returned value can be
passed into a parameter as a string, json, or yaml object.  The
file extension determines the return type.  By default, contents from
any files that do not end in `.json` or `.yaml` will be passed in as
a string.

Motivation:

Unlike the `file_contents`_ resolver which can only pass strings to parameters
this resolver can also pass file content in as json and yaml object.  This
resolver can also resolve remote file contents from the web.

Syntax:

```yaml
parameters|sceptre_user_data:
  <name>: !file /path/to/local/file
```

```yaml
parameters|sceptre_user_data:
  <name>: !file URL/To/File
```

Examples:

**Local file**

*text*
Get file content and pass it to the parameter as a text string:

tags/departments.txt
```text
"HR, Governance, Engineering, Marketing"
```

```yaml
parameters:
  departments: !file tags/departments.txt
```

*json*
Get file contents and pass it to the parameter as a json object:

tags/departments.json
```json
[
  "HR",
  "Governance",
  "Engineering",
  "Marketing"
]
```

```yaml
parameters:
  departments: !file tags/departments.json
```

*yaml*
Get file contents and pass it to the sceptre_user_data as a yaml object:

tags/departments.yaml
```yaml
- "HR"
- "Governance"
- "Engineering"
- "Marketing"
```

```yaml
sceptre_user_data:
  departments: !file tags/departments.yaml
```

__Note__: will do the same for files with `.yml` extension.


**Remote file**
Works similarly to local file except this will get file contents from the web.

*URL*
Get file contents from a URL reference:

```yaml
sceptre_user_data:
  departments: !file https://my-bucket.s3.us-east-1.amazonaws.com/tags/departments.json
```


file_contents
~~~~~~~~~~~~~

**deprecated**: Consider using the `file`_ resolver instead.

Reads in the contents of a file.

Syntax:

.. code-block:: yaml

   parameters|sceptre_user_data:
     <name>: !file_contents /path/to/file.txt

Example:

.. code-block:: yaml

   sceptre_user_data:
     iam_policy: !file_contents /path/to/policy.json

stack_output
~~~~~~~~~~~~

Fetches the value of an output from a different Stack controlled by Sceptre.

Syntax:

.. code-block:: yaml

   parameters | sceptre_user_data:
     <name>: !stack_output <stack_name>.yaml::<output_name>

Example:

.. code-block:: yaml

   parameters:
       VpcIdParameter: !stack_output shared/vpc.yaml::VpcIdOutput

Sceptre infers that the Stack to fetch the output value from is a dependency,
and builds that Stack before the current one.

This resolver will add a dependency for the Stack in which needs the output
from.

stack_output_external
~~~~~~~~~~~~~~~~~~~~~

Fetches the value of an output from a different Stack in the same account and
region. You can specify a optional AWS profile to connect to a different
account/region.

If the Stack whose output is being fetched is in the same StackGroup, the
basename of that Stack can be used.

Syntax:

.. code-block:: yaml

   parameters/sceptre_user_data:
     <name>: !stack_output_external <full_stack_name>::<output_name> <optional-aws-profile-name>

Example:

.. code-block:: yaml

   parameters:
     VpcIdParameter: !stack_output_external prj-network-vpc::VpcIdOutput prod

Custom Resolvers
----------------

Users can define their own resolvers which are used by Sceptre to resolve the
value of a parameter before it is passed to the CloudFormation template.

A resolver is a Python class which inherits from abstract base class
``Resolver`` found in the ``sceptre.resolvers module``.

Resolvers are require to implement a ``resolve()`` function that takes no
parameters and to call the base class initializer on initialisation.

Resolvers may have access to ``argument``, ``stack_config``,
``stack_group_config`` and ``connection_manager`` as an attribute of ``self``.
For example ``self.stack_config``.

Sceptre uses the ``sceptre.resolvers`` entry point to locate resolver classes.
Your custom resolver can be written anywhere and is installed as Python
package.
In case you are not familiar with python packaging, `this is great place to start`_.

Example
~~~~~~~

The following python module template can be copied and used:

.. code-block:: text

   custom_resolver
   ├── custom_resolver.py
   └── setup.py

The following python module template can be copied and used:

custom_resolver.py
^^^^^^^^^^^^^^^^^^

.. code-block:: python

        from sceptre.resolvers import Resolver


        class CustomResolver(Resolver):
            """
            The following instance attributes are inherited from the parent class Resolver.

            Parameters
            ----------
            argument: str
                The argument of the resolver.
            stack: sceptre.stack.Stack
                The associated stack of the resolver.

            """

            def __init__(self, *args, **kwargs):
                super(CustomResolver, self).__init__(*args, **kwargs)

            def resolve(self):
                """
                resolve is the method called by Sceptre. It should carry out the work
                intended by this resolver. It should return a string to become the
                final value.

                To use instance attribute self.<attribute_name>.

                Examples
                --------
                self.argument
                self.stack

                Returns
                -------
                str
                    Resolved value
                """
                return self.argument


setup.py
^^^^^^^^

.. code-block:: python

   from setuptools import setup

   setup(
       name='<custom_resolver_package_name>',
       py_modules=['<custom_resolver_module_name>'],
       entry_points={
           'sceptre.resolvers': [
               '<custom_resolver_command_name> = <custom_resolver_module_namef>:CustomResolver',
           ],
       }
   )

Then install using ``python setup.py install`` or ``pip install .`` commands.

This resolver can be used in a Stack config file with the following syntax:

.. code-block:: yaml

   template_path: <...>
   parameters:
     param1: !<custom_resolver_command_name> <value> <optional-aws-profile>


resolver arguments
^^^^^^^^^^^^^^^^^^
Resolver arguments can be a simple string or a complex data structure.

.. code-block:: yaml

    template_path: <...>
    parameters:
      Param1: !ssm "/dev/DbPassword"
      Param2: !ssm {"name": "/dev/DbPassword"}
      Param3: !ssm
        name: "/dev/DbPassword"

.. _Custom Resolvers: #custom-resolvers
.. _this is great place to start: https://docs.python.org/3/distributing/
