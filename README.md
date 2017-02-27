[title]: - “Large Scale Computation with HTCondor’s Queue Command”
[TOC]


## Overview


Many large scale computations require the ability to process multiple jobs concurrently.  Consider the extensive
sampling for a multi-dimensional Monte Carlo integration or molecular
dynamics simulation with several initial conditions. These type of calculations require 
submitting many jobs. About a million CPU hours per day are available to OSG users
on an opportunistic basis.  Learning how to scale up and control large
numbers of jobs is essential to realizing the full potential of distributed high
throughput computing on the OSG.

The `Queue` command in HTCondor can handle running multiple jobs from  a single job description file. In this tutorial, we will see how to scale up the calculations for a 
simple python example using the HTCondor’s Queue command


Once we understand the basic HTCondor script, it is easy
to scale up.

    $ tutorial ScalingUp-python
    $ cd tutorial-ScalingUp-python


As we discussed in the previous section on HTCondor scripts, we need to
prepare the job execution and the job submission scripts. 

## Python script and the optimization function

Let us take a look at our objective function that we are trying to optimize.

    def rosenbrock(x):   # The rosenbrock function
        f = 100.0*(1 - x[0])**2 + (x[1] - x[0]**2)**2
        return f

This a two dimensional Rosenbrock function. Clearly, the minimum is located at (1,1). Rosenbrock
function is one of the test function used to test the robustness of an optimization method.

