===========================================================================
Industry: Migrate a MySQL/MariaDB database to Open Telekom Cloud RDS with Apache NiFi
===========================================================================

`Apache NiFi <https://nifi.apache.org/>`__ (**nye-fye**) is a project from the Apache Software Foundation aiming
to automate the flow of data between systems in form of workflows. It supports powerful and scalable directed graphs
of data routing, transformation, and system mediation logic. We are going to utilize these features in order to
consolidate all the manual, mundane and error prone steps of an on-premises MySQL/MariaDb database migration
to Open Telekom Cloud RDS in a form of a highly scalable workflow.

.. note::
 Historically, NiFi is based on the "*NiagaraFiles*", a system that was developed by the US National Security Agency (NSA).
 It was open-sourced as a part of NSA's technology transfer program in 2014.

Overview
========

With zero cost in 3rd party components and in less than 15 minutes we are going to transform a highly error prone and
demanding use-case, as the migration of an MySQL or MariaDB to the cloud, to a fully automated, repeatable and scalable procedure.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/placeholder-image.jpg

.. image:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-etc.png
    :scale: 75
    :target: https://docs.otc.t-systems.com/en-us/usermanual/rds/en-us_topic_dashboard.html
    :alt: Relational Database Service (RDS)
.. image:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-etn.png
    :scale: 75
    :target: https://docs.otc.t-systems.com/en-us/usermanual/rds/en-us_topic_dashboard.html
.. image:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-f0m.png
    :scale: 75
    :target: https://docs.otc.t-systems.com/en-us/usermanual/rds/en-us_topic_dashboard.html
.. image:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-f0q.png
    :scale: 75
    :target: https://docs.otc.t-systems.com/en-us/usermanual/rds/en-us_topic_dashboard.html
.. image:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-f0u.png
    :scale: 75
    :target: https://docs.otc.t-systems.com/en-us/usermanual/rds/en-us_topic_dashboard.html

.. seealso::

   `Github Repository <https://www.youtube.com/watch?v=Phk-dP45QBI>`_

   `Apache Nifi Workflow Template <https://www.youtube.com/watch?v=PzBNkObWUXc>`_

Provision a MySQL instance in RDS
=================================

If you don't have an RDS instance in place, let's create one in order to demonstrace this use-case.
Under Relational Database Service in  Open Telekom Cloud Console,

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20221004-fw6.png

choose *Create DB Instance* and go through the creation wizard:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220926-9b5.png

1. Choose the basic details of your database engine. You need to stick for this use-case to MySQL engine v8.0.
Whether you create a single instance database or a replicated one with primary and standby instances is fairly
irrelevant in regards to our use-case.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220926-9c5.png

2. Create a new Security Group that will allow port 3306 in its inbound ports, and assign this Security Group
as Security Group of the ECS instances of your database (still in the database creation wizard)

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220926-9d4.png

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220926-9h7.png

3. After the database is successfully created, enable SSL support:

.. warning::
    It's not recommended transfering production data without having SSL enalbed

Provision an Apache Nifi Server
==============================

We are going to deploy the Apache NiFi server as a **docker container** using the following command
(replace first the required credentials with the ones of your choice):

.. code-block:: shell

    docker run --name nifi \
      -p 8443:8443 \
      -d \
      -e SINGLE_USER_CREDENTIALS_USERNAME={{USERNAME}} \
      -e SINGLE_USER_CREDENTIALS_PASSWORD={{PASSWORD}} \
      apache/nifi:latest

and then open your browser and navigate to the following URL address:

.. code-block:: shell

    https://localhost:8443/nifi/

enter your credentials and you will land on an empty workflow canvas:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-lt4.png

Create the migration workflow
============================

1. Add a **Processor** of type **GenerateFlowFile**, as the entry point of our workflow (as is instructed in the following picture):

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-lvz.png

2. Add a **Processor** of type **ExecuteStreamCommand**, as the step that will dump and export our source database — and call it ExportMysqlDump:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-m0k.png

and let’s configure the external command we want this component to execute:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-m2m.png

