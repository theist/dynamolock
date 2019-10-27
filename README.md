# Fork differences from cirello.io/dynamolock

This is a custom fork I've made to support two features that were out of the
scope of the original project. I choose cirello-io package because I find it
easy to modify and because I find brilliant all the code layout with the options
and client usage.

## Acquire Locks on same owner

First option is for immediately acquire the lock when the lock owner is the same
of the owner that created the lock. This allows to immediately reclaim an
unreleased lock without having to wait for the lease time to consider it
expired.

Sample usage:

```golang
	lockedItem, err := c.AcquireLock("bones",
		dynamolock.WithNoWaitOnSameOwner(),
	)
```

When this code finds an unexpired lock check if the Existing lock owner is the
same than the client owner and consider the existing lock expired.

## Acquire Locks when it is expired according to the local clock

Second option are for checking the expiration date against the local clock. For
this the lock needs a field with the creation date that is retrieved as an
additional attribute. This is a quite unsafe operation for distributing tasks
because it relies on the local clocks and the local clocks can be skewed. Be
advised that this method could not be suitable for coordination with small
lease times or with tight tolerances.

This requires an option for the client to add the creation time to each lock it
creates, and an option to acquire lock to check that field against the local
clock.

```golang
	c, err := dynamolock.New(svc,
		"locks",
		dynamolock.WithLeaseDuration(3*time.Minute),
		dynamolock.WithUtcCreationDate(),
	)

	// [...]

	lockedItem, err := c.AcquireLock("bones",
		dynamolock.WithUnsafeLocalClockExpire(),
	)
```

When this code finds an unexpired lock check if the creation date stored in the
lock plus the lease duration **in the client** is less than the current time.
Log times are in UTC.

## Usage

Unless you need some of the features described above, please, use original
library. This is a custom fork I did quickly co cover two needs for a pet
project And it is tested just to the extent of my needs. Also I won't update it
or accept feature PR's as this is a derivative work.

You will need also use `go mod` to allow overwriting the original library with
mine in your project.

And now the original doc:

# DynamoDB Lock Client for Go

[![GoDoc](https://godoc.org/cirello.io/dynamolock?status.svg)](https://godoc.org/cirello.io/dynamolock)
[![Build Status](https://travis-ci.org/cirello-io/dynamolock.svg?branch=master)](https://travis-ci.org/cirello-io/dynamolock)
[![Coverage Status](https://coveralls.io/repos/github/cirello-io/dynamolock/badge.svg?branch=master)](https://coveralls.io/github/cirello-io/dynamolock?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/cirello-io/dynamolock)](https://goreportcard.com/report/github.com/cirello-io/dynamolock)
[![SLA](https://img.shields.io/badge/SLA-95%25-brightgreen.svg)](https://github.com/cirello-io/public/blob/master/SLA.md)

This repository is covered by this [SLA](https://github.com/cirello-io/public/blob/master/SLA.md).

The dymanoDB Lock Client for Go is a general purpose distributed locking library
built for DynamoDB. The dynamoDB Lock Client for Go supports both fine-grained
and coarse-grained locking as the lock keys can be any arbitrary string, up to a
certain length. Please create issues in the GitHub repository with questions,
pull request are very much welcome.

It is a port in Go of Amazon's original [dynamodb-lock-client](https://github.com/awslabs/dynamodb-lock-client).

## Use cases
A common use case for this lock client is:
let's say you have a distributed system that needs to periodically do work on a
given campaign (or a given customer, or any other object) and you want to make
sure that two boxes don't work on the same campaign/customer at the same time.
An easy way to fix this is to write a system that takes a lock on a customer,
but fine-grained locking is a tough problem. This library attempts to simplify
this locking problem on top of DynamoDB.

Another use case is leader election. If you only want one host to be the leader,
then this lock client is a great way to pick one. When the leader fails, it will
fail over to another host within a customizable leaseDuration that you set.

## Getting Started
To use the DynamoDB Lock Client for Go, you must make it sure it is present in
`$GOPATH` or in your vendor directory.

```sh
$ go get -u cirello.io/dynamolock
```

This package has the `go.mod` file to be used with Go's module system. If you
need to work on this package, use `go mod edit -replace=cirello.io/dynamolock@yourlocalcopy`.

Then, you need to set up a DynamoDB table that has a hash key on a key with the
name `key`. For your convenience, there is a function in the package called
`CreateTable` that you can use to set up your table, but it is also possible to
set up the table in the AWS Console. The table should be created in advance,
since it takes a couple minutes for DynamoDB to provision your table for you.
The package level documentation comment has an example of how to use this
package. Here is some example code to get you started:

```Go
package main

import (
	"log"

	"cirello.io/dynamolock"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
)

func main() {
	svc := dynamodb.New(session.Must(session.NewSession(&aws.Config{
		Region: aws.String("us-west-2"),
	})))
	c, err := dynamolock.New(svc,
		"locks",
		dynamolock.WithLeaseDuration(3*time.Second),
		dynamolock.WithHeartbeatPeriod(1*time.Second),
	)
	if err != nil {
		log.Fatal(err)
	}
	defer c.Close()

	log.Println("ensuring table exists")
	c.CreateTable("locks",
		dynamolock.WithProvisionedThroughput(&dynamodb.ProvisionedThroughput{
			ReadCapacityUnits:  aws.Int64(5),
			WriteCapacityUnits: aws.Int64(5),
		}),
		dynamolock.WithCustomPartitionKeyName("key"),
	)

	data := []byte("some content a")
	lockedItem, err := c.AcquireLock("spock",
		dynamolock.WithData(data),
		dynamolock.ReplaceData(),
	)
	if err != nil {
		log.Fatal(err)
	}

	log.Println("lock content:", string(lockedItem.Data()))
	if got := string(lockedItem.Data()); string(data) != got {
		log.Println("losing information inside lock storage, wanted:", string(data), " got:", got)
	}

	log.Println("cleaning lock")
	success, err := c.ReleaseLock(lockedItem)
	if !success {
		log.Fatal("lost lock before release")
	}
	if err != nil {
		log.Fatal("error releasing lock:", err)
	}
	log.Println("done")
}
```

## Selected Features
### Send Automatic Heartbeats
When you create the lock client, you can specify `WithHeartbeatPeriod(time.Duration)`
like in the above example, and it will spawn a background goroutine that
continually updates the record version number on your locks to prevent them from
expiring (it does this by calling the `SendHeartbeat()` method in the lock
client.) This will ensure that as long as your application is running, your
locks will not expire until you call `ReleaseLock()` or `lockItem.Close()`

### Read the data in a lock without acquiring it
You can read the data in the lock without acquiring it, and find out who owns
the lock. Here's how:
```Go
lock, err := lockClient.Get("kirk");
```

## Logic to avoid problems with clock skew
The lock client never stores absolute times in DynamoDB -- only the relative
"lease duration" time is stored in DynamoDB. The way locks are expired is that a
call to acquireLock reads in the current lock, checks the RecordVersionNumber of
the lock (which is a GUID) and starts a timer. If the lock still has the same
GUID after the lease duration time has passed, the client will determine that
the lock is stale and expire it.

What this means is that, even if two different machines disagree about what time
it is, they will still avoid clobbering each other's locks.
