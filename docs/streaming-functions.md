# Data Stream events

Kubeless lets you trigger any Kubeless function in response to ingested records into a data stream. Kubeless currently supports AWS Kinesis streaming service. 

## AWS Kinesis

To trigger Kubeless functions in response to ingested records into the AWS kinesis stream you need to deploy Kubeless AWS Kinesis trigger controlle. Please use this manifest to deploy Kubeless AWS Kinesis triggers controller.

```
kubectl create -f https://github.com/kubeless/kubeless/releases/download/$RELEASE/kinesis-$RELEASE.yaml
```

Once you deploy the manifest you shall see Kinesis trigger controller running in the Kubeless namespace as below.

```console
± kubectl get pods -n kubeless
NAME                                           READY     STATUS    RESTARTS   AGE
kinesis-trigger-controller-65c78f9f44-v5flq    1/1       Running   0          1h
kubeless-controller-manager-6b7cdcdc76-x6gsd   1/1       Running   0          13h
```

You shall also notice a CRD resource type `kinesistriggers.kubeless.io` created as below.
```console
± kubectl get crd
NAME                          AGE
cronjobtriggers.kubeless.io   13h
functions.kubeless.io         13h
httptriggers.kubeless.io      13h
kinesistriggers.kubeless.io   13h
```

Kubeless cli lets you create Kubeless triggers of Kinesis type. Kubeless cli provides necessary functionality to manage life cycle of Kinesis triggers.

```console
± kubeless trigger kinesis
kinesis trigger command allows user to create, list, update, delete Kinesis triggers running on Kubeless

Usage:
  kubeless trigger kinesis SUBCOMMAND [flags]
  kubeless trigger kinesis [command]

Available Commands:
  create      Create a Kinesis trigger
  delete      Delete a Kinesis trigger
  list        list all Kinesis triggers deployed to Kubeless
  publish     publish message to a Kinesis stream
  update      Update a Kinesis trigger

Flags:
  -h, --help   help for kinesis

Use "kubeless trigger kinesis [command] --help" for more information about a command.
```

In order to deploy a Kinesis trigger and associate a Kubeless function to be invoked in response to ingested records in Kinesis data stream, you need to first let Kubeless know the credentials required to acess your AWS Kinesis stream. Kubeless will leverage Kubernetes secrets to store the credentails in the cluster and use them to acess the Kinesis stream.

First you need to creat Kubernetes secret that can store you AWS `aws_access_key_id` and `aws_secret_access_key`. Usually if you are using AWS cli your keys will be present in `~/.aws/credentials` or you can create access keys from AWS console.

```console
kubectl create secret generic ec2 --from-literal=aws_access_key_id=ABCAIFDB7GEGGD37HN5Q --from-literal=aws_secret_access_key=gAckl2wtNrY9aaR/OvzAOCaAg6wSkABCsyXazXrx
```

Once you have created a secret you are ready to deploy Kubeless Kinesis trigger as below.

```console
kubeless trigger kinesis create test-trigger --function-name post-python --aws-region us-west-2 --shard-id shardId-000000000000 --stream my-kinesis-stream --secret ec2
```

Lets look into the flags expected. `--aws-region` is the AWS region in which your Kinesis stream is avilable. `--shard-id` is the id of shard into which records are placed. You should be able to get the `shard-id` from the stream description. `--stream` is the name of the Kinesis stream.

```console
± aws kinesis describe-stream --stream-name my-kinesis-stream
{
    "StreamDescription": {
        "RetentionPeriodHours": 24,
        "StreamName": "my-kinesis-stream",
        "Shards": [
            {
                "ShardId": "shardId-000000000000",
                "HashKeyRange": {
                    "EndingHashKey": "340282366920938463463374607431768211455",
                    "StartingHashKey": "0"
                },
                "SequenceNumberRange": {
                    "StartingSequenceNumber": "49584495912138607235774073050889122383423872293029281794"
                }
            }
        ],
        "StreamARN": "arn:aws:kinesis:us-west-2:159706291352:stream/my-kinesis-stream",
        "EnhancedMonitoring": [
            {
                "ShardLevelMetrics": []
            }
        ],
        "StreamStatus": "ACTIVE"
    }
}
```

Once you deploy Kinesis trigger you shall see a `kinesistrigger` CRD object as below.

```console
± k get kinesistriggers.kubeless.io test -o yaml
apiVersion: kubeless.io/v1beta1
kind: KinesisTrigger
metadata:
  labels:
    created-by: kubeless
  name: test
  namespace: default
spec:
  aws-region: us-west-2
  function-name: post-python
  secret: ec2
  shard: shardId-000000000000
  stream: my-kinesis-stream
```

At this point you shall be able to publish record in to the stream either through Kubeless CLI or using AWS cli as below.

```console
kubeless  trigger kinesis publish --aws-region us-west-2 --aws_access_key_id AKIDIDCB7NEGGD37HN5Q --aws_secret_access_key gBckl2wtMrY9aaR/OvzAOCaKg6wSkEZAsyXazXrx --partition-key "123" --stream my-kinesis-stream  --message "hello world"
```

or

```console
aws kinesis put-record --stream-name my-kinesis-stream --partition-key 123 --data testdata1
aws kinesis put-record --stream-name my-kinesis-stream --partition-key 123 --data testdata2
aws kinesis put-record --stream-name my-kinesis-stream --partition-key 123 --data testdata3
```

You shall see the log of recived messages in the function pod associated with the Kinesis trigger.

```console
± k logs post-python-59f7fc4b54-4nhbb
Bottle v0.12.13 server starting up (using CherryPyServer())...
Listening on http://0.0.0.0:8080/
Hit Ctrl-C to quit.

{'event-time': '2018-05-18 05:40:42.881137473 +0000 UTC', 'extensions': {'request': <LocalRequest: POST http://post-python.default.svc.cluster.local:8080/>}, 'event-type': 'application/x-www-form-urlencoded', 'event-namespace': 'kinesistriggers.kubeless.io', 'data': 'testdata12', 'event-id': 'bDRMSN3NPC81ktU'}
172.17.0.7 - - [18/May/2018:05:40:42 +0000] "POST / HTTP/1.1" 200 10 "" "Go-http-client/1.1" 0/11758
{'event-time': '2018-05-18 05:40:44.891994208 +0000 UTC', 'extensions': {'request': <LocalRequest: POST http://post-python.default.svc.cluster.local:8080/>}, 'event-type': 'application/x-www-form-urlencoded', 'event-namespace': 'kinesistriggers.kubeless.io', 'data': 'testdata22', 'event-id': 'uHdiWN-lzeKYQyQ'}
172.17.0.7 - - [18/May/2018:05:40:44 +0000] "POST / HTTP/1.1" 200 10 "" "Go-http-client/1.1" 0/8983
{'event-time': '2018-05-18 05:40:45.878361324 +0000 UTC', 'extensions': {'request': <LocalRequest: POST http://post-python.default.svc.cluster.local:8080/>}, 'event-type': 'application/x-www-form-urlencoded', 'event-namespace': 'kinesistriggers.kubeless.io', 'data': 'testdata32', 'event-id': 'sRRjSasGVApy8tA'}
```