go to **Properties** from the tab menu:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-m44.png

As **Command Path** set :

.. code-block:: shell

    /usr/bin/mysqldump

and as **Command Arguments** fill in the mysql-client arguments, but separated by a semicolon
(replace the highlighted values with your own):

.. code-block:: shell

    -u;root;-P;3306;-h;{{HOSTNAME_OR_CONTAINER_IP}};-p{{PASSWORD}};
    --databases;employees;--routines;--triggers;--single-transaction;
    --order-by-primary;--gtid;--force

Connect the two Processors by dragging a connector line from the first to the latter.
You should be able to observe now that a **Queue** component is injected between them:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-m8y.png

We will see later how these Queues contribute to the workflow and how we can use them
to gain useful insights or debug our workflows.

3. Open Telekom Cloud RDS for MySql will **not** permit SUPER privileges or the SET_USER_ID privilege to any user,
and this will lead to the following error when you will try to run the migration workflow for the first time:

.. code-block:: shell

    ERROR 1227 (42000) at line 295: Access denied;
    you need (at least one of) the SUPER or SET_USER_ID privilege(s) for this operation

The error above may occur while executing CREATE VIEW, FUNCTION, PROCEDURE, TRIGGER OR EVENT with DEFINER statements
as part of importing a dump file or running a script. In order to preactively mitigate this situation, we are going to add
a second **Processor** of type **ExecuteStreamCommand**. This Processor (let’s call it ReplaceDefinersCommand)
will edit the dump file script and replace the DEFINER values with the appropriate user with admin permissions
who is going to perform the import or execute the script file.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-ni2.png

As **Command Path** set :

.. code-block:: shell

    sed

and as **Command Arguments** (*in one line*):

.. code-block:: shell

    -e;"s/DEFINER[ ]*=[ ]*[^*]*\*/\*/";
    -e;"s/DEFINER[ ]*=.*FUNCTION/FUNCTION/";
    -e;"s/DEFINER[ ]*=.*PROCEDURE/PROCEDURE/";
    -e;"s/DEFINER[ ]*=.*TRIGGER/TRIGGER/";
    -e;"s/DEFINER[ ]*=.*EVENT/EVENT/"

Connect the two ExecuteCommandStream Processors, by dragging a connector line from the first to the second.
You should be able to observe now that a second Queue component is added between them on the canvas.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-ngs.png

4. Add a third **Processor** of type **ExecuteStreamCommand** (same drill as with ExportMysqlDump).
This step will import the dump to our target database — call it ImportMysqlDump. Let’s configure it:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220810-mf6.png

As **Command Path** set :

.. code-block:: shell

    /usr/bin/mysql

and as **Command Arguments** (*in one line*):

.. code-block:: shell

    -u;root;-P;3306;-h;{{EIP}};-p{{PASSWORD}};--ssl-ca;/usr/bin/ca-bundle.pem;--force

Connect the ReplaceDefinersCommand with this new Processor, by dragging a connector line from the first to the second.
You should be able to observe now that a second Queue component is added between them on the canvas:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-nfj.png

5. Add a **Processor** of type **LogAttribute**; this component will emit attributes of the FlowFile for a predefined log level.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-dsr.png

Then drag a connection between the ExportMysqlDump and the LogAttribute Processors, and in the Create Connection popup
let’s define two new relationships: *original* and *nonzero status*. The former is the original queue message that was
processed from the Processor and the latter bears the potential errors (*non zero results*) that were thrown during
this step of the workflow. Every relationship will inject a dedicated queue in the workflow. Repeat the same steps for
the ReplaceDefinersCommand Processor. For ImportMySqlDump and LogAttribute Processors, activate all 3 available relationship options.
The output stream will log the successful results of our import workflow step.

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-dum.png

Eventually, our LogAttribute Processor and its dependencies should now look like this on the canvas:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-nk1.png

6. Start the Processors. As you will notice on the left-hand upper corner of every Processor on the canvas appears a stop sign.
That means that the Processors will not execute any commands even if we kick off a new instance of the workflow.
In order to start them press, for every single one of them — except LogAttribute, the start button marked with blue in the picture below:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-e7c.png

