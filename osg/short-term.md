---
layout: page
title: Discovery Environment (DE) Docs
---

# Table of Contents

* [Short-Term OSG Integration](#short-term-osg-integration)
    * [Image Overview](#image-overview)
        * [Requirements](#requirements)
    * [Ticket List Files](#ticket-list-files)
    * [Job Config Files](#job-config-files)
        * [arguments](#arguments)
        * [irods_host](#irods_host)
        * [irods_port](#irods_port)
        * [irods_job_user](#irods_job_user)
        * [irods_user_name](#irods_user_name)
        * [irods_zone_name](#irods_zone_name)
        * [input_ticket_list](#input_ticket_list)
        * [output_ticket_list](#output_ticket_list)
        * [status_update_url](#status_update_url)
        * [stdout](#stdout)
        * [stderr](#stderr)
    * [Initializing the iRODS Connection](#initializing-the-irods-connection)
    * [Sending Job Status Updates](#sending-job-status-updates)
        * [Status Codes](#status-codes)
        * [Example Implementation](#example-implementation)
    * [Retrieving Input Files](#retrieving-input-files)
    * [Running the Job](#running-the-job)
    * [Uploading Output Files](#uploading-output-files)
    * [JSON Parsing](#json-parsing)
    * [Caveats](#caveats)
        * [File Creator Problem](#file-creator-problem)


# Short-Term OSG Integration

The DE can submit jobs to OSG, but the current integration mechanism is very limited in scope and places much of the
burden on the app integrator. For OSG jobs, the DE performs the following tasks:

- obtains iRODS read tickets for each of the job input files and creates a file listing the tickets and paths
- obtains an iRODS write ticket for the job output folder and creates a file listing the tickets and paths
- creates a JSON configuration file containing information needed to run the job
- submits the job to OSG

The app integrator is expected to provide a wrapper script that performs the following tasks:

- reads and parses the JSON configuration file
- initializes the iRODS connection based on information available in the configuration file
- sends job status updates back to the DE using a URL provided in the configuration file
- parses the input ticket list file and retrieves all of the input files from iRODS
- runs the job using the arguments provided in the configuration file
- redirects job output to the file names provided in the configuration file
- parses the output ticket list file and uploads all job outputs to iRODS

**Note:** a username for the iRODS connection is provided in the job configuration file, but the wrapper script is not
required to use it. If the wrapper script uses custom iRODS credentials then the script itself is responsible for
obtaining the credentials and using them to initialize the iRODS connection.

## Image Overview

OSG creates the Singularity images in its CVM-FS repository by pulling Docker images from Dockerhub and converting them
to Singularity images. This means that the first step to create a Singularity image for OSG is to create a Docker image
and push it to Dockerhub. Instructions for getting Docker images into OSG's CVM-FS repository can be found [here][1].

### Requirements

The primary requirements are that the image must contain an executable file called `/usr/bin/wrapper`, and this program
must read its configuration settings from a file called `config.json` in the current working directory. (The initial
working directory cannot be known in advance even if a working directory is specified in the Docker image.)  The
implementation language is unimportant as long as all of the tasks required of the wrapper can be performed. CyVerse has
a reference implementation of such a wrapper script implemented in Python 2.7 available [here][2].

Each image must also have a way to communicate with iRODS. Usually, this means going with one of two options. If you're
using a JVM language such as Java, Scala or Clojure for the wrapper script then you can communicate with iRODS using
Jargon. If you're not using a JVM language then the easiest option is to use the iRODS icommands. The wrapper script
reference implementation provided by CyVerse uses the icommands which are installed as a step in the [corresponding
Dockerfile][3].

This image should also contain an empty directory called `/cvmfs`. When the job is executed, OSG's CVM-FS repository
will be mounted to this path.

## Ticket List Files

The input and output ticket list files are comma-delimited files with each significant line containing an iRODS ticket
string and a file path. Blank lines and lines beginning with a hash mark (`#`) are ignored. The first line in the file
is a comment line containing the MIME type, `application/vnd.de.tickets-path-list+csv; version=1`.

As mentioned above, each significant line in the file contains the iRODS ticket string followed by a comma and the path
corresponding to the ticket. The ticket string is guaranteed not to contain commas, but the path may. None of the fields
in the ticket list file are ever quoted. Instead, parsers must rely on the fact that tickets will never contain commas
to parse the field. For example, suppose the following line is found in a ticket list file:

```
521CDB78-8EA4-4F14-94FF-D506DB0D45D7,/iplant/home/nobody/the foo, the bar, and the baz
```

In this example the ticket string is `521CDB78-8EA4-4F14-94FF-D506DB0D45D7` and the path is `/iplant/home/nobody/the
foo, the bar, and the baz`.

The [wrapper script reference implementation][2] contains a convenient context manager class that can be used to
correctly parse a ticket list file:

``` python
# This is a simple context manager class designed to make it easier to read and write iRODS ticket list files.
class TicketListReader:
    def __init__(self, path):
        self.path = path

    def __enter__(self):
        self.fd = open(self.path, "r")
        self.r = itertools.imap(lambda l: l.strip().split(",", 1), itertools.ifilter(lambda l: l[0] != '#', self.fd))
        return self.r

    def __exit__(self, type, value, traceback):
        self.fd.close()
```

## Job Config Files

The job configuration file is a JSON file named `config.json` containing configuration settings for the job. The
following configuration settings are provided:

### arguments

This setting is an array of strings containing the command-line arguments generated by the DE based on the app
configuration and the parameter values selected by the user. In cases where the wrapper script runs the tool itself as a
subprocess, this argument list can generally be passed to the subprocess as-is. In other cases, the wrapper script must
interpret the argument array directly.

### irods_host

This setting contains the host name to use when connecting to iRODS.

### irods_port

This setting contains the port number to use when connecting to iRODS.

### irods_job_user

This setting contains the username of the person who submitted the job. This user should have ownership permissions on
the output files after the job completes.

### irods_user_name

This setting contains the username to use when authenticating to iRODS. This setting can be ignored if you're using
custom iRODS credentials.

### irods_zone_name

This setting contains the zone name to use when authenticating to iRODS and can be ignored if you're using custom iRODS
credentials.

### input_ticket_list

This setting contains the name of the input ticket list file.

### output_ticket_list

This setting contains the name of the output ticket list file.

### status_update_url

This setting contains a URL that should be used to send job status updates back to the DE. This will be described in
more detail later.

### stdout

This setting contains the name of the file that should contain the redirected standard output produced by the tool.

### stderr

This setting contains the name of the file that should contain the redirected standard error output produced by the
tool.

## Initializing the iRODS Connection

As long as you're using version 4.x of the icommands and not using custom iRODS credentials, initializing the iRODS
connection is very easy. `config.json` is made to be compatible with the file that the icommands use to
specify the server connection settings. The wrapper script can simply copy `config.json` to
`~/.irods/irods_environment.json`. And start using the icommands.

If you're using custom iRODS credentials or an older version of the icommands then (assuming you're not using Jargon)
`iinit` must be called before using any of the iRODS icommands. `iinit` only runs in interactive mode, so it will be
necessary to send the password to `iinit` using redirection. Using older icommands versions or custom iRODS credentials
is expected to be rare, so precise instructions for running `iinit` are beyond the scope of this
documentation. Similarly, using Jargon is beyond the scope of this documentation.

## Sending Job Status Updates

Job status updates must be sent to the DE using HTTP POST requests. The complete URL used to send job status updates for
the current job is provided in the job configuration file. The request body is a JSON object consisting of three
fields. The `state` field contains the current job status. The `message` field contains a brief description of what the
job is currently doing. The `hostname` field contains the host name or IP address that the request is being sent from.

### Status Codes

There are three job status codes that can be used in the wrapper script: `running`, `completed` and `failed`. A status
update with the status code `running` should be sent immediately after the job configuration file is parsed. This
informs the user that the job has started. It is also helpful to send job status updates when the wrapper script is
about to start downloading files, about to kick off the analysis itself, and about to start uploading output
files. These job status updates can be very helpful when troubleshooting failed jobs in the DE, so it's recommended to
err on the side of sending too many status updates rather than too few.

The other two status codes, `completed` and `failed`, are intended to be sent exactly once just before the wrapper
script exits. If the job completes successfully then the status code should be `completed`. Otherwise, the status code
should be `failed`. The DE uses these status codes to notify the user that the job has finished and to trigger cleanup
tasks, so it's important to make sure that one of these updates is sent for each analysis if possible.

### Example Implementation

If you're using Python, you can use this class from the [reference implementation][2] to send status updates:

``` python
# A clean way to print to stderr.
def eprint(*args, **kwargs):
    print(*args, file=sys.stderr, **kwargs)

# This is a simple class used to send job status update messages.
class JobStatusUpdater:
    def __init__(self, url):
        self.url = url

    def print_update(self, status, message):
        print("{0}: {1}".format(status, message))

    def send_update(self, status, message):
        self.print_update(status, message)
        body = {"state": status, "message": message, "hostname": socket.gethostname()}
        r = requests.post(self.url, json=body)
        if r.status_code < 200 or r.status_code > 299:
            eprint("unable to send job status update: {0} {1}: {2}".format(status, message), r.content)

    def failed(self, message):
        self.send_update("failed", message)

    def running(self, message):
        self.send_update("running", message)

    def completed(self, message):
        self.send_update("completed", message)
```

These example bash functions provide similar functionality:

``` shell
STATUS_UPDATE_URL="https://rest.example.org/job/job-id/status" # URL comes from config JSON

send_status() {
   STATUS=$1
   MSG=$2

   curl -s -d "{\"state\": \"${STATUS}\", \"message\": \"${MSG}\", \"hostname\": \"$(hostname)\"}" "$STATUS_UPDATE_URL"

   if [[ $? != 0 ]]; then
       echo "Failed to send status update ${STATUS}: ${MSG}"
   fi
}

status_failed() {
   send_status "failed" "$1"
}

status_running() {
   send_status "running" "$1"
}
```

## Retrieving Input Files

Input files can retrieved using `iget`, which is the method that will be described here. Other file retrieval methods
are available, but they are beyond the scope of this documentation.

The first step in retrieving a file from iRODS is to extract the ticket string and path from the input ticket list
file. Once you have the ticket and path, `iget` can be executed as follows:

```
iget -rt <ticket> "<path>"
```

Note: if you're using a command shell to execute `iget` then the quotes around the path are important if the path
contains whitespace. In cases where the wrapper script is not a UNIX shell, the arguments can usually explicitly be
passed in as a list. For example, this function can be used in Python:

``` python
# Download a file or directory from iRODS.
def download_file(ticket, src):
    rc = subprocess.call(["iget", "-rt", ticket, src])
    if rc != 0:
        raise Exception("could not download {0}".format(src))
```

## Running the Job

Once the input files have been downloaded, the job itself can be executed. In many cases, the job is executed as a
subprocess of the wrapper script. This is a useful practice because it provides a clear boundary between the code that
interacts with the DE and the code that performs the analysis itself. This Python example simply calles `wc` passing in
the arguments retrieved from the configuration file:

``` python
# Run the word count job.
def run_job(arguments, output_filename, error_filename):
    with open(output_filename, "w") as out, open(error_filename, "w") as err:
        rc = subprocess.call(["wc"] + arguments, stdout=out, stderr=err)
        if rc != 0:
            raise Exception("wc returned exit code {0}".format(rc))
```

## Uploading Output Files

Uploading output files is a lot more complicated for two reasons. First, uploading output files in nested directories
requires some additional steps. Second, file permissions must be altered after the files are uploaded.

The basic algorithm used to upload the files is as follows:

- For each file to upload:
  - Use `iput -t <ticket>` to upload the file.
  - Use `ichmod` to grant file ownership to the user who submitted the job.
  - Use `ichmod` to revoke file ownership from the current iRODS user.
- For each directory to upload:
  - Use `iput -rt <ticket>` to create the directory, ignoring any errors. (Permissions errors will occur when it tries
    to upload the directory contents and `imkdir` can't be used to create the directory because it doesn't support
    tickets.)
  - For each file in the directory:
    - Use `iput` to upload the file. (The ticket isn't used because the newly created directory is owned by the current
      iRODS user.)
  - For each subdirectory in the directory:
    - Use `-put -r` to upload the subdirectory and its contents. (Once again the ticket isn't used.)
  - Use `ichmod -r` to grant directory ownership to the user who submitted the job.
  - Use `ichmod -r` to revoke directory ownership from the current iRODS user.

An example Python script can be found [here][4].

## JSON Parsing

If you're writing your wrapper script using a Unix shell, it's helpful to have a versatile tool for parsing JSON
files. One tool that we use for this purpose at CyVerse is [jq][5].

## Caveats

### File Creator Problem

There's a slight issue with the way files are uploaded in this short-term solution. The username passed in the
configuration file is not `anonymous`, but since the password is empty the account is effectively an anonymous
account. When a file is uploaded using tickets, the file creator in iRODS is always set to the currently authenticated
user. This means that anyone with access to the iRODS username passed in the job configuration file has the ability to
grant themselves permission to the output files from any job that was executed at OSG.

This problem is probably best illustrated using an example. Suppose the username that is used to log into iRODS by a job
is `foo` and the person who submitted the job has a username of `someuser`. The job runs successfully and a file called
`secret-stuff.txt` is produced by the job and uploaded to iRODS. The upload process happens as follows:

- `secret-stuff.txt` is uploaded to the job output directory and file ownership is granted to `foo`
- the wrapper script grants ownership permission on `secret-stuff.txt` to `someuser`
- the wrapper script revokes ownership permission on `secret-stuff.txt` from `foo`

So far, everything is okay. The permissions on the file are correct and everyone who should be able to view the file can
do so. There is one slight problem, however: iRODS maintains a record of the user who created the file and the creator
is _always_ permitted to grant themselves permissions on any file.

Now suppose that a nefarious user, `dr-nefario`, learns of an interesting analysis and is aware that `foo` is the
creator of the output file and that account has the empty string for a password. This means that `dr-nefario` can log
into iRODS as `foo` and use the icommands to grant permissions on the file to anyone.

This is one reason to consider using custom credentials to log into iRODS; if the password is not empty then
`dr-nefario` would have to know both the username and the password of the account that uploaded the file (or one of the
accounts that currently has access to the file) in order to be able to access the file.

The drawback to using custom credentials to log into iRODS, however, is that it's difficult to use those credentials
without exposing them to anyone. If a nefarious user obtains access to those credentials then a similar problem can
still occur.

The DE development team has some ideas for avoiding this issue, but these ideas won't be implemented immediately. We
plan to have this problem fixed in the medium-term solution for OSG integration.

[1]: https://support.opensciencegrid.org/support/solutions/articles/12000024676-docker-and-singularity-containers
[2]: https://github.com/cyverse/docker-builds/blob/master/osg-word-count/wrapper
[3]: https://github.com/cyverse/docker-builds/blob/master/osg-word-count/Dockerfile
[4]: https://github.com/cyverse/docker-builds/blob/master/osg-word-count/upload-files
[5]: https://stedolan.github.io/jq/
