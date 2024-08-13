# How to create graphs from runtime statistics 

**Outline:**
[TOC]

**Reference:**
[stats_monitor.py](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/ci-scripts/stats_monitor.py?ref_type=heads)

## Introduction

`stats_monitor.py` is a Python script from OAI designed for collecting, analyzing, and visualizing statistical data from 5G NR (New Radio) network components. It's primarily used for monitoring performance metrics of gNB (5G base stations) or eNB (4G base stations).

:::spoiler `stats_monitor.py` from OAI
```python=
"""
To create graphs and pickle from runtime statistics in L1,MAC,RRC,PDCP files
"""

import subprocess
import time
import shlex
import re
import sys
import pickle
import matplotlib.pyplot as plt
import numpy as np
import yaml
import os


class StatMonitor():
    def __init__(self,cfg_file):
        with open(cfg_file,'r') as file:
            self.d = yaml.load(file)
        for node in self.d:#so far we have enb or gnb as nodes
            for metric_l1 in self.d[node]: #first level of metric keys
                if metric_l1!="graph": #graph is a reserved word to configure graph paging, so it is disregarded
                    if self.d[node][metric_l1] is None:#first level is None -> create array
                        self.d[node][metric_l1]=[]
                    else: #first level is not None -> there is a second level -> create array
                        for metric_l2 in self.d[node][metric_l1]:
                            self.d[node][metric_l1][metric_l2]=[]                



    def process_gnb (self,node_type,output):
        for line in output:
            tmp=line.decode("utf-8")
            result=re.match(r'^.*\bdlsch_rounds\b ([0-9]+)\/([0-9]+).*\bdlsch_errors\b ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['dlsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['dlsch_err_perc_round_1'].append(percentage)
            result=re.match(r'^.*\bulsch_rounds\b ([0-9]+)\/([0-9]+).*\bulsch_errors\b ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['ulsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['ulsch_err_perc_round_1'].append(percentage)

            for k in self.d[node_type]['rt']:
                result=re.match(rf'^.*\b{k}\b:\s+([0-9\.]+) us;\s+([0-9]+);\s+([0-9\.]+) us;',tmp)
                if result is not None:
                    self.d[node_type]['rt'][k].append(float(result.group(3)))


    def process_enb (self,node_type,output):
        for line in output:
            tmp=line.decode("utf-8")
            result=re.match(r'^.*\bPHR\b ([0-9]+).+\bbler\b ([0-9]+\.[0-9]+).+\bmcsoff\b ([0-9]+).+\bmcs\b ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['PHR'].append(int(result.group(1)))
                self.d[node_type]['bler'].append(float(result.group(2)))
                self.d[node_type]['mcsoff'].append(int(result.group(3)))
                self.d[node_type]['mcs'].append(int(result.group(4)))


    def collect(self,testcase_id,node_type):
        if node_type=='enb':
            files = ["L1_stats.log", "MAC_stats.log", "PDCP_stats.log", "RRC_stats.log"]
        else: #'gnb'
            files = ["nrL1_stats.log", "nrMAC_stats.log", "nrPDCP_stats.log", "nrRRC_stats.log"]
        #append each file's contents to another file (prepended with CI-) for debug
        for f in files:
            if os.path.isfile(f):
                cmd = 'cat '+ f + ' >> CI-'+testcase_id+'-'+f
                subprocess.Popen(cmd,shell=True)  
        #join the files for further processing
        cmd='cat '
        for f in files:
            if os.path.isfile(f):
                cmd += f+' '
        process=subprocess.Popen(shlex.split(cmd), stdout=subprocess.PIPE)
        output = process.stdout.readlines()
        if node_type=='enb':
            self.process_enb(node_type,output)
        else: #'gnb'
            self.process_gnb(node_type,output)


    def graph(self,testcase_id, node_type):
        for page in self.d[node_type]['graph']:#work out a set a graphs per page
            col = 1
            figure, axis = plt.subplots(len(self.d[node_type]['graph'][page]), col ,figsize=(10, 10))
            i=0
            for m in self.d[node_type]['graph'][page]:#metric may refer to 1 level or 2 levels 
                metric_path=m.split('.')
                if len(metric_path)==1:#1 level
                    metric_l1=metric_path[0]
                    major_ticks = np.arange(0, len(self.d[node_type][metric_l1])+1, 1)
                    axis[i].set_xticks(major_ticks)
                    axis[i].set_xticklabels([])
                    axis[i].plot(self.d[node_type][metric_l1],marker='o')
                    axis[i].set_xlabel('time')
                    axis[i].set_ylabel(metric_l1)
                    axis[i].set_title(metric_l1)
                    
                else:#2 levels
                    metric_l1=metric_path[0]
                    metric_l2=metric_path[1]
                    major_ticks = np.arange(0, len(self.d[node_type][metric_l1][metric_l2])+1, 1)
                    axis[i].set_xticks(major_ticks)
                    axis[i].set_xticklabels([])
                    axis[i].plot(self.d[node_type][metric_l1][metric_l2],marker='o')
                    axis[i].set_xlabel('time')
                    axis[i].set_ylabel(metric_l2)
                    axis[i].set_title(metric_l2)
                i+=1                

            plt.tight_layout()
            #save as png
            plt.savefig(node_type+'_stats_monitor_'+testcase_id+'_'+page+'.png')


if __name__ == "__main__":

    cfg_filename = sys.argv[1] #yaml file as metrics config
    testcase_id = sys.argv[2] #test case id to name files accordingly, especially if we have several tests in a sequence
    node = sys.argv[3]#enb or gnb
    mon=StatMonitor(cfg_filename)

    #collecting stats when modem process is stopped
    CMD='ps aux | grep modem | grep -v grep'
    process=subprocess.Popen(CMD, shell=True, stdout=subprocess.PIPE)
    output = process.stdout.readlines()
    while len(output)!=0 :
        mon.collect(testcase_id,node)
        process=subprocess.Popen(CMD, shell=True, stdout=subprocess.PIPE)
        output = process.stdout.readlines()
        time.sleep(1)
    print('Process stopped')
    with open(node+'_stats_monitor.pickle', 'wb') as handle:
        pickle.dump(mon.d, handle, protocol=pickle.HIGHEST_PROTOCOL)
    mon.graph(testcase_id, node)

```

