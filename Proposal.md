# Outline
The talk is structured as follows: 
- Introduction and problem statement [2 mins]
- Background and Terminology [2 mins]
- Alerting on Critical Components [1 min]
- Pressure Relief via Throttling [20 mins]
  - Implementing a barebone throttler [4 mins + demo]
  - Abstraction and Capability Expansion [16 mins + demo]
- Recovery [14 mins]
	- Reusing backoff throttler code to implement recovery [4 mins + demo]
	- Efficiently switching between modes using context cancellation [10 mins + demo]
- Setup and Configuration [2 mins]
- Conclusion [2 mins]


# Introduction

Software systems are becoming increasingly complex, so developers must proactively implement safeguards to protect the integrity of the platform under high load. Systems that don't have such measures in place are prone to service slowdowns, and in the worst case, service outages. The problem can have a cascading effect in an event-driven microservice architecture that deals with high load, especially when the load can be spiky.

To mitigate this problem at [company], we built a self-regulating pressure relief valve in Go that automatically adjusts a service’s processing rate to maximize throughput without sacrificing system stability. The key to success is to identify metrics that can signal a potential problem, monitor those metrics through an automated workflow, and set up processes to automatically alleviate or eliminate the problem before they negatively impact the system.

In more detail, by leveraging Go concepts including interfaces, channels, tickers, and context cancellation, this pressure relief valve:
1. Detects when a critical component is reaching a resource limit
2. Automatically begins to throttle consumption of noncritical messages to relieve pressure on the component
3. Incrementally increases processing rate after component has returned to a stable state
4. Surfaces configuration options in a minimal, backwards-compatible manner that minimizes developer overhead

The image below demonstrates the effect of the pressure relief valve on a given metric (ex. CPU) over time, automatically containing its oscillation in proximity of a soft limit, resulting in maximum throughput without reaching the metric limit.

<img width="804" alt="Throughput over time" src="https://github.com/anonymousgopher/Gophercon2024/assets/161173606/32533589-73e9-4ef3-aedb-ef68a6bea4e3">

Today we will walk through a simplified version of this pressure relief valve that has been deployed in our backend ecosystem at [company]. This talk showcases some powerful features within the Go programming language and illustrates how they can be used together to solve a non-trivial problem. The goal is to inspire the audience to incorporate some of these concepts into their own solutions. In the interest of time, all code snippets will be written ahead of time and a live code walkthrough and demo will be done.

# Background and Terminology

Before we dive into the details, let's set the scene and go over some concepts and terminology that will be used throughout this presentation. 

<img width="804" alt="Setup" src="https://github.com/anonymousgopher/Gophercon2024/assets/161173606/f60a49c4-1b7f-488a-8617-a87e50218960">

The illustration above depicts a portion of the event-driven architecture and the components involved, which communicate primarily asynchronously using a message queue. The sender publishes messages with a topic, routing key, and message body. Messages are stored on different queues based on their binding (topic/key pair), and delivered to the respective consumer function FIFO (first in, first out) through a Go channel. On service startup, the recipient registers its own handlerFunc with each consumer, and when the consumer is ready to deliver the message to the recipient, it invokes the handlerFunc.


# Alerting on Critical Components

The first step in this process is to accurately detect when a critical component is overwhelmed. At [company], we addressed this by having a centralized service that monitored component metrics such as CPU and memory usage, and published a message on a dedicated “alerts” topic. In subsequent sections of this proposal, we will explore how we can responsively address these alerts by building an automated mechanism to alleviate pressure on components until the alert is resolved. Overall, this is the behavior we are striving for:

<img width="804" alt="Throttling on Alerts" src="https://github.com/anonymousgopher/Gophercon2024/assets/161173606/4387fc00-b92a-4693-88a8-4f00c8165ef7">

In this illustration, we have three separate processes that are consuming varying amounts of CPU over time. Process A is considered a critical process that would cause noticeable issues to the end user if interrupted. process B and C are non-critical processes that would not cause functional problems, but should still be processed in a timely manner. When CPU usage hits the soft limit, an alert is triggered and each process is adjusted to alleviate pressure on our CPU. Specifically, process A is allowed to proceed normally, process B may proceed at a slower rate, and process C is temporarily halted. Once CPU usage returns to a healthy state, the alert is resolved and process B and C slowly return to their normal processing rate.

# Pressure Relief via Throttling
The primary mechanism we'll use today to relieve pressure on the system is throttling. We'll start by implementing a barebone throttler that gets the job done, and by the end of this section we'll have a solution that:

