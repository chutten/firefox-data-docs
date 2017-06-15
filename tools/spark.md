Introduction
------------

Spark is a data processing engine designed to be fast and easy to use.
We have setup Jupyter workbooks that use Spark to analyze our Telemetry
data. Jupyter workbooks can be easily shared and updated among
colleagues to enable richer analysis than SQL alone.

The Spark clusters can be spun up on analysis.telemetry.mozilla.org,
which is abbreviated as atmo. The Spark Python API is called pyspark.

Setting Up a Spark Cluster On atmo
----------------------------------

1.  Go to analysis.telemetry.mozilla.org
2.  Click “Launch an ad-hoc Spark cluster”.
3.  Enter some details:
    1.  The “Cluster Name” field should be a short descriptive name,
        like “chromehangs analysis”.
    2.  Set the number of workers for the cluster. Please keep in mind
        to use resources sparingly; use a single worker to write and
        debug your job.
    3.  Upload your SSH public key.

4.  Click “Submit”.
5.  A cluster will be launched on AWS preconfigured with Spark, IPython
    and some handy data analysis libraries like pandas and matplotlib.

Once the cluster is ready, you can tunnel IPython through SSH by
following the instructions on the dashboard, and running the ssh shell
command. For example:

ssh -i \~/.ssh/id\_rsa -L 8888:localhost:8888
hadoop@ec2-54-70-129-221.us-west-2.compute.amazonaws.com

Finally, you can launch IPython in Firefox by visiting
<http://localhost:8888>.

The Python Jupyter Notebook
---------------------------

When you access <http://localhost:8888>, two example Jupyter notebooks
are available to peruse. To create a new Jupyter notebook, select
new -\> Python 2.

