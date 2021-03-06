Audit trail
===========

For every built artifact Bob records all involved sources and build steps that
lead to a particular artifact. The recorded information contains at least, but
not limited to, the following records:

* state of the recipes,
* recipe name, package path,
* build host/time,
* environment,
* dependencies (with their respective audit trail),
* state of SCMs (e.g. commit id, dirty, ...).

The information in the audit trail records may be extended with additional
information in the future. An application that parses the audit trail should
ignore unknown fields for future compatibility.

Storage format
--------------

Audit trails are stored as gzip compressed JSON documents. For local builds the
audit trail is stored as ``audit.json.gz`` file next to the workspace. Jenkins
builds only rely on binary artifacts where the same file is stored in the
compressed tar file in a ``meta/`` directory. The general structure of an audit
trail looks like the following::

    {
        "artifact" : {
            // audit record
        },
        "references" : [
            {
                // audit record
            },
            ...
        ]
    }

The audit information about the involved artifact is stored under the
``artifact`` key. Any audit records about the dependencies that were used to
create the artifact are included in the list under the ``references`` key. This
includes all transitive records too. A correct audit trail must include the
full transitive information to be accepted by Bob.

Records
-------

The following sections describe the various keys and their semantics that can
be found in an audit record.

Basic information
~~~~~~~~~~~~~~~~~

``artifact-id``
    Hexadecimal number that identifies a particular artifact. This is also the
    primary key for audit records.

``variant-id``
    The Variant-Id as described in :ref:`concepts-implicit-versioning`.

``build-id``
    The Build-Id as described in :ref:`concepts-implicit-versioning`.

``result-hash``
    A hash sum across the content of the workspace after the artifact was
    built.

``env``
    Dump of the bash environment as created by ``declare -p``. See
    `bash declare`_.

.. _bash declare: https://www.gnu.org/software/bash/manual/html_node/Bash-Builtins.html#index-declare

Recipes
~~~~~~~

If Bob recognizes that the recipes are managed in a supported SCM (currently
git or svn) there will be a ``recipes`` key in the audit record. The format of
the object under this key is described in :ref:`audit-trail-scms`.


Dependencies
~~~~~~~~~~~~

Each step can have any number of dependencies. They will be recorded under a
``dependencies`` key. The other step is referenced by the Artifact-Id and their
audit record will be found in the ``references`` list of the audit trail. There
are three types of dependencies to other steps that each have their different
representation in audit record:

``arguments``
    Ordered list of all dependencies whose result was input to this step. They
    correspond to the ``$1`` to ``$n`` arguments of the script that was
    executed.

``tools``
    Object that maps all available tools by their name to the Artifact-Id.

``sandbox``
    Used sandbox during execution.

Example::

    "dependencies" : {
        "args" : [
            "b0a6632c6e7677220e46e4ae9c528efb949137c6"
        ],
        "tools" : {
            "toolchain" : "0b1c5e3489bed347ccf8e0e1e12dc70c92b09472"
        },
        "sandbox" : "3473b28df3891046618420428b530418ce006ad9"
    }

.. _audit-trail-scms:

SCMs
~~~~

All SCMs are recorded after the checkout step was run. The audit record will
contain a list of objects under the ``scms`` key. Each object has at least a
``type`` key that identifies the kind ob SCM and a ``dir`` key for the relative
directory (or file) that was managed by the SCM in the workspace.

See the following list for the additional information that each SCM adds to the
record:

git
    The git SCM records all remotes, the current commit that HEAD points to and
    if the tree is dirty. The output of ``git describe`` is also recorded.

    Example::

        {
            "commit": "6e986014563b70ecd867fb6a6e1adeb408f63dd6",
            "description": "v0.11.0-59-g6e98601-dirty",
            "dir": ".",
            "dirty": true
            "remotes": {
                "origin": "git@github.com:BobBuildTool/bob.git"
            },
            "type": "git",
        }

svn
    Example::

        {
            "dir" : ".",
            "dirty" : false,
            "repository" : {
                "root" : "http://svn.haiku-os.org/oldhaiku",
                "uuid" : "a95241bf-73f2-0310-859d-f6bbb57e9c96",
            },
            "revision" : 43238,
            "type" : "svn",
            "url" : "http://svn.haiku-os.org/oldhaiku/haiku/",
        }

url
    Example::

        {
            "digest" : {
                "algorithm" : "sha1",
                "value" : "697b7c87c73eb53bf80e19b65a4ac245214d530c" 
            },
            "dir" : "author.txt",
            "type" : "url",
        }


Meta data
~~~~~~~~~

There can be any number of key-value meta data pairs. They will be contained
under the ``meta`` key and typically hold at least the following information:

``bob``
    Bob version string.

``package``
    Package path of the artifact that was built. Note that there might be
    multiple packages that produce the same result. Only one will be built by
    Bob without recording all possible package paths here.

``recipe``
    Name of the recipe that declared the package.

``step``
    The executed step for this audit record. Can be ``src``, ``build`` or
    ``dist``.

Example::

    "meta" : {
        "bob" : "0.11.0-56-g9b3d2c6-dirty",
        "package" : "root/lib"
        "recipe" : "lib",
        "step" : "src",
    },

