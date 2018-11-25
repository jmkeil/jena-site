Title: Fuseki Logging

Fuseki logs operation details and also provides a standard NCSA request log.  

Logging is via [slj4j](http://slf4j.org/) over 
[Apache Log4J](http://logging.apache.org/log4j/).
Logging output is controlled via log4j.

## Server Logs

| Full Log name                   | Usage |
|---------------                  |-------|
| org.apache.jena.fuseki.Server   | General Server Messages              |
| org.apache.jena.fuseki.Request  | NCSA request Log.                    |
| org.apache.jena.fuseki.Fuseki   | The HTTP request log                 |
| org.apache.jena.fuseki.Admin    | Administration operations            |
| org.apache.jena.fuseki.Builder  | Dataset and service build operations |
| org.apache.jena.fuseki.Config   | Configuration                        |

## NCSA request Log. 

This log is in NCSA extended/combined log format.  
Many web log analysers can process this format.

This log is normally off.

When run as a WAR file inside a webapp container 
(e.g. [Apache Tomcat](http://tomcat.apache.org/)), the webapp container
or reverse proxy will log access requests anyway. 

## Setting logging

The Fuseki engine looks for the log4j configuration as follows:

* Use system property `log4j.configuration` if defined (as usual for log4j).
* Use `file:log4j.properties` (current directory) if it exists
* Use file `log4j.properties` is the directory defined by `FUSEKI_BASE`
* Use java resource `log4j.properties` on the classpath.
* Use java resource `org/apache/jena/fuseki/log4j.properties` on the classpath.
* Use a built-in configuration.

The last step is a fallback to catch the case where Fuseki has been repackaged
into a new WAR file and `org/apache/jena/fuseki/log4j.properties` omitted, or run from
the base jar.  It is better to include `org/apache/jena/fuseki/log4j.properties`.

The preferred customization is to use a custom `log4j.properties` file in
`FUSEKI_BASE`.  For the WAR file, `FUSEKI_BASE` defaults to `/etc/fuseki`
on Linux.  For the standalone server, `FUSEKI_BASE` defaults to directory
`run/` within the directory where the server is run.

## Default setting

The [default log4j.properties](https://github.com/apache/jena/blob/master/jena-fuseki2/jena-fuseki-core/src/main/resources/org/apache/jena/fuseki/log4j.properties).

## Logrotate

Below is an example logrotate(1) configuration (to go in `/etc/logrotate.d`)
assuming the log file has been put in `/etc/fuseki/logs/fuseki.log`.

It rotates the logs once a month, compresses logs on rotation and keeps them for 6 months.

It uses `copytruncate`.  This may lead to at most one broken log file line.

    /etc/fuseki/logs/fuseki.log
    {
        compress
        monthly
        rotate 6
        create
        missingok
        copytruncate
        # Date in extension.
        dateext
        # No need
        # delaycompress
    }