Starting out, we recommend looking through the [Telemetry Hello
World](https://github.com/mozilla/mozilla-reports/blob/master/tutorials/telemetry_hello_world.kp/orig_src/Telemetry%20Hello%20World.ipynb)
notebook. It gives a nice overview of Jupyter and analyzing telemetry
data using pyspark and the RDD API.

### Using Jupyter

Jupyter Notebooks contain a series of cells. Each cell contains code or
markdown. To switch between the two, use the dropdown at the top. To run
a cell, use shift-enter; this either compiles the markdown or runs the
code. To create new cell, select Insert -\> Insert Cell Below.

A cell can output text or plots. To output plots inlined with the cell,
run the following command, usually below your import statements:

` %pylab inline`

The notebook is setup to work with Spark. See the "Using Spark" section
for more information.

### Schedule a periodic job

Scheduled Spark jobs allow a Jupyter notebook to be updated
consistently, making a nice and easy-to-use dashboard.

To schedule a Spark job:

1.  Visit the analysis provisioning dashboard at
    telemetry-dash.mozilla.org and sign in using Persona with an
    @mozilla.com email address.
2.  Click “Schedule a Spark Job”.
3.  Enter some details:
    1.  The “Job Name” field should be a short descriptive name, like
        “chromehangs analysis”.
    2.  Upload your IPython notebook containing the analysis.
    3.  Set the number of workers of the cluster in the “Cluster Size”
        field.
    4.  Set a schedule frequency using the remaining fields.

Now, the notebook will be updated automatically and the results can be
easily shared. Furthermore, all files stored in the notebook's local
working directory at the end of the job will be automatically uploaded
to S3, which comes in handy for simple ETL workloads for example.

For reference, see [Simple Dashboard with Scheduled Spark Jobs and
Plotly](https://robertovitillo.com/2015/03/13/simple-dashboards-with-scheduled-spark-jobs-and-plotly).

### Sharing a Notebook

Jupyter notebooks can be shared in a few different ways.

#### Sharing a Static Notebook

An easy way to share is using a gist on github.

1.  Download file as .ipynb
2.  Upload to a gist on [gist.github.com](http://gist.github.com)
3.  Enter the gist URL at [Jupyter
    nbviewer](http://nbviewer.jupyter.org/)
4.  Share with your colleagues!

#### Sharing a Scheduled Notebook

Setup your scheduled notebook. After it's run, do the following:

1.  Go to the 'Schedule a Spark job' tab in atmo
2.  Get the URL for the notebook (under 'Currently Scheduled Jobs')
3.  Paste that URL into [Jupyter nbviewer](http://nbviewer.jupyter.org/)

Zeppelin Notebooks
------------------

We also have \*experimental\* support for [Apache
Zeppelin](http://zeppelin.apache.org/) notebooks. The notebook server
for that is running on port 8890, so you can connect to it just by
tunneling the port (instead of port 8888 for Jupyter). For example:

ssh -i \~/.ssh/id\_rsa -L 8890:localhost:8890
hadoop@ec2-54-70-129-221.us-west-2.compute.amazonaws.com

Using Spark
-----------

Spark is a general-purpose cluster computing system - it allows users to
run general execution graphs. APIs are available in Python, Scala, and
Java. The Jupyter notebook utilizes the Python API. In a nutshell, it
provides a way to run functional code (e.g. map, reduce, etc.) on large,
distributed data.

Check out [Spark Best
Practices](https://robertovitillo.com/2015/06/30/spark-best-practices/)
for tips on using Spark to it's full capabilities.

### SparkContext (sc)

Access to the Spark API is provided through SparkContext. In the Jupyter
notebook, this is the \`sc\` object. For example, to create a
distributed RDD of monotonically increasing numbers 1-1000:

`   numbers = range(1000)`\
`   #no need to initialize sc in the Jupyter notebook`\
`   numsRdd = sc.parallelize(numbers)`\
`   nums.take(10) #no guaranteed order`

### Spark RDD

The Resilient Distributed Dataset (RDD) is Spark's basic data structure.
The operations that are performed on these structures are distributed to
the cluster. Only certain actions (such as collect() or take(N)) pull an
RDD in locally.

RDD's are nice because there is no imposed schema - whatever they
contain, they distribute around the cluster. Additionally, RDD's can be
cached in memory, which can greatly improve performance of some
algorithms that need access to data over and over again.

Additionally, RDD operations are all part of a directed, acyclic graph.
This gives increased redundancy, since Spark is always able to recreate
an RDD from the base data (by rerunning the graph), but also provides
lazy evaluation. No computation is performed while an RDD is just being
transformed (a la map), but when an action is taken (e.g. reduce, take)
the entire computation graph is evaluated. Continuing from our previous
example, the following gives some of the peaks of a sin wave:

`   import numpy as np`\
`   #no computation is performed on the following line!`\
`   sin_values = numsRdd.map(lambda x : np.float(x) / 10).map(lambda x : (x, np.sin(x)))`\
`   #now the entire computation graph is evaluated`\
`   sin_values.takeOrdered(5, lambda x : -x[1])`

For jumping into working with Spark RDD's, we recommend reading the
[Spark Programming
Guide](https://spark.apache.org/docs/latest/programming-guide.html).

### Spark SQL and Spark Dataframes/Datasets

Spark also supports traditional SQL, along with special data structures
that require schemas. The Spark SQL API can be accessed with the
\`spark\` object. For example:

`   longitudinal = spark.sql('SELECT * FROM longitudinal')`

creates a DataFrame that contains all the longitudinal data. A Spark
DataFrame is essentially a distributed table, a la Pandas or R
Dataframes. Under the covers they are an RDD of Row objects, and thus
the entirety of the RDD API is available for DataFrames, as well as a
DataFrame specific API. For example, a sql-like way to get the count of
a specific OS:

`   longitudinal.select("os").where("os = 'Darwin'").count()`

To Transform the DataFrame object to an RDD, simply do:

`  longitudinal_rdd = longitudinal.rdd`

In general, however, the DataFrames are performance optimized, so it's
worth the effort to learn the DataFrame API.

For more overview, see the [SQL Programming
Guide](https://spark.apache.org/docs/latest/sql-programming-guide.html).
See also the [Longitudinal
Tutorial](https://github.com/mozilla/emr-bootstrap-spark/blob/master/examples/Longitudinal%20Dataset%20Tutorial.ipynb),
one of the available example notebooks when you start a cluster.

### Available Data Sources for SparkSQL

For information about available queryable data sources (e.g.
Longitudinal dataset), see [Telemetry Datasets
Documentation](https://wiki.mozilla.org/Telemetry/Available_Telemetry_Datasets_and_their_Applications).

These datasets are optimized for fast access, and will far out-perform
analysis on the raw telemetry ping data.

### Accessing the Spark UI

Go to localhost:8888/spark after ssh-ing into the spark cluster to see
the Spark UI. It has information about job statuses and task completion,
and may help you debug your job.

The MozTelemetry Library
------------------------

We have provided a library that gives easy access to the raw telemetry
ping data. For example usage, see the [Telemetry Hello
World](http://reports.telemetry.mozilla.org/post/tutorials/telemetry_hello_world.kp)
example notebook. Detailed documentation for the library can be found at
the [Python MozTelemetry
Documentation](http://python-moztelemetry.readthedocs.io).

### Using the Raw Ping Data

First off, import the moztelemetry library using the following:

`   from moztelemetry import get_pings`

If you need any other functions, just comma separate them with
get\_pings.

The ping data is an RDD of JSON elements. For example, using the
following:

`   pings = get_pings(sc, app="Firefox", channel="nightly", build_id=("20160901000000", "20160901999999"), fraction=0.01)`

returns an RDD of 1/100th of Firefox Nightly JSON pings for all builds
from September 1 2016. Now, because it's JSON, pings are easy to access.
For example, to get the count of each OS type:

`   os_names = pings.map(lambda x : (x['environment']['system']['os']['name'], 1))`\
`   os_counts = os_names.reduceByKey(lambda x, y : x+y)`\
`   os_counts.collect()`

Alternatively, moztelemetry provides the \`get\_pings\_properties\`
function, which will gather the data for you:

`   subset = get_pings_properties(pings, ["environment/system/os/name"])`\
`   subset.map(lambda x : (x["environment/system/os/name"], 1)).reduceByKey(lambda x, y : x+y).collect()`

FAQ
---

Please add more FAQ as questions are answered by you or for you.

### How can I load parquet datasets in a Jupyter notebook?

Use spark.read.parquet, e.g.:

` dataset = spark.read.parquet("s3://the_bucket/the_prefix/the_version")`

### I got a REMOTE HOST IDENTIFICATION HAS CHANGED! error

AWS recycles hostnames, so removing the offending key from
\$HOME/.ssh/known\_hosts will remove the warning. You can find the line
to remove by finding the line in the output that says

` Offending key in /path/to/hosts/known_hosts:2`

Where 2 is the line number of the key that can be deleted. Just remove
that line, save the file, and try again.

### Why is my notebook hanging?

There are a couple common causes for this:

1\. Currently, our Spark notebooks can only run a single Python kernel at
a time. If you open multiple notebooks on the same cluster and try to
run both, the second notebook will hang. Be sure to close notebooks
using "Close and Halt" under the "File" dropdown.

2\. The connection from PySpark to the Spark driver might be lost.
Unfortunately the best way to recover from this for the moment seems to
be spinning up a new cluster.

3\. Canceling execution of a notebook cell doesn't cancel any spark jobs
that might be running in the background. If your spark commands seem to
be hanging, try running \`sc.cancelAllJobs()\`.

### How can I keep running after closing the notebook?

For long-running computation, it might be nice to close the notebook
(and the ssh session) and look at the results later. Unfortunately,
**all cell output will be lost when a notebook is closed** (for the
running cell). To alleviate this, there are a few options:

1\. Have everything output to a variable. These values should still be
available when you reconnect.

2\. Put %%capture at the beginning of the cell to store all output. [See
the
documentation](https://ipython.org/ipython-doc/3/interactive/magics.html#cellmagic-capture).

### How do I load an external library into the cluster?

Assuming you've got a url for the repo, you can create an egg for it
this way:

` !git clone `<repo url>` && cd `<repo-name>` && python setup.py bdist_egg`\
` sc.addPyFile('`<repo-name>`/dist/my-egg-file.egg')`

Alternately, you could just create that egg locally, upload it to a web
server, then download and install it:

` import requests`\
` r = requests.get('`<url-to-my-egg-file>`')`\
` with open('mylibrary.egg', 'wb') as f:`\
`   f.write(r.content)`\
` sc.addPyFile('mylibrary.egg')`

You will want to do this **before** you load the library. If the library
is already loaded, restart the kernel in the ipython notebook.