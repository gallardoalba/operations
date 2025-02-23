
### Galaxy Job handling 

Reassign jobs from handler 11 to handler 3 with gxadmin:

```bash
gxadmin tsvquery queue-detail-by-handler handler_main_11  | cut -f1 | xargs -I{} -n1 gxadmin mutate reassign-job-to-handler {} handler_main_3 --commit

```

-----

### Change htcondor labels on-the-fly:

```bash
sed -i 's/"compute"/"dockerhack"/g' /etc/condor/condor_config.local; systemctl reload condor
```

Test with:

```bash
condor_status -autoformat Machine GalaxyGroup GalaxyDockerHack | grep hack | sort -u
```

-----

### fail all jobs of a particular user using gxadmin

The following command is failing all jobs of the service-account user.

```bash
gxadmin query queue-details | grep  service-account | awk '{print $3}' |  xargs -I {} sh -c "gxadmin local fail-job {}"
```

### fail all jobs on the nodes, in cases when condor_rm does not do the job

This cmd will find all jobs matching a string (here "obabel"), returns the group-pid and kills those group process. This seems to be the only way
to remove jobs from the condor nodes when condor_rm was not able to kill the jobs.

```bash
pdsh -g cloud 'ps xao pgid,cmd | grep "[o]babel" | awk "{ print \$1 }" | xargs -I {} sudo kill -9 {}'
```

-----
### Jobs running into a specific host

```bash
condor_q -autoformat ClusterID JobDescription RemoteHost | grep cn032
```

### Number of cores available

```bash
condor_status -autoformat Name Cpus | cut -f2 -d' ' | paste -s -d'+' | bc
```

### Debugging of a Condor job that was giving back an empty file as result
As input we had a galaxy job id `11ac94870d0bb33a8013642012e063ec (11384941)` and a note of an empty file as result. The job was a step of a big collection where the other steps were successful.

To understand the reason for the problem, I proceeded as follows:

```bash
condor_history | grep 11384941
```
to retrieve the condor id

```bash
condor_history -l 6545461
```
to retrieve all the job detail, and here, I found this error message:  
`"Error from slot1_1@cn030.bi.uni-freiburg.de: Failed to open '/data/dnb03/galaxy_db/job_working_directory/011/384/11384941/galaxy_11384941.o' as standard output: No such file or directory (errno 2)"`

A quick check into the compute node
```[root@cn030 ~]# cd /data/dnb03
-bash: cd: /data/dnb03: No such file or directory
```
showed it was not mounting properly the NFS export.

### Reserve an handler for a tool and move all running jobs to it

Add a new handler to the `job_conf.xml`

```xml
	<handlers assign_with="db-skip-locked" max_grab="8">
		<handler id="handler_key_0"/>
		<handler id="handler_key_1"/>
		<handler id="handler_key_2"/>
		<handler id="handler_key_3"/>
		<handler id="handler_key_4"/>
		<handler id="handler_key_5"/>
		<handler id="handler_key_6" tags="special_handlers"/>
	</handlers>
```
associate the tool to that handler

```xml
	<tools>
		<tool id="upload1" destination="gateway_singlerun" />
		<tool id="toolshed.g2.bx.psu.edu/repos/chemteam/gmx_sim/gmx_sim/2020.4+galaxy0" destination="gateway_singlerun" resources="usegpu" />
		<tool id="toolshed.g2.bx.psu.edu/repos/chemteam/gmx_sim/gmx_sim/2019.1.5.1" destination="gateway_singlerun" resources="usegpu" />
		<tool id="gmx_sim" destination="gateway_singlerun" resources="usegpu" />
		<tool id="param_value_from_file" handler="special_handlers" />
```
restart workflow schedulers
and
move all running jobs to the new handler

```bash
for j in `gxadmin query queue-detail --all| grep param_value_from_file |grep -v handler_key_6 | cut -f2 -d'|' | paste -s -d ' '`; do gxadmin mutate reassign-job-to-handler $j handler_key_6 --commit;done
```

### How to shut down/start up a single execute node without killing jobs

This “peaceful” shutdown of a startd will cause that daemon to wait indefinitely for all existing jobs to exit before shutting down. During this time, no new jobs will start.
```bash
condor_off -peaceful -startd vgcnbwc-worker-c125m425-8231.novalocal
```

To begin running or restarting all daemons (other than condor_master) given in the configuration variable DAEMON_LIST on the host:
```bash
condor_on vgcnbwc-worker-c125m425-8231.novalocal
```

Both commands can be executed from the submitter node.
