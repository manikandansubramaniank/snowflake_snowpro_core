# Connectors, Drivers & Partner Connect
True or False: A third-party tool that supports standard JDBC or ODBC but has no Snowflake-specific driver will be
unable to connect to Snowflake.

A. True<br>B. False

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>B</strong>

The statement is false. While Snowflake provides specific drivers for optimal performance and advanced
features, it also supports standard JDBC and ODBC interfaces. These industry-standard interfaces allow
applications and tools, even those without Snowflake-specific drivers, to connect and interact with Snowflake
databases. JDBC (Java Database Connectivity) and ODBC (Open Database Connectivity) are widely adopted
APIs enabling software applications to connect to various databases. When a third-party tool utilizes these
standard protocols, it can establish a connection to Snowflake through the designated Snowflake JDBC or
ODBC driver which is a prerequisite that supports the required protocols. The tool does not require an explicit
"Snowflake-specific driver", as the generic drivers will translate the standard SQL requests into a language
understood by the Snowflake engine. However, it’s crucial that the Snowflake JDBC or ODBC driver installed
on the machine where the third-party software is running matches the protocol of the 3rd party tool. The
Snowflake-specific drivers often include performance enhancements and Snowflake-specific features, but a
basic connection can still be achieved through standard JDBC or ODBC. Therefore, a third-party tool with
generic JDBC or ODBC capabilities can indeed connect to Snowflake, making the statement false. Essentially,
Snowflake supports standard connection protocols, expanding its compatibility with various tools.
Authoritative Links:
Snowflake JDBC Driver: https://docs.snowflake.com/en/developer-guide/jdbc/jdbc
Snowflake ODBC Driver: https://docs.snowflake.com/en/developer-guide/odbc/odbc
JDBC Overview: https://www.oracle.com/java/technologies/javase/jdbc.htmlODBC Overview: https://learn.microsoft.com/en-us/sql/odbc/odbc
</details>


---
Which of the following connectors are available in the Downloads section of the Snowflake Web Interface (UI)?
(Choose two.)

A. SnowSQL<br>B. ODBC<br>C. R<br>D. HIVE

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>A, B</strong>

The correct answer is A and B. Snowflake's web interface provides direct links for downloading client
connectors and drivers to interact with the Snowflake data platform. Among the available options, SnowSQL
and ODBC connectors are prominently featured in the downloads section.
SnowSQL is Snowflake's command-line client, allowing users to execute SQL queries and perform various
administrative tasks. It is essential for scripting and automating workflows involving Snowflake. The direct
download link in the web UI streamlines access to this core tool.
ODBC (Open Database Connectivity) is a standard API for accessing database systems. Snowflake provides an
ODBC driver, enabling applications and tools that support ODBC to connect to Snowflake. This connectionallows BI (Business Intelligence) tools, data integration platforms, and custom applications to seamlessly
access data stored within Snowflake. Similar to SnowSQL, the web UI provides easy access to this connector
for download.
Options C and D, representing R and HIVE connectors, are not directly listed in the "Downloads" section of the
web interface. While Snowflake can be connected to with these using specific drivers or connectors, they're
not as prominently featured for download within the Snowflake web interface. Instead, those are typically
acquired through other channels, like CRAN (Comprehensive R Archive Network) for R and through driver
repositories for HIVE.
Therefore, the primary purpose of the downloads section in the Snowflake UI is to provide readily accessible
download links for core connectivity tools directly supported by Snowflake, making options A (SnowSQL) and
B (ODBC) the correct choice.
Authoritative Links:
Snowflake Documentation - Downloading Drivers & Clients: https://docs.snowflake.com/en/user-guide/uidownloads.html
Snowflake Documentation - SnowSQL: https://docs.snowflake.com/en/user-guide/snowsql.html
Snowflake Documentation - ODBC Driver: https://docs.snowflake.com/en/user-guide/odbc.html
</details>


---
Which connectors are available in the downloads section of the Snowflake web interface (UI)? (Choose two.)


A. SnowSQL<br>B. JDBC<br>C. ODBC<br>D. HIVE<br>E. Scala

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>A, C</strong>

