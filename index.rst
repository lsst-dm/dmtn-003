..
  Content of technical report.

  See http://docs.lsst.codes/en/latest/development/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label
     :target: http://target.link/url

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-report-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

Description of Alert Production Simulator v1.0
==============================================
This document describes the current state of the Alert Production Simulator, which was written to test the workflow described in LDM-230, revision 1.2 dated Oct 10, 2013.   

The system consists of twenty-five virtual machines simulating over two hundred machines, simulating the workflow required for one thread of processing.   Please note that image archiving, science image processing, catch-up archiver, and EFD replicator are not implemented in this simulator.

Operations
----------

Systems
^^^^^^^

The AP simulator runs on VMware virtual machines in the following configuration:
 
- lsst-base - main Base DMCS system
- lsst-base2 - failover Base DMCS system
 
- lsst-rep - replicator HTCondor master node
- lsst-rep1 - HTCondor replicator job execution node, running 11 slots
- lsst-rep2 - HTCondor replicator job execution node, running 11 slots
 
- lsst-dist - distributor node, running 22 distributor node processes
 
- lsst-work - worker HTCondor master node
- lsst-run1 through lsst-run14 (14 virtual machines) - each configured with 13 slots
- lsst-run15 - configured with 7 slots.
 
- lsst-archive - main Archive DMCS system
- lsst-archive2 - failover Archive DMCS system
 
- lsst-ocs - LSST OCS system - sends simulated DDS events, "startIntegration", "nextVisit", and "startReadout";  also runs simulated file server for camera readout.

Software packages
^^^^^^^^^^^^^^^^^

The software packages:
 
- ctrl_ap
- htcondor (this is not currently integrated in the LSST stack, but contains the python software install on the systems lsst-base and lsst-base2)
 
must be setup on each of the following machines:
 
- lsst-base
- lsst-base2
- lsst-archive
- lsst-archive2
- lsst-rep
- lsst-dist
- lsst-work
- lsst-ocs
 
After setup, execute the following commands on the following machines:

- lsst-base - baseDMCS.py - starts the base DMCS master (via config)
- lsst-base2 - baseDMCS.py - starts the base DMCS failover (via config)
- lsst-archive - archiveDMCS.py - starts the Archive DMCS master (via config)
- lsst-archive2 - archiveDMCS.py - starts the Archive DMCS failover (via config)
- lsst-rep - launchReplicators.sh - starts the replicator node processes on lsst-rep1 and lsst-rep2 (22 processes, 11 on each system)
- lsst-dist - launchDistributors.sh - starts the distributor node processes on lsst-dist (22 processes)
- lsst-ocs - ocsFileNode.py - starts a server the delivers files to replicator jobs that request them.

There are two HTCondor pools.  The Replicator pool is set up with lsst-rep as the master node, and lsst-rep1 and lsst-rep2 containing  11 HTCondor slots each.   The Worker pool is setup with lsst-work as the master node, and lsst-run1 through lsst-run15 running 189 HTCondor slots in total.

Workflow
--------

After all the services are configured, an external command is used to send commands as if they had been sent from the OCS, and images pulled from a simulated CDS service through to the entire workflow through worker jobs. 

Events are sent from the OCS system to the Base site and trigger specific operations.   There are three event types that the OCS system sends:  nextVisit, startIntegration, and startReadout.  

The nextVisit event starts worker jobs on the HTCondor Worker pool.  

The startIntegration event starts replicator jobs on the HTCondor Replicator pool.   

The startReadout event triggers the replicator jobs to make a CDS library call to read rafts.

Replicator jobs send each raft to it’s pair distributor.  Each distributor breaks up the raft into CCDs.   Worker jobs rendezvous with the distributors and process each CCD individually.

