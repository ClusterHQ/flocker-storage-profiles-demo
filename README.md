# Flocker Storage Profiles Demo

This is a simple demonstration of Flocker Storage Profiles on AWS using the EBS
driver for flocker with docker 1.9.0+ and flocker 1.7.0+.

*Status*: Work in progress. Would like review before making public.

*TODO*: Put asciinema cast of the demo here.

*TODO*: Add pretty pictures and directions for using the other fio
		encapsulation.

# Problem

Your storage provider offers a variety of different storage types, and you
would like to take advantage of that in your project. In your dev clusters
you'd like to use inexpensive slow magnetic disks, but in production, you'd
like to be able to use faster storage to back your databases.

For instance, on Amazon EBS you'd like to be able to take advantage of the
provisioned IOPS volumes within your Flocker cluster. Enter
[Flocker Storage Profiles](https://docs.clusterhq.com/en/latest/concepts/storage-profiles.html)
to assist in exposing these varied storage options to your cluster.

# Prerequisites

* The Unofficial Flocker Tools (Step 1 of
  https://docs.clusterhq.com/en/1.7.0/labs/installer-getstarted.html) installed
  on your local machine.
* An Amazon AWS account / credentials / ssh key. You could do a similar demo
  with another storage backend that has support for storage profiles with
  minimal modifications to the demo. However setting up such a cluster is left
  as an exercise for the reader.

# Demo

Note that this demo has some steps that are intended to be executed on your
`local $` machine, and some that are intended to be executed on a `node $` in
your cluster. The prompts will be labelled as such to assist in determining
where to run the commands.

## Setting up the cluster.

For starters we will set up a single node cluster on amazon using the automated
tools, basically following
https://docs.clusterhq.com/en/1.7.0/labs/installer-getstarted.html.

Get the sample files:
```
local $ uft-flocker-sample-files
```

Edit `terraform.tfvars` to have your amazon credentials. Also you can change
`agent_nodes = "2"` to be `agent_nodes = "1"` since we only need 1 node to
demonstrate profile support.
```
local $ cp terraform/terraform.tfvars.sample terraform/terraform.tfvars
local $ vim terraform/terraform.tfvars
```

Spin up the nodes:
```
local $ time uft-flocker-get-nodes --ubuntu-aws >& /dev/null && echo Done!
```

Install and configure flocker on the nodes:
```
uft-flocker-install cluster.yml && uft-flocker-config cluster.yml && uft-flocker-plugin-install cluster.yml
```

Inspect the nodes and the current volumes:
```
local $ uft-flocker-volumes list-nodes
local $ uft-flocker-volumes list
```

You may have noticed the public IP address of the node in the logs above. If
not, you can find the IP address from the name of some of the keys that were
generated in the configuration step:
```
local $ ls | grep plugin
```

Set an environment variable for use later:
```
IP=<ip address of node>
```

## Constructing the volumes with different profiles.

Create a new terminal and ssh into the node:
```
local $ ssh ubuntu@$IP
```

Drop into a superuser shell, or at least verify that you have privileges to run
`docker ps`, and that `docker` is at least version `1.9.0`.
```
node $ sudo -s
node $ docker --version
node $ docker ps
```

Create two volumes using docker with the flocker plugin for docker, specifying
the profile using volume options:
```
node $ docker volume create --name bronze-volume -o profile=bronze -d flocker
node $ docker volume create --name gold-volume -o profile=gold -d flocker
```

Back on our local machine, we can inspect the flocker cluster to observe that
the volumes are created, and they have a special metadata tag
`clusterhq:flocker:profile` which is handed to compatible drivers to create
volumes with a specified profile.
```
local $ uft-flocker-volumes list
```

## Verifying the different profiles.

Back on the node we'll run fio to test the speed of the volumes.
```
node $ docker run \
-e REMOTEFILES="https://gist.githubusercontent.com/sarum90/ec8b798c9f7e0fe9ac33/raw/fb251d5684bd03a0d969838267cb56fb71cb104f/randwrite.fio" \
-v /tmp/fio-data:/tmp/fio-data \
-v gold-volume:/gold \
-v bronze-volume:/bronze \
-e JOBFILES=randwrite.fio clusterhq/fio-tool
```

You can produce graphs of your test by running the following commands, the graphs will be in /tmp/fio-data on your `node`
```
node $ docker run -v /tmp/fio-data:/tmp/fio-data clusterhq/fio-genplots \
-t NameOfYourTestForGraphs -b -g -p *_bw*

node $ docker run -v /tmp/fio-data:/tmp/fio-data clusterhq/fio-genplots \
-t NameOfYourTestForGraphs -b -g -p *_iops*
```

If you want to serve you graphs using HTTP to view them in a browser run the following command. 
```
node $ docker run -p 80:8000 -d \
-v /tmp/fio-data:/tmp/fio-data \
clusterhq/fio-plotserve
```

You should be able to go to `http://node` to view your graphs. Here is an example.

*TODO* Put in pretty 2 line graph showing the difference between gold and bronze

*TODO* Show how to run all the above command in one using `clusterhq/fiotools_aio`

Scrolling up through the logs of the `fio` command you should see that the
bronze job had about 100-300 IOPS whereas the gold job had around 2250 iops.
This matches the expected IOPS for magnetic disks (Amazon docs say a burst to a
few hundred IOPS) and IOPS for a maximally provisioned 75 GiB volume on EBS (30
IOPS per GiB for 75 GiB is 2250 IOPS).

This ends the feature part of the demo, the rest is cleanup.

## Cleanup

Remove all containers on the node, specifically the fio one that was using the volumes.
```
node $ docker ps -a
node $ docker rm ids of the containers.
node $ docker ps -a # should see nothing
```

Remove the docker volumes:
```
node $ docker volume rm bronze-volume  gold-volume
```

No more work to be done on the node:
```
node $ exit
```

List the volumes in the flocker cluster:
```
local $ uft-flocker-volumes list
```

Flocker currently does not delete volumes just because docker volumes asks them
to be deleted, so we have to explicitly delete them using the flocker API:
```
local $ for D in <dataset-ids of volumes>
      > do
      > uft-flocker-volumes destroy --dataset=$D
      > done
```

Wait for the volumes to be deleted:
```
local $ uft-flocker-volumes list
```

Destroy the nodes:
```
local $ uft-flocker-destroy-nodes
```
