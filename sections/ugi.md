<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
  
# UGI


If there is one class guaranteed to strike fear into anyone with experience in Hadoop+Kerberos code it is `UserGroupInformation`, abbreviated to "UGI"


## What does UGI do?

Here sre some of the things it can do

1. Handles the initial login process, using any environmental `kinit`-ed tokens or a keytab.
1. Spawn off a thread to renew the TGT
1. Provides an operation for-on demand verification/re-init of kerberos tickets details before issuing a request.




## UGI strengths

* It's one place for almost all Kerberos/User authentication to live.
* Being fairly widely used, once you've learned it, your knowledge works through
the entire Hadoop stack.
 

## UGI troublespots

* It's a singleton. Don't expect to have one "real user" per process. 
This does sort of makes sense. Even a single service has its "service" identity; as the 

* Once initialized, it stays initialized *and cannot be reset*.
This makes it critical to load in your configuration information including keytabs and principals,
before that first initialization of the UGI.
(There is actually a `UGI.reset()` call, but it is package scoped and purely to allow tests to
reset the information).
* UGI initialization can take place in code which you don't expect.
 A specific example is in the Hadoop filesystem APIs.
 Create a Hadoop filesystem instance and UGI is likely to be inited immediately, even if it is a local file:// reference.
 As a result: init before you go near the filesystem, with the principal you want.
* It has to do some low-level reflection-based access to Java-version-specific Kerberos internal classes.
This can break across Java versions, and JVM implementations. Specifically Java 8 has classes that Java 6 doesn't; the IBM JVM is very different.
* All its exceptions are basic `IOException` instancss, so hard to match on without looking at the text, which is very brittle.
* Some invoked operations are relayed without the stack trace (this should now be fixed).
* Diagnostics could be improved. (this is one of those British understatements, it really means "it would be really nice if you could actually get any hint as to WTF is going inside the class as otherwise you are left with nothing to go on other than some message that a user at a random bit of code wasn't authorized)

The issues related to diagnostics, logging, exception types and inner causes could be addressed. It would be nice to also have an exception cached at init time, so that diagnostics code could even track down where the init took place. Volunteers welcome. That said, here are some bits of the code where patches would be vetoed

* The text of exceptions. We don't know what is scanning for that text, or what documents go with them.
* All exceptions must be subclasses of IOException.
* Logging must not leak secrets, such as tokens.


## Core UGI Operations


### `isSecurityEnabled()`

One of the foundational calls is the `UserGroupInformation.isSecurityEnabled()`

It crops up in code like this

    if(!UserGroupInformation.isSecurityEnabled()) {
        stayInALifeOfNaiveInnocence();
     } else {
        sufferTheEnternalPainOfKerberos();
     }

Having two branches of code, the "insecure" and "secure mode" is actually dangerous: the entire
security-enabled branch only ever gets executed when run against a secure Hadoop cluster

**Important**

*If you have separate secure and insecure codepaths, you must test on a secure cluster*
*alongside an insecure one. Otherwise coverage of code and application state will be*
*fundamentally lacking.*

*Unless you put in the effort, all your unit tests will be of the insecure codepath.*

*This means there's an entire codepath which won't get exercised until you run integration*
*tests on a secure cluster, or worse: until you ship.*

What to do? Alongside the testing, the other strategy is: keep the differences between
the two branches down to a minimum. If you look at what YARN does, it always uses
renewable tokens to authenticate communication between the YARN Resource Manager and
a deployed application. As a result, one codepath for token creation, while token propagation
and renewal is automatically tested on all applications.

Could your applications do the same? Certainly as far as token- and delegation-token based
mechanisms for callers to show that they have been granted access rights to a service.


## Environment variable managed UGI Initialization

There are some environment variables which configure UGI.

| HADOOP_PROXY_USER | identity of a proxy user to authenticate as |
| HADOOP_TOKEN_FILE_LOCATION | local path to a token file |

Why Environment variables? They offer some features

1. Hadoop environment setup scripts can set them
1. When launching YARN containers, they may be set as environment variables.

As the UGI code is shared across all clients of HDFS and YARN; these environment
variables can be used to configure *any* application which communicates with Hadoop
services via the UGI-authenticated clients. Essentially: all Java IPC clients and
those REST clients using the Hadoop-implemented REST clients