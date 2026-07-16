# Purpose

AWS SSM has been sufficient to achieve virtually all ssh type needs for some time.  However, the credentials used to get a
session on a remote instance inherently provide a lot of power.  Once you have a session on an instance, you can
use the instance's profile to read quite a lot of secret information.

eg
```
# Fetch the user data script
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/user-data
```
or even
```
# Fetch the private cert
aws --region eu-west-1 s3 cp s3://my-bucket-of-secrets/my-stack/PROD/my-secure-app/secret-cert.json
```
or EVEN
```
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/[role-name]
```

Ideally, therefore, we would use the same approach as devcontainers, and not put "bare" credentials on our laptops 
(where they can potentially be exfiltrated by attackers) at all.

This is a simple setup to achieve that.

# Setup

Build a small docker image which includes some tooling, and crucially the ssm session manager plugin for aws.

```
docker build -t aws-shell .
```

# Running

```
docker run -it aws-shell bash
```

# Configuring

Grab your aws credentials as aws configure commands.  Make sure the profile name is `default` or set `AWS_PROFILE`.  Paste them into the docker shell.

# Examples

## Starting a remote ssm session

```
docker run -it aws-shell bash
```
...paste in your creds...

```
> session myApp myStack myStage
Connecting to i-0c7f9cb1234567890

Starting session with SessionId: justin.rowles-rycskjdsfjkgflkbgdflksdfakl
```

## Starting a tunnel to a remote host

### Ports

There will be three ports in play.  

 * Container port exposed by docker
 * SSM port inside the container, exposed by aws ssm
 * Remote port on the remote host, mediated by a tunnel to the remote instance

The following command starts the container, listening on 9000, which is forwarded to 9000 internally.  A `socat`
process is then automatically started which forwards CONTAINER_PORT internally to SSM_PORT.

```
docker run -it -e CONTAINER_PORT=9000 -e SSM_PORT=9001 -e REMOTE_PORT=9000 -p 9000:9000 aws-shell bash
```
...paste in your creds...
```
host-tunnel myApp myStack myStage theirHostName
```
to open a tunnel on the oldest instance with those tags from SSM_PORT to &lt;theirHostName&gt;:REMOTE_PORT 

A similar approach can be used to get to remote RDS hosts, looking them up with tags, using `rds-tunnel.

At this point, you can run a client to the remote host communicating on localhost:CONTAINER_PORT.  This
client can be anything - psql, curl, ftp...

# Notes

The socat listener is required solely because the aws cli does not bind to all interfaces.  This may change.