1. Throttles messages for non-critical workloads to relieve pressure on a component before it gets overwhelmed
2. Automatically responds to changes in component statuses by adjusting throttling strategy


Let’s start by working on the basic barebone throttler.

## Barebone Throttler

In this implementation, the main goal is to set up a separate process that receives non-time-sensitive messages from the main message queue and processes them at a slower rate. This preliminary step becomes the foundation to our throttler! 

Visually, this is what we would like to achieve:

<img width="804" alt="Simple Throttling Setup" src="https://github.com/anonymousgopher/Gophercon2024/assets/161173606/a9a306e7-06ed-461d-bbe2-2e21be56e911">

For each binding (topic/key pair) to be throttled, we need a dedicated queue to store the throttled messages. Since every binding could have a different throttling configuration, there must be a separate queue and consumer per configured binding. The secondary queue's consumer is responsible for delivering messages to the handler at the configured rate. In the original consumer's logic, when throttling is enabled, messages that have a matching binding are published to the secondary queue instead of getting delivered to the handler directly.

The regular consumer loops over each message received on the channel and checks if there’s an active alert and if the message binding is in the list of bindings to throttle. If both are true, the message gets published to the secondary throttled queue. Otherwise, the recipient’s handlerFunc is called to deliver the message to the service. 


``` go
// HandlerFunc contains the recipient's processing logic to process incoming messages
type HandlerFunc func(message msg) error

func consume(deliveryChan <- chan msg, handler HandlerFunc) {
	for d := range deliveryChan {
		ctx := context.Background()
		// alertActive is set when an alert is processed
		if alertActive && isThrottledBinding() {
			// publish to secondary queue
			b.Publish(ctx, topic, key, msg)
			continue
		}
		// recipient's handlerFunc
		err := handler(ctx, msg)
		if err != nil {
			log.Printf("Handler returned error: %s", err)
		}
	}
}
```

The consumer of the throttled queue will then apply a delay before calling the recipient's handlerFunc. A simple way to do this is using the `Sleep()` function from Go’s time package:

``` go
func consumeThrottled(deliveryChan <- chan msg, handler HandlerFunc) {
	for d := range deliveryChan {
		// wait 30 seconds before delivering message to handler
		time.Sleep(30)

		// recipient's handlerFunc
		err := handler(msg)
		if err != nil {
			log.Printf("Handler returned error: %s", err)
		}
	}
}
```

We have just implemented a basic throttler that waits 30 seconds before delivering each message to the handlerFunc! In the next section, let's expand our throttler's capabilities.


## Abstraction and Capability Expansion

In some scenarios, a fixed delay between messages is a sufficient throttling strategy. However, in other scenarios, a different policy may be more appropriate. In this section, we'll explore and carry out 3 different policies:

1. **Block throttling**: Stops message consumption completely
2. **Interval throttling**: Applies a fixed delay in between messages
3. **Backoff throttling**: Applies a delay that increases over time


It's important to point out here that queue consumers are not particularly interested in _how_ throttling is actually carried out. We can abstract the implementation logic by defining a throttler interface that includes an `apply()` method which waits for the configured amount of time. Then, we pass the throttler into the consumer as a parameter and call `apply()` instead of directly using `time.Sleep()`. 


```go
type Throttler interface {
	apply()
}

func consumeThrottled(deliveryChan <- chan msg, handler HandlerFunc, throttler Throttler) {
	for d := range deliveryChan {
		// instead of a delay, use the throttler method
		throttler.apply()

		// recipient's handlerFunc
		err := handler(ctx, msg)
		if err != nil {
			log.Printf("Handler returned error: %s", err)
		}
	}
}
```
Extracting the implementation logic into an interface decouples the consumer logic from the throttling logic, which, in turn makes the code more testable and maintainable. New policies can be added easily just by implementing the interface methods. 

That being said, let's go ahead and implement the three policies we listed above!


### Block throttling

For the block throttler, we are looking for a way to wait forever. One easy way this can be done is by using an empty select statement:

```go
type blockThrottler struct {}

func (b *blockThrottler) apply() {
	select{}
}
```

In practice, we probably don't want to be waiting _forever_... but we’ll address this later in the proposal.


### Interval throttling

In the barebone implementation we have already done this by using the `Sleep()` function in Go’s time package, but we can slightly improve the code to use the delay configured for the throttler:

```go
type intervalThrottler struct {
   delay time.Duration
}

func (i *intervalThrottler) apply() {
   time.Sleep(i.delay)
}
```

### Backoff throttling

