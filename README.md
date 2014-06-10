HERD
====

A Highly Efficient key-value system for RDMA

This version of HERD has been tested for the following configuration:

1. Software
  * OS: Ubuntu 12.04 (kernel 3.2.0)
  * RDMA drivers: `mlx4` from MLNX OFED 2.2. I suggest using the MLNX OFED version for Ubuntu 12.04.
2. Hardware
  * RNICs: 
    * ConnectX-3 353A (InfiniBand) with PCIe 2.0 x8
	* ConnectX-3 313A (RoCE) with PCIe 2.0 x8
	* ConnectX-3 354A (InfiniBand) with PCIe 3.0 x8

Initial setup:
-------------

* I assume that the machines are named: `node-i.RDMA.fawn.apt.emulab.net` starting from `i = 1`.
  * The experiment requires at least `(1 + (NUM_CLIENTS / num_processes))` machines.
	`node-1` is the server machine.
  	`NUM_CLIENTS` is the total number of client processes, defined in `common.h`.
	`num_processes` is the number of client processes per machine, defined in
	`run-machine.sh`.
  * To modify HERD for your machine names: 
    * Make appropriate changes in `bomb.sh` and `run-servers.sh`.
	* Change the server's machine name in the `servers` file. Clients use this file to
	  connect to server processes.

  * Make sure that ports 5500 to 5515 are available on the server machine. Server process `i`
	listens for clients on port `5500 + i`.

* Execute the following commands at the server machine:
```bash
cd ~
git clone https://github.com/anujkaliaiitd/HERD.git
export PATH=~/HERD/scripts:$PATH
cd HERD
sudo ./shm-init.sh	# Increase shmmax and shmall
sudo hugepages-create.sh 0 4096		# Create hugepages on socket 0. Do for all sockets.
```

* Mount the HERD folder on all client machines via NFS.

Quick start:
-----------

* Run `make` on the server machine to build the executables.

* To run the clients automatically along with the server:

```bash	
# At node-1 (server)
./run-servers.sh
```

* If you do not want to run clients automatically from the server, delete the 
2nd loop from `run-servers.sh`. Then:
	
```bash	
# At node-1 (server)
./run-servers.sh
# At node-2 (client 0)
./run-machine.sh 0
# At node-i (client i - 2)
./run-machine.sh (i - 2)
```

* To kill the server processes, run `kill-local.sh` at the server machine. To kill the 
client processes remotely, run `kill-remote.sh` at the server machine.


<!---
Algorithm details:
====

SERVER's ALGORITHM (one iteration)

1. Poll for a new request. The polling must be done on the last byte
of the request area slot. We must check (char) key != 0 and not just
key != 0. The latter can lead to a situation where the request is 
detected before the key is written entirely by the HCA (for example,
only the first 4 bytes have been writtesn). 

If no new request is found in FAIL_LIM tries, go to 2.

2. Move the pipeline forward and get a pipeline item as the return
value. The pipeline item contains the request type, the client
number from which this request was received, and the request area
slot (RAS) from which this request was received.
	2.1. If the request type is a valid type (GET_TYPE or PUT_TYPE),
	send a response to the client. Otherwise, do nothing.

3. Add the new request to the pipeline. The item that we're adding
is the one that was polled in step 1.

We zero out the polled field of the request and store it into the
pipeline item. This is a must do. Here's what happens if we don't
zero out the polled field. Although the client will not write
into the same request slot till we send a response for the slot, the 
server's round-robin polling will detect this request again.

We also zero out the len field of the request. This is useful because
clients do not WRITE to the len field for GETs. So, when a new
request is detected in (1), len == 0 means that the request is a
GET, otherwise it's a PUT.

OUTSTANDING REQUESTS / RESPONSES:
----

The number of outstanding responses from a server is WS_SERVER.
A server polls for SEND completions once per WS_SERVER SENDs.

The number of outstanding requests from a client is WINDOW_SIZE.
A client polls for a RECV completion WINDOW_SIZE iterations after
a request was posted. The client polls for SEND completions *very*
rarely: once every S_DEPTH iterations. This is because the RECV
completions, which are polled frequently, give an indication of 
SEND completions.

The client uses parameters CL_BTCH_SZ and CL_SEMI_BTCH_SZ to post
RECV batches.
--->
