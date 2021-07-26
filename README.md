PEGR NGS Pipeline
===========================================================

This repository contains the PEGR NGS processing pipeline. The PEGR
processing pipeline can be installed by cloning this repository and
configuring the pipeline for use within the file system into which it has
been installed.  This pipeline is used to retrieve raw sequenced datasets
from a remote server, convert them to fastqsanger.gz data format, import
them into Galaxy data libraries, and use them as the inputs to workflows
(contained within the Galaxy instance), automatically executing the workflows
for every sample in the run.

Note: any reference to a ~/ file location in the text below implies the
root installation directory of a clone of this repository.

Configuring the pipeline
========================

The pipeline configuration file is located ~/config/pegr_config.ini.sample,
and the sample included with the repository should be copied to
~/config/pegr_config.ini, which is then used as the pipeline's configuration
file.  The file system configuration settings within this file allow for the
pipeline to easily be moved to new environments over time if necessary.

These configuration settings are used by each of the components of the
pipeline, and we'll categorize these components as follows.

1. Remote server configuration settings for retrieving the raw sequenced data.
  The following settings are used primarily by the copy_raw_data.py script.  The
  exception is the REMOTE_WORKFLOW_CONFIG_DIR_NAME setting, which is used by the
  start_workflows.py script.

  RAW_DATA_LOGIN - the login information for the remote server.

  RAW_DATA_DIR - the location for retrieving the raw sequenced data from the
  remote server.

  REMOTE_RUN_COMPLETE_FILE - The file named RunCompletionStatus.xml, which
  is produced by the Illumina sequencer.  This file name configuration setting
  can be changed if different sequencers are used over time.

  REMOTE_RUN_INFO_FILE - the full path to the pegr_run_info.txt on the remote
  server.  This file is produced for each run, and is the configuration engine
  used by the pipeline for each run.  This file is retrieved from the remote
  server by the copy_raw_data.py script and stored locally within the ~/config
  directory.

  REMOTE_WORKFLOW_CONFIG_DIR_NAME - This is the name of the directory that
  contains the XML files for each run.  This value cannot be a full path, but
  must be restricted to the directory name (e.g., pegr_config).

2. Local file system configuration settings.  The follwoing settings are used
  primarily by the bcl2fastq.py and the send_data_to_galaxy.py scripts.

  ANALYSIS_PREP_LOG_FILE_DIR - the local directory where the pre-processing
  pipeline will generate and store all log files.  This is typically ~/log

  BCL2FASTQ_BINARY - the name of the bcl2fastq package - this cannot be a full
  path.

  BCL2FASTQ_REPORT_DIR - the full path to the location that the bcl2fastq
  package will generate its reports.

  FASTQ_VALIDATOR_BINARY - the full path to the installed fastQValidator package.
  The fastQValidator package is available here:
  http://genome.sph.umich.edu/w/images/2/20/FastQValidatorLibStatGen.0.1.1a.tgz
  Due to the way bcl2fastq compresses files (it does not include an end of file
  block), this enhancement was added manually to the ~/src/FastQValidator.cpp
  file:
  https://github.com/statgen/fastQValidator/commit/0b7decb8b502cd8d9d6bf27dbd9e319ed8478b53.
  The package was then compiled normally.

  RUN_INFO_FILE - the full path to the local pegr_run_info.txt file.

  SAMPLE_SHEET - the full path to the sample sheet file produced by the
  bcl2fastq.py script.

  LIBRARY_PREP_DIR - the full path to the files that have been converted by the
  bcl2fastq. py script into fastqsanger data format.  This path is used by the
  send_data_to_galaxy.py script to import the files into Galaxy data libraries.

  BLACKLIST_FILTER_LIBRARY_NAME - the name of the Galaxy data library that
  contains all of the blacklist filter datasets used by the ChIP-exo workflows.

3. Galaxy instance and bioblend API configuration settings.

  GALAXY_HOME - the full path to the Galaxy installation root directory.

  GALAXY_BASE_URL - the Galaxy URL, including port.

  API_KEY - the Galaxy api key associated with the user that executes the
  ChIP-exo workflows for each run.

The pipeline's four custom programs
===================================

The pipeline consists of four primary custom programs that perform its tasks.
Each of these programs includes quality assurance components that automatically
halt processing if errors occur, logging the details for review and correction.
Each program can be executed independently (assuming that the previous program
in the pipeline has completed successfully) allowing for a certain step to be
re-executed after corrections are made.

These programs are located in ~/scripts/api and are executed in the following
order.  The programs are executed via the start_processing.sh shell script
located in the same directory.

1. ~/scripts/api/copy_raw_data.py - This script copies a directory of raw data
files produced by the sequencer from a remote host to a local directory.

2. ~/scripts/api/bcl2fastq.py - This script reads a local directory of raw
sequenced datasets and executes the bcl2fastq converter on each file,
converting the raw sequenced data into the fastqsanger format.

3. ~/scripts/api/send_data_to_galaxy.py - This script uses the bioblend API
to create and populate Galaxy data libraries with the data produced by the
bcl2fastq.py script.

4. ~/scripts/api/start_workflows.py - This script parses the pegr_run_info.txt
file to automatically execute a selected workflow for each dbkey defined for
every sample in the defined run.  This script retrieves data library datasets
that were imported by the send_data_to_galaxy.py script and imports them into
a new Galaxy history for each workflow execution.  The history is named with a
combination of the workflow name (e.g., sacCer3_cegr_paired), the workflow
version (e.g., 001) the sample (e.g., 02) and the history name id (e.g., 001).
Using these examples, the complete history name is
sacCer3_cegr_paired_001-02_001.  The values of both the workflow version and
the history name id can be passed as command line parameters if desired.

Executing these scripts requires logging into the file system unless cron is
used to execute them automatically.  The scripts are executed by executing the
~/scripts/api/start_processing.sh shell script (e.g., sh start_processing.sh).
This script can be executed manually from the command line or cron can be
configured to execute it at a specified time.

When the copy_raw_data.py script is executed, it will first check the remote
server (specified by the RAW_DATA_LOGIN configuration setting) to see if the
file specified by the REMOTE_RUN_COMPLETE_FILE configuration setting exists.
If it does, the script will continue.  If it does ot. the script will sleep
for 5 minutes and check again.  This polling process will continue until the
REMOTE_RUN_COMPLETE_FILE is found, at which time the script will stop polling
and continue its execution.

Each of these scripts, at its conclusion, will produce a file named the same
as the script, but with a .complete extension.  These files are created in
the ~/log directory.  For example, when the ~/scripts/api/copy_raw_data.py
script finishes, it creates the file ~/log/copy_raw_data.py.complete.  The
~/scripts/api/start_processing.sh shell script looks for these files, and
does not execute a script if an associated .complete file exists for it.  This
allows a user (or cron) to execute certain steps in the pipeline while skipping
others.  This is very useful when an error occurs within a step, requiring that
step to be re-executed without having to execute the previous steps in the
pipeline since they completed successfully.  It is also very useful to use this
feature to keep downstream steps in the pipeline from executing if you want to
check the results of the current step before allowing the next step to start.

When all 4 of these scripts have completed, the
~/scripts/api/start_processing.sh script will delete each of the .complete
files created in the ~/log directory by each of the scripts, allowing the next
run to start

Authors
======

William KM Lai

Gretta Kellogg

Ali Nematbakhsh

Greg Von Kuster