![fig 1](https://raw.githubusercontent.com/OSGConnect/tutorial-matlab-SimulatedAnnealing/master/Figs/RosenBrockFunction.png)

Here, we are going to use the brute force optimization approach to evaluate the two dimensional 
Rosenbrock function on grids of points. The boundary values for the grid points are 
randomly assigned inside the python script. However, these default values may be replaced by 
user supplied values.

For random assigned boundary values, the script is executed without any argument

    python rosen_brock_brute_opt.py

The boundary values are supplied as input argument to the python script

    python rosen_brock_brute_opt.py x_low x_high y_low y_high

where x_low and x_high are low and high values along x direction, and y_low and y_high are the low and high values along the y direction.

For example, the following arguments mean the boundary of x direction is (-3, 3) and the boundary of y direction is (-2, 3).

    python rosen_brock_brute_opt.py  -3 3 -2 2

The directory `Example1` runs the python script with the default random values. The directories `Example2`, `Example3` and `Example4` deal with supplying the boundary values as input arguments. 

##Execution Script 


Let us take a look at the execution script, `scalingup-python-wrapper.sh`

    #!/bin/bash

    module load python/3.4
    module load all-pkgs

    python ./rosen_brock_brute_opt.py  $1 $2 $3 $4

The wrapper loads the the relevant modules and then executes the python script `rosen_brock_brute_opt.py`. The python script takes four argument but they are optional. If we don't supply these optional
arguments, the values are internally assigned.

## Submitting jobs concurrently

Now let us take a look at job description file 

    cd Example1
    cat ScalingUp-PythonCals.submit

If we want to submit several jobs, we need to track log, out and error  files for each
job. An easy way to do this is to add the `$(Cluster)` and `$(Process)` variables to the file names. 

    # The UNIVERSE defines an execution environment. You will almost always use VANILLA.
    Universe = vanilla

    # These are good base requirements for your jobs on the OSG. It is specific on OS and
    # OS version, core, and memory, and wants to use the software modules. 
    Requirements = OSGVO_OS_STRING == "RHEL 6" && TARGET.Arch == "X86_64" && HAS_MODULES == True 
    request_cpus = 1
    request_memory = 1 GB

    # executable is the program your job will run It's often useful
    # to create a shell script to "wrap" your actual work.
    executable = scalingup-python-wrapper.sh 

    # files transferred into the job sandbox
    transfer_input_files = rosen_brock_brute_opt.py

    # error and output are the error and output channels from your job
    # that HTCondor returns from the remote host.
    output = Log/job.out.$(Cluster).$(Process)
    error = Log/job.error.$(Cluster).$(Process)


    # The log file is where HTCondor places information about your
    # job's status, success, and resource consumption.
    log = Log/job.log.$(Cluster).$(Process)

    # Send the job to Held state on failure. 
    on_exit_hold = (ExitBySignal == True) || (ExitCode != 0)  

    # Periodically retry the jobs every 60 seconds, up to a maximum of 5 retries. 
    # The RANDOM_INTEGER(60, 600, 120) means random integers are generated between 
    # 60 and 600 seconds with a step size of 120 seconds. The failed jobs are 
    # randomly released with a spread of 1-10 minutes.  Releasing multiple jobs at 
    # the same time causes stress for the login node, so the random spread is a 
    # good approach to periodically release the failed jobs. 

    PeriodicRelease = ( (CurrentTime - EnteredCurrentStatus) > $RANDOM_INTEGER(60, 7200, 120) ) && ((NumJobStarts < 5))

    # Queue is the "start button" - it launches any jobs that have been
    # specified thus far.
    queue 10

Note the `Queue 10`.  This tells Condor to queue 10 copies of this job as one cluster.  

Let us submit the above job

    $ condor_submit ScalingUp-PythonCals.submit
    Submitting job(s)..........
    10 job(s) submitted to cluster 329837.

Apply your `condor_q` and `connect watch` knowledge to see this job progress. After all 
jobs finished, execute the `post_script.sh  script to sort the results. 

    ./post_script.sh

## Other ways to use Queue command

Now we will explore other ways to use Queue command. In the previous example, we did not pass 
any argument to the program and the program generated random boundary conditions.  If we have some guess about what could be a better boundary condition, it is a good idea to supply the boundary 
condition as arguments. 

It is possible to use a single file to supply multiple arguments. We can take the job description 
file from the previous example, and modify it slightly to submit several jobs.  The modified job 
description file is available in `Example2` directory.  Take a look at the job description file `ScalingUp-PythonCals.submit`.  

    $ cd Example2
    $ cat  ScalingUp-PythonCals.submit
    
    ...
    #Supply arguments 
    arguments = -9 9 -9 9

    # Queue is the "start button" - it launches any jobs that have been
    # specified thus far.
    queue 

    arguments = -8 8 -8 8
    queue 

    arguments = -8 8 -8 8
    queue 
    ...

Let us submit the above job

    $ condor_submit ScalingUp-PythonCals.submit
    Submitting job(s)..........
    10 job(s) submitted to cluster 329838.

Apply your `condor_q` and `connect watch` knowledge to see this job progress. After all 
jobs finished, execute the `post_script.sh  script to sort the results. 

    ./post_script.sh


A major part of the job description file looks same as the previous example. The main 
difference is that the addition of  `arguments` keyword.  Each time the queue command appears 
in the script, the expression(s) before the queue would be added to the job description. 


We may get tired of typing the argument and queue expressions again and again in the above 
job description file. There is a way to implement compact queue expression and  expand the 
arguments for each job. Take a look at the job description file in Example3. 

    $ cat Example3/ScalingUp-PythonCals.submit
    ...
    queue arguments from (
    -9 9 -9 9 
    -8 8 -8 8 
    -7 7 -7 7 
    -6 6 -6 6 
    -5 5 -5 5 
    -4 4 -4 4 
    -3 3 -3 3 
    -2 2 -2 2 
    -1 1 -1 1 
    )
    ...

Let us submit the above job

    $ condor_submit ScalingUp-PythonCals.submit
    Submitting job(s)..........
    10 job(s) submitted to cluster 329839.

Apply your `condor_q` and `connect watch` knowledge to see this job progress. After all 
jobs finished, execute the `post_script.sh  script to sort the results. 

    ./post_script.sh


In fact, we could define variables and assign them to HTCondor's expression. This is
illustrated in Example4. 

    $ cd Example4
    $ cat ScalingUp-PythonCals.submit

    ...
    arguments = $(x_low) $(x_high) $(y_low) $(y_high)

    # Queue command  
    queue x_low, x_high, y_low, y_high from (
    -9 9 -9 9 
    -8 8 -8 8 
    -7 7 -7 7 
    -6 6 -6 6 
    -5 5 -5 5 
    -4 4 -4 4 
    -3 3 -3 3 
    -2 2 -2 2 
    -1 1 -1 1 
   )

The  queue command defines the variables x_low, x_high, y_low, and y_hight.  These variables are passed on to the 
argument command (`arguments = $(x_low) $(x_high) $(y_low) $(y_high)`). 
 
Let us submit the above job

    $ condor_submit ScalingUp-PythonCals.submit
    Submitting job(s)..........
    10 job(s) submitted to cluster 329840.

Apply your `condor_q` and `connect watch` knowledge to see this job progress. After all 
jobs finished, execute the `post_script.sh  script to sort the results. 

    ./post_script.sh


## Key Points
- [x] Scaling up the computational resources on OSG is crucial to taking full advantage of grid computing.
- [x] Changing the value of `Queue` allows the user to scale up the resources.
- [x] `Arguments` allows you to pass parameters to a job script.
- [x] `$(Cluster)` and `$(Process)` can be used to name log files uniquely.
- [x]  Check the HTCondor manual to learn more about the `Queue` command (https://research.cs.wisc.edu/htcondor/manual/latest/2_5Submitting_Job.html).

## Getting Help
For assistance or questions, please email the OSG User Support team  at <mailto:user-support@opensciencegrid.org> or visit the [help desk and community forums](http://support.opensciencegrid.org).
