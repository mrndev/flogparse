# `flogparse` - a command line utility to parse flow logs
`flogparse` is a shell script utility to parse cloud flow logs into more readable and self explanatory format. 
The script optionally utilizes the ionos APIs to fetch interface name information from 
the virtual data center. 

In short - this:

`2 31931785 d8963239-e78b-4e18-8ee4-42b22dc83047 51.195.90.237 212.227.224.120 60890 22 6 1 60 1682968081 1682968081 REJECT OK`

will be converted to something like this:

`2023-05-01 19:08:38+7s: 240 bytes from 139.59.27.154:20460 to ApiLB:22 (SSH Remote Login Protocol) over TCP rejected by ApplicationLoadBalancer`


the script is provided 'as is' and can be used for adhoc trouble shoouting or as an example 
how to parse and interpret the flow log information. See [here](https://docs.ionos.com/cloud/compute-engine/networks/how-tos/flow-logs/configure-flow-logs) for more information on the flow logs.

## Examples:

**Example 1**: Parse flow logs into more readable format and add protocol/port information
<pre>
<b>
mkdir logs
s3cmd get --skip-existing s3://mylogbucket-123/* ./logs/
flogparse.sh ../logs/*.log.gz</b>
2023-04-28 05:30:18+0s: 44 bytes from 95.79.55.27:59845 to 212.227.224.120:23 (<b>Telnet</b>) over <b>TCP</b> rejected by d8963239-e78b-4e18-8ee4-42b22dc83047
2023-04-28 05:30:34+0s: 125 bytes from 198.235.24.100:52852 to 212.227.224.120:1900 (<b>UPnP SSDP</b>) over <b>UDP</b> rejected by d8963239-e78b-4e18-8ee4-42b22dc83047
</pre>

**Example 2**: like above but replace the interface UUID with the name/identifier on the log filename (the s3 prefix)

<pre>
./flogparse.sh -F <b>ApplicationLoadBalancer</b>-1682967612.log.gz
2023-05-01 19:08:49+0s: 78 bytes from 198.235.24.57:56658 to 212.227.224.120:137 (NETBIOS Name Service) over UDP rejected by <b>ApplicationLoadBalancer</b>
2023-05-01 19:08:08+0s: 40 bytes from 95.214.55.85:36691 to 212.227.224.120:9091 over TCP rejected by <b>ApplicationLoadBalancer</b>
</pre>

**Example 3**: Parse flow logs and resolve the interface UUID and IP address with data from the virtual data center (requires IONOS_TOKEN env variable)
<pre>
<b>./flogparse.sh -i $DCID ./logs/*.log.gz </b>
2023-04-27 00:11:15+0s: 311 bytes from 10.7.224.254:67 to <b>API2_Private_NIC</b>:68 (Bootstrap Protocol Client) over UDP accepted by <b>API2_Private_NIC</b>
2023-04-27 00:16:16+0s: 311 bytes from 10.7.224.254:67 to <b>API2_Private_NIC</b>:68 (Bootstrap Protocol Client) over UDP accepted by <b>API2_Private_NIC</b>
</pre>


**Example 4**: Fetch logs with the s3cmd and parse the logs to stdout
You will need to have the s3cmd configure for your bucket
<pre>
s3cmd get --skip-existing s3://mybucket-123/* ./logs/
./flogparse.sh -F -i $DCID ./logs/*.gz
</pre>

## Help

<pre>


This script parses the cloud flow logs into a more readable format by resolving
protocol number and port number into clear text. If you give a data center id
(UUID) with the '-i' option, the script will attempt to get clear text names
for interfaces and IP addresses from the APIs. 

the parsed data will look somethin like this:
2023-04-27 12:03:14+0s: 120 bytes from 45.134.144.235:52876 to ApiLB:80 (World Wide Web HTTP) over TCP accepted by ApplicationLoadBalancer


OPTIONS:
-h      
        print help

-i      <dcid>
        Datacenter ID (UUID). Used to get interface information from the APIs
        (swap UUIDs and IP addresses to clear text names). The data is stored
        in dcinfo.<dcid>.lib file. use the '-c' parameter below to prevent
        fetching the information on every call (makes the script faster).
        Requires IONOS_TOKEN environment variable set.

-c      
        Use the locally stored cache file dcinfo.<dcid>.lib instead of trying
        to fetch the data from the APIs

-F      
        Use the device/interface identifier on the log file name instead of the
        UUID in the log records. See teh example above where the string
        ApplicationLoadBalancer from the filename is used as the interface
        identifier in the listed log lines. The s3 prefix can be defined when
        setting the flow log in the DCD or with ionosctl. Give for example
        's3://mybucket/myprefix' instead of just 's3://mybucket'.

ARGUMENTS:
        *.log or *.log.gz files that conform to the log format 2


EXAMPLES:
        parse all gzipped and clear text logs in the logs directory.
        flogparse/flogparse.sh ./logs/*.log.gz ./logs/*.log

        replace the interface UUID in the logs with the file prefix
        flogparse/flogparse.sh -F myprefix.123345123345.log.gz

        get data center information from the APIs and use it to replace UUIDs and IP
        addresses with the interface names (requires IONOS_TOKEN environment variable)
        flogparse/flogparse.sh -i $DCID ./logs/*.log.gz

        # the same but use the cached data center information (faster)
        flogparse/flogparse.sh -c -i $DCID ./logs/*.log.gz

        # run without parameters to generate the flogparse.awk that can be used in pipes
        flogparse/flogparse.sh
        gunzip -c *.log.gz | flogparse.awk

NOTES: 
        the script will generate a self contained flogparse.*.awk script in the
        same directory. The parselogs.qwk reads the logs from stdin and can be
        used to parse logs in data pipes. Example:
        ./flogparse.*.awk < alb.1234523423.log

DEPENDENCIES:
        jq (if -i option is used)

</pre>

## Examples for configuring the flow logs and S3 tools
Configure the s3cmd for fetching the log files
<pre>
# what you need for the S3 bucket is the key and the secret from the DCD S3 panel
# ionos endpoins are listed here
# https://docs.ionos.com/cloud/managed-services/s3-object-storage/endpoints
s3cmd --configure \
         --host 's3-eu-central-1.ionoscloud.com' \
         --host-bucket '%(bucket)s.s3-eu-central-1.ionoscloud.com' \
         --region de
         

# Change the website endpoint, which for some reason cannot be set with the command line
# parametes above.
sed -i 's!website_endpoint.*!website_endpoint = %(bucket)s.s3-website-de-central.profitbricks.com!g' .s3cfg
</pre>

here is an example how you can set the s3 prefix when creating the flowlogs
<pre>
ionosctl flowlog create \
   --datacenter-id $DCID \
   --server-id $SERVER_ID \
   --nic-id $NIC_ID \
   --name "Flowlog for NIC1" \
   --s3bucket s3://my-bucket-12322/<b>myPrefix</b>
</pre>
now, the flowlogs system will use `myPrefix` instead of the interface UUID in the flow log file names

