---
layout: post
title: "Reactive Streams Design Patterns"
date: 2017-02-17 09:00:00 -0600
categories: java
---

In the past, we've covered the [basics of using Reactor][prevpost] along with
a preview on how that works with the upcoming Spring Framework 5 release. Today,
we're going to take a deeper dive into the [reactive streams][rs] standard,
exploring how reactive streams are implemented and how to implement some common
design patterns using [Reactor][reactor].

## Reactive Streams API

To dive into the [reactive streams API][rsapi], we're going to create some
simple example implementations of the various interfaces by using
`java.io.InputStream` and `java.io.OutputStream` as delegates for implementations
of `Publisher<Byte>` and `Subscriber<Byte>` respectively. Note that all code
samples are published in [this GitHub project][example].

### Publisher

The first class we'll take a look at is implementing a `Publisher<T>` class
which will also cover the purpose of the `Subscription` interface as well.
The purpose of a `Publisher<T>` is to provide a source of data that can be
subscribed to by a `Subscriber<T>`. When a `Subscriber<T>` subscribes to
a `Publisher<T>`, the publisher will create a `Subscription` object which
the subscriber will be notified about. At this point, the subscriber can
either immediately being requesting data from the publisher, or it can
wait for something else to happen before requesting data. It is the
responsibility of the subscriber to request an amount of data that it can
reasonably process at a time.

```java
public class InputStreamPublisher implements Publisher<Byte> {

    private final Supplier<InputStream> streamSupplier;

    public InputStreamPublisher(final Supplier<InputStream> streamSupplier) {
        this.streamSupplier = Objects.requireNonNull(streamSupplier);
    }

    @Override
    public void subscribe(final Subscriber<? super Byte> subscriber) {
        Objects.requireNonNull(subscriber);
        InputStream is = streamSupplier.get();
        InputStreamSubscription s = new InputStreamSubscription(is, subscriber);
        subscriber.onSubscribe(s);
        if (is == null) {
            s.error(new NullPointerException("InputStream returned from Supplier was null"));
        }
    }
}
```

In this class, we'll use a `Supplier<InputStream>` so that each `Subscriber` to
this will be able to iterate over a new `InputStream` instead of complicating things
by supporting multicasting. There are two reactive streams rules that we followed
in the above: throw a `NullPointerException` if the subscriber is null, and call
`subscriber.onSubscribe()` with the newly created `Subscription`. There is a third
rule considered here that is implemented in our `Subscription` class that almost
all exceptions can only be signalled via the `Subscriber.onError()` method.
Things get more complicated as we examine our `Subscription` implementation.

### Subscription

```java
public class InputStreamSubscription implements Subscription {

    private final InputStream stream;
    private final Subscriber<? super Byte> subscriber;

    private volatile boolean cancelled;
    private final AtomicNonNegativeLong requested = new AtomicNonNegativeLong();

    private InputStreamSubscription(
        final InputStream stream,
        final Subscriber<? super Byte> subscriber
    ) {
	this.stream = stream;
	this.subscriber = subscriber;
    }

    @Override
    public void request(final long n) {
	if (cancelled) return;
	if (n <= 0) {
	    error(new IllegalArgumentException("Rule 3.9"));
	    return;
	}
	if (requested.getAndAdd(n) == 0) {
	    if (n == Long.MAX_VALUE) {
		requestUnbounded();
	    } else {
		requestBounded(n);
	    }
	}
    }

    private void requestBounded(final long n) {
	long i = 0, req = n;
	while (!cancelled) {
	    while (i < req) {
		if (cancelled) return;
		requestNextByte();
		i++;
	    }
	    req = requested.get();
	    if (i == req) {
		req = requested.addAndGet(-i);
		if (req == 0) return;
		i = 0;
	    }
	}
    }

    private void requestUnbounded() {
	while (!cancelled) {
	    requestNextByte();
	}
    }

    private void requestNextByte() {
	int read;
	try {
	    read = stream.read();
	} catch (IOException e) {
	    error(e);
	    return;
	}
	if (cancelled) return;
	if (read == -1) {
	    subscriber.onComplete();
	    cancel();
	    return;
	}
	subscriber.onNext((byte) read);
    }

    @Override
    public void cancel() {
	if (cancelled) return;
	try {
	    stream.close();
	} catch (IOException e) {
	    subscriber.onError(e);
	} finally {
	    cancelled = true;
	}
    }

    void error(final Throwable t) {
	if (cancelled) return;
	subscriber.onError(t);
	cancelled = true;
    }
}
```

