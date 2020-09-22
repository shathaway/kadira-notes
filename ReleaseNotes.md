# Release Notes - OSP Kadira APM

This GitHub project 
![shathaway/kadira-server](https://github.com/shathaway/kadira-server)
is forked from the GitHub
![kadira-open/kadira-server](https://github.com/kadira-open/kadira-server)
repository.

Signifcant updates to the code base are first being tested with in-house
deployment before submitting to the GitHub
![shathaway/kadira-server](https://github.com/shathaway/kadira-server)
repository.

The goal is to have a serviceable local hosting Kadira APM capability
that uses current Meteor and NodeJs packages. When we started our own
in-house work on Kadira APM, the code base was not maintained for over
three years. At time of our posting back to GitHub, the kadira-open
maintenance has been stale for over four years.

## Version 0.0.0
This is the initial *kadira-open/kadira-server* code base.

This code base is declared non-operational.

## Version 1.0.0
This is an operational kadira-server on a single CentOs-7 host.

This is an operational baseline system with local hosting.
GitHub packaging is available.

Work is being updated to the *v1.0.0* branch of
![shathaway/kadira-server](https://github.com/shathaway/kadira-server).
- Meteor 1.4.3.2
- NodeJs 4.8.x
- MongoDb 3.6 via mongodb@2.2.x drivers
- NGINX web application proxy server

Application Directories Supported
- kadira-engine
- kadira-rma
- kadira-ui

Test Suites (not validated)

Installation Guide is yet to be published.

## Version 2.0.0
This is an operational kadira-server with significant updates.

This is an operational updated system with local hosting.
GitHub packaging is work in progress.

Work is being updated to the *v2.0.0* branch of the
![shathaway/kadira-server](https://github.com/shathaway/kadira-server).
- Meteor 1.8.3
- NodeJs 8.17.0
- MongoDb 3.6 via mongodb@3.2.x drivers
- NGINX web application proxy server

Application Directories Supported
- kadira-engine (updated)
- kadira-rma (updated)
- kadira-ui (updated)
- packages - local
  - mocha-mongo (updated)
  - mongo-sharded-cluster (updated)
  - pick-mongo-primary (updated)

Test Suites
- kadira-engine/test updated and validated
- kadira-ui/tests (not usable)

**Version 3.0.0** (future) to contain current package releases,

Work is being updated to the *v3.0.0* branch of the **shathaway/kadira-server**.
- Meteor 1.11.0
- NodeJs 12.18.3
- MongoDb 4.2 and 4.4
- NGINX web application proxy server




