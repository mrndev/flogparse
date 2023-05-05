# `flogparse` - a command line utility to parse flow logs
`flogparse` is a shell script utility to parse cloud flow logs into more readable and self explanatory format. 
The script optionally utilizes the ionosctl tool to fetch interface name information from 
the virtual data center.

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

**Example 2**: Parse flow logs and resolve the interface UUID and IP address with information from the virtual data center
<pre>
<b>./flogparse.sh -i $DCID ./logs/*.log.gz </b>
2023-04-27 00:11:15+0s: 311 bytes from 10.7.224.254:67 to <b>API2_Private_NIC</b>:68 (Bootstrap Protocol Client) over UDP accepted by <b>API2_Private_NIC</b>
2023-04-27 00:16:16+0s: 311 bytes from 10.7.224.254:67 to <b>API2_Private_NIC</b>:68 (Bootstrap Protocol Client) over UDP accepted by <b>API2_Private_NIC</b>
</pre>

**Example 3**: Like above but use the S3 log file prefix as the interface identifier
<pre>
<b>./flogparse.sh -F -i $DCID</b> ../logs/<b>ApplicationLoadBalancer</b>-1682659839.log.gz 
2023-04-28 05:30:18+0s: 44 bytes from 95.79.55.27:59845 to ApiLB:23 (Telnet) over TCP rejected by <b>ApplicationLoadBalancer</b>
2023-04-28 05:30:34+0s: 125 bytes from 198.235.24.100:52852 to ApiLB:1900 (UPnP SSDP) over UDP rejected by <b>ApplicationLoadBalancer</b>
2023-04-28 05:30:07+0s: 44 bytes from 162.216.149.112:51336 to ApiLB:2082 over TCP rejected by <b>ApplicationLoadBalancer</b>
</pre>

**Example 4**: Fetch logs with the s3cmd and parse the logs to stdout
You will need to have the s3cmd configure for your bucket
<pre>
s3cmd get --skip-existing s3://mybucket-123/* ./logs/
./flogparse.sh -F -i $DCID ./logs/*.gz
</pre>

 Example how the s3cmd can be configured
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


## OPTIONS:
-i      Datacenter ID (same as --datacenter-id with ionosctl). Used to
        get interface information with ionosctl to swap UUIDs and IP
        addresses to clear text names

-c      Use the cached datacenter information (generated with -i). Faster than
        generating it on every call.

-F      Use the device/interface identifier on the log file name instead of the
        UUID in the log records. See teh example above where the string
        ApplicationLoadBalancer from the filename is used as the interface
        identifier in the listed log lines. The s3 prefix can be defined when setting
        the flow log in the DCD or with ionosctl. Give for example
        's3://mybucket/myprefix' instead of just 's3://mybucket' in the DCD or ionosctl.
        Normally you would not need this if you give the interfaces (NICs) proper names
        in the virtual data center. For certain components, such as the application load balancer
        the individual interfaces cannot be named so this may be a useful option.


## ARGUMENTS:
*.log or *.log.gz files that conform to the log format 2

## DEPENDENCIES:
- ionosctl (needed to fetch information from the virtual datacenter)
- jq (needed to parse json data received with ionosctl)
