**OddEye Agent**
---------


OE-Agent3 is python3 metrics collection daemon for OddEye monitoring suite.

[OddEye](#oddeye)

We have developed OE-Agent to support number of other OpenSource backends :

[OpenTSDB](#opentsdb)

[KairosDB](#kairosdb)

[InfluxDB](#influxdb)

[Graphite](#graphite-carbon)


It uses REST and Pickle protocols with bulk uploads to talk to endpoints, 
so make sure your endpoint i configured properly. 

Main idea behind OddEye Agent is simplicity and less as possible dependencies, it is tested on Debian and Ubuntu systems, but should work on any Linux system.   

To install OddEye Agent just clone our repository. Base program requires only pycurl as external dependency. 
On Debian/Ubuntu you can install it via apt-get
 
    apt-get install python3-pycurl 
On RedHat systems:
 
    yum install python3-pycurl
Via python pip: 

    pip3 install pycurl

Some checks requires additional modules for example check_mysql requires MySQLdb. So make sure to install it before using MySQL check 
Debian/Ubuntu : 

    apt-get install python3-mysqldb

Python pip: 
    
    pip3 install MySQL-python
    
Make your changes if needed in config.ini and run 

    ./oddeye.sh start
Python daemon process will start, run all python scripts from checks_available directory as well as all check_* files scripts_available directory. 

### Main Config

OddEye client (OddEye Agent) uses simple ini files to configure main service and all checks. Configs are splitted into sections. Section [SelfConfig] contains base config parameters like checks interval, log/pid file location as well as some basic tags. 

    [SelfConfig]
    check_period_seconds = 5
    log_file = /var/log/oddeye.log
    pid_file = /var/run/oddeye.pid
    cluster_name = OddEye
    host_group = workers
    tmpdir = /tmp/oddeye_tmp

cluster_name and host_group are placeholders for tags for better manageability. 

In section [TSDB] you should set correct backend and uri. 

### Back End Config
To make it run you need to change **uuid** to one which you got during registration and start OddEye Agent, optionally change run user from oddeye.sh and start 

    ./oddeye.sh start 

OddEye server is native backend, but OddEye Agent can for with number of other open source backends. All configs are done at TSDB section of config.ini. Only one TSDB can be set at once, so make sure that all other are ether commented out or deleted from config file. 

#### **OddEye** 

    [TSDB]
    tsdtype = OddEye
    url = https://barlus.oddeye.co/oddeye-barlus/put/tsdb
    uuid = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    sandbox = False
    err_handler = 2

As OddEye Agent send metrics with small bulks you should enable chunked requests in opentsdb.conf

	tsd.http.request.enable_chunked = true

#### **OpenTSDB**

	[TSDB]
	tsdtype: OpenTSDB
	address: http://opentsdb_address:4242
	datapoints: /api/put
	user: netangels
	pass: bololo
	auth: False

OpenTSDB is designed to run in private networks and does not supports authentication, but if you want it to be public available, you can use any proxy server like Haproxy or NginX with basic auth enabled and configure credentials in  config.ini. If you do not need authentication, just set auth param to False and some placeholders as user/pass.  **Do not delete user/pass/auth parameters.**

#### **KairosDB** 
 
	[TSDB]
	address: http://kairosdb_address:8088
	datapoints: /api/v1/datapoints
	user: netangels
	pass: bololo
	auth: True
	tsdtype: KairosDB

Enable or disable auth: in accordance to your KairosDB setup 

#### **InfluxDB**

    [TSDB]
    address: http://influxdb_address:8086
    auth: False
    user: netangels
    pass: bololo
    database: test
    tsdtype: InfluxDB

Enable or disable authentication.
 
#### **Graphite Carbon** 

	[TSDB]
	address: carbon_host:2004
	user: netangels
	pass: bololo
	auth: false
	tsdtype: Carbon

OddEye Agent uses Carbon pickle, default port is 2004

OddEye Agent is completely stateless, so if you want to scale Backend, you can use any load balancing mechanism including DNS Round Robin.  

For all types of REST Backens (OpenTSDB, KairosDB, InfluxDB) config fields user/pass are mandatory even if you do not user authentication at backend.
So **Do not delete authentication parameters**,  just write something meaningless and use it as placeholder. 

### Configure modules

By default all checks are disabled . To enable check you need to create symlink  or copy check module from OE-AGENT_HOME/checks_available to OE-AGENT_HOME/checks_enable checks-available

    cd $OE-AGENT_HOME/checks_enabled
    ln -s ../checks_available/check_cpustats.py ./
    ../oddeye.sh restart 

Some of checks needs to be configured before you can use it, for example `check_nginx.py` needs to know NginX status url and username password if authentication on NginX status is enabled. All config files are located in conf directory. `conf/webservers.ini`is where `check_nginx.py` will look for configuration parameters. 

    [NginX]
    address: http://127.0.0.1:8888
    stats: /nginx_status
    auth: True
    user: netangels
    pass: bololo

Some checks depends on non standard python modules, like check_mysql.py depends on MySQL-python , so be sure to install all dependent modules before running checks. information about modules that should be installed before using checks are inside module files as comments. 

    head  checks-available/check_mysql.py 

    '''
    This check required Python MySQLDB, On Debian like systems do
    apt-get install python3-mysqldb
    or
    pip3 install MySQL-python
    '''

### Create own python module 

Create file in checks_enabled directory with name check_checkname.py, 
inside script import lib.basecheck and subclass Check inherited from lib.basecheck.CheckBase . 
in subclass you should have function precheck, where your actual check should leave. 
Example below demonstrates how you can send single metric with random value from 100 to 200 using custom module      

**Create scripts which which send metrics as it comes :**

```python
import lib.puylogger
import lib.basecheck
import random 

check_type = 'system'

class Check(lib.basecheck.CheckBase):
    def precheck(self):
        try:
            value = random.randint(100, 200)
            self.local_vars.append({'name': 'my_check_name', 'timestamp': self.timestamp, 'value': value, 'check_type': check_type})
        except Exception as e:
            lib.puylogger.print_message(__name__ + ' Error : ' + str(e))
            pass
```

This will import needed libs to generate and send to time series server needed json files, so you do not have to deal with generating and pushing it manually. 

**Create scripts which calculates value rates :** 

```python
import lib.puylogger
import lib.basecheck
import random 

check_type = 'system'

class Check(lib.basecheck.CheckBase):
    def precheck(self):
        try:
            value = random.randint(100, 200)
            rated = self.rate.record_value_rate('my_check_name', value, self.timestamp)
            self.local_vars.append({'name': 'my_check_name', 'timestamp': self.timestamp, 'value': rated, 'check_type': check_type})
        except Exception as e:
            lib.puylogger.print_message(__name__ + ' Error : ' + str(e))
            pass
```

### Create custom non python module 

To run custom script  like Bash, Perl etc.. Create scripts in format *check_name.extension* in folder scripts_enabled. 
All is needed from custom is to system out values in right order, Below is sample Bash scripts, which generates random number and send to collector for graphing: Make sure to have check_style parameter (stack/rate). This is for telling main program if it should calculate value rates or just push data as it comes. 

	#!/bin/bash
		
	myvalue=$RANDOM
	mytype=random_gen
	check_type=my_bash_random
	check_style=stack
	
	myvalue2=$RANDOM
	mytype2=random_gen2
	check_type2=my_bash_random2
	check_style2=rate
	
	echo $mytype $myvalue $check_type $check_style
	echo -n $mytype2 $myvalue2 $check_type2 $check_style2

### OddEye Specific 

As OddEye is completely push based and our servers does not have any direct access to your infrastructure we need special check which will determine if particular host is alive or not. Thus we made small module, which will call our servers and send response times as any other module do. We will generate host alive parameter based based on this. If you want to have host aliveness test just enable check_oddeye.py as you will do with any other python check:

    cd ${puypuy_home}/checks enabled
    ln -s ../checks-available/check_oddeye.py ./ 
    ../oddeye.sh restart 

OddEye is dynamic system based on machine learning, but if you want to have statically defined alerts you can use  `send_special` method in python module. Example below demonstrates how custom alerts can be configured its taken from check_load_average: 

```python
import lib.getconfig
import lib.pushdata
import lib.basecheck

warn_level = int(lib.getconfig.getparam('System Thresholds', 'load_high'))
crit_level = int(lib.getconfig.getparam('System Thresholds', 'load_severe'))
check_type = 'system'
reaction = -3


class Check(lib.basecheck.CheckBase):

    def precheck(self):
        cpucount = 0
        procstats = open("/proc/stat", "r")
        for line in procstats:
            if 'cpu' in line:
                cpucount += 1
        cpucount -= 1
        procstats.close()

        try:
            loadavg = open("/proc/loadavg", "r")
            proc_loadavg = loadavg.readline().split()
            curr_level = float(proc_loadavg[0]) * 100 / cpucount
            if curr_level < warn_level:
                health_value = 0
                err_type = 'OK'
                health_message = err_type + ': System Load average is at ' + str(curr_level) + ' percent of available  resources'
                self.jsondata.send_special("Load-Average", self.timestamp, health_value, health_message, err_type)
            if warn_level <= curr_level < crit_level:
                health_value = 8
                err_type = 'WARNING'
                health_message = err_type + ': System Load average is at ' + str(curr_level) + ' percent of available  resources'
                self.jsondata.send_special("Load-Average", self.timestamp, health_value, health_message, err_type)
            if curr_level >= crit_level:
                health_value = 16
                err_type = 'ERROR'
                health_message = err_type + ': System Load average is at ' + str(curr_level) + ' percent of available  resources'
                self.jsondata.send_special("Load-Average", self.timestamp, health_value, health_message, err_type)

            self.local_vars.append({'name': 'sys_load_1', 'timestamp': self.timestamp, 'value': proc_loadavg[0]})
            self.local_vars.append({'name': 'sys_load_5', 'timestamp': self.timestamp, 'value': proc_loadavg[1], 'reaction': reaction})
            self.local_vars.append({'name': 'sys_load_15', 'timestamp': self.timestamp, 'value': proc_loadavg[2], 'reaction': reaction})

            loadavg.close()
        except Exception as e:
            lib.pushdata.print_error(__name__ , (e))
            pass
```
You can disable or change dynamic alerting for particular checks, by passing `reaction` parameter to `jsondata.gen_data` . Its done this way : 

    reaction = -3 # (Disables dynamic alerting and learning on this check)
    reaction = -1 # (Disables dynamic alerting if values are smaller than expected)
    reaction = -2 # (Disables dynamic alerting if values are bigger than expected)
    reaction = 0  # (Default : Enable dynamic alerting and learning on this check)
    jsondata.gen_data(txname, timestamp, value, lib.pushdata.hostname, check_type, cluster_name, reaction)

We have created another **OddEye** specific optional parameter. This is to tell back-end type of incoming messages. It accepts "Rate" and "Counter" arguments.

**Rate**: is used to tell **OddEye** that incoming metrics are rated so back-end knows **better** how to calculate  dynamic rules.

**Counter**: Is set when we have increasing counter for metrics values. This is needed to do better calculation of regression, and to reset it when next value is smaller than previous . For doing this we have special reason, which may not feet your requirements, so if you find this not suitable for your needs, do not set parameter even when your metrics values are increasing counter, but keep in mind to manually drop regression when you counter resets. Otherwise regression counter will think that something happens as new values are out of expected regression scopes and will set high level of alerting to check. 


    jsondata.gen_data(txname, timestamp, value, lib.pushdata.hostname, check_type, cluster_name, reaction, 'Counter')

[Full documentation](https://www.oddeye.co/documentation/)
