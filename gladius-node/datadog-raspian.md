# monitoring with datadog
## intro
This document shows how to integrate a gladius node into datadog.
The monitored device is based on a native raspberry pi installation - documented [here](./raspberry.md).

The datadog setup as described here is not production ready but good enough for the beta test ;-)

## Setup Datadog
Datadog is a monitoring as a service solution. It provides different agents to integrate all common operating systems.
The cool thing is that you can monitor up to 5 nodes for free. Therefore, datadog is a very good fit for the beta testers. 

### Create a datadog account
First you need to go to the [datadog homepage](https://www.datadoghq.com/) and create an account.

### Install the agent from github
The latest goagent version 6 must be installed via github. There is no automated installer for arm based systems yet

First we need to install go and build dependencies
```bash
# install deb packages
sudo apt-get install -y python-pip git sysstat

# install python dependencies
sudo pip install invoke
```

Next we need to install a current version of go
```bash
# install go 1.10+ 
# https://gist.github.com/simoncos/49463a8b781d63b5fb8a3b666e566bb5
cd /tmp
wget https://storage.googleapis.com/golang/go1.10.1.linux-armv6l.tar.gz
sudo tar -xzf go1.10.1.linux-armv6l.tar.gz -C /usr/local
rm go1.10.1.linux-armv6l.tar.gz
```

Before we can start the installation process we need to setup the PATH and GOPATH
```bash
# setup gopath
export GOPATH="/home/pi/go"
echo 'export GOPATH="/home/pi/go"' >> ~/.bashrc

# setup path with go 1.10+
export PATH=$GOPATH/bin:/usr/local/go/bin:$PATH
echo 'export PATH=$GOPATH/bin:/usr/local/go/bin:$PATH' >> ~/.bashrc

# create the gopath directory
mkdir $GOPATH
```

Now we can clone the datadog repository and build the agent
```bash
# clone the datadog repository
git clone https://github.com/DataDog/datadog-agent.git $GOPATH/src/github.com/DataDog/datadog-agent
cd $GOPATH/src/github.com/DataDog/datadog-agent

# install agent dependencies
invoke deps

# build the agent
# more details here: https://github.com/DataDog/datadog-agent/blob/master/docs/dev/agent_build.md
#invoke agent.build --build-exclude=snmp,jmx
invoke agent.build --build-include=gce,process
```
The installation will take 45 to 60 minutes.

Next we create a link to the datadog binary and setup the datadog configuration directories
```bash
# add a link to the agent binary in the PATH
sudo ln -s $GOPATH/src/github.com/DataDog/datadog-agent/bin/agent/agent /usr/local/bin/datadog-agent

# create datadog config directories
sudo mkdir -p /etc/datadog-agent/conf.d/process.d/

# copy the default configuration file
sudo cp $GOPATH/src/github.com/DataDog/datadog-agent/bin/agent/dist/datadog.yaml /etc/datadog-agent/
sudo chmod 600 /etc/datadog-agent/datadog-agent.yaml
```

For the datadog agent to work properly you need to specify the API key too.
First get your API key from the [API list in datadog](https://app.datadoghq.com/account/settings#api)
Then add the key to the configuration file
```bash
export KEY=ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
sudo sed -i "s/^api_key:.*$/api_key: $KEY/g" /etc/datadog-agent/datadog.yaml
``` 

*Attention:* The api key is not valid. You need to get your api key from the [API list in datadog](https://app.datadoghq.com/account/settings#api)

### Install service unit
To run the datadog agent as a service we need to create a systemd unit for it.

First we create the service definition.
```bash
sudo bash -c 'cat << EOF > /etc/systemd/system/datadog-agent.service
[Unit]
Description=datadog egent
After=network.target

[Service]
Environment="PATH=/usr/local/go/bin:/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
Type=simple
User=root
ExecStart=/usr/local/bin/datadog-agent -c /etc/datadog-agent/datadog.yaml start
WorkingDirectory=/home/pi/go/src/github.com/DataDog/datadog-agent
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF'
``` 

Next we need to tell the systemd about the new service
```bash
sudo systemctl daemon-reload
```

Enable the datadog agent on startup
```bash
sudo systemctl enable  datadog-agent
```

Now we can start the datadog agent service
```bash
sudo systemctl start datadog-agent
```

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
  - system.load.1 - host:raspberrypi
- Add Timeseries
  - system.mem.free - host:raspberrypi
  - system.mem.used - host:raspberrypi
  - system.swap.used - host:raspberrypi
- Add Timeseries
  - system.net.bytes_rcvd - host:raspberrypi
  - system.net.bytes_sent - host:raspberrypi
- Add Timeseries
  - system.processes.cpu.pct - host:raspberrypi, gladius-control
  - system.processes.cpu.pct - host:raspberrypi, gladius-edge
- Add Timeseries
  - system.processes.mem.pct - host:raspberrypi, gladius-control
  - system.processes.mem.pct - host:raspberrypi, gladius-edge

And thats it. You know have a small dashboard with the most important metrics.

## Next steps
Datadog provides a lot more functionality the we used in this document. 
For example we could stream all logs of the gladius services or setup triggers based on metric to get alerted if for example disk space is running out.
At the moment it is to early to invest a lot of time into detailed monitoring and alerting. The further the beta progresses the more we can invest into monitoring.