The correct answer is AC: SnowSQL and ODBC driver. Let's break down why:
Snowflake provides several ways to interact with its data warehouse. These interactions involve tools and
connectors that allow users and applications to connect and communicate. The "Downloads" section within
the Snowflake web interface provides direct access to essential tools.
SnowSQL: This is Snowflake's command-line client, which is a fundamental tool for executing SQL queries
and performing administrative tasks. It is available for download as it's a primary way for users to interact
directly with Snowflake. (https://docs.snowflake.com/en/user-guide/snowsql.html)
ODBC Driver: Open Database Connectivity (ODBC) is a standard API that enables applications to access
databases. Snowflake provides an ODBC driver, allowing various applications (like reporting tools and custom
applications) to connect to Snowflake. The ODBC driver facilitates integration with existing systems that use
ODBC. (https://docs.snowflake.com/en/user-guide/odbc.html)
Let's look at why the other options are incorrect:
JDBC Driver: While Snowflake does provide a JDBC driver (Java Database Connectivity), it is usually
downloaded from the Maven Central Repository rather than directly from the "Downloads" section of the
Snowflake web UI. It is common to manage java dependencies through a dependency management tool.
HIVE: HIVE is related to Hadoop ecosystem and not native to Snowflake. There is no HIVE connector available
in the Snowflake web UI.
Scala: Scala is a programming language often used with Spark. While Snowflake can integrate with Spark,
the Scala language itself is not a connector directly available for download within Snowflake's UI. Spark
connectors are often integrated in cloud platforms using external functions or snowpark.
</details>


---
Which of the following SQL statements will list the version of the drivers currently being used?

A. Execute SELECT CURRENT_ODBC_CLIENT(); from the Web UI<br>B. Execute SELECT CURRENT_JDBC_VERSION(); from SnowSQL<br>C. Execute SELECT CURRENT_CLIENT(); from an application<br>D. Execute SELECT CURRENT_VERSION(); from the Python Connector

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>C</strong>

The correct answer is C, Execute SELECT CURRENT_CLIENT(); from an application. The CURRENT_CLIENT()
function in Snowflake is specifically designed to return a string that identifies the client application or driver
that is currently connected to the Snowflake instance. This includes information about the driver type and
version. Options A, B, and D, while related to Snowflake information, do not serve the purpose of showing the
specific driver version in use. CURRENT_ODBC_CLIENT() is not a valid Snowflake function.
CURRENT_JDBC_VERSION() would provide the version of the JDBC driver, but it is not as general as
CURRENT_CLIENT(). Similarly, CURRENT_VERSION() returns the version of the Snowflake database system
itself, not the connected client application. Using CURRENT_CLIENT() within the application itself will reveal
the client driver, including its version, as it is the application using that specific driver for the connection. This
is useful for debugging and ensuring compatibility. The query is designed to be run from within the connected
application, not directly from the Snowflake UI. Therefore, option C specifically reflects the intended use case
of identifying the version of the driver used by a specific application connection.
Relevant Snowflake documentation for reference:
CURRENT_CLIENT: This documentation directly outlines the purpose of CURRENT_CLIENT() function.
CURRENT_VERSION: For understanding the use case of CURRENT_VERSION().
Overview of Snowflake Drivers and Connectors: To understand why driver versions are important.
</details>


---
Snowflake Partner Connect is limited to users with a verified email address and which role?

A. SYSADMIN<br>B. SECURITYADMIN<br>C. ACCOUNTADMIN<br>D. USERADMIN

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>C</strong>

The correct answer is C, ACCOUNTADMIN. Snowflake's Partner Connect is a feature designed to facilitate
easy integration with various third-party tools and services. Access to this functionality is intentionally
restricted to higher-level roles due to the potential impact on the entire Snowflake account. Specifically, the
ACCOUNTADMIN role holds the highest level of privileges, encompassing all actions and aspects of a
Snowflake account. This includes the authority to manage users, set global parameters, configure security
settings, and crucially, establish and manage connections through Partner Connect. The verification of an
email address serves as a prerequisite, ensuring that users accessing such powerful capabilities are
legitimate and identifiable. Other roles like SYSADMIN, SECURITYADMIN, and USERADMIN, while powerful in
their own domains, do not possess the overarching authority required to manage Partner Connect
configurations. SYSADMIN handles system level configurations, SECURITYADMIN focuses on security
aspects, and USERADMIN manages users, none of which align directly with the account-wide implications of
Partner Connect access. Therefore, maintaining tight control over Partner Connect access with the
ACCOUNTADMIN role mitigates potential security risks and ensures consistent, authorized configuration.For further research, refer to the official Snowflake documentation:
Snowflake Partner Connect
Snowflake Access Control
Snowflake Account-Level Roles
</details>


---
A user wants to upload a file to an internal Snowflake stage using a PUT command.
Which tools and/or connectors could be used to execute this command? (Choose two.)

A. SnowCD<br>B. SnowSQL<br>C. SQL API<br>D. Python connector<br>E. Snowsight worksheets

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>B, D</strong>

The correct answer is B. SnowSQL and D. Python connector. Here's why:
SnowSQL is Snowflake's command-line client, specifically designed for interacting with
Snowflake using SQL and other commands. The PUT command is a core Snowflake SQL command,
and SnowSQL is a natural choice for executing it. It allows users to interact with the database
directly from their terminal, providing a flexible and scriptable interface for data loading
operations like uploading files to stages. (Source: SnowSQL Documentation)
The Python connector provides a programmatic way to interact with Snowflake. It allows
developers to embed Snowflake operations directly within their Python code. This includes
executing SQL statements such as PUT to upload files. The Python connector offers flexibility for
automating data workflows and incorporating Snowflake into Python-based data processingpipelines. (Source: Snowflake Python Connector)
SnowCDB (Option A) is not a standard Snowflake tool or connector; it appears to be a misspelling
or a non-existent option. SQL API (Option C) provides a REST-based interface for executing SQL
queries and DDL operations, however, the 'PUT' command requires direct file handling and
streaming and is not well suited for typical REST request-response patterns with file payloads.
Snowsight worksheets (Option E) are primarily designed for running queries and exploring data
within the Snowflake web interface, and are not used for direct file manipulation like the PUT
command. While worksheets can execute SQL including PUT, they do not upload files directly,
instead, the web ui requires you to upload it then the query will refer to it.Therefore, SnowSQL and
the Python connector are the most suitable options for executing the PUT command for uploading
files to Snowflake stages.
</details>


---
What does Snowflake recommend a user do if they need to connect to Snowflake with a tool or technology that is
not listed in Snowflake’s partner ecosystem?

A. Use Snowflake’s native API.<br>B. Use a custom-built connector.<br>C. Contact Snowflake Support for a new driver.<br>D. Connect through Snowflake’s JDBC or ODBC drivers.

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>D</strong>

The correct answer is D: Connect through Snowflake’s JDBC or ODBC drivers. Here's why:Snowflake provides standardized interfaces, namely JDBC (Java Database Connectivity) and
ODBC (Open Database Connectivity) drivers, designed for broad compatibility. These drivers act
as translation layers, allowing diverse applications and tools to communicate with Snowflake's
database engine using standard SQL. If a specific tool isn't directly supported by Snowflake's
partner ecosystem, relying on JDBC or ODBC ensures a common ground for data exchange.
Snowflake's architecture is built to support these drivers as primary connection mechanisms
beyond its native API. Option A, using Snowflake’s native API, is helpful for direct interaction but
might require more custom coding than using pre-built drivers. Option B, building a custom
connector, is also a possibility but involves significant development effort, making it less ideal for
standard tools. Contacting support (Option C) isn't suitable as Snowflake encourages using
readily available JDBC/ODBC. JDBC and ODBC are cloud computing standards for database
connectivity, ensuring broad interoperability and reducing vendor lock-in. Therefore, utilizing
Snowflake’s JDBC/ODBC drivers is the most efficient and recommended approach when
connecting with tools not in their partner list.
Authoritative Links:
Snowflake JDBC Driver Documentation: https://docs.snowflake.com/en/developerguide/jdbc/index
Snowflake ODBC Driver Documentation: https://docs.snowflake.com/en/developerguide/odbc/index
Snowflake Drivers and Connectors: https://www.snowflake.com/partners/technology-partners/
(While not directly stating this use case, it shows they offer these drivers).
</details>


---
Which drivers or connectors are supported by Snowflake? (Choose two.)

A. Perl Connector<br>B. MongoDB Rust Driver<br>C. Go Snowflake Driver<br>D. Cobol Driver<br>E. Snowflake Connector for Python

<details>
<summary><strong>✅ Answer : </strong></summary>
<strong>C, E</strong>

The Correct answer is ["C","E"]
</details>