This is the trickiest policy to implement so far since the throttler's delay changes over time. These updates should be done on an interval in parallel to the throttling itself. Luckily, the time package in Go has a ticker, which delivers “ticks” over a channel at an interval. Since the delay has to be updated in parallel with the throttling, we can spin up another goroutine that uses a ticker to update the current delay. For better code organization and reusability, let’s extract this logic into its own method. We can define a new struct that contains a `time.Duration` field, then create a method that updates the delay in a separate goroutine.


```go
type adjustableDelay struct {
	val time.Duration
}

type backoffThrottler struct {
	currentDelay adjustableDelay

	// user input
	initialDelay time.Duration
	maxDelay time.Duration
	multiplier float64
	backoffInterval time.Duration
}

func (a *adjustableDelay) adjust(interval time.Duration, multiplier float64, maxDelay time.Duration) {
	go func() {
		t := time.NewTicker(interval)
		for {
			// block until tick received
			select {
			case <-t.C:
				a.val = time.Duration(a.val.Seconds()*multiplier) * time.Second
				if a.val.Seconds() >= maxDelay.Seconds() {
					// stop ticker and exit goroutine
					t.Stop()
					return
				}
			}
		}
	}()
}

func (bo *backoffThrottler) apply() {
	// on first call, initialize throttler values
	if bo.currentDelay == 0 {
		bo.currentDelay = bo.initialDelay
		bo.currentDelay.adjust(bo.backoffInterval, bo.multiplier, bo.maxDelay)
	}
	time.Sleep(bo.currentDelay)
}
```

On every iteration of the loop, we should check if the max delay has been reached so we can stop the ticker and exit the goroutine as soon as the job is done. It’s noteworthy to mention that the value of the multiplier provided must be greater than 1, otherwise we would end up consuming messages at a faster rate.

In the next section, we will dive into the other side of our pressure relief valve and talk about how we transition between the two states.

# Recovery
They say what goes up must come down. In the context of the pressure relief valve we have built so far, this means that once the pressure dies down, the valve should slowly return to its original position. In other words, when the alert is cleared, we want to slowly increase the rate of processing until we return to the normal rate of processing. 

The exciting part is that we’ve already put together the functionality needed to do this. In the previous section when writing the backoff throttler, we discussed how “... the value of the multiplier provided must be greater than 1, otherwise we would end up consuming messages at a faster rate.” This happens to be exactly what we need to implement recovery! The only adjustment required is to add an additional exit condition to `adjust()`:
- Throttling mode: multiplier is greater than 1 AND current delay value is greater than or equal to maxDelay
- Recovery mode: multiplier is less than 1 AND current delay value is less than or equal to 1

Here’s what it looks like in code:

```go
if (multiplier > 1 && a.val.Seconds() >= maxDelay.Seconds()) || (multiplier < 1 && a.val.Seconds() <= 1) {
    // stop ticker and exit goroutine
    t.Stop()
    return
}
```

Now we have all the building blocks of our self-regulating pressure relief valve! The next logical step is to automate the switch between throttling and recovery states when alerts are received. 

In our barebone throttler, we have already configured the normal consumer to publish to the throttled queue if an alert is active, or call the recipient’s handlerFunc instead if no alert is active. Something similar needs to happen in the consumer of the throttled queue. We could do the same active alert check before calling the throttler’s `apply()` method, but we actually run into some problems with this approach. First, if we take a another look at the block throttler implementation, we’ll notice that there’s no exit condition for our `apply()` function. Ultimately, this means that we’re stuck throttling the message forever with no way to switch to recovery. Along similar lines, for the other throttling/recovery policies, there’s no mechanism to exit `apply()` early if the alert value has changed. For fairly small delay configurations, this may not be an issue. However, it’s important to ensure our solution is efficient for a wide range of delay values. 

This is where we can leverage Go’s context package! For each message received on the message channel, let’s create a context with cancellation. We can pass this context to all methods that might need to exit early if the alert value is changed. To listen to changes in alert values, we can have a channel that receives values in a separate goroutine and cancels the current message’s context when a value is received. 

Our throttle consumer now looks like this:

```go
func consumeThrottled(deliveryChan <- chan msg, alert <- chan bool, handler HandlerFunc, throttler Throttler) {
	for d := range deliveryChan {
		func() {
			// create context with cancel
			ctx, cancel := context.WithCancel(context.Background())

			// resources associated with the context should be released when done
			defer cancel()

			go func() {
				// listen to changes in alert over channel
				<-alert
				// when value received, cancel the context
				cancel()
			}()
			// instead of a delay, use the throttler method
			throttler.apply(ctx)

			// recipient's handlerFunc
			err := handler(msg)
			if err != nil {
				log.Printf("Handler returned error: %s", err)
			}
		}()
	}
}
```