This entire process is described  in greater detail below:

 
#. A “nextVisit” event is sent from the OCS system, lsst-ocs.
#. The message is received by the Base DMCS system, lsst-base.
#. The Base DMCS system submits 189 worker jobs (with the information about CCD image they’re looking for) to the worker master node, lsst-work, which controls the worker pool.
#. The worker jobs begin to execute on slots available on the HTCondor worker cluster.
#. The worker jobs subscribe to the OCS “startReadout” event.
#. The worker jobs contact the Archive DMCS to ask which distributor has the image they’re looking for.
#. A “startIntegration” event is sent from the OCS system.
#. The Base DMCS system submits 21 replicator jobs (containing the visit id, exposure sequence number of the RAFT they’re looking for) to the replicator master node, lsst-rep, which controls the replicator pool.
#. The replicator jobs begin to execute on the HTCondor replicator cluster.
#. The replicator jobs send the visit id, exposure sequence number and the raft to the replicator node process, which passed it to the paired distributor.
#. The distributor sends a messages to the Archive DMCS telling it which CCDs it will be handling for this RAFT.
#. The workers waiting for the distributor information from the Archive DMCS are told which distributors to rendezvous with when that information arrives.
#. The workers connect to the distributor with their image information, and block until the image is received.
#. A “startReadout” event is sent from the OCS system.
#. The replicator jobs receive the startReadout event from the OCS system
#. The replicator jobs contact the OCS system to read the cross-talk corrected image.
#. The replicator jobs send the image data to the distributor node processes and exit.
#. The distributor node processes break up the raft images into ccd images
#. The distributor send the image to the worker job.
#. Steps 5 through 14 are repeated for a second exposure, and then move on to Step 18
#. The worker jobs process the images they receive and exit.


Status
^^^^^^

Components transmit their activity as status via the ctrl_event messages.   Each status message includes the source of the message, the activity type, and an optional free form message, in addition to other standard event data (publish time, host, etc). Each component in the system uses a subset of activity types.   When any of these activities occurs a status messages is sent.   Other programs have been written to capture this status information, and we've used it to drive external animations of the overall AP simulator activity.   An example of this can be seen here: https://lsst-web.ncsa.illinois.edu/~srp/alert/alert.html

Components
----------

OCS
^^^

The OCS system, lsst-ocs, is used to run a simulated CDS server that delivers images to Replicator Jobs.   The OCS events (nextVisit, startIntegration, startReadout) are sent from this system.
 
We use the commands ocsTransmitter.py and automate.py to send OCS events to the Base DMCS.   The ocsTransmitter.py command sends the specified event for each command invocation.  The automate.py command can send groups of events at a specific cadence.
We use the ocsFile.py server process as the simulated CDS server that delivers images.
 
**Notes**:  We created the ocsFile.py server process as a substitute for the CDS library call that will be made by the replicator jobs to retrieve images from the CDS.  We do not know what mechanisms will be used to server or deliver the images, other that it is through a library call.
 
The OCS commands are sent via the ctrl_events ActiveMQ message system, not through the DDS system.

Base DMCS
^^^^^^^^^

The Base DMCS receives messages from the OCS and controls the job submission to the replicator cluster and the worker cluster.

Two Base DMCS processes are started, one on primary system and one on a secondary system.  These can be started in any order.  On start up, the processes contact each other to negotiate the role each has, either “main” or ‘failover”.  Both processes subscribe and receive messages from the simulated OCS, but only the the process currently designated as “main” acts on the messages.  A heartbeat thread is maintained by both processes.  If the failover process detects that the main process is no longer alive, it’s role switches to main.  When the process that had been designated as main returns, it’s role is now reassigned to “failover”.   The following describes the actions of the Base DMCS in the “main” role.

OCS messages are implemented as DM software events, since the OCS DDS library was not available when the simulator was written.  The Base DMCS process subscribes to one topic, and receives all events on that topic.   It responds only to the “startIntegration” and “nextVisit” events, and submits jobs to the appropriate HTCondor pool.   These jobs are submitted via the Python API that HTCondor software provides.
On receipt of the “nextVisit” event, the Base DMCS submits 189 Worker jobs and 4 Wavefront jobs to the Worker master node.   Each job is given the visit id, number of exposures to be taken, boresight pointing, filter id, and CCD id.  The jobs are place holders for the real scientific code, and are described here.
On receipt of the “startIntegration” event, the Base DMCS submits 22 replicator jobs, one for each raft, and a single job for the wavefront sensors.  The replicator jobs are described here.