:::

## Features

-   Collects statistics from log files
-   Supports both gNB and eNB node types
-   Generates charts for performance metrics
-   Customizable monitoring metrics and chart configurations

## System Requirements

-   Python 3.x
-   matplotlib library
-   PyYAML library
-   NumPy library

## Configuration

The script uses a YAML configuration file to define metrics and chart layouts. 
- Example structure:

```yaml=
gnb :
  dlsch_err:
  dlsch_err_perc_round_1:
  ulsch_err:
  ulsch_err_perc_round_1:

  graph : 
    page1:
      dlsch_err:
      dlsch_err_perc_round_1:
      ulsch_err:
      ulsch_err_perc_round_1:
```

## Usage

### Modify Log file PATH 
- line 73: Modify to correct log path.
```python=69
def collect(self,testcase_id,node_type):
        if node_type=='enb':
            files = ["L1_stats.log", "MAC_stats.log", "PDCP_stats.log", "RRC_stats.log"]
        else: #'gnb'
            files = ["nrL1_stats.log", "nrMAC_stats.log", "nrPDCP_stats.log", "nrRRC_stats.log"]
```


### Run

:::warning
**The script will continue to collect data while the modem process is running. Make sure to start your modem process after running the script.**

- Run gNB
```bash
cd /home/oai72/FH_7.2_dev/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.273prb.fhi72.4x4-liteon.conf --sa --reorder-thread-disable 1 --thread-pool 1,3,5,7,9,11,13,15 > /home/oai72/Heidi_LOG/0812monitor/0812test.log 2>&1
```
:::

Run the script with the following command:

```bash=
python stats_monitor.py <config_file> <test_case_id> <node_type>
```

-   `<config_file>`: Path to the YAML configuration file.
-   `<test_case_id>`: Unique identifier for the test run.
-   `<node_type>`: Either 'gnb' or 'enb'.

Example:

```bash=
python stats_monitor.py stats_monitor_conf.yaml test_001 gnb
```

## Result
```bash=
╭─oai72@Dell-R740-Server ~/Heidi_Log/0812monitor 
╰─$ python stats_monitor.py stats_monitor_conf.yaml test_001 gnb                                              1 ↵
/home/oai72/Heidi_Log/0812monitor/stats_monitor.py:20: YAMLLoadWarning: calling yaml.load() without Loader=... is deprecated, as the default Loader is unsafe. Please read https://msg.pyyaml.org/load for full details.
  self.d = yaml.load(file)
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4365 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4380 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4395 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4410 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4410 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4425 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4440 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4455 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4471 lines of output
Starting collection for gnb
About to execute command: cat 0812test.log 
Waiting for process output...
Received 4486 lines of output
Process stopped
```
:::warning
**When modem process is stopped you will have log file and metrics pictures that you define.** 
- Example:
	- CI-test_001-nrL1_stats.log
	- CI-test_001-nrMAC_stats.log  
	- CI-test_001-nrRRC_stats.log
	- gnb\_stats\_monitor\_test\_001_page1.png  
	- gnb\_stats\_monitor\_test\_001_page2.png 
