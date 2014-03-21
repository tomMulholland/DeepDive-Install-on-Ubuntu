DeepDive Install on Ubuntu 12.04
===========
After having [many problems][1] installing the [DeepDive project][2] on Ubuntu, I decided to write out a detailed guide. These problems were based on the output from the `test.sh` file provided with the source - I can't speak to the functionality of the source just yet (just starting to learn).

Because I messed up a few configuration files, and this is still early in my Ubuntu experience, I decided to reinstall the OS (Precise 12.04) and redo everything from scratch. So, this guide is based off of a clean version of Ubuntu 12.04, after installing all relevant updates (via Update Manager) as of 20-Mar-2014.

DeepDive gives us a few prerequisites: Java, Python 2.X, PostgreSQL, and SBT. Ubuntu 12.04 already has Python 2.X, so we'll worry about the others.

We're going to go with the Ubuntu recommended OpenJDK-7. Enter the following in the terminal.

    sudo apt-get update
    sudo apt-get install openjdk-7-jdk icedtea-7-plugin

Now, let's install SBT. Use the following link to [download the debian file][3] or get it [from the site][4]. SBT has a dependence on curl, so first I'll install that.

    sudo apt-get install curl
    cd /home/tom/Downloads
    sudo dpkg -i sbt.deb

Now I need to install PostgreSQL, which was by far the trickiest part. In this tutorial, I'm assuming that the computer you're working on will also be the PostgreSQL host. It's also important to note that DeepDive uses JSON, which is apparently not supported by PostgreSQL 9.1 and under. To install version 9.3, I'm going to use the instruction given by "Danny" in [this StackExchange post][5], and just change the numbers to 9.3: 

    wget -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
    sudo gedit /etc/apt/sources.list.d/pgdg.list

Add the following line **to the file**, then save and close:

    deb http://apt.postgresql.org/pub/repos/apt/ precise-pgdg main

Note that "precise-pgdg" corresponds to your Ubuntu version. Now let's update and install.

    sudo apt-get update
    sudo apt-get install pgdg-keyring postgresql-9.3

Now we'll install DeepDive. First, I need to install git, since I'm on a fresh version of the OS. Then, the instructions come from the [DeepDive page][6]. I'm going to install DeepDive in my home directory, but if you want it somewhere else, modify the `cd` line.

    sudo apt-get install git
    cd
    git clone https://github.com/dennybritz/deepdive.git
    cd deepdive
    sbt compile

If we run the deepdive test now, it will give us some errors:

cd deepdive
./test.sh

    [info] Run completed in 8 seconds, 322 milliseconds.
    [info] Total number of tests run: 71
    [info] Suites: completed 18, aborted 0
    [info] Tests: succeeded 69, failed 2, canceled 0, ignored 0, pending 3
    [info] *** 2 TESTS FAILED ***
    [error] Failed tests:
    [error] 	org.deepdive.test.integration.LogisticRegressionApp
    [error] 	org.deepdive.test.unit.InferenceManagerSpec
    [error] Error during tests:
    [error] 	org.deepdive.test.unit.PostgresInferenceDataStoreSpec
    [error] 	org.deepdive.test.unit.PostgresExtractionDataStoreSpec
    [error] (test:test) sbt.TestsFailedException: Tests unsuccessful
    [error] Total time: 29 s, completed Mar 20, 2014 6:45:30 PM

To fix this, we need to set up PostgreSQL. First, let's activate the local and TCP/IP connections.

    sudo gedit /etc/postgresql/9.3/main/postgresql.conf

**Modify the following line** in the "Connections and Authentication" from:

`#listen addresses = 'localhost'`

to:

`listen_addresses = 'localhost, 127.0.0.1, 192.168.1.10'`

Note that you should check your own IP address in your network connections and use that instead of mine, which ends in .10.  It's also worth noting that localhost and 127.0.0.1 are equivalent. Now, you'll need to make sure the port 5432 is activated/open on your router. For me, it was something like the following: Access the router from a browser typing 192.168.1.0 -> Virtual servers -> enable port 5432 for IP Address 192.168.1.10

Now we need to set up the postgres superuser for the first time. The following line will open psql as user postgres (thanks to the [Ubuntu-PostgreSQL community Wiki][7])

    sudo -u postgres psql postgres

You should see `postgres=#` and a cursor. Type the following, then enter your password of choice:

    \password postgres

While we're still in psql as the postgres superuser, let's go ahead and create a regular user, who has the same name as your Ubuntu user account. This will make life easier (for me at least). You can use `\du` to check the characteristics of your users.

    CREATE ROLE tom WITH SUPERUSER CREATEDB CREATEROLE REPLICATION LOGIN;
    \du

Now add a password to that user too, then quit psql.

    ALTER ROLE tom WITH PASSWORD 'your_pa$$w0rd';
    \q

Check that you are now user 'tom' again, and not user 'postgres-tom.' If the latter, type `exit`

We now need one additional dependency to be error-free.

    sudo apt-get install gnuplot-x11

Lastly, we need to slightly modify the `test.sh` file in the deepdive directory. It seems like there's a bug, where the test 'forgets' the password you provided in the middle of the run. So, let's just hardwire it in there.

    cd
    gedit deepdive/test.sh

You'll notice the following lines right at the top.

    # Set username and password
    export PGUSER=${PGUSER:-`whoami`}
    export PGPASSWORD=${PGPASSWORD:-}

If you'd like to save the original file, then change the name to `test_original.sh`. We're going to change those lines to the following (as per your case):

    # Set username and password
    export PGUSER=tom
    export PGPASSWORD=your_pa$$w0rd

OK, now go to your deepdive folder, and run the test!

    cd deepdive
    ./test.sh

Success! Sweet, sweet success! You should see the following: 

    [info] Run completed in 21 seconds, 280 milliseconds.
    [info] Total number of tests run: 90
    [info] Suites: completed 20, aborted 0
    [info] Tests: succeeded 90, failed 0, canceled 0, ignored 0, pending 3
    [info] All tests passed.
    [success] Total time: 23 s, completed Mar 20, 2014 7:27:21 PM

Don't ask me what tests are "pending." No idea.


  [1]: http://stackoverflow.com/questions/22469188/deepdive-installation-postgresql-error
  [2]: http://deepdive.stanford.edu/
  [3]: http://repo.scala-sbt.org/scalasbt/sbt-native-packages/org/scala-sbt/sbt/0.13.1/sbt.deb
  [4]: http://www.scala-sbt.org/release/docs/Getting-Started/Setup.html
  [5]: http://askubuntu.com/questions/186610/how-do-i-upgrade-to-postgres-9-2
  [6]: http://deepdive.stanford.edu/doc/installation.html
  [7]: https://help.ubuntu.com/community/PostgreSQL