**Notes**: Both the replicator jobs and worker jobs are submitted to their respective master nodes, and have no mechanism for monitoring their progress.  In data challenges, work was submitted via HTCondor’s DAGman, which provided a mechanism for resubmitting jobs that failed automatically for a configured number of times.  Furthermore, it provided a resubmission DAG for jobs that completed failed after that set number of times.   HTCondor itself does not resubmit failed jobs automatically;  it will, however resubmit a job if a HTCondor job slot in which it was running has an error of some kind.
None of this is desired behavior.   We need to monitor job progress, success and failure. Resubmitting a job on failure of software or hardware to run given the time constraints is not feasible.  Using DAGman to submit files would require us to keep track of the resubmit files, which seems like overkill.   We need a mechanism that logs success and the reason for failure so that we can take appropriate action.

Replicator Master Node
^^^^^^^^^^^^^^^^^^^^^^

The HTCondor master node for the Replicator pool is configured on lsst-rep.   This system acts as the HTCondor master for two VMs, lsst-rep1 and lsst-rep2, which are configured with 2 CPUs and 4 gig of memory each.   HTCondor is configured on lsst-rep1 and lsst-rep2 to have 11 job slots.
 
This master node accepts job submissions from lsst-base, and runs those jobs on lsst-rep1 and lsst-rep2.

Replicator Execution Node
^^^^^^^^^^^^^^^^^^^^^^^^^

A replicator node is part of the HTCondor pool, which is controlled by the HTCondor master.    Each node accepts replicator Jobs scheduled by the HTCondor master.   There are 22 worker nodes, one for each raft (including wavefront).

**Notes**:  Due to limitations capacity at the time the simulator was written, this was simulated across 2 VMs.   Each VM was configured with 2 CPUs, and 4 gig of memory.   HTCondor will ordinarily make the number of slots for jobs equal to the number of CPUs, but we overrode this to configure 11 slots per for each VM.    We noted varying startup times from the time the job was submitted to the HTCondor master.  The pool was configured to retain ownership of the slot, which increased the speed at which jobs were matched to slots. In general, the start up was very quick, but there were times when we noted start up times of between 15 and 30 seconds.   It was difficult to determine why exactly this occurred, but given the limited capacity of the VMs themselves, we believe this is a contributing factor.

Replicator Node Process
^^^^^^^^^^^^^^^^^^^^^^^

The Replicator Node Process receives messages and data from the replicator job, and transmits this information to its paired Distributor Node process.
 
Twenty-two replicator node processes are started, eleven each on lsst-rep1 and lsst-rep2.  The processes are started with a paired distributor address and port to connect to.  The distributor node process does not have to be running at the time the replicator node process starts because it will continue to attempt a connection until it is successful.
 
Once successful, a heartbeat thread is started which sends messages to the paired distributor node heartbeat thread at regular intervals in order to monitor the health of the connection.  If this connection fails, the process attempts to connect to its paired distributor until it succeeds.
 
A network server port is also opened for connections from a Replicator Job process.   When the Replicator Job process connects it sends the Replicator Node process the visit id, exposure id, and raft id it was assigned.  The Replicator Node process sends this information to the paired distributor.  The Replicator Node process then waits to receive the crosstalk-corrected image location from the replicator job process.   When this is received, the file is transferred to the paired distributor.   The connection to the Replicator Job process is closed (since the job dies at this point), and the cycle repeats.
 
**Notes**: The replicator node process only handles transmitting to the paired distributor.  It needs to handle the case where the connection to the distributor is down or interrupted.

