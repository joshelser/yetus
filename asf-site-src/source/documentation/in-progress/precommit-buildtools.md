<!---
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

Build Tool Support
===================

test-patch has the ability to support multiple build tools.  Build tool plug-ins have some extra hooks to do source and object maintenance at key points. Every build tool plug-in must have one line in order to be recognized:

```bash
add_build_tool <pluginname>
```

# Global Variables

* BUILDTOOLCWD

    - If the build tool does not always run from the ${BASEDIR} and instead needs to change the current working directory to work on a specific module, then set this to true.  The default is false.

* UNSUPPORTED\_TEST

    - If pluginname\_modules\_worker is given a test type that is not supported by the build system, set UNSUPPORTED\_TEST=true.  If it is supported, set UNSUPPORTED\_TEST=false.

For example, the gradle build tool does not have a standard way to execute checkstyle. So when checkstyle is requested, gradle\_modules\_worker sets UNSUPPORTED\_TEST to true and returns out of the routine.

# Required Functions

* pluginname\_buildfile

    - This should be an echo of the file that controls the build system.  This is used for module determination. If the build system wishes to disable module determination, it should echo with no args.

* pluginname\_executor

    - This should be an echo of how to run the build tool, any extra arguments, etc.

* pluginname\_modules\_worker

    - Input is the branch and the test being run.  This should call `modules_workers` with the generic parts to run that test on the build system.  For example, if it is convention to use 'test' to trigger 'unit' tests, then `module_workers` should be called with 'test' appended onto its normal parameters.

* pluginname\_builtin_personality\_modules

    - Default method to determine how to enqueue modules for processing.  Note that personalities may override this function.

* pluginname\_builtin_personality\_file\_tests

    - Default method to determine which tests to trigger.  Note that personalities may override this function.

# Optional Functions

* pluginname\_postapply\_install

    - After the install step, this allows the build tool plug-in to do extra work.

* pluginname\_parse\_args

    - executed prior to any other above functions except for pluginname\_usage. This is useful for parsing the arguments passed from the user and setting up the execution environment.

* pluginname\_initialize

    - After argument parsing and prior to any other work, the initialize step allows a plug-in to do any precursor work, set internal defaults, etc.

* pluginname\_count\_(test)\_probs

    - Certain language test plug-ins require assistance from the build tool to count problems in the compile log due to some tools having custom handling for those languages.  The test plug-in name should be in the (test) part of the function name.

* pluginname\_(test)_calcdiffs

    - Some build tools (e.g., maven) use custom output for certain types of compilations (e.g., java).  This allows for custom log file difference calculation used to determine the before and after views.

* pluginname\_docker\_support

    - If this build tool requires extra settings on the `docker run` command line, this function should be defined and write those options into a file called `${PATCH_DIR}/buildtool-docker-params.txt`.  This is particularly useful for things like mounting volumes for repository caches.

       **WARNING**: Be aware that directories that do not exist MAY be created by root by Docker itself under certain conditions.  It is HIGHLY recommend that `pluginname_initialize` be used to create the necessary directories prior to be used in the `docker run` command.

# Ant Specific

## Command Arguments

test-patch always passes -noinput to Ant.  This forces ant to be non-interactive.

## Docker Mode

In Docker mode, the `${HOME}/.ivy2` directory is shared amongst all invocations.

# Gradle Specific

The gradle plug-in always rebuilds the gradlew file and uses gradlew as the method to execute commands.

In Docker mode, the `${HOME}/.gradle` directory is shared amongst all invocations.

# Maven Specific

## Command Arguments

test-patch always passes --batch-mode to maven to force it into non-interactive mode.  Additionally, some tests will also force -fae in order to get all of messages/errors during that mode. Some tests are executed with -DskipTests.  Additional arguments should be handled via the personality.

##  Per-instance Repositories

Under many common configurations, maven (as of 3.3.3 and lower) may not properly handle being executed by multiple processes simultaneously, especially given that some tests require the `mvn install` command to be used.

To assist, `test-patch` supports a `--mvn-custom-repo` option to set the `-Dmaven.repo.local` value to a per-instance repository directory keyed to the project and branch being used for the test.  If the `--jenkins` flag is also passed, the instance will be tied to the Jenkins `${EXECUTOR_NUMBER}` value.  Otherwise, the instance value will be randomly generated via `${RANDOM}`.  If the repository has not been used in 30 days, it will be automatically deleted when any test run for that project (regardless of branch!).

By default, `test-patch` uses `${HOME}/yetus-m2` as the base directory to store these custom maven repositories.  That location may be changed via the `--mvn-custom-repos-dir` option.

The location of the `settings.xml` may be changed via the `--mvn-settings` option.

## Docker Mode

In Docker mode, `${HOME}/.m2` is shared amongst all invocations.  If `--mvn-custom-repos` is used, all of `--mvn-custom-repos-dir` is shared with all invocations.  The per-instance directory will be calculated and configured after Docker has launched.

## Test Profile

By default, test-patch will pass -Ptest-patch to Maven. This will allow you to configure special actions that should only happen when running underneath test-patch.

## Custom Maven Tests

* Maven will trigger a maven install as part of the precompile.
* Maven will test eclipse integration as part of the postcompile.
* If src/site is modified, maven site tests are executed.
