The osquery "public API" or SDK is the set of osquery headers and a subset of the source "cpp" files implementing what we call osquery **core**. The core code can be thought of as the framework or platform, it is everything except for the SQLite code and most table implementations. The public headers can be found in [osquery/include/osquery/](https://github.com/facebook/osquery/tree/master/include/osquery).

osquery is organized into a **core**, **additional**, and **testing** during a default build from source. We call the set of public headers implementing **core** the 'osquery SDK'. This SDK can be used to build osquery outside of our CMake build system with a minimum set of dependencies. This organization better isolates OS API dependencies from development tools and libraries and provides a logical separation between code needed for extensions and module compiling.

The public API and SDK headers are documented via **doxygen**. To generate web-based documentation, you will need to install doxygen, run `make docs` from the repository root, then open *./build/docs/html/index.html*.

## Extensions

Extensions are separate processes built using osquery **core** designed to register one or more plugins. An extension may be compiled and linked using an external build system, against proprietary code, and will be version-compatible with our publicly-built binary packages on [https://osquery.io/downloads](https://osquery.io/downloads).

osquery extensions should statically link the **core** code and use the `<osquery/sdk.h>` helper include file. Let's walk through a basic example extension (source for [example_extension.cpp](https://github.com/facebook/osquery/blob/master/osquery/examples/example_extension.cpp)):

```cpp
// Note 1: Include the sdk.h helper.
#include <osquery/sdk.h>

using namespace osquery;

// Note 2: Define at least one plugin.
class ExampleTablePlugin : public tables::TablePlugin {
 private:
  tables::TableColumns columns() const {
    return {{"example_text", "TEXT"}, {"example_integer", "INTEGER"}};
  }

  QueryData generate(tables::QueryContext& request) {
    QueryData results;
    Row r;

    r["example_text"] = "example";
    r["example_integer"] = INTEGER(1);
    results.push_back(r);
    return results;
  }
};

// Note 3: Use REGISTER_EXTERNAL to define your plugin.
REGISTER_EXTERNAL(ExampleTablePlugin, "table", "example");

int main(int argc, char* argv[]) {
  // Note 4: Start logging, threads, etc.
  osquery::Initializer runner(argc, argv, OSQUERY_EXTENSION);

  // Note 5: Connect to osqueryi or osqueryd.
  auto status = startExtension("example", "0.0.1");
  if (!status.ok()) {
    LOG(ERROR) << status.getMessage();
  }

  // Finally shutdown.
  runner.shutdown();
  return 0;
}
```

Extensions use osquery's [thrift API](https://github.com/facebook/osquery/blob/master/osquery.thrift) to communicate between osqueryi or osqueryd and the extension process. They may be written in any language that supports [Thrift](https://thrift.apache.org/). Only the osquery SDK provides the simple `startExtension` symbol that manages the life of your process including thrift service threads and a watchdog. C++ extensions should link: boost, thrift, glog, gflags, and optionally rocksdb for eventing.

The osqueryi or osqueryd processes start an "extension manager" thrift service thread that listens for extension register calls on a UNIX domain socket. Extensions may only communicate if the processes can read/write to this socket. An extension process running as a non-privileged user cannot register plugins to an osqueryd process running as root. Both osqueryi/osqueryd and C++ extensions using `startExtension` will deregister plugins if the communication becomes latent. Both are configurable using gflags or config options.

### Using the example extension

Please see the deployment [guide on extensions](../deployment/extensions.md) for a more-complete overview of how and why extensions are used.

If you [build from source](../development/building.md), you will build an example extension. The code can be found in the [`osquery/examples`](https://github.com/facebook/osquery/blob/master/osquery/examples/example_extension.cpp) folder; it adds a config plugin called "example" and additional table called "example". There are two ways to run an extension: load the extension at an arbitrary time after shell or daemon execution, or request an "autoload" of extensions. The auto-loading method has several advantages such as dependencies on external config plugins, and the same management and process monitoring applied to osquery worker processes.

To load the example extension in the shell try:
```
$ ./build/darwin/osquery/osqueryi
osquery> select path from osquery_extensions;
+-------------------------------------+
| path                                |
+-------------------------------------+
| /Users/USERNAME/.osquery/shell.em   |
+-------------------------------------+
osquery> ^Z
[1]  + 98777 suspended  ./build/darwin/osquery/osqueryi
```

Here we have started a shell process, inspected the UNIX domain socket path used for extensions, and suspended the process temporarily.

```
$ ./build/darwin/osquery/example_extension.ext --help
osquery 1.7.0, your OS as a high-performance relational database
Usage: osqueryi [OPTION]...

osquery extension command line flags:

    --interval VALUE  Seconds delay between connectivity checks
    --socket VALUE    Path to the extensions UNIX domain socket
    --timeout VALUE   Seconds to wait for autoloaded extensions

osquery project page <https://osquery.io>.
$ ./build/darwin/osquery/example_extension.ext --socket /Users/USERNAME/.osquery/shell.em &
```

Before executing the extension we've inspected the potential CLI flags, which are a subset of the shell or daemon's [CLI flags](../installation/cli-flags.md). The example extension is executed in the background so we can resume the shell and use the provided 'example' table.

```
[2] 98795
$ fg
[1]  - 98777 continued  ./build/darwin/osquery/osqueryi
osquery> select * from example;
+--------------+-----------------+
| example_text | example_integer |
+--------------+-----------------+
| example      | 1               |
+--------------+-----------------+
osquery>
```

If the responsible shell or daemon process ends the extension will soon after detect the loss of communication and also shutdown. Read more about the lifecycle of extensions in the deployment guide.

## Thrift API

[Thrift](https://thrift.apache.org/) is a code-generation/cross-language service development framework. osquery uses thrift to allow plugin extensions for config retrieval, log export, table implementations, event subscribers, and event publishers. We also use thrift to wrap our SQL implementation using SQLite.

**Extension API**

An extension process should implement the following API. During an extension's set up it will "broadcast" all the registered plugins to a shell or daemon process. Then the extension will be asked to start a UNIX domain socket and thrift service thread implementing the `ping` and `call` methods.

```thrift
service Extension {
  /// Ping to/from an extension and extension manager for metadata.
  ExtensionStatus ping(),
  /// Call an extension (or core) registry plugin.
  ExtensionResponse call(
    /// The registry name (e.g., config, logger, table, etc.).
    1:string registry,
    /// The registry item name (plugin name).
    2:string item,
    /// The thrift-equivilent of an osquery::PluginRequest.
    3:ExtensionPluginRequest request),
}
```

When an extension becomes unavailable, the shell or daemon process will automatically deregister those plugins.

**Extension Manager API (osqueryi/osqueryd)**

```thrift
service ExtensionManager extends Extension {
  /// Return the list of active registered extensions.
  InternalExtensionList extensions(),
  /// Return the list of bootstrap or configuration options.
  InternalOptionList options(),
  /// The API endpoint used by an extension to register its plugins.
  ExtensionStatus registerExtension(
    1:InternalExtensionInfo info,
    2:ExtensionRegistry registry),
  ExtensionStatus deregisterExtension(
    1:ExtensionRouteUUID uuid,
  ),
  /// Allow an extension to query using an SQL string.
  ExtensionResponse query(
    1:string sql,
  ),
  /// Allow an extension to introspect into SQL used in a parsed query.
  ExtensionResponse getQueryColumns(
    1:string sql,
  ),
}
```
