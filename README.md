# AWSnycast

[![Build Status](https://travis-ci.org/bobtfish/AWSnycast.svg)](https://travis-ci.org/bobtfish/AWSnycast)

AWSnycast is a routing daemon for AWS route tables, to simulate an Anycast like service, and act as an
extension of in-datacenter Anycast. It can also be used to provide HA NAT service.

# NAT

A common pattern in AWS is to route egressing traffic (from private subnets) through a NAT instance
in a public subnet. AWSnycast can manage one or more NAT instances for you automatically, allowing
you to deploy one or more NAT instances per VPC.

You can deploy 2 NAT machines, in different availability zones, and configure a private routing table
for each AZ. AWSnycast can then be used to manage the routes to these NAT machines for High Availability,
such that by default, both machines will be active (for their AZ - to avoid cross AZ transfer costs + latency),
however if one machine fails, then the remaining machine will take over it's route for the duration that
it's down.

# Anycast in AWS?

In datacenters, a common pattern is to have a /24 network for Anycast, and then in each datacenter,
use systems like [exabgp](https://github.com/Exa-Networks/exabgp) and [bird](http://bird.network.cz/)
on hosts to publish BGP routing information for individual /32 IPs of services they're running.

In AWS we can configure VPN tunnels, or Direct Connects, and publish routes to the Amazon
routing tables using BGP, including the Anycast /24. AWSnycast then runs locally on your AWS nodes,
healthchecks local services, and publishes more specific routes to them into the AWS route table.

This means that all your systems in AWS can use *the same* network for Anycast services as your
in datacenter machines, *and* talk to services locally, as you bring them up in AWS. This is
super useful for bootstrapping a new VPC (before you have any local services running), or for
providing high availability.

You don't *have* to have a datacenter or VPN connection for AWSnycast to be useful to you, you
still get route publishing based on healthchecks, and HA/failover, just not bootstrapping
from in-datacenter.

N.B. Whilst publishing routes to *from* AWS into your datacenter's BGP would be useful, at this
time that is beyond the goals for this project.

# Trying it out

In the tests/integration folder, there is [Terraform](terraform.io) code which will build
a simple AWS VPC, with 2 AZs and 2 NAT machines (with HA and failover), and AWSnycast setup.

To try this, you can just run _make_ then _terraform apply_ in that directory to build
the example network, then _make sshnat_ to log into one of the machines,
and _journalctl -u awsnycast_ to view the logs.

Try terminating one of the machines and watch routes fail over!

Or try running:

    sudo iptables -I OUTPUT -d 8.8.8.8 -j DROP

to kill the healthcheck on one machine, and watch it drop the route and the other
machine pick it up.

# Installation

You need go installed to build this project (go 1.4 or 1.5). 

Once you have go installed, and a GOPATH setup, you should be able
to install with:

    go get github.com/bobtfish/AWSnycast
    go install github.com/bobtfish/AWSnycast

This will install the software to the bin directory in the first part of your GOPATH.

Run this to find it:

    echo $(echo $GOPATH|cut -d: -f1)/bin/AWSnycast

# Running it

You can run AWSnycast -h to get a list of helpful options:

    Usage of AWSnycast:
      -debug
            Enable debugging
      -f string
            Configration file (default "/etc/awsnycast.yaml")
      -noop
            Don't actually *do* anything, just print what would be done
      -oneshot
            Run route table manipulation exactly once, ignoring healthchecks, then exit

Once you've got it fully setup, you shouldn't need any options.

To run it also needs permissions to access the AWS API. This can be done either by
supplying the standard *AWS_ACCESS_KEY_ID* and *AWS_SECRET_ACCESS_KEY* environment
variables, or by applying an IAM Role to the instance running AWSnycast (recommended).

An example IAM Policy is shown below:

    {
      "Version": "2012-10-17",
      "Statement": [
            {
                "Action": [
                    "ec2:ReplaceRoute",
                    "ec2:CreateRoute",
                    "ec2:DescribeRouteTables"
                ],
                "Effect": "Allow",
                "Resource": "*"
            }
        ]
    }

# Configuration

Which routes to advertise into which route tables is configured with a YAML config file.

By default AWSnycast will look for this in /etc/awsnycast.yaml

An example config is shown below:

        ---
        healthchecks:
            public:
                type: ping
                destination: 8.8.8.8
                rise: 2  # How many need to succeed in a row to be up
                fall: 10 # How many need to fail in a row to be down
                every: 1 # How often, in seconds
            localservice:
                type: tcp
                destination: 192.168.1.1
                rise: 2
                fall: 2
                every: 30
                send: HEAD / HTTP/1.0 # String to send (optional)
                expect: 200 OK        # Response to get back (optional)
        routetables:
            # This is our AZ, so always try to takeover routes always
            a:
                find:
                    type: by_tag
                    config:
                        key: Name
                        value: private a
                manage_routes:
                  - cidr: 0.0.0.0/0     # NAT box, so manage the default route
                    instance: SELF
                    healthcheck: public
                  - cidr: 192.168.1.1/32 # Manage an AWSnycast service on this machine
                    instance: SELF
                    healthcheck: localservice
            # This is not our AZ, so only takeover routes only if they don't exist already, or
            # the instance serving them is dead (terminated or stopped).
            # Every backup AWSnycast instance should have if_unhealthy: true set for route tables
            # it is the backup server for, otherwise multiple instances can cause routes to flap
            b:
                find:
                    type: by_tag
                    config:
                        key: Name
                        value: private b
                manage_routes:
                  - cidr: 0.0.0.0/0
                    if_unhealthy: true # Note this is what causes routes only to be taken over not present currently, or the instance with them has failed
                    instance: SELF
                    healthcheck: public
                  - cidr: 192.168.1.1/32
                    if_unhealthy: true
                    instance: SELF
                    healthcheck: localservice

## Healthchecks

### ping

### tcp

## Route tables

### Finding them

### Managing them

# TODO

This project is currently under heavy development.

Here's a list of the features that I'm planning to work on next, in approximate order:

  * Making healthchecks actively notify routing tables on state change (so that changes happen more often than once every 5m)
  * Make us autodetect the VPC this instance is running in, and refuse to adjust routing tables in other VPCs
  * Make route table finding more flexible - be able to search by more than tag, and be able to interpolate current AZ
  * Make how often we poll AWS for route tables configurable
  * Enable the use of multiple different healthchecks for a route (to only consider it down if multiple checks fail)
  * Make how often we poll AWS for route tables flexible (max/min times, and backoff)
  * Implement 'negative' healthchecks, to be able to check the routes currently managed by other instances,
    and takeover faster than the other instance deleting it's route + polling time.
  * Add serf gossip between instances, to allow faster and reliable failover/STONITH
  * Add the ability to have external clients participate in healthchecks in the serf network.
  * Add a web interface to be able to get the state (and manually initiate failovers?)

# Contributing

Bug reports, success(or failure) stories, questions, suggestions, feature requests and (documentation or code) patches are all very welcome.

Please feel free to ping t0m on Freenode or @bobtfish on Twitter if you'd like help configuring/using/debugging/improving this software.

Please note however that this project is released with a Contributor Code of Conduct. By participating in this project you agree to abide by its terms.

# Copyright

Copyright Tomas Doran 2015

# License

Apache2 - See the included LICENSE file for more details.

