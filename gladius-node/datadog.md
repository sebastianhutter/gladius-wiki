# monitoring with datadog
## intro
This document shows how to integrate a gladius node into datadog.
The monitored device is based on a native raspberry pi installation - documented [here](./raspberry.md).

## Setup Datadog
Datadog is a monitoring as a service solution. It provides different agents to integrate all common operating systems.
The cool thing is that you can monitor up to 5 nodes for free. Therefore, datadog is a very good fit for the beta testers.

### Create a datadog account
First you need to go to the [datadog homepage](https://www.datadoghq.com/) and create an account.

### Install the debian agent
Raspbian is a debian based distribution. This allows us to install the debian agent on the pi.
Every datadog agent installation needs a `datadog api key` set to work. This api key is different for every datadog account.

Navigate to the [debian agents page](https://app.datadoghq.com/account/settings#agent/debian) to get the comamnd to install the agent with your api key. 

```bash
DD_API_KEY=ABCDEFGHIJKLMNOPQRSTUVWXYZ012345 bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh)"
```

*Attention:* The api key is not valid. You need to get your api key from the [datadog agents page](https://app.datadoghq.com/account/settings#agent/debian)

### Check installation
First check if the datadog agent is running as expected with systemctl
```bash
sudo systemctl status datadog-agent
```

After a minute or two you will see the raspberry pi listed in the [infrastructure list](https://app.datadoghq.com/infrastructure)

We have datadog installed now and can now look into monitoring the gladius processes and the host itself.

## Monitoring gladius
### enable process monitoring
By default the datadog agent does not collect process stats. To gather useful data we need to enable the process collector for the agent.

You can find more information in the [live process monitoring](https://docs.datadoghq.com/graphing/infrastructure/process/) guide of datadog.

```bash
# add process config parameter to datadog configuration file
sudo bash -c 'cat << EOF >> /etc/datadog-agent/datadog.yaml
process_config:
  enabled: "true"
EOF'

# restart the datadog agent
sudo systemctl restart datadog-agent
```

### gather metrics for gladius processes
Next we need to tell the datadog agent to monitor the two gladius nodejs processes and gather metrics for them.
You can find more details about process monitoring in the [datadog documentation](https://docs.datadoghq.com/integrations/process/).

```
# add the processes to monitor
sudo bash -c 'cat << EOF > /etc/datadog-agent/conf.d/process.d/gladius.yaml
init_config:

instances:
  - name: gladius-edge
    search_string: ['gladius-edge']
    exact_match: false
  - name: gladius-control
    search_string: ['gladius-control']
    exact_match: false
EOF'

# restart the datadog agent
sudo systemctl restart datadog-agent
```

### Dashboard
With the process monitoring enabled its time to create the first dashboard to make monitoring the node and the gladius processes easy.
Datadog knows two different types of dashboards. For the moment the TimeBoard is what we need - it allows us to display multiple metrics synchronized by time.

The dashboard we create will show cpu / memory usage of the gladius processes, basic metrics of the node and incoming / outgoing network traffic

- Navigate to the [dashbord lists](https://app.datadoghq.com/dashboard/lists) and hit 'New Dashboard'.
- Choose 'New TimeBoard'
- Add Timeseries
  - system.load.1 - host:raspberry
- Add Timeseries
  - system.mem.free - host:raspberry
  - system.mem.used - host:raspberry
  - system.swap.used - host:raspberry
- Add Timeseries
  - system.net.bytes_rcvd - host:raspberry
  - system.net.bytes_sent - host:raspberry
- Add Timeseries
  - system.processes.cpu.pct - host:raspberry, gladius-control
  - system.processes.cpu.pct - host:raspberry, gladius-edge
- Add Timeseries
  - system.processes.mem.pct - host:raspberry, gladius-control
  - system.processes.mem.pct - host:raspberry, gladius-edge

## Next steps
Datadog provides a lot more functionality the we used in this document. 
For example we could stream all logs of the gladius services or setup triggers based on metric to get alerted if for example disk space is running out.
At the moment it is to early to invest a lot of time into detailed monitoring and alerting. The further the beta progresses the more we can invest into monitoring.

