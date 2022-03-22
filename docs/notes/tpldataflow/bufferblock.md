# BufferBlock\<T>
The BufferBlock is a **propagator** that provides support for **unbounded** or **bounded** **buffering** of messages of type T. Posting to the block causes values to be stored in FIFO order by the block. 

![Buffer Block Basics](./docs/buffer-block-basics.png)

## FIFO buffering 
When we call the Post extension method on a buffer block it causes values to be stored in FIFO order. The following code posts two values into an unbounded buffer.  

```cs
var b = new BufferBlock<int>();
b.Post(1);
b.Post(2);

WriteLine(b.Receive());
WriteLine(b.Receive());
WriteLine("All received");
```

The output is 

```
1
2
All received
```

## Full bounded buffer rejects when full
What happens when we post to a bounded buffer which is at capacity? The answer is that the BufferBlock will reject the posted message and return false. The following code shows this happening. 

If the buffer is already full then calling Post causes the buffer to reject the messags as we can see from the running the following code wich sets up a buffer with a bounded capacity of 1 and tries to post two consecutive values into it. 

``` cs
var b = new BufferBlock<int>(new DataflowBlockOptions {BoundedCapacity = 3});

for (int i = 0; i < 5; i++)
{
	WriteLine(@$"Thread {Thread.CurrentThread.ManagedThreadId} Sending {i} {(b.Post(i) 
		? "accepted" 
		: "rejected")}");
}

```
The output is.

```
Thread 1 Sending 0 accepted
Thread 1 Sending 1 accepted
Thread 1 Sending 2 accepted
Thread 1 Sending 3 rejected
Thread 1 Sending 4 rejected
```

## Receive Blocks on Empty BufferBlock
Calling *Receive* on a buffer with no values is a blocking operation. Note in the below sample “All received” is never written to the console. 

**BufferBlock receive**
```cs
var b = new BufferBlock<int>();
b.Post(1);
b.Post(2);

WriteLine(b.Receive());
WriteLine(b.Receive());
WriteLine(b.Receive());
WriteLine("All received");
```

The output is

```
1
2
```

## ReceiveAsync does not block

``` cs
var b = new BufferBlock<int>();
b.Post(1);
b.Post(2);

WriteLine(b.ReceiveAsync());
WriteLine(b.ReceiveAsync());
WriteLine(b.ReceiveAsync());

WriteLine("Called ReceiveAsync 3 times");
Sleep(3000);
b.Post(3);

```

We temporarily see the following **if we run the fragment in LINQPad**

```
1
2
awaiting...
Called ReceiveAsync 3 times

```

Then the final post comes in and it changes to

```
1
2
3
Called ReceiveAsync 3 times
```

## Synchronous Producer/Consumer with BufferBlock
The following code shows how to use a BufferBlock to create a synchronous Producer/Consumer. 

``` cs
BufferBlock<int> buffer = new BufferBlock<int>();

var consumerTask = Task.Factory.StartNew(() =>
{
	while (true)
	{
		int item = buffer.Receive();
		WriteLine($"Consumed item {item}");
	}
});

var producerTask = Task.Factory.StartNew(() =>
{
	for (int i = 0; i < 10; i++)
	{
		buffer.Post(i);
		Sleep(1000);
	}
});

Task.WaitAll(consumerTask, producerTask);
```

## ASynchronous Producer/Consumer with BufferBlock
The following code shows how to use a BufferBlock to create an asynchronous Producer/Consumer. 

```cs
BufferBlock<int> buffer = new BufferBlock<int>();

var consumerTask = Task.Factory.StartNew(async () =>
{
	while(true)
	{
		int item = await buffer.ReceiveAsync();
		WriteLine($"Consumed item {item}");
	}
});

var producerTask = Task.Factory.StartNew(() =>
{
	for (int i = 0; i < 10; i++)
	{
		buffer.Post(i);
		Sleep(1000);
	}
});


Task.WaitAll(consumerTask, producerTask);

```

## ASynchronous Producer/Consumer with Throttle

```cs
[STAThread]
public static void Main()
{
	// Create a buffer with a bound 
	var producerConsumer = new ProducerConsumer(3);

	Window w = new Window() { SizeToContent = SizeToContent.WidthAndHeight };
	StackPanel s = new StackPanel();

	Button b = new Button() { Content = "Consume" };
	b.Click += async (object sender, RoutedEventArgs e)
		=> await producerConsumer.Consume();
	s.Children.Add(b);

	Button c = new Button() { Content = "Produce" };
	c.Click += async (object sender, RoutedEventArgs e)
		=> await producerConsumer.Produce();
	s.Children.Add(c);

	w.Content = s;
	w.Show();
	Dispatcher.Run();
}


public class ProducerConsumer
{
	private BufferBlock<int> _buffer;
	private int _pcCnt = 0;

	public ProducerConsumer(int capacity)
	{
		var options = new DataflowBlockOptions { BoundedCapacity = 3 };
		_buffer = new BufferBlock<int>(options);

		// Fill buffer to capacity
		for (_pcCnt = 0; _pcCnt < capacity; _pcCnt++)
		{
			_buffer.Post(_pcCnt);
		}
	}

	public async Task Consume() => WriteLine(await _buffer.ReceiveAsync());

	public async Task Produce() => WriteLine(await _buffer.SendAsync(_pcCnt++));
```