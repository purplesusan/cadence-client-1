# Go framework for Cadence [![Build Status](https://travis-ci.org/uber-go/cadence-client.svg?branch=master)](https://travis-ci.org/uber-go/cadence-client) [![Coverage Status](https://coveralls.io/repos/uber-go/cadence-client/badge.svg?branch=master&service=github)](https://coveralls.io/github/uber-go/cadence-client?branch=master)
[Cadence](https://github.com/uber/cadence) is a distributed, scalable, durable, and highly available orchestration engine we developed at Uber Engineering to execute asynchronous long-running business logic in a scalable and resilient way.

`cadence-client` is the framework for authoring workflows and activities in Go.

## Samples

For samples, see [Cadence Samples](https://github.com/samarabbas/cadence-samples).

## Run Cadence Server

Run Cadence Server using Docker Compose:

    curl -O https://raw.githubusercontent.com/uber/cadence/master/docker/docker-compose.yml
    docker-compose up

If this does not work, see instructions for running the Cadence Server at https://github.com/uber/cadence/blob/master/README.md.

## Sample workflow

Following code demonstrates a sample workflow. For detailed information about the code, see [Workflows](docs/workflows.md).

```go
package simple

import (
	"time"


	"go.uber.org/cadence/workflow"
	"go.uber.org/zap"
)

func init() {
	workflow.Register(SimpleWorkflow)
}

// SimpleWorkflow is a sample Cadence workflow that accepts one parameter and
// executes an activity to which it passes the aforementioned parameter.
func SimpleWorkflow(ctx workflow.Context, value string) error {
	options := workflow.ActivityOptions{
		ScheduleToStartTimeout: time.Second * 60,
		StartToCloseTimeout:    time.Second * 60,
	}
	ctx = workflow.WithActivityOptions(ctx, options)

	var result string
	err := workflow.ExecuteActivity(ctx, activity.SimpleActivity, value).Get(ctx, &result)
	if err != nil {
		return err
	}
	workflow.GetLogger(ctx).Info(
		"SimpleActivity returned successfully!", zap.String("Result", result))

	workflow.GetLogger(ctx).Info("SimpleWorkflow completed!")
	return nil
}
```

## Contributing
We'd love your help in making Cadence-client great. Please review our [instructions](CONTRIBUTING.md).

## License
MIT License, please see [LICENSE](LICENSE) for details.
