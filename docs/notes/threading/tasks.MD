# Tasks
Tasks separate the specification and co-ordination of units of work from the details of how they are scheduled. Unlike threads, it is easy to get a return value from a task. Tasks can be chained together by specifying that one task continues when another completes. Large operations can be formed by combining smaller ones.

> The code fragments for this document are LINQPad fragments and can be found [here](https://github.com/kennyrnwilson/linqpad/tree/master/Queries/Notes/Threading/02%20Tasks). 

## Starting Tasks 
**1**
```cs
var t1 = Task.Factory.StartNew(() =>  Console.WriteLine("TaskFactory.StartNext"));
Console.WriteLine("TaskFactory.StartNext"));
```
**2**
```cs
var t2 = Task.Run(() => Console.WriteLine("Task.Run"));
```
**3**
```cs
var t3 = new Task(() => Console.WriteLine("new Task()"));
t3.Start();
```

## Unit of Work definition separate from scheduling
The following piece of code creates two separate but identical units of work and then schedules each one differently

**Separating Scheduling and UOW**
```cs
Console.WriteLine(Thread.CurrentThread.ManagedThreadId);

void Function()
{
	Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
}

Task t1 = new Task(Function);
Task t2 = new Task(Function);

t1.Start(TaskScheduler.Default);
t2.Start(TaskScheduler.Current);

```

## Chaining Tasks
 Tasks can be chained together using Continuations. Continuations are set up using the ContinueWith method. The following code shows how to chain two tasks together. We have two very basic methods **(1)**,**(2)** that carry out the work we want to do. We want **(2)** to execute once **(1)** is complete. We create a task **(3)** and pass in an Action delegate that invokes TaskOneFunction. We then add a continuation that will execute our second method. Note however we must add a wrapper method **(5)** that takes the antecedent task and calls TaskTwoFunction

**Chaining (1)**

 ```cs
 void Main()
{
	MyExtensions.SetupLog4Net();
	LogManager.GetLogger("Chaining Tasks").Info(nameof(Main));

	Task taskOne = /* (3) */ new Task(new Action(TaskOneFunction));

	/* (4) */
	taskOne.ContinueWith(new Action<Task>(TaskTwoFunctionWrapper));

	taskOne.Start();
}

/* (1) */
public void TaskOneFunction()
{
	LogManager.GetLogger("Chaining Tasks").Info(nameof(TaskOneFunction));
}

/* (2) */
public void TaskTwoFunction()
{
	LogManager.GetLogger("Chaining Tasks").Info(nameof(TaskTwoFunction));
}

/* (5) */
public void TaskTwoFunctionWrapper(Task antecedent)
{
	TaskTwoFunction();
}

 ```

 ## Chaining Tasks
 Now we move on to consider how to chain value returning tasks together. If our first task is of type *Task\<int>* then the continuations delegate function must take an argument of type *Task\<int>*.

**Chaining (2)**
``` cs
static ILog logger;

void Main()
{
	MyExtensions.SetupLog4Net();
	logger = LogManager.GetLogger("Chaining Value Returning Tasks");

	Task<int> taskOne = new Task<int>(new Func<int>(TaskOneFunction));

	Task<int> continuation = taskOne.ContinueWith(new Func<Task<int>, int>(TaskTwoFunctionWrapper));

	taskOne.Start();
	logger.Info(continuation.Result);
}


public int TaskOneFunction()
{
	logger.Info(nameof(TaskOneFunction));
	return 5;
}

public int TaskTwoFunction(int x)
{
	logger.Info(nameof(TaskTwoFunction));
	return x * 2;
}

public int TaskTwoFunctionWrapper(Task<int> antecedent)
{
	return TaskTwoFunction(antecedent.Result);
}
```

## Sub-tasks and Unwrap
Sometimes one task will want to use another task within its body. Where the tasks return values this can lead to a type of **(1)** *Task\<Task\<Tresult>>* which is rather inconvenient. We can get around this using the static method **(2)** Unwrap which creates a proxy task that only completes when the outer and inner tasks complete. It achieves this internally without blocking using callbacks.

**Unwrapping**
``` cs 
static ILog log = LogManager.GetLogger("Sub Tasks");

public static void Main()
{
	MyExtensions.SetupLog4Net();
	
	Task<double> GetSpot() => Task.Run(() => 100.0);
	
	Task<double> GetForward(double spot) => Task.Run(() =>spot * Math.Exp(0.1));
	
	/* (1) */ Task<Task<double>> forward = GetSpot()
		.ContinueWith(x => GetForward(x.Result));	
		
	// We can solve this using the Unwrap extension method.
	// This method creates a proxy Task that only completes
	// when both the outer and inner Tasks are ready	
	forward
		/* (1) */ .Unwrap()
		.ContinueWith(f => log.Info(f.Result));
}
```

The following example show how *Unwrap* and *ContinueWith* can be used to chain together value returing tasks where the output from one function forms the input to the next function

**Unwrapping 2**
``` cs
static ILog log = LogManager.GetLogger("Sub Tasks");

public static void Main()
{
	MyExtensions.SetupLog4Net();
	
	Task<int> Increment(int x)
	{
		return Task.Run(() =>
	    {
			log.Info(nameof(Increment));
			return ++x;
	    });
	}
	
	var result = Increment(0)
	.ContinueWith(f => Increment(f.Result))
	.Unwrap().ContinueWith(f => Increment(f.Result))
	.Unwrap().ContinueWith(f => Increment(f.Result))
	.Unwrap().ContinueWith(f => Increment(f.Result))
	.Unwrap();

	result.ContinueWith(r => log.Info(r.Result));
}
```

## Scheduling Tasks
The examples of creating and starting tasks in the previous section all utilised default scheduling which schedules tasks on a thread pool. If we want to specify the scheduler, we can use the following call.	

**Explicit Scheduling**
```
TaskScheduler scheduler = TaskScheduler.Default;
Task t3 = new Task(() => Console.WriteLine("new Task()"));
t3.Start(scheduler);
```

Again, we use the default scheduler but this time we explicitly specify it. The following built in options are supported

**Scheduling Options**

```cs
TaskScheduler s1 = TaskScheduler.Default;

TaskScheduler s2 = TaskScheduler.Current;

TaskScheduler s3 = TaskScheduler.FromCurrentSynchronizationContext();
```

It is often useful to add a level of indirection to the schedulers used. In this way we can switch in different schedulers as suits our needs. I.e., we might want to use a single threaded scheduler rather than thread pool when unit testing. Often a team will create something along the lines of *ISchedulerProvider* to provide this extra level of indirection.
If the out of the box .NET schedulers don’t do what you want you can always implement your own by sub classing *TaskScheduler*.  

## Scheduling Example
The following example show how to chain and schedule in a WPF Environment

```cs
[STAThread]
public static void Main()
{
	Window w = new Window();
	w.Loaded += WOnLoaded;
	w.Show();
	Dispatcher.Run();
}

private static void WOnLoaded(object s, RoutedEventArgs e)
{
	/* (1) */
	int Id() => Thread.CurrentThread.ManagedThreadId;

	Console.WriteLine($@"Handler {Id()}");

	var rateTask = new Task<double>(() =>
	{
		Thread.Sleep(1000);
		Console.WriteLine($@"Task1 {Id()}");
		return 0.1;
	});

	var fwdTask = rateTask.ContinueWith(task =>
	{
		Thread.Sleep(1000);
		Console.WriteLine($@"Task2 {Id()}");
		return 100 * Math.Exp(task.Result);
	}, TaskScheduler.Default); /* (3) */

	fwdTask.ContinueWith(task =>
	{
		Console.WriteLine($@"Task3 {Id()}");
		double result = task.Result;
		Console.WriteLine($@"Task3 Result = {result} {Id()}");
	}, TaskScheduler.FromCurrentSynchronizationContext()); /* (4) */

	rateTask.Start(TaskScheduler.Default);  /* (2) */

}
```

**(1)** The button click itself is executed by the UI thread and as such has an associated SynchronizationContext associated with it. **(2)** The first task in the chain is started and explicitly scheduled on the default thread pool scheduler. **(3)** The first continuation is also explicitly scheduled on the default scheduler. **(4)** The final continuation uses the special TaskScheduler.FromCurrentSynchronizationContext() to execute the task on the SynchronizationContext captured at the point the final task is created.

If we want the second background task to execute on the same thread pool thread as the first background task we can use the TaskContinuationOptions.ExecuteSynchronously **(5 )**option as follows

**TaskContinuationOptions.ExecuteSynchronously**
```cs
var fwdTask = rateTask.ContinueWith(task =>
{
	Thread.Sleep(1000);
	Console.WriteLine($@"Task2 {Id()}");
	return 100 * Math.Exp(task.Result);
}, TaskContinuationOptions.ExecuteSynchronously); /* (5) */

```

## Exception Handling 
Another benefit of tasks is that any uncaught exception raised from task code is re-thrown such that it can be caught by the caller when it invokes Task.Wait or Task.Result.

**Exception Handling**
```cs
public static void Main()
{
	MyExtensions.SetupLog4Net();
	TaskExceptionTest();
}


public static Task GetExceptionTask()
{
	return Task.Run(() =>
	{
		LogManager.GetLogger("GetExceptionTask").Info("exception raised");
		throw new ArgumentException();
	});
}

public static void TaskExceptionTest()
{
	try
	{
		var exceptionTask = GetExceptionTask();
		exceptionTask.Wait();
	}
	catch (Exception e)
	{
		LogManager.GetLogger("GetExceptionTask").Info($"Exception {e.GetType()}");
		LogManager.GetLogger("GetExceptionTask").Info($"Inner exception {e.InnerException.GetType()}");
	}
}
```


Because a task can cause multiple exceptions calling *Wait* or *GetResult* results in any exceptions being wrapped by an *AggregateException*. 

## Cancelling Tasks 
The threading API provides two types that standardise cancellation 
 * CancellationTokenSource 
 * CancellationToken

By using two types the API seeks to separate the responsibilities of listening to and responding to cancellation events from the raising of cancellations. The following code shows a simple use case. 

```cs
void Main()
{
	/* (1) */
	var source = new CancellationTokenSource();

	/* (2) */
	Task t = LongRunningAsyncMethod(source.Token);
	
	/* (3) */
	source.CancelAfter(2);

	try
	{
		/* (4) */
		t.Wait();
	}
	catch (Exception ex)
	{
		Console.WriteLine($"Caught {ex.InnerException}");
	}

	Console.WriteLine($"Final status: {t.Status}");
}

static async Task LongRunningAsyncMethod(CancellationToken token)
{                                              /* (2) */
	await Task.Delay(TimeSpan.FromSeconds(20), token);
}
```

**(1)** We start off by creating a CancellationTokenSource which we then pass into a long running operation **(2)** that will take about 20 seconds to run. We then instruct the *CancellationTokenSource* to carry out a cancellation after 2 seconds **(3)** and then carry out a wait on the task **(4)**. The output from executing this fragment is

```
Caught System.Threading.Tasks.TaskCanceledException: A task was canceled.
Final status: Canceled 

```