![image](https://hackmd.io/_uploads/S1Vtduu9R.png =50%x)
- gnb\_stats\_monitor\_test\_001_page1.png
![image]
:::
## Advanced Usage

### Custom Metrics

You can add new metrics in the YAML file under the appropriate node type (gnb or enb).

### Example: (Add UL/DL BLER and MCS metrices)
- File: `stats_monitor_conf.yaml`
- lines: 6~9, 28~31
```yaml=
gnb :
  dlsch_err_perc_round_1:
  ulsch_err_perc_round_1:
  dl_bler:
  ul_bler:
  dl_mcs:
  ul_mcs:

  graph : 
    page1:
      dlsch_err_perc_round_1:
      ulsch_err_perc_round_1:
    page2:
      dl_bler:
      ul_bler:
    page3:
      dl_mcs:
      ul_mcs:
```

- File: `stats_monitor.py`
- function: `process_gnb`

```python=
def process_gnb (self,node_type,output):
        for line in output:
            tmp=line.decode("utf-8")
            result=re.match(r'^.*\bdlsch_rounds\b ([0-9]+)\/([0-9]+).*\bdlsch_errors\b ([0-9]+).*\bBLER ([0-9]+\.[0-9]+) MCS \([0-9]+\) ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['dlsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['dlsch_err_perc_round_1'].append(percentage)
                dlbler = float(result.group(4))
                dlmcs = int(result.group(5))
                self.d[node_type]['dl_bler'].append(dlbler)
                self.d[node_type]['dl_mcs'].append(dlmcs)
            result=re.match(r'^.*\bulsch_rounds\b ([0-9]+)\/([0-9]+).*\bulsch_errors\b ([0-9]+).*\bBLER ([0-9]+\.[0-9]+) MCS \([0-9]+\) ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['ulsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['ulsch_err_perc_round_1'].append(percentage)
                ulbler = float(result.group(4))
                ulmcs = int(result.group(5))
                self.d[node_type]['ul_bler'].append(ulbler)
                self.d[node_type]['ul_mcs'].append(ulmcs)
        # ...
```

### Modifying Chart Layout

1. You can adjust the `graph` section in the `stats_monitor_conf.yaml` YAML file to change chart arrangements.
2. You can adjust the `graph` function in the `stats_monitor.py` to draw the figure.


## Appendix 
### Heidi Case
```python=
import subprocess
import time
import shlex
import re
import sys
import pickle
import matplotlib.pyplot as plt
import numpy as np
import yaml
import os



class StatMonitor():
    def __init__(self,cfg_file):
        with open(cfg_file,'r') as file:
            self.d = yaml.load(file)
        for node in self.d:#so far we have enb or gnb as nodes
            for metric_l1 in self.d[node]: #first level of metric keys
                if metric_l1!="graph": #graph is a reserved word to configure graph paging, so it is disregarded
                    if self.d[node][metric_l1] is None:#first level is None -> create array
                        self.d[node][metric_l1]=[]
                    else: #first level is not None -> there is a second level -> create array
                        for metric_l2 in self.d[node][metric_l1]:
                            self.d[node][metric_l1][metric_l2]=[]                



    def process_gnb (self,node_type,output):
        for line in output:
            tmp=line.decode("utf-8")
            #result=re.match(r'^.*\bdlsch_rounds\b ([0-9]+)\/([0-9]+).*\bdlsch_errors\b ([0-9]+)',tmp)
            result=re.match(r'^.*\bdlsch_rounds\b ([0-9]+)\/([0-9]+).*\bdlsch_errors\b ([0-9]+).*\bBLER ([0-9]+\.[0-9]+) MCS \([0-9]+\) ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['dlsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['dlsch_err_perc_round_1'].append(percentage)
                dlbler = float(result.group(4))
                dlmcs = int(result.group(5))
                self.d[node_type]['dl_bler'].append(dlbler)
                self.d[node_type]['dl_mcs'].append(dlmcs)
            #result=re.match(r'^.*\bulsch_rounds\b ([0-9]+)\/([0-9]+).*\bulsch_errors\b ([0-9]+)',tmp)
            result=re.match(r'^.*\bulsch_rounds\b ([0-9]+)\/([0-9]+).*\bulsch_errors\b ([0-9]+).*\bBLER ([0-9]+\.[0-9]+) MCS \([0-9]+\) ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['ulsch_err'].append(int(result.group(3)))
                percentage=float(result.group(2))/float(result.group(1))
                self.d[node_type]['ulsch_err_perc_round_1'].append(percentage)
                ulbler = float(result.group(4))
                ulmcs = int(result.group(5))
                self.d[node_type]['ul_bler'].append(ulbler)
                self.d[node_type]['ul_mcs'].append(ulmcs)


    def process_enb (self,node_type,output):
        for line in output:
            tmp=line.decode("utf-8")
            result=re.match(r'^.*\bPHR\b ([0-9]+).+\bbler\b ([0-9]+\.[0-9]+).+\bmcsoff\b ([0-9]+).+\bmcs\b ([0-9]+)',tmp)
            if result is not None:
                self.d[node_type]['PHR'].append(int(result.group(1)))
                self.d[node_type]['bler'].append(float(result.group(2)))
                self.d[node_type]['mcsoff'].append(int(result.group(3)))
                self.d[node_type]['mcs'].append(int(result.group(4)))


    def collect(self, testcase_id, node_type):
        LOG_DIRECTORY = "/home/oai72/FH_7.2_dev/openairinterface5g/cmake_targets/ran_build/build/"
        if node_type == 'enb':
            files = ["L1_stats.log", "MAC_stats.log", "PDCP_stats.log", "RRC_stats.log"]
        else:  # 'gnb'
            # files = ["nrL1_stats.log", "nrMAC_stats.log", "nrRRC_stats.log"]
            files = ["nrMAC_stats.log"]

        print(f"Starting collection for {node_type}")
        print(f"Looking for files in: {LOG_DIRECTORY}")

        existing_files = []
        for f in files:
            full_path = os.path.join(LOG_DIRECTORY, f)
            if os.path.isfile(full_path):
                existing_files.append(full_path)
                cmd = f'cat {full_path} >> CI-{testcase_id}-{f}'
                subprocess.Popen(cmd, shell=True)
                print(f"Appended {full_path} to CI-{testcase_id}-{f}")
            else:
                print(f"File not found: {full_path}")

        if not existing_files:
            print("No log files found!")
            return

        cmd = 'cat ' + ' '.join(existing_files)
        print("About to execute command:", cmd)
        process = subprocess.Popen(shlex.split(cmd), stdout=subprocess.PIPE)
        print("Waiting for process output...")
        output = process.stdout.readlines()
        print(f"Received {len(output)} lines of output")

        if node_type == 'enb':
            self.process_enb(node_type, output)
        else:  # 'gnb'
            self.process_gnb(node_type, output)


    def graph(self,testcase_id, node_type):
        for page in self.d[node_type]['graph']:#work out a set a graphs per page
            col = 1
            figure, axis = plt.subplots(len(self.d[node_type]['graph'][page]), col ,figsize=(10, 10))
            i=0
            for m in self.d[node_type]['graph'][page]:#metric may refer to 1 level or 2 levels 
                metric_path=m.split('.')
                if len(metric_path)==1:#1 level
                    metric_l1=metric_path[0]
                    major_ticks = np.arange(0, len(self.d[node_type][metric_l1])+1, 1)
                    axis[i].set_xticks(major_ticks)
                    axis[i].set_xticklabels([])
                    axis[i].plot(self.d[node_type][metric_l1],marker='o')
                    axis[i].set_xlabel('time')
                    axis[i].set_ylabel(metric_l1)
                    axis[i].set_title(metric_l1)
                    
                else:#2 levels
                    metric_l1=metric_path[0]
                    metric_l2=metric_path[1]
                    major_ticks = np.arange(0, len(self.d[node_type][metric_l1][metric_l2])+1, 1)
                    axis[i].set_xticks(major_ticks)
                    axis[i].set_xticklabels([])
                    axis[i].plot(self.d[node_type][metric_l1][metric_l2],marker='o')
                    axis[i].set_xlabel('time')
                    axis[i].set_ylabel(metric_l2)
                    axis[i].set_title(metric_l2)
                i+=1                

            plt.tight_layout()
            #save as png
            plt.savefig(node_type+'_stats_monitor_'+testcase_id+'_'+page+'.png')


if __name__ == "__main__":

    cfg_filename = sys.argv[1] #yaml file as metrics config
    testcase_id = sys.argv[2] #test case id to name files accordingly, especially if we have several tests in a sequence
    node = sys.argv[3]#enb or gnb
    mon=StatMonitor(cfg_filename)

    #collecting stats when modem process is stopped
    CMD='ps aux | grep modem | grep -v grep'
    process=subprocess.Popen(CMD, shell=True, stdout=subprocess.PIPE)
    output = process.stdout.readlines()
    while len(output)!=0 :
        mon.collect(testcase_id,node)
        process=subprocess.Popen(CMD, shell=True, stdout=subprocess.PIPE)
        output = process.stdout.readlines()
        time.sleep(1)
    print('Process stopped')
    with open(node+'_stats_monitor.pickle', 'wb') as handle:
        pickle.dump(mon.d, handle, protocol=pickle.HIGHEST_PROTOCOL)
    mon.graph(testcase_id, node)
```