Configure the Apache Nifi Server
==============================

At this point we are not ready yet to run our workflow. The Apache Nifi server is lacking two additional resources.
The two ExecuteStreamCommand Processors will execute an export and import from and to remote MySQL instances using
the mysql-client, but the Apache NiFi container doesn’t have any knowledge of this package. We have to connect to our
container and install the required client.

Let's connect first to the Apache Nifi container as root:

.. code-block:: shell

    docker exec -it -u 0 nifi /bin/bash

and install the client (in this case is the *mariadb-client* package):

.. code-block:: shell

    apt-get update -y
    apt-get install -y mariadb-client

A quick sanity check to make sure that everything is in place. For that matter go to `/usr/bin/` and make sure you
that `mysqldump` and `mysql` are properly symlinked:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-eii.png

Next we have to copy to the Apache Nifi container the SSL certificate we downloaded from the Open Telekom Cloud console.

.. code-block:: shell

    docker cp ca-bundle.pem nifi:/usr/bin

.. attention::
    For the time being, let's skip the step above in order to simulate an error in the migration workflow and we will
    come back later to this.

Start a Migration Workflow
=========================

Open the cascading menu of the *GenerateFlowFile* component and click *Run Once*:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-f0t.png

The current active Processor will be marked with this sign on right-hand upper corner on the canvas:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-f32.png

Let’s see what happened and if the migration went through, and if no how could we debug and trace the source of our problem.
The canvas now will be updated with some more data in every *Processor* and *Queue*:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-nsn.png

*GenerateFlowFile* Processor is informing us that has sent 1 request down the pipeline (*Out* 1 — in box marked in blue).
The *ExecuteMysqlDump* Processor ran successfully and wrote out a dump in the size of 160.59MB. Its logging queues show
us that we have a new entry in *original* and zero entries in *nonzero status*. (The latter indicates that the Processor ran **without any error**).
Let’s see what was written in the original queue. Open the queue:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-fap.png

and under the *Properties* tab of the Queue, we can see which command was executed by our Processor:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-fc21.png

Now let's focus on the second ExecuteStreamCommand Processor, the one that is responsible to import the dump to the target database.
We can see that it received an input of 160.59MB (that is our dump file, generated from the previous Processor);
it pushed it down in the *original* queue but it seems that migration didn’t go through as planned,
because we have items in the *nonzero status* queue. As a first step finding the culprit, we will inspect in the original queue
(open the *List Queue* and pick the element that corresponds to this very workflow instance under the *Details* tab).
We can either inspect the generated dump file that was handed over by the ExportMysqlDump Processor by either viewing or download it,

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-fhz.png

or inspect the command that was executed to see if there is a helpful error message (in our case there is one):

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-fhm1.png

A faster way though, figuring out what went wrong, is hovering over the red sign (that will appear in case of error)
in the upper right-hand corner of our Processor that threw the error:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220812-flv.png

Now that we saw how we can, in principle, debug and investigate errors during the execution of our workflows, go back
to previous chapter guidelines and, this time, do copy the SSL certificate to the Apache Nifi container.

We are now set to start a new migration instance. You will observe that after a while the *ImportMysqlDump* Processor goes
in execution mode, for the small sign on the right upper-hand corner that indicates the active threads currently running
on this component. After a while, when the workflow will:

* not have any more active threads in any processor
* have an additional message in the outcome queue of the ImportMysqlDump Processor
* have no additional messages in the nonzero status queue of the ImportMysqlDump Processor

then check your database — the migration would have successfully completed:

.. figure:: https://architecture-center-poc-images.obs.eu-de.otc.t-systems.com/rds-migration/SCR-20220926-bhx.png

References
==========

.. seealso::

   `Relational Database Service: Accessing RDS <https://www.youtube.com/watch?v=Phk-dP45QBI>`_

   `Database Services Overview with RDS Deep Dive <https://www.youtube.com/watch?v=PzBNkObWUXc>`_