The replicator node process starts knowing which distributor process to connect to via a command line argument.   We should look into using a configuration manager which the replicator node process could contact to retrieve the host/port of the distributor it should connect to.  This would make the system more robust if nodes go down and need to be replaced.

Replicator Job
^^^^^^^^^^^^^^

Replicator Jobs are submitted by the Base DMCS process to be executed on the Replicator Node HTCondor pool.   Each job is given a raft, visit id, and exposure sequence id.   This information is transmitted to the Replicator Node Process on the same host the job is running on.   The replicator job then makes a method call to retrieve that particular raft.

The replicator job retrieves the raft, and then sends a message with the location of the data to the Replicator Node Process, and exits.
 
**Notes**: The process of starting a new replicator job for every exposure seems to be quite a bit of overkill to transfer one image to the node, and then on to the distributor.   One of the issues that we’ve been trying to mitigate is the start up time for the job.  Generally this is pretty quick, but we’ve seen some latency in the start of the process when submitted through HTCondor.   I don’t think it makes sense to have a process be started and stopped through HTCondor, and depend on it starting at such a rapid pace.  We should explore having Replicator Node Processes transfer the files and use redis to do the raft assignments, and eliminate the Replicator Jobs completely.

Distributors
^^^^^^^^^^^^

The machine lsst-dist runs twenty-two distributor processes.   These processes receive images from their paired replicators and split the images into nine CCDs.   Each CCD is later retrieved by Worker Jobs running on nodes in the HTCondor Worker Pool.

The distributor is started with a parameter of which port to use as it’s incoming port.   Before the distributor send messages, it sets up several things.  First, it starts a thread for incoming Archive DMCS requests.  This event triggers a dump of all CCD identification data to the Archive DMCS so the Archive DMCS can replenish it’s cache for worker requests.  This is done in case the Archive DMCS goes offline and loses it’s cached information.  Next the Distributor also sends it’s own identification information to the Archive DMCS.   When the Archive DMCS receives this information, it removes all previous information the Distributor sent it.   The Distributor does not maintain a cache of it’s information.

At this point, the distributor can receive connects from worker jobs and its paired replicator.   Once the replicator contacts the distributor, the network connection is maintained throughout its lifetime.   If the connection is dropped for some reason, the distributor goes back and waits for the replicator to reconnect. 

 At this point messages from the Replicator Node Process can be received, generally in pairs. The first type of message serves as a notice to the Distributor with information about the raft it is about to receive.   The Distributor can at this point send that information to the Archive DMCS.  (The Archive DMCS informs any workers waiting for Distributor information for a particular CCD of the Distributor’s location).

The second type of message that can be received from the Replicator Node Process is the data itself.  Header information in the transmission describes the raft data being sent, and a length for the data payload.  The data is read by the Distributor, and split into 9 CCDs.

Workers contact the Distributors, and request that a CCD be transmitted.  If the CCD is not yet available, the worker blocks until it is received by the distributor.  The waiting workers get the image once the CCD is received.  Once the worker receives the image, the connection to the distributor is broken.

**Notes**:  The distributor/replicator pairing maintains a continuous connection until one side is brought down, or an error is detected.   The location of the distributor is specified on invocation;  it might be better to have something like REDIS keep this information.
It might also be good to keep distributor/ccd location information in REDIS, eliminating the pairing software that currently exists in the Archive DMCS.

Archive DMCS
^^^^^^^^^^^^

The Archive DMCS process is a rendezvous point between Worker Jobs and Distributors.   On startup, the Archive DMCS sends a messages to all distributors asking for image data meta data that they currently hold.   The Archive DMCS process opens a port on which Worker Jobs can contact it, and subscribes to an event topic on which it can get advisory messages from the Distributors.  Distributors send two types of messages.  The first is notifies the Archive DMCS that it is starting.  This clears all entries in the Archive DMCS for that Distributor, since the Distributors do not have an image cache and may be a completely new Distributor with no previous knowledge of images.  The second is an information event that tells the Archive DMCS which image it has available.