You may have noticed something that looks unusual... right inside the loop ranging over messages on the deliveryChan there's an anonymous function. This was added to address a potential issue with using a defer statement inside a loop. According to the [Go language spec](https://go.dev/ref/spec#Defer_statements), defer “invokes a function whose execution is deferred to the moment the surrounding function returns.” For our consumer logic, this means that the context is **not** cancelled when we're done processing the current message. By wrapping all the logic inside an anonymous function, the defer now runs at the end of every iteration of the loop instead.

Next, we have to adjust the `apply()` method for each throttler to exit early if the context provided is cancelled before the specified duration has passed. For the block throttler, instead of an empty select statement, we just block until the context is cancelled:

```go
type blockThrottler struct {}

func (b *blockThrottler) apply(ctx context.Context) {
	<- ctx.Done()
}
```

For the other throttlers, we face a small problem. The `Sleep()` function does not take a context parameter, so we’re not able to exit early. The next best thing is using a select with `time.After()` instead of `time.Sleep()` right? Something like this:

```go
func (i *interval) apply(ctx context.Context) {
	select {
	case <-ctx.Done():
		return
	case <-time.After(i.delay):
		return
	}
}
```

This select will block until the context has been cancelled or the entire duration of the delay has passed, whichever happens first. However, if we take a closer look at the [documentation](https://pkg.go.dev/time#After) for `time.After()`, we find that “The underlying Timer is not recovered by the garbage collector until the timer fires. If efficiency is a concern, use NewTimer instead and call Timer.Stop if the timer is no longer needed.” 

Let’s make that adjustment:

```go
func (i *interval) apply(ctx context.Context) {
	t := time.NewTimer(i.delay)
	select {
	case <-ctx.Done():
		t.Stop()
		return
	case <-t.C:
		return
	}
}
```

We can extend this idea to the backoff throttler as well as the `adjust()` function to stop adjusting the delay if the context has been cancelled before the final value has been reached. Code snippets for those have been omitted in this proposal, but will be shown in the presentation.

# Setup and Configuration
Now that we have built all the elements of our pressure relief valve, we need to expose these configuration options to the developer of the microservice. We can easily provide functions that set the initial values for each throttler. Only one throttling policy is shown below, but all three will be included in live demo.

```go
type Binding struct {
	Key          string
	Topic        string
	throttler    Throttler
}

// WithBackoffThrottleConfig configures the backoff throttler with
// initialDelay: initial duration to wait before consuming the next message
// backoffInterval: time after which delay will be increased
// multiplier: factor to increase current delay
// maxDelay: max duration for delay
func (b Binding) WithBackoffThrottleConfig(initialDelay time.Duration, backoffInterval time.Duration, multiplier float64, maxDelay time.Duration) Binding {
	return Binding{
		Key:      b.Key,
		Topic: b.Topic,
		throttler: &backoff{
			initialDelay:    initialDelay,
			backoffInterval: backoffInterval,
			multiplier:      multiplier,
			maxDelay:        maxDelay,
		},
	}
}
```

As a general rule of thumb, user inputs should always be validated. To prevent unintended behavior from using a throttler with invalid values, this validation should be done when the throttler is configured on service startup. All we need to do is add a `validate()` method to the throttler interface and complete the logic for each throttler we've defined. In the interest of keeping this proposal at an appropriate length, the code for validation has been omitted. During the demo they will be briefly shown in the presentation.


Now the throttler interface looks like this:

```go
type Throttler interface {
	apply()
	validate() error
}
```

The use of Go's `time.Duration` to express delay values makes the configuration easy to understand at a quick glance (as opposed to expressing delay in seconds as an integer value). Requiring the developer to convert back and forth between units is inconvenient and error-prone. Using the functions we’ve defined above, a developer would only need to add one line of code per binding throttled!

# Conclusion

For systems dealing with large volumes of spiky traffic in an event-based microservice architecture, heavy spiky load can  put immense pressure on critical components, often resulting in cascading effects that can compromise the system's stability. In today's talk, we explored a Go-based solution for a self-regulating pressure relief valve that automatically adjusts the overall data processing rate based on alerts for component resource limits. By using Go concepts such as channels, tickers, and context cancellation, we implemented a solution that temporarily slows down processing of noncritical workflows until components regain sufficient bandwidth to handle them. With this solution in place, we are able to strike a healthy balance between maximizing system throughput and protecting system stability.

