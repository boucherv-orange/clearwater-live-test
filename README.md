Clearwater Live Test Framework
==============================

This framework allows scripted testing of a Clearwater deployment to be used as a system-wide test of a fix/feature.

Project Clearwater is an open-source IMS core, developed by [Metaswitch Networks](http://www.metaswitch.com) and released under the [GNU GPLv3](http://www.projectclearwater.org/download/license/). You can find more information about it on [our website](http://www.projectclearwater.org/) or [our wiki](http://clearwater.readthedocs.org/en/latest/).

Pre-Requisites
--------------

The test framework requires Ruby 1.9.3 and bundler to be installed.

    sudo apt-get install build-essential git --yes
    curl -L https://get.rvm.io | bash -s stable
    source ~/.rvm/scripts/rvm
    rvm autolibs enable
    rvm install 1.9.3
    rvm use 1.9.3

Installation
------------

To install the framework, clone the repository:

    git clone git@github.com:Metaswitch/clearwater-live-test.git

Then type `bundle install` inside the newly-created `clearwater-live-test` folder to install the required gems.

Running The Tests
-----------------

To run the tests against a deployment, use:

    rake test[<deployment_name>] SIGNUP_CODE=<code>

where `<code>` is the signup code you supplied when configuring Ellis (see /etc/clearwater/config on the Ellis server).

There are various modifiers you can use to determine which subset of tests you wish to run, to use these modifiers, add them to the end of the `rake` command:

 - `ELLIS_API_KEY=<key>` - this should match [the ellis_api_key configuration on Ellis](https://github.com/Metaswitch/clearwater-docs/wiki/Manual%20Install#configuring-the-inter-node-hostnamesip-addresses) and allows the test scripts to provision specific SIP URIs on the system instead of having them randomly assigned by Ellis.
 - `TESTS="<glob>"` - to only run tests whos name matches the given glob.
 - `PSTN=true` - to run the PSTN-specific tests (your deployment must have PSTN numbers allocated).
 - `LIVENUMBER=<number>` - to allow running of tests that dial out to real devices (your deployment must have an IBCF node and a working PSTN) the live number given may be dialled as part of running the test and the test will expect it to be answered (so make it a real one!).
 - `REPEAT=<number>` - to allow the suite of tests to be run multiple times.
 - `TRANSPORT=<transports>` - Comma-separated transports to test with.  Allowed tranports are `TCP` and `UDP`.  If not specified, all tests will be run twice, for each transport type.
 - `PROXY=<host>` - to force the tests to run against a particular Bono instance. Useful when running against an AIO node, or when the Bono domain isn't DNS resolvable.
 - `ELLIS=<host>` - to override the default FQDN for Ellis.  Useful when running against an AIO node, or when the Ellis domain isn't DNS resolvable.
 - `HOSTNAME=<host>` - publicly accessible hostname of the machine running the tests, used for the dummy AS.
 - `EXPIRES=<number>` - maximum Expires header expected from Sprout, used for the dummy AS.
 - `GEMINI=<host>` - hostname of the the Gemini cluster. If the Gemini application server is integrated with Sprout rather than running as a standalone, this should be set to the Sprout cluster.
 - `MEMENTO_SIP=<host>` - hostname of the Memento (SIP) cluster. If the Memento application server is integrated with Sprout rather than running as a standalone, this should be set to the Sprout cluster.
 - `MEMENTO_HTTP=<host>` - hostname of the Memento (HTTP) cluster.
 - `PROVISIONAL_RESPONSES_IGNORED=TRUE` - set this to interoperate with devices that absorb second and subsequent provisional responses (so that if a call is forked and both endpoints send a 180 Ringing, only one will reach the caller)
 - `EXCLUDE_TESTS="test1 (TCP),test2 (UDP)"` - a comma-separated list of tests to ignore. Useful for working around known bugs with tests in particular environments (e.g. skipping the B2BUA test in cases where the EC2 security group settings won't allow it)
 - `ELLIS_USER=<email>` - to override the default email used for Ellis (live.tests@example.com). Useful to allow multiple live test instances to run simultaneously without deleting each other's lines.
 - `SNMP=Y` - to verify the SNMP statistics produced in the test run.
 - `OFF_NET_TEL=<number>` - an off-net number that should be routed back to this machine, for testing BGCF functionality. See [the BGCF Testing doc](BGCF_Testing.md) for more detail.

For example, to run all the call barring tests (including the international number barring tests) on the test deployment, run:

    rake test[test.cw-ngv.com] TESTS="Call Barring*" PSTN=true

Framework Structure
---------------

Tests in the framework are essentially short Ruby programs. These programs use the Quaff library to talk over SIP to Clearwater nodes for calls, and the rest-client library to communicate with Ellis for provisioning.

Any Quaff error logs are saved off in the event of a failure and are very useful for tracking down issues/bugs in the scripts.

The test framework is very punctilious about cleaning up after itself, so there should be no issue with running the tests in any order or running the framework multiple times on one deployment (even at the same time!).

Writing A New Test
------------------

The test definitions are found in `lib/tests/*.rb` and should be pretty self-explanatory.  A basic test structure is as follows:

```
TestDefinition.new("Basic Call - Messages - Pager model") do |t|
  caller = t.add_endpoint
  callee = t.add_endpoint

  t.add_quaff_setup do
    caller.register
    callee.register
  end

  t.add_quaff_scenario do
    call = caller.outgoing_call(callee.uri)

    call.send_request("MESSAGE", "hello world\r\n", {"Content-Type" => "text/plain"})
    call.recv_response("200")
    call.end_call
  end

  t.add_quaff_scenario do
    call2 = callee.incoming_call
    call2.recv_request("MESSAGE")
    call2.send_response("200", "OK")
    call2.end_call
  end

  t.add_quaff_cleanup do
    caller.unregister
    callee.unregister
  end
end
```

This example would create two numbers in ellis/homestead, register them, then send a MESSAGE transaction between them before deregistering and destroying the numbers. Because the deregistration takes place in a cleanup block, this happens even if the main scenario fails or hits an exception.

There are different `skip` functions that can be included in a test. These control whether a test case is run, based on the command line options given at run time:
 
 - `skip_unless_pstn`: PSTN test, requires that `PSTN=true` be passed to rake or the test will be skipped.
 - `skip_unless_live`: Live call test, requires that a live number is given to rake as `LIVENUMBER=...`.
 - `skip_unless_<application server>`: Application Server test, requires that a hostname for the particular server is passed to rake as (for example) `GEMINI=...`.
 - `skip_if_udp`: Used to mark a test that's only valid when using TCP
 - `skip_unless_ellis_api_key`: Test that requires the Ellis API key, for example a test that creates specific endpoints
 - `skip_unless_offnet_tel`: Test that requires an off-net number (specified by `OFF_NET_TEL`) to be routed back to this machine.
 - `skip`: Used to mark out currently broken tests, tests should not be left in this state for longer than necessary.


Creating Endpoints
------------------

As you saw above, a test can create an endpoint in ellis with `test.add_endpoint`.  It may need an endpoint that is not a Clearwater number (for example for off-net calling), in which case `add_fake_endpoint(<DN>, <domain>)` may be used instead.

To create a PSTN number use `test.add_pstn_endpoint`.  These numbers can make calls out to the PSTN and should be used for live calling/international number dialing tests.  When using these numbers, include in the test `skip_unless_pstn` to ensure it is skipped if PSTN numbers are not available on the system under test.

By default, `test.add_endpoint` has a random SIP URI assigned from Ellis' pool of numbers. `test.add_specific_endpoint "2345"` will assign the specific number `sip:2345@DOMAIN`. This should only be used when absolutely necessary for a specific test - it requires the Ellis API key to be provided in order to have sufficient privileges to create arbitrary numbers.

To create a new public identity for a line, use the `test.add_quaff_public_identity` function.  The returned endpoint will share a private ID with the passed in one but will have its own public identity.

When using Clearwater endpoints, a common thing to need to do is to (un)register the endpoint and so `ep.register` and `ep.unregister` have been supplied that return the scenario entries needed to do this (see _Basic Call - Mainline_ for an example of how this is used).

Modifying Simservs
------------------

To change a simservs document for a Clearwater endpoint, use `ep.set_simservs` which takes a hash of options for the document.  See `templates/simservs.xml.erb` for how the options are used and see `EllisEndpoint::default_simservs` for the options that will be used if not specified in your call to `set_simservs`.

Modifying iFCs
------------------

To change the iFC document for a Clearwater endpoint, use `ep.set_ifcs` which takes a hash of options for the document.  See `templates/ifcs.xml.erb` for how the options are used and see `EllisEndpoint::default_ifcs` for the options that will be used if not specified in your call to `set_ifcs`.

Sending Messages
----------------

To send a SIP message, use the `ep.send_request(<method>)` or `ep.send_response(<status code>, <reason phrase>)` commands. These are commands from the Quaff Ruby library for SIP, documented at https://github.com/rkday/quaff.

Receiving Messages
------------------

To receive a SIP message, simple add `ep.recv_response(<status code>)` or `ep.recv_request(<method>)` to the scenario.

Pausing
-------

Pauses are expressed using standard Ruby syntax - `sleep 5`. In general, the live tests are simply blocks of Ruby code using a SIP library, so anything that is possible in Ruby is possible within a testcase.

Acknowledgements
----------------
The Clearwater Live Test Framework depends on the following files from the [sipp project](http://sipp.sourceforge.net/).  These are distributed under the [GPL](http://sipp.sourceforge.net/doc/license.html).

*   g711a.cap - pcap file for test announcement
*   sipp - precompiled binary from git commit 3268f48, with RTP/pcap relay enabled