On acceptance of a connection from the worker,  a thread is spawned and a lookup for the requested Distributor is performed.  If the Distributor is not found, the worker thread blocks until that data arrives, or until a TTL times out.

When the Distributor receives information from its paired Replicator about the raft image it will handle, events are sent to the Archive DMCS containing the information for all the CCDs in the raft.

When the Archive DMCS receives this information from the Distributor, it adds those entries into its cache, and notifies the waiting workers that new data is available.  If a Worker Job receives the information it was looking for, it disconnects from the Archive DMCS and contacts the Distributor, which will send the job CCD image it requested.

There is a passive Archive DMCS that shadows the active Archive DMCS, and will respond to requests if the main Archive DMCS fails.   Both Worker Jobs and Distributors are configured with the active and failover host/port and can respond to a failed connection (or unreachable host) appropriately.

**Notes**:

There are a number of issues which were solved in creating the Archive DMCS, including timeouts of TTL counters for threads, cleanup on workers that disconnected while waiting, and caching of information that multiple threads were requesting.    Some issues, such as expiring data deemed out of daea (since no worker would request that old data from a distributor after a certain amount of time), were not addressed.   While a duplicate Archive DMCS was built and can act as a failover shadow, this implementation does not seem ideal because of the way the workers and distributors need to be configured for failover.  Additionally, a better mechanism than having Worker Job threads connect at wait for data from the Archive DMCS, seem possible via DM Event Services.

Since this portion of the simulator was written, we’ve found that there are some open source packages, such as Redis and Zookeeper, that seem to duplicate the functionality we’ve written and address the issues listed above.   Some small bit of code may have to be written to clear data cache on Distributor startup. Both packages have Python interfaces.  These are in widespread use in other projects and companies.   This seems to be worth investigating.

Worker Master Node
^^^^^^^^^^^^^^^^^^

The HTCondor master node for the Worker pool is configured on lsst-work.   This system acts as the HTCondor master for fifteen VMs, lsst-run1 through lsst-run15, which are configured with 2 CPUs and 4 gig of memory each.   HTCondor is configured on lsst-run1 through lsst-run14 to have 13 job slots each, and lsst-run15 is configured with 7 slots.
 
This master node accepts job submissions from lsst-base, and runs those jobs on lsst-run1 through lsst-run15.

Worker Execution Node
^^^^^^^^^^^^^^^^^^^^^

A worker node is part of the HTCondor pool, which is controlled by the HTCondor master.    Each node accepts Worker Jobs scheduled by the HTCondor master.   There are 189 worker nodes, one for each CCD.

**Notes**:  Due to limitations capacity at the time the simulator was written, this was all simulated across 15 VMs.   Each VM was configured with 2 CPUs, and 4 gig of memory.   HTCondor will ordinarily make the number of slots for jobs equal to the number of CPUs, but we overrode this to configure 13 slots per for the first 14 VMs, and 7 for the last one.    As with the replicator jobs, we noted varying startup times from the time the job was submitted to the HTCondor master.  The pool was configured to retain ownership of the slot, which increased the speed at which jobs were matched to slots. In general, the start up was very quick, but there were times when we noted start up times of between 15 and 30 seconds.   It was difficult to determine why exactly this occurred, but given the limited capacity of the VMs themselves, we believe this is a contributing factor.

Worker Job
^^^^^^^^^^

The Worker Job starts with the CCD id, visit id, raft id, foresight, filter id, and the number of exposures to consume.  A job termination thread is started.  If the timer in the thread expires, the job is terminated. The job contacts the Archive DMCS with the CCD information and blocks until it receives the distributor location for that CCD.  Once the worker retrieves the distributor location, it contacts that distributor and asks for that CCD.  If the distributor has no information about the CCD, the worker returns to the Archive DMCS and requests the information again.  Once all exposures for that CCD are retrieved, the work jobs sleep for a short time to simulate processing.  When this completed, the worker job exits
