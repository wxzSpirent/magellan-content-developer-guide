# Magellan: A Content Developer's Perspective
+ [Overview](#overview)
  - [The Grand Scheme of Magellan](#the-grand-scheme-of-magellan)
  - [Roadmap](#roadmap)
+ [BLL](#bll)
  - [File Locations](#file-locations-1)
  - [Dimension Set](#dimension-set)
  - [Result Set](#result-set)
  - [Debug Tips](#bll-debug-tips)
+ [Views and Profiles](#views-and-profiles)
  - [File Locations](#file-locations-2)
  - [Standard View](#standard-view)
  - [Drill Down View](#drill-down-view)
  - [Generator Scripts](#generator-scripts)
  - [Profiles]
  - [Debug Tips]
+ [Advanced features :sparkles:](#advanced-features)
  - [Health Indicator](#health-indicator)
  - [Chart View](#chart-view)
  - [Report](#report)
## Overview ##
### The Grand Scheme of Magellan ###
Magellan is a framework consisting of existing and new components that loosely coupled together:
* BLL (C++)
* STC UI (C#)
* Automation (Python)
* orion-res (Go)
* Web UI (HTML, Javascript).

Content developers are farmiliar with components such as BLL and STC UI, but not those new components which are developed with various tools or languages. Without the knowledge of how these pieces work together as a whole, debugging for Magellan can be difficult.

As shown in the image below, each component is running in its own process:

![processes](images/magellan_processes.png)

Component|Process (Windows)
---------|------------
STC UI | TestCenter.exe
BLL | TestCenterSession.exe
Orion-res | orion-res.exe
Web UI (embedded) | CefSharp.BrowserSubprocess.exe

Each component provides a set of functionalities related to Magellan:
* STC UI
    * Enable/disable Magellan (TestCenter IQ)
    * Brings up Embedded Web UI if Magellan is enabled
    * Result Selector that specify what results to subscribe for live results
* BLL
    * Brings up orion-res process during start up
    * Subscribe to IL for results (as before, but in a different way)
    * Write results to orion-res through Rest API
* Orion-res
    * Transfer data received from BLL to backend database service(PostgreSQL by default)
    * Respond to queries from Web UI
* Web UI
    * The visual part of Magellan to end users

![scheme](images/magellan_scheme.png)

The figure above summaries the communication between components:

* BLL subscribes to IL for results and send data to orion-res upon receiving. There is no communication between IL and orion-res.
* Communication between BLL and orion-res is always initiated by BLL. Data (test results, user information etc.) almost always flow from BLL to orion-res, except during apply, BLL may request to sync with orion-res (for test information ect.) 
* There’s no direct communication between Web UI and BLL, i.e., Web UI cannot send data to BLL, and vice versa.
* There’s no communication between STC UI and Web UI, although the former contains an embeded version of the latter. It's possible for STC UI to sync with Web UI (which is the case for test info). But it's not possible to retrieve data from STC UI in Web UI.
* STC UI/Automation client communicate with BLL through STAK commands. It can also query results from orion-res through REST Api as Python provides this capability.
* Orion-res pushes data (from BLL) to backend DB. It’s usually not a concern for content developers. But knowledge on how to work with the DBs is helpful for debugging.


### Roadmap ###
For migrating to Magellan, a recommended approach is described below. Each step is self-contained and serves as prerequisite for the next:
1. Start with BLL development, including dimension sets, result sets. Verify that data retrieved from IL is correctly populated in Postgres database.
2. Write view files so that web UI show the results correctly in embedded view and supported browser.
3. Add advanced features such as Heath indicator, chart view, report etc.

## BLL
BLL development is the start point for migrating an existing protocol to Magellan. Conceptually, dimension sets usually corresponding to a data-model class (e.g., *BGP_BgpRouterConfig*) and keep track of the changes to properties of interest. Result sets create queries (by utilizing related dimension sets) and subscribe to IL. When responses are received, result sets process the messages and write data to orion-res.

### Data Model Schema ###
A key difference between Magellan and classic results is how the data is stored in the backend (PostgreSQL) and consequently the query deployed by the front end (Web UI). Specifically, in Magellan, data storage is modeled with [**star schema**](https://www.vertabelo.com/blog/technical-articles/data-warehouse-modeling-the-star-schema). Orion-res provides [**query api**](https://github.com/SpirentOrion/orion-api/blob/master/res/doc/orion-res-query-api.md) for clients to retrieve data (private repo, check with orion team if you don't have access).

Most elements in star schema have a schema file which typically serves as the manual for authoring YAML files. Schema files rarely changes and is used for validation sometimes (e.g., for view generation). All schema files are located at:

`//TestCenter/integration/framework/bll/core/STAKCommands/results/schema/`

:exclamation:Whenever have doubts about YAML file, check out its schema file.

### File Locations ###
#### YAML Files ####
YAML files for dimension set and result set can be found in:
`//TestCenter/integration/framework/bll/core/STAKCommands/results/dim/`
`//TestCenter/integration/framework/bll/core/STAKCommands/results/res/`

As seen in these folders, about a dozen protocols have migrated to Magellan. Each protocol should create its own sub-directory, under which contains the YAML files. Result set YAML files should end with *_stats*.

:exclamation:These files are loaded when STC start up to create dynamic objects used by BLL later. When BLL is compiled, these files will be copied over to the debug folder. However, if you only modify the YAML file without changing BLL code, these files will not get updated. In this case, don’t forget to manually copy it to the debug folder:

`(STC_BUILD_ROOT)/bin/Debug/STAKCommandsspirent/results`

STC also needs to be restarted for the change to take effect.

#### c++ code ####
By convention, c++ files are located under:

`(content)/bll/src/magellan`

`(content)/bll/include/magellan`

`init.cpp` also need to be updated to register the dimension set and result set handlers.

### Dimension Set ###
The framework provides several dimension set classes to accommondate various use cases. Content developer needs to decide which class to use based on factors such as where the properties are savedXXX
#### Data-model based ####
#### IL driven ####
#### Special cases ####
streams, test
### Result Set ###
### BLL Debug Tips ###

#### 1. Sanity check ####
If live results are not showing, check Result Selector is set properly.

#### 2. Check DB Tables ####
Dimension and result sets work together to populate tables in backend db, so that Web UI can query upon. To determine if BLL is working properly, the quickest way is to check if the db is intact.
Each dimension set will create one table while each result set will have two. The name of the tables are determined by the name in the YAML file.

For example, for tx port dimension set, the name defined in tx_port.yaml is `tx_port`. So the corresponding table named is `db_id_tx_port` where `db_id` is a UUID automatically generated for each test.

For result sets, two tables are created: one for live results and one for end of test (EOT) results (i.e., snapshot). For example, in `tx_port_basic_stats.yaml`, the name is `tx_port_basic_stats`. Correspondingly, the EOT result table name is `db_id_tx_port_basic_stats`, and live result table name is `db_id_tx_port_basic_live_stats`.

Here are the steps to visually check the tables (on Windows):
1. Search for program ‘pgAdmin’ and run it. It should be installed along with STC.
2. Go to Databases/stc/Schemas/Tables as shown below: ![pgAdmin1](images/pgAdmin_1.png)
3. Get the db id from browser (embedded browser won’t work, use pop out one): ![pgAdmin2](images/pgAdmin_2.png)
4. Back to pgAdmin, Check the tables start with db id from last step.  Snapshots are written to tables ends with “_stats” (not “_live_stats”, those are for live results and will be empty if not checked in result selector),  e.g., “db_id_rx_port_basic_stats”, “db_id_rx_stream_stats”. ![pgAdmin3](images/pgAdmin_3.png)
5. Right click on the table of interest, the pop up menu have commands to count the rows, show rows etc. ![pgAdmin4](images/pgAdmin_4.png)

The columns in dimension tables are defined by the YAML files (attributes)
The columns in result set tables are defined by the YAML files and related dimension set. 

Common things to check:
Are dimension tables updated correctly when configuration changes? 


#### Check Chassis log for query ####
Magellan query has a different message name, use chassis log viewer, search magellan 
two ways when bll query results: notification based, pull (similar to drv)
## Views and Profiles
### File Locations
#### Generated view files (Json) ####
#### View data files (YAML) ####
#### Generator scripts ####
#### Tables in DB ####
view, profile table
### Standard view ###
### Drill down view ###
### Generator Scripts ###
* how to use generator script
* Facts ended with "rate" is special
### Profiles ###
:bulb: update testinfo class
### Debug Tips ###
* python script to check query
* json parser
* browser dev tool (for HI)
## Advanced Features ##
### Health Indicator ###
#### Basic Query ####
#### Link ####
#### Rate HI ####
### Chart view ###
### Report ###

