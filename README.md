Introduction {#mainpage}
============

The PostgreSQL Frontend (Pgfe) is client C++ API to PostgreSQL servers. **ATTENTION, this
software is "beta" quality, and the API is a subject to change**. Any feedback (*especially
results of testing*) is highly appreciated! Together we can make Pgfe library *really*
production-ready!

Hello, World
============

```cpp
#include <dmitigr/pgfe.hpp>
#include <iostream>

int main()
{
  namespace pgfe = dmitigr::pgfe;
  try {
    const auto conn = pgfe::Connection_options::make()->
      set(pgfe::Communication_mode::tcp)->
      set_tcp_host_name("localhost")->
      set_database("pgfe_test")->
      set_username("pgfe_test")->
      set_password("pgfe_test")->
      make_connection();

    conn->connect();
    conn->execute("SELECT generate_series($1::int, $2::int) AS natural", 1, 3);
    conn->for_each([](const auto* const row)
      {
        std::cout << pgfe::to<int>(row->data("natural")) << "\n";
      });
    std::cout << "The " << conn->completion()->operation_name() << " query is done.";
  } catch (const std::exception& e) {
    std::cout << "Oops: " << e.what() << std::endl;
  }
}
```

Features
========

Current API allows to work with:

  - database connections (in both blocking and non-blocking IO manner);
  - prepared statements (named parameters are supported);
  - SQLSTATE codes (as simple as with enums);
  - dynamic SQL;
  - extensible data type conversions (including support of PostgreSQL arrays to/from STL containers conversions).

Features of the near future
---------------------------

The urgent TODO-list includes:

  - support for Large Objects via IO streams of the Standard C++ library;
  - support of COPY command;
  - support of work with SQL queries separately of C++ code;
  - `dmitigr::pgfe::Composite` data type to work with composite types;
  - `dmitigr::pgfe::Dynamic_array` data type to work with arrays of variable dimensions;
  - C API.

Tutorial
========

Client programs that use Pgfe should include header file `dmitigr/pgfe.hpp` and
must link with `dmitigr_pgfe` library. Logically Pgfe library consists of the
following parts:

  - Main (client/server communication);
  - Large objects (future feature, see the above TODO-list);
  - Data types (future feature, see the above TODO-list);
  - Data types conversions;
  - Errors (exceptions and error codes);
  - Utilities.

**WARNING** Headers other than `dmitigr/pgfe.hpp` should be *avoided* to use in
applications since that headers are subject to reorganize. Also, namespaces
`dmitigr::pgfe::detail` and `dmitigr::pgfe::internal` contains implementation
details and an internal stuff and *should not* be used in applications.

Connecting to a server
----------------------

The class dmitigr::pgfe::Connection is a central abstraction of Pgfe library. By
using methods of this class it is possible to:

  - send requests to a server;
  - receive responses from a server (see dmitigr::pgfe::Response);
  - receive signals from a server (see dmitigr::pgfe::Signal);
  - perform other operations that depend on a server data (such as dmitigr::pgfe::Connection::to_quoted_literal()).

To make an instance of the class dmitigr::pgfe::Connection, the instance of the
class dmitigr::pgfe::Connection_options is required. A copy of this instance
is always *read-only* accessible via dmitigr::pgfe::Connection::options().

Example 1. Creation of the connection with customized options:

```cpp
std::unique_ptr<dmitigr::pgfe::Connection> create_customized_connection()
{
  return pgfe::Connection_options::make()->
    set(Communication_mode::tcp)->
    set_tcp_host_name("localhost")->
    set_database("db")->
    set_username("user")->
    set_password("password")->
    make_connection();
}
```

Example 2. Creation of the connection with default options:

```cpp
std::unique_ptr<dmitigr::pgfe::Connection> create_default_connection_1()
{
  const auto opts = pgfe::Connection_options::make();
  return pgfe::Connection::make(opts.get());
}
```

Example 3. Creation of the connection with default options:

```cpp
std::unique_ptr<dmitigr::pgfe::Connection> create_default_connection_2()
{
  return pgfe::Connection::make();
}
```

After creation of an object of type dmitigr::pgfe::Connection there are two ways to connect available:

  - synchronous by using dmitigr::pgfe::Connection::connect();
  - asynchronous by using dmitigr::pgfe::Connection::connect_async().

Executing commands
------------------

SQL commands can be executed through either of two ways:

  1. by using "simple query" protocol (which implies parsing and executing a
     query by a server on each request) with dmitigr::pgfe::Connection::perform();
  2. by using "extended query" protocol (which implies using of parameterizable prepared statements):
     + by explicitly preparing a statement with dmitigr::pgfe::Connection::prepare_statement()
       and executing it with dmitigr::pgfe::Prepared_statement::execute();
     + by implicitly preparing and executing an unnamed prepared statement with
       dmitigr::pgfe::Connection::execute().

Commands can be executed and processed asynchronously, i.e. without need of waiting
a server response(-s), and thus, without thread blocking. For this purpose the
methods of the class dmitigr::pgfe::Connection with the suffix `_async` shall be used,
such as dmitigr::pgfe::Connection::perform_async() or dmitigr::pgfe::Connection::prepare_statement_async().

Prepared statements can be parameterized with either positional or named parameters.
In order to use the named parameters, a SQL string must be preparsed by Pgfe.
Preparsed SQL strings are represented by the class dmitigr::pgfe::Sql_string.
Unparameterized prepared statements, or prepared statements parameterized by
only positional parameters *does not* require to be preparsed. Thus, there is no
need to create an instance of dmitigr::pgfe::Sql_string and `std::string` should
be used instead.

To set a value of a prepared statement's parameter it should be converted to an
object of the class dmitigr::pgfe::Data. For convenience, there is the templated
method dmitigr::pgfe::Prepared_statement::set_parameter(std::size_t, T&&) which
do such a conversion by using one of the specialization of the template structure
dmitigr::pgfe::Conversions.

Example 1. Simple querying.

```cpp
void simple_query(dmitigr::pgfe::Connection* conn)
{
  conn->perform("SELECT generate_series(1, 3) AS num");
}
```

Example 2. Implicit execution of the unnamed prepared statement.

```cpp
void implicit_prepare_and_execute(dmitigr::pgfe::Connection* conn)
{
  conn->execute("SELECT generate_series($1::int, $2::int) AS num", 1, 3);
}
```

Example 3. Explicit execution of the named prepared statement with named parameters.

```cpp
void explicit_prepare_and_execute(const std::string& name, dmitigr::pgfe::Connection* conn)
{
  static const auto sql = dmitigr::pgfe::Sql_string::make("SELECT generate_series(:infinum::int, :supremum::int) AS num");
  auto ps = conn->prepare_statement(sql.get(), name);
  ps->set_parameter("infinum",  1);
  ps->set_parameter("supremum", 3);
  ps->execute();
}
```

Responses handling
------------------

Server responses are represented by the classes, inherited from dmitigr::pgfe::Response:

  - Responses that are server errors are represented by the class dmitigr::pgfe::Error.
    Each server error is identifiable by a so-called SQLSTATE code. In Pgfe *each* such
    a code is represented by the member of the enum class dmitigr::pgfe::Server_errc,
    integrated in framework for reporting errors provided by the standard library in
    `<system_error>`. Therefore, working with SQLSTATE codes is as simple and safe as
    with `std::error_code` and enumerated types!

  - Responses that are rows are represented by the class dmitigr::pgfe::Row. Objects of
    this class can be accessed by using dmitigr::pgfe::Connection::row() and/or
    dmitigr::pgfe::Connection::release_row(). However, it is best to use the method
    dmitigr::pgfe::Connection::for_each() for rows processing. Be aware, that before
    executing the subsequent operations, all of the rows **must** be processed.

  - Responses that are prepared statements are represented by the class
    dmitigr::pgfe::Prepared_statement. Prepared statements are accessible via the
    method dmitigr::pgfe::Connection::prepared_statement().

  - Responses that indicates success of operations are represented by the class
    dmitigr::pgfe::Completion. Such responses can be accessed by calling
    dmitigr::pgfe::Connection::completion() and/or dmitigr::pgfe::Connection::release_completion().
    Alternatively, to process completion responses the method dmitigr::pgfe::Connection::complete()
    can be used.

To *initiate* asynchronous retrieving of the *first* response (i.e. with no blocking the thread),
methods of the class dmitigr::pgfe::Connection with the suffix "_async" must be used. Otherwise,
Pgfe will wait for the first response and if that response is dmitigr::pgfe::Error, an object of
type dmitigr::pgfe::Server_exception will be thrown as exception. This object provides access to
the retrieved object of type dmitigr::pgfe::Error, which contains the error details.

Server responses can be *retrieved*:
  - synchronously by using the methods such as dmitigr::pgfe::Connection::wait_response()
    and dmitigr::pgfe::Connection::wait_last_response();
  - asynchronously by using the methods such as dmitigr::pgfe::Connection::collect_server_messages()
    and dmitigr::pgfe::Connection::socket_readiness();

Data type conversions
---------------------

The class dmitigr::pgfe::Data is designed to store:

  - the values of prepared statements' parameters;
  - the data retrieved from PostgreSQL server.

The template structure dmitigr::pgfe::Conversions are used by:

  - dmitigr::pgfe::Prepared_statement::set_parameter(std::size_t, T&&) - to perfrom data
    conversions from objects or type `T` to objects of type dmitigr::pgfe::Data;
  - dmitigr::pgfe::to() - to perform data conversions from objects of type
    dmitigr::pgfe::Data to objects of the specified type `T`.

There is the partial specialization of the template structure dmitigr::pgfe::Conversions to
perform conversions from/to PostgreSQL arrays representation to any combination of the STL
containers. (At the moment, arrays conversions are only implemented for dmitigr::pgfe::Data_format::text
format.) Any PostgreSQL array can be represented as `Container<Optional<T>>`, where:

  - `Container` - is the template class of the container such as `std::vector` or `std::list`;
  - `Optional` - is the template class of the optional value holder such as `std::optional` or `boost::optional`.
    The special value of `std::nullopt` represents the SQL `NULL`;
  - T - is the type of elements of the array. It can be `Container<Optional<T>>` to represent
    the multidimensional array. For example, the type `std::vector<std::optional<std::list<std::optional<int>>>>`
    can be used to represent 2-dimensional array of integers!

User-defined data conversions could be implemented by either:

  - implementing the `operator<<(std::ostream&, const T&)` and `operator>>(std::istream&, T&)`;
  - specializing the template structure dmitigr::pgfe::Conversions. (With this approach overheads
    of standard IO streams can be avoided.)

Signals handling
----------------

Server signals are represented by the classes, inherited from dmitigr::pgfe::Signal:

  - signals that are server notices are represented by the class dmitigr::pgfe::Notice;
  - signals that are server notifications are represented by the class dmitigr::pgfe::Notification.

Signals can be handled:

  - synchronously, by using the signal handlers (see dmitigr::pgfe::Connection::set_notice_handler(),
    dmitigr::pgfe::Connection::set_notification_handler());
  - asynchronously, by using the methods that provides access to the retrieved signals directly (see
    dmitigr::pgfe::Connection::notice(), dmitigr::pgfe::Connection::notification()).

Signal handlers, being set, called by dmitigr::pgfe::Connection::handle_signals(). The latter is
called automatically while waiting a response. If no handler is set, corresponding signals will be
collected in the internal storage and can be popped up by using dmitigr::pgfe::Connection::pop_notice()
and/or dmitigr::pgfe::Connection::pop_notification() depending on the type of signal.

**WARNING** If signals are not popped up from the internal storage it may cause memory exhaustion!
Thus, signals must be handled anyway!

Dynamic SQL
-----------

The standard tools like `std::string` or `std::ostringstream` can be used to make
SQL strings dynamically. However, in some cases it is more convenient to use the
class dmitigr::pgfe::Sql_string for this purpose.

Consider the following statement:

```cpp
auto sql = dmitigr::pgfe::Sql_string::make("SELECT :expr::int, ':expr'");
```

This statement has one named parameter `expr` and one string constant `':expr'`. If prepare this
statement with dmitigr::pgfe::Connection::prepare_statement(), the actual prepared statement
parsed by the server will looks like:

```sql
SELECT $1::int, ':expr'
```

Before preparing the statement, it is possible to replace the named parameters of the SQL
string with another SQL string by using dmitigr::pgfe::Sql_string::replace_parameter().
For example:

```cpp
auto sql = dmitigr::pgfe::Sql_string::make("SELECT :expr::int, ':expr'");
sql->replace_parameter("expr", "sin(:expr1::int), cos(:expr2::int)");
```

Now the statement has two named parameters, and looks like:

```sql
SELECT sin(:expr1::int), cos(:expr2::), ':expr'
```

Note, that the quoted string `:expr` is not affected by the replacement operation!

Exceptions
----------

Pgfe may throw:

  - an instance of the type `std::logic_error` when:
      + API contract requirements are violated;
      + an assertion failure has occurred (it is possible only with the "debug" build of Pgfe);
  - an instance of the types `std::runtime_error` or dmitigr::pgfe::Client_exception
    when some kind of runtime error occured on the client side;
  - the instance of the type dmitigr::pgfe::Server_exception when some error occured
    on the server side and the methods like dmitigr::pgfe::Connection::wait_response_throw() is in use
    (which is case when using dmitigr::pgfe::Connection::perform(), dmitigr::pgfe::Connection::execute() etc).

Thread safety
-------------

By default, if not explicitly documented, all functions and methods of Pgfe are *not*
thread safe. Thus, in most cases, some of the synchronization mechanisms (like mutexes)
must be used to work with the same object from several threads.

Documentation
=============

The documentation is located at <http://dmitigr.ru/pgfe/doc/>.

Download
========

Pgfe can be downloaded from Github - <https://github.com/dmitigr/pgfe>.

Installation
============

Dependencies
------------

- CMake build system version 3.10+;
- C++17 compiler (GCC 8+ or Microsoft Visual C++ 15.7+);
- libpq library (the underlying engine).

Build time settings
-------------------

Settings that may be specified at build time by using CMake variables are:
  1. the type of the build;
  2. the flag of build the shared library;
  3. the flag of building the tests (default is on);
  4. installation directories;
  5. default values of the connection options.

Details:

|CMake variable|Possible values|Default on Unix|Default on Windows|
|:-------------|:--------------|:--------------|:-----------------|
|**The type of the build**||||
|CMAKE_BUILD_TYPE|Debug \| Release \| RelWithDebInfo \| MinSizeRel|Debug|Debug|
|**The flag of build the shared library**||||
|BUILD_SHARED_LIBS|On \| Off|On|On|
|**The flag of building the tests**||||
|PGFE_BUILD_TESTS|On \| Off|On|On|
|**Installation directories**||||
|CMAKE_INSTALL_PREFIX|*an absolute path*|"/usr/local"|"%ProgramFiles%\dmitigr"|
|PGFE_SHARE_INSTALL_DIR|*a path relative to CMAKE_INSTALL_PREFIX*|"share/dmitigr"|"share"|
|PGFE_LIBRARY_INSTALL_DIR|*a path relative to CMAKE_INSTALL_PREFIX*|"lib"|"lib"|
|PGFE_INCLUDES_INSTALL_DIR|*a path relative to CMAKE_INSTALL_PREFIX*|"include/dmitigr"|"include/dmitigr"|
|**Default values of the connection options**||||
|PGFE_CONNECTION_COMMUNICATION_MODE|uds \| tcp|uds|tcp|
|PGFE_CONNECTION_UDS_DIRECTORY|*an absolute path*|/tmp|*unavailable*|
|PGFE_CONNECTION_UDS_FILE_EXTENSION|*a string*|5432|*unavailable*|
|PGFE_CONNECTION_UDS_REQUIRE_SERVER_PROCESS_USERNAME|*a string*|*not set*|*unavailable*|
|PGFE_CONNECTION_TCP_KEEPALIVES_ENABLED|On \| Off|Off|Off|
|PGFE_CONNECTION_TCP_KEEPALIVES_IDLE|*non-negative number*|*null (system default)*|*null (system default)*|
|PGFE_CONNECTION_TCP_KEEPALIVES_INTERVAL|*non-negative number*|*null (system default)*|*null (system default)*|
|PGFE_CONNECTION_TCP_KEEPALIVES_COUNT|*non-negative number*|*null (system default)*|*null (system default)*|
|PGFE_CONNECTION_TCP_HOST_ADDRESS|*IPv4(v6) address*|127.0.0.1|127.0.0.1|
|PGFE_CONNECTION_TCP_HOST_NAME|*a string*|localhost|localhost|
|PGFE_CONNECTION_TCP_HOST_PORT|*a number*|5432|5432|
|PGFE_CONNECTION_USERNAME|*a string*|postgres|postgres|
|PGFE_CONNECTION_DATABASE|*a string*|postgres|postgres|
|PGFE_CONNECTION_PASSWORD|*a string*|""|""|
|PGFE_CONNECTION_KERBEROS_SERVICE_NAME|*a string*|*null (not used)*|*null (not used)*|
|PGFE_CONNECTION_SSL_ENABLED|On \| Off|Off|Off|
|PGFE_CONNECTION_SSL_SERVER_HOST_NAME_VERIFICATION_ENABLED|On \| Off|Off|Off|
|PGFE_CONNECTION_SSL_COMPRESSION_ENABLED|On \| Off|Off|Off|
|PGFE_CONNECTION_SSL_CERTIFICATE_FILE|*an absolute path*|*null (libpq's default)*|*null (libpq's default)*|
|PGFE_CONNECTION_SSL_PRIVATE_KEY_FILE|*an absolute path*|*null (libpq's default)*|*null (libpq's default)*|
|PGFE_CONNECTION_SSL_CERTIFICATE_AUTHORITY_FILE|*an absolute path*|*null (libpq's default)*|*null (libpq's default)*|
|PGFE_CONNECTION_SSL_CERTIFICATE_REVOCATION_LIST_FILE|*an absolute path*|*null (libpq's default)*|*null (libpq's default)*|

Installation on Linux
---------------------

    $ git clone https://github.com/dmitigr/pgfe.git
    $ mkdir -p pgfe/build
    $ cd pgfe/build
    $ cmake -DBUILD_TYPE=Debug ..
    $ make
    $ sudo make install

The value of the `BUILD_TYPE` could be replaced.

Installation on Microsoft Windows
---------------------------------

Run the Developer Command Prompt for Visual Studio and type:

    > git clone https://github.com/dmitigr/pgfe.git
    > mkdir pgfe\build
    > cd pgfe\build
    > cmake -G "Visual Studio 15 2017 Win64" ..
    > cmake --build -DBUILD_TYPE=Debug ..

Next, run the Elevated Command Prompt (i.e. the command prompt with administrator privileges) and type:

    > cd pgfe\build
    > cmake -DBUILD_TYPE=Debug -P cmake_install.cmake

If the target architecture is Win32 or ARM, then "Win64" should be replaced by "Win32" or "ARM" accordingly.
The value of the `BUILD_TYPE` could be also replaced.

**WARNING** The target architecture must corresponds to the bitness of libpq to link!

License
=======

Pgfe library is distributed under zlib license. For conditions of distribution and use,
see files `LICENSE.txt` or `pgfe.hpp`.

Contributions, sponsorship, partnership
=======================================

Pgfe has been developed on the own funds. Donations are welcome!

If you are using Pgfe for commercial purposes it is reasonable to donate or
even sponsor the further development of Pgfe.

To make a donation, please go to [here](https://www.paypal.com/cgi-bin/webscr?cmd=_donations&business=38TY2K8KYKYJC&lc=US&item_name=Pgfe%20library&item_number=1&currency_code=USD&bn=PP%2dDonationsBF%3abtn_donateCC_LG%2egif%3aNonHosted) or
[here](https://paypal.me/dmitigr).

If you need a commercial support, or you need to develop a custom client-side or
server-side software based on PostgreSQL, please contact us by sending email to <dmitigr@gmail.com>.

Pgfe is a free software. Enjoy using it!

Copyright
=========

Copyright (C) Dmitry Igrishin
