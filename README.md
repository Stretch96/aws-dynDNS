# aws-dynDNS

This is a simple bash script that obtains the Public ip and updates a record in AWS, emulating DynamicDNS services

It is best ran as a cron (The interval depends on your requirements)

## Requirements

#### jq

```
apt-get install jq
```

#### AWS Cli

```
apt-get install awscli
```

#### AWS Requirements

Create a Hosted Zone within AWS:

https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html

(The current price per hosted zone is $0.50, or $0.10 if you already have more than 25 hosted zones)

Create an IAM user with Route53 permissions
The `AmazonRoute53FullAccess` policy is the easiest option

## Usage

```
Usage: aws-dyndns [OPTIONS]
  -h                   - help
  -z <zone_domain>     - AWS Zone domain name
  -r <record_value>    - Record value eg. home
  -c                   - create record if it doesn't exist
  -p <aws_profile>     - AWS Profile (defaults to default)
```

## Recommended setup

I have this installed on a Raspberry Pi, as it's a cheap little device that can be left running 24/7

#### Initial setup on the Raspberry Pi (or other compatible host)

1. Create the directory `~/.aws`

```
$ mkdir ~/.aws
```

2. Create/Edit the `~/.aws/config` file

```
$ vim ~/.aws/config
[default]
region=<your-default-region>
cli_follow_urlparam=false
```

Replace `<your-default-region>` with your preferred region

3. Create/Edit the `~/.aws/credentials` file

```
$ vim ~/.aws/credentials
[aws-dyndns]
aws_access_key_id=xxxxxxx
aws_secret_access_key=xxxxxxx
```

Enter the `aws_access_key_id` and `aws_secret_access_key` generated when you created the IAM user

4. Clone this repo

```
git clone git@github.com:Stretch96/aws-dynDNS
cd aws-dynDNS
```

5. Add a symlink from the `aws-dyndns` script to `/usr/local/bin`:

```
ln -s aws-dyndns /usr/local/bin/aws-dyndns
```

#### Testing the setup

1. Once you have created a Hosted Zone on AWS, the following command should produce an error:

(Replace `<hosted_zone_name>` with your Hosted Zone domain name)

```
$ aws-dyndns -z <hosted_zone_name> -r test123 -p aws-dyndns
Error: test record doesn't exist in <hosted_zone_name> zone
       Either create the record first in AWS,
       or add the -c flag to create it
```

If any other error appears, there may be something wrong with the setup

2. Now run the same command, but with the `-c` flag, which should create the record:

```
$ aws-dyndns -z <hosted_zone_name> -r test123 -c -p chris_qa_dns
Creating record
Record <hosted_zone_name> updated to <your-public-ip> - Status: PENDING
```

If anything other than `PENDING` for the Status is produced, something has gone wrong

#### Creating the cronjob

Decide what DNS record you would like to be created to point towards your public IP address, eg. `home`

The following configuration will run the script every 5 minutes

**Warning**:
  This script uses the ipinfo.io service to obtain the Public ip
  ipinfo.io currently have a Free rate limit of 50,000 requests per day
  If you require more than this, consider creating a paid account: https://ipinfo.io/pricing
  (For reference, there are 1440 minutes in a day)

3. Create/Edit the file `/etc/cron.d/aws-dyndns`:
(Replace `<username>` with the user that the cron should run as)
(Replace `<hosted_zone_name>` with your Hosted Zone domain name)

```
$ vim /etc/cron.d/aws-dyndns
*/5 * * * * <username> /usr/local/bin/aws-dyndns -z <hosted_zone_name> -r home -c -p aws-dyndns | /usr/bin/logger -t aws-dyndns
```

4. Check `/var/log/syslog` for any errors

```
tail -f /var/log/syslog
```

Every 5 minutes, you should see something like:

```
Nov 28 21:00:01 raspberrypi CRON[1234]: (<username>) CMD (/usr/local/bin/aws-dyndns -z <hosted_zone_name> -r home -c -p aws-dyndns | /usr/bin/logger -t aws-dyndns)
```

If it needs to create the record, you will see:

```
Nov 28 21:00:01 raspberrypi CRON[1234]: (<username>) CMD (/usr/local/bin/aws-dyndns -z <hosted_zone_name> -r home -c -p aws-dyndns | /usr/bin/logger -t aws-dyndns)
Nov 28 21:00:11 raspberrypi aws-dyndns: Creating record
Nov 28 21:00:16 raspberrypi aws-dyndns: Record <new-record-name> updated to <public-ip> - Status: PENDING
```

If your Public ip changes, it will produce the output:

```
Nov 28 21:00:01 raspberrypi CRON[1234]: (<username>) CMD (/usr/local/bin/aws-dyndns -z <hosted_zone_name> -r home -c -p aws-dyndns | /usr/bin/logger -t aws-dyndns)
Nov 28 21:00:16 raspberrypi aws-dyndns: Record <new-record-name> updated to <public-ip> - Status: PENDING
```

5. Confirming DNS record creation

Just run the following command and it should produce your Public ip:

```
$ host <domain-name>
1.2.3.4 # (Your Public ip address)
```

It may take a minute or two to update