Since our publisher is simply a wrapper around an `InputStream`, our `Subscription`
only needs to know about that as well as the subscriber. We also need to keep track
of when the subscription is cancelled so we can stop sending data to the subscriber,
and we also have an interesting bit of code surrounding the implementation of
[rule 3.3][rule303] preventing unwanted recursion in a specific use case that we'll
handle by keeping track of bounded requests until they complete.

One thing to keep in mind in this implementation is that we try to cancel the
subscription as fast as possible, though the standard allows us to get away with
lazier cancellation. This implementation detail provides a performance tradeoff
depending on the use case.

First, let's examine the `request()` method. We keep track of requests by
tracking the value given to us by the method and only sending data if a request
is not already in progress. This prevents an interesting recursion error that
would otherwise be a problem if the subscriber's `onNext()` method called this
subscription's `request()` method for some reason.

When a value of `Long.MAX_VALUE` is given, this is used to indicate that the
subscriber wants an unbounded stream of data. To handle this, we implement
`requestUnbounded()` to continually read from our `InputStream` until it ends
at which point we signal completion and close the stream. Any errors are
reported directly to the subscriber via `onError()`.

In the bounded case where the number of elements requested is between 1 and
`Long.MAX_VALUE - 1`, we make sure to read up to that many elements and signal
the subscriber with each value. When the input stream ends, we signal
completion and close the stream. Again, errors are signaled via `Subscriber.onError()`.

The other method implemented for a `Subscription` is `cancel()` which sets our
`cancelled` flag and closes the `InputStream`, reporting any exceptions from
that.

### Subscriber

The next class to implement is `Subscriber<T>`, and in this case we'll wrap
an `OutputStream` into a `Subscriber<Byte>`. Note that this implementation
is more demonstrative of what a subscriber's responsibilities are and is not
really optimized for real world usage. A better implementation may wish to
buffer bytes before writing them to the stream for instance.

```java
public class OutputStreamSubscriber implements Subscriber<Byte> {

    private final OutputStream stream;
    private Subscription subscription;
    private volatile boolean cancelled;

    public OutputStreamSubscriber(final OutputStream stream) {
        this.stream = Objects.requireNonNull(stream);
    }

    @Override
    public void onSubscribe(final Subscription subscription) {
        Objects.requireNonNull(subscription);
        if (cancelled) return;
        if (this.subscription != null) {
            subscription.cancel();
            return;
        }
        this.subscription = subscription;
        this.subscription.request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(final Byte b) {
        if (cancelled) return;
        try {
            stream.write(Objects.requireNonNull(b));
        } catch (IOException e) {
            e.printStackTrace();
            subscription.cancel();
            closeAndCancel();
        }
    }

    @Override
    public void onError(final Throwable throwable) {
        Objects.requireNonNull(throwable);
        if (cancelled) return;
        throwable.printStackTrace();
        closeAndCancel();
    }

    @Override
    public void onComplete() {
        closeAndCancel();
    }

    private void closeAndCancel() {
        if (cancelled) return;
        try {
            stream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        cancelled = true;
    }
}
```

As with a `Subscription`, we need to keep track of cancellation state. To
follow the standard properly, our `onSubscribe()` method only accepts the
first `Subscription` provided; subsequent subscriptions are rejected and
cancelled. Since the `OutputStream` this class wraps will be ready to
receive data on construction, we also begin requesting an unbounded
stream of data from the subscription. The rest of the code here is rather
straightforward as we handle each byte in our `onNext()` method by writing
them to the `OutputStream`. Exceptions are handled by printing their
stacktrace and cancelling the subscriber. Completion is handled almost the
same as an error but without printing an exception.

[prevpost]: http://musigma.org/java/2016/11/21/reactor.html
[rm]: http://www.reactivemanifesto.org/
[rs]: http://www.reactive-streams.org/
[rsapi]: http://www.reactive-streams.org/reactive-streams-1.0.0-javadoc/
[reactor]: https://projectreactor.io/
[example]: https://github.com/jvz/reactive-streams-examples
[rule303]: https://github.com/reactive-streams/reactive-streams-jvm/blob/v1.0.0/README.md#3.3
