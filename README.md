# nhttpsnoop: Trace Node.js HTTP server activity

Synopsis:

    usage: nhttpsnoop [-ln] [-o col,...] [-p pid,...]

nhttpsnoop traces Node.js HTTP server activity.  By default, all requests from
all Node.js servers on the system are traced, and each one is displayed as it
completes:

    # nhttpsnoop
    TIME            PID    LATENCY METHOD PATH
    [  0.508108]  10110    1.998ms GET    /wf_runners/869de259-5bdf-4efe
    [  0.490785]  10110    1.386ms GET    /search/wf_jobs
    [  0.363200]  37476    3.036ms HEAD   /agentprobes

With -l, a record is emitted when requests are received as well in addition to
when the response is sent:

    # nhttpsnoop -l
    TIME            PID       LATENCY METHOD PATH                
    [  6.807218]  10110 ->          - GET    /wf_runners/869de259-5bdf-4efe
    [  6.869112]  10110 <-    2.061ms GET    /wf_runners/869de259-5bdf-4efe
    [  6.888441]  10110 ->          - GET    /search/wf_jobs
    [  6.899112]  10110 <-    1.386ms GET    /search/wf_jobs
    [  6.808441]  37476 ->          - HEAD   /agentprobes
    [  6.869112]  37476 <-    3.036ms HEAD   /agentprobes

You can also select individual fields for display with -o, much like ps(1) and
other tools:

    # nhttpsnoop -otime,method,path
    TIME         METHOD PATH                
    [  2.381936] GET    /wf_runners/869de259-5bdf-4efe
    [  2.965854] GET    /search/wf_jobs
    [  2.960546] GET    /agentprobes

See below for the list of columns available.

Finally, you can select only individual processes with -p.  See below for details.

## Options

    -l            Display two lines for each request: one when the request is
                  received, and one when the response is sent (instead of just
                  one line when the response is sent).  Note that these lines
                  may not be adjacent, since multiple requests may be received
                  before any responses are sent if the request handling is
                  asynchronous.

    -n            Don't actually run DTrace, but instead just print out the D
                  script that would be used.

    -o col,...    Display the specified columns instead of the default output.
                  Available columns include:

                latency time in microseconds between the request being received
                        and the response being sent.  With -l, this field is
                        blank for the line denoting the start of the request.

                method  Request's HTTP method

                path    Request's HTTP URL path (excludes query parameters)

                pid     process identifier

                raddr   Client's IPv4 address

                rport   Client's TCP port

                time    relative time of the event from when nhttpsnoop started

                url     Request's full HTTP URL, including query parameters

    -p pid,...    Only trace the specified processes.

## Selecting processes

"-p" is the only way to select processes, but you can use this with pgrep(1)
for more sophisticated selections:

    # nhttpsnoop -p "$(pgrep -f restify)"  # select only "restify" processes
    # nhttpsnoop -p "$(pgrep -z myzone)"   # select processes in zone "myzone"
    # nhttpsnoop -p "$(pgrep -u dap)"      # select processes with user "dap"
    
With "-p", all Node processes are actually traced, but only requests from the
selected processes are printed.

## DTrace

This tool uses the Node.js DTrace provider and dtrace(1M).  You must have
permissions to run dtrace(1M) and use this provider.  It works on illumos-based
systems, and OS X should work once [Node issue
3617](https://github.com/joyent/node/issues/3617) is resolved.

If you see an error like the following:

    dtrace: failed to compile script /var/tmp/nhttpsnoop.81961.d: line 20: failed to resolve translated type for args[1]
    nhttpsnoop: failed to invoke dtrace(1M)

then "dtrace" didn't find the expected Node translator file.  Node installs
this file into $PREFIX/lib/dtrace/node.d, but dtrace only knows about
/usr/lib/dtrace/node.d.  So if you have installed Node into a prefix other than
/usr, then you must specify this via DTRACE\_OPTS in your environment:

    # export DTRACE_OPTS='-L $PREFIX/lib/dtrace'

where `$PREFIX` is where you've installed Node (e.g., "/usr/local" or
"/opt/local").  nhttpsnoop passes DTRACE\_OPTS to "dtrace", which in this case
causes "dtrace" to look for the node.d translator file in the directory
specified by -L.  See the "-L" option in dtrace(1M) for details.

On older versions of illumos, you may see errors like this:

    # dtrace: error on enabled probe ID 34 (ID 67807: node6112:node:_ZN4node26DTRACE_HTTP_SERVER_REQUESTERKN2v89ArgumentsE:http-server-request): invalid kernel access in action #2 at DIF offset 348

This has been fixed in versions of SmartOS after 20120531T220306Z.  See "uname
-v" to see what release you're running.
