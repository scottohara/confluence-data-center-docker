### Stand up a 4-node Confluence Data Center cluster using Docker Compose

Acknowledgements
================
Most of this was inspired by Ilia Sadykov's talk from AtlasCamp 2018: [Testing DC/Multi-Product Atlassian Server Apps Made Easy](https://www.atlassian.com/atlascamp/watch-sessions/advanced-atlassian-development/testing-dcmulti-product-atlassian-server-apps-made-easy) 

Assumptions/Prerequisites
=========================
Data Centre requires a standalone shared database (you cannot use the embedded H2 database in a clustered setup).
In my case, I already had Postgres installed on the host machine, so I'm using that.

If you don't have an existing database, recommend adding an additional Postgres container to the `docker-compose.yml` file.
You can use [Ilia's JIRA example](https://bitbucket.org/iasadykov/atlascamp2018/src/e430e84dd567021e690b762e1521332cb8377a3c/samples/jiradc/docker-compose.yml?at=master&fileviewer=file-view-default) as a guide on how to do this.

You will also need a Confluence Data Centre license key. For testing/evaluation purposes, you can either request an evaluation license from Atlassian, or alternatively you can use a timebomb license such as [10 user Confluence Data Center license, expires in 3 hours](https://developer.atlassian.com/platform/marketplace/timebomb-licenses-for-testing-server-apps/).
Using a timebomb license means that you will need to restart your cluster every 3 hours (the license is valid for 3 hours from when the cluster was started).

Finally, you will (obviously) need Docker installed. The instructions below assume you are using Docker on Mac (which runs in a VM, so some of the commands to copy or set permissions on volumes have to be executed inside the VM).
If you are running Docker on another platform, you will need to adjust accordingly.

Each Confluence node requires approximately 2Gb of memory. By default, Docker on Mac is configured to use only 2Gb of memory in total; so click on the Docker logo in the menu bar and go to Preferences... > Advanced and increase the memory slider.
If you can't allocate enough memory to run all four nodes, you can either reduce the number of nodes (by removing some from the compose file), or alternatively you can try limiting the amount of resources each one uses as follows:

```yaml
services:
  confluence1:
    deploy:
      resources:
        limits:
          memory: 1024M
        reservations:
          memory: 1024M
```

If you don't have enough resources to run all 4 Confluence nodes on a single machine, you may want to consider running [Confluence Data Centre in Google Kubernetes Engine](https://github.com/scottohara/confluence-data-center-gke).

Overview
========
To get the cluster configured, the process is as follows:
1. Start the first Confluence node (only), and browse to it
2. Complete the first run configuration, including cluster & database setup
3. Stop the first Confluence node
4. Duplicate the contents of the node 1 home directory to the other nodes
5. Start all four Confluence nodes

First run setup
==============
1. Start postgres on the host machine (`postgres -D /usr/local/var/postgres`)
2. Create a new empty database, called "confluence" (`createdb confluence`)
3. For nodes 2-4, temporarily override their default command in `docker-compose.yml` so that they don't start (`command: /bin/true`)
4. Start the containers (`docker-compose up`)
5. Update permissions on the shared home volume (`docker run --rm -it -v /:/vm-root alpine:edge chmod -R 777 /vm-root/var/lib/docker/volumes/confluence-data-center_sharedHome`)
6. Browse to the initial confluence node (http://localhost:8090)
7. When prompted, set the cluster name (eg. "confluence-dc"), shared home path ("/var/atlassian/application-data/shared")
8. When prompted, set the db host ("host.docker.internal"), port (5432), database name ("confluence") and user
9. When prompted, create the admin user
10. Kill docker-compose (CTRL-C)

Additional nodes
================
11. Copy node 1 home to node 2 (`docker run --rm -it -v /:/vm-root alpine:edge cp -R /vm-root/var/lib/docker/volumes/confluence-data-center_confluence1Home/_data /vm-root/var/lib/docker/volumes/confluence-data-center_confluence2Home/`)
12. Repeat the above for nodes 3 & 4
12. Remove the command overrides (from step #3 above)
13. Start the containers (`docker-compose up`)