# Goroutines in Go: A Comprehensive Guide

## Chapter 1: Understanding Goroutines

### 1.1 What are Goroutines?

Goroutines are Go's lightweight threads of execution. Think of them as workers in a factory who can work independently and simultaneously. Unlike traditional operating system threads that are heavy and expensive to create, goroutines are managed by Go's runtime and are extremely lightweight.

**Key characteristics:**
- **Lightweight**: Start with only 2KB of stack space (vs 1-2MB for OS threads)
- **Managed by Go runtime**: Not managed by the operating system
- **Concurrent execution**: Can run simultaneously with other goroutines
- **Easy to create**: Simple `go` keyword starts a new goroutine
- **Efficient scheduling**: Go runtime handles scheduling and context switching

### 1.2 Goroutines vs Traditional Threads

| Aspect | Goroutines | OS Threads |
|--------|------------|-------------|
| **Memory** | 2-8KB initial | 1-2MB |
| **Creation** | Instant | Slow |
| **Management** | Go runtime | OS kernel |
| **Context switching** | Fast | Slow |
| **Maximum count** | Millions | Thousands |
| **Scheduling** | Cooperative | Preemptive |

**Real-world analogy:** Think of goroutines as employees in a modern office building. Traditional threads are like having individual offices for each employee (expensive, takes time to set up). Goroutines are like having a flexible workspace where employees can quickly grab a desk and start working (cheap, instant setup).

### 1.3 Why Use Goroutines?

1. **Concurrency**: Handle multiple tasks simultaneously
2. **Efficiency**: Better resource utilization than threads
3. **Simplicity**: Easy to reason about concurrent code
4. **Scalability**: Can handle thousands of concurrent operations
5. **Performance**: Better performance for I/O-bound tasks

## Chapter 2: Basic Goroutine Usage

### 2.1 Starting a Goroutine

The simplest way to start a goroutine is using the `go` keyword followed by a function call.

#### 2.1.1 Basic Syntax

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Start a goroutine
    go sayHello("World")
    
    // Main function continues immediately
    fmt.Println("Main function continues...")
    
    // Wait a bit to see goroutine output
    time.Sleep(time.Second)
}

func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}
```

**What happens:**
1. `go sayHello("World")` starts a new goroutine
2. Main function continues executing immediately
3. The goroutine runs concurrently in the background
4. Without `time.Sleep`, the main function might exit before the goroutine finishes

#### 2.1.2 Anonymous Function Goroutines

```go
func main() {
    // Start goroutine with anonymous function
    go func() {
        fmt.Println("Anonymous goroutine running...")
        time.Sleep(time.Millisecond * 500)
        fmt.Println("Anonymous goroutine finished!")
    }()
    
    fmt.Println("Main function continues...")
    time.Sleep(time.Second)
}
```

**Benefits of anonymous functions:**
- No need to define separate functions for simple tasks
- Can capture variables from surrounding scope
- Good for one-off concurrent operations

#### 2.1.3 Multiple Goroutines

```go
func main() {
    // Start multiple goroutines
    for i := 1; i <= 5; i++ {
        go func(id int) {
            fmt.Printf("Goroutine %d starting...\n", id)
            time.Sleep(time.Millisecond * time.Duration(id*100))
            fmt.Printf("Goroutine %d finished!\n", id)
        }(i)
    }
    
    fmt.Println("All goroutines started...")
    time.Sleep(time.Second)
}
```

**Important note:** The loop variable `i` is captured by reference in the closure. That's why we pass `i` as a parameter to the anonymous function.

### 2.2 Goroutine Lifecycle

Understanding the lifecycle of goroutines is crucial for writing correct concurrent code.

#### 2.2.1 Lifecycle Stages

```go
func goroutineLifecycle() {
    fmt.Println("1. Goroutine starting...")
    
    // Simulate some work
    time.Sleep(time.Millisecond * 100)
    
    fmt.Println("2. Goroutine working...")
    
    // Simulate more work
    time.Sleep(time.Millisecond * 100)
    
    fmt.Println("3. Goroutine finishing...")
}

func main() {
    go goroutineLifecycle()
    
    fmt.Println("Main function...")
    time.Sleep(time.Millisecond * 300)
    fmt.Println("Main function done")
}
```

**Lifecycle stages:**
1. **Creation**: `go` keyword creates the goroutine
2. **Execution**: Goroutine runs concurrently with other code
3. **Completion**: Goroutine finishes when function returns
4. **Cleanup**: Go runtime cleans up resources

#### 2.2.2 Goroutine Termination

```go
func main() {
    // Start a goroutine
    go func() {
        fmt.Println("Goroutine starting...")
        time.Sleep(time.Millisecond * 500)
        fmt.Println("Goroutine finishing...")
    }()
    
    fmt.Println("Main function exiting...")
    // Main function exits immediately
    // Goroutine is terminated when main exits
}
```

**Critical behavior:** When the main function exits, all goroutines are terminated immediately, regardless of whether they've finished their work.

#### 2.2.3 Preventing Premature Termination

One of the most critical challenges in concurrent programming is ensuring that goroutines have sufficient time to complete their work before the main program terminates. Without proper coordination, you risk losing work, leaving resources in an inconsistent state, or missing important results.

**The Problem: Uncoordinated Termination**

When the main function exits, Go's runtime immediately terminates all running goroutines, regardless of their current state. This can lead to several issues:

- **Data loss**: Goroutines might be in the middle of processing data
- **Resource leaks**: Open files, network connections, or database transactions might not be properly closed
- **Incomplete operations**: Long-running tasks might be cut short
- **Inconsistent state**: Shared resources might be left in an intermediate state

**Solution 1: Channel-Based Coordination**

The most straightforward approach is to use channels as synchronization points between the main function and goroutines:

```go
func main() {
    // Create a coordination channel
    done := make(chan bool)
    
    // Start goroutine with coordination
    go func() {
        fmt.Println("Goroutine: Starting work...")
        
        // Simulate some meaningful work
        time.Sleep(time.Millisecond * 500)
        
        // Perform cleanup or finalization
        fmt.Println("Goroutine: Finalizing results...")
        time.Sleep(time.Millisecond * 100)
        
        fmt.Println("Goroutine: Work completed successfully!")
        
        // Signal completion to main function
        done <- true
    }()
    
    fmt.Println("Main: Waiting for goroutine to complete...")
    
    // Block until goroutine signals completion
    <-done
    
    fmt.Println("Main: Goroutine finished, proceeding with exit...")
    fmt.Println("Main function exiting gracefully...")
}
```

**How it works:**
1. The main function creates a channel (`done`) for coordination
2. The goroutine performs its work and sends a signal when complete
3. The main function blocks on `<-done` until the signal is received
4. Only after coordination does the main function proceed to exit

**Solution 2: Multiple Goroutine Coordination**

When you have multiple goroutines, you need to coordinate with all of them:

```go
func main() {
    // Create coordination channels for multiple goroutines
    worker1Done := make(chan bool)
    worker2Done := make(chan bool)
    worker3Done := make(chan bool)
    
    // Start multiple workers
    go worker("Worker 1", 300, worker1Done)
    go worker("Worker 2", 500, worker2Done)
    go worker("Worker 3", 200, worker3Done)
    
    fmt.Println("Main: All workers started, waiting for completion...")
    
    // Wait for all workers to complete
    <-worker1Done
    <-worker2Done
    <-worker3Done
    
    fmt.Println("Main: All workers completed, exiting...")
}

func worker(name string, duration time.Duration, done chan<- bool) {
    fmt.Printf("%s: Starting work...\n", name)
    
    // Simulate work
    time.Sleep(time.Millisecond * duration)
    
    fmt.Printf("%s: Work completed!\n", name)
    
    // Signal completion
    done <- true
}
```

**Solution 3: Using sync.WaitGroup for Multiple Goroutines**

For scenarios with many goroutines, `sync.WaitGroup` provides a more elegant solution:

```go
import "sync"

func main() {
    var wg sync.WaitGroup
    
    // Start multiple workers
    for i := 1; i <= 5; i++ {
        wg.Add(1) // Increment counter before starting goroutine
        
        go func(workerID int) {
            defer wg.Done() // Decrement counter when goroutine finishes
            
            fmt.Printf("Worker %d: Starting...\n", workerID)
            
            // Simulate work
            workDuration := time.Duration(workerID*100) * time.Millisecond
            time.Sleep(workDuration)
            
            fmt.Printf("Worker %d: Completed after %v\n", workerID, workDuration)
        }(i)
    }
    
    fmt.Println("Main: All workers started, waiting for completion...")
    
    // Wait for all goroutines to complete
    wg.Wait()
    
    fmt.Println("Main: All workers completed, exiting...")
}
```

**Solution 4: Context-Based Cancellation with Coordination**

For more sophisticated scenarios, you can combine context cancellation with coordination:

```go
import "context"

func main() {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)
    defer cancel()
    
    // Create coordination channel
    done := make(chan bool)
    
    // Start worker that respects context
    go func() {
        defer func() {
            done <- true // Always signal completion
        }()
        
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Worker: Context cancelled, cleaning up...")
                time.Sleep(time.Millisecond * 100) // Simulate cleanup
                fmt.Println("Worker: Cleanup completed")
                return
            default:
                fmt.Println("Worker: Processing...")
                time.Sleep(time.Millisecond * 200)
            }
        }
    }()
    
    fmt.Println("Main: Worker started, waiting for completion...")
    
    // Wait for worker to finish (either naturally or due to cancellation)
    <-done
    
    fmt.Println("Main: Worker finished, exiting...")
}
```

**Best Practices for Preventing Premature Termination**

1. **Always coordinate**: Never assume goroutines will finish before main exits
2. **Use appropriate mechanisms**: 
   - Channels for simple one-to-one coordination
   - `sync.WaitGroup` for multiple goroutines
   - Context for cancellation scenarios
3. **Handle cleanup**: Ensure goroutines can clean up resources before terminating
4. **Consider timeouts**: Use context with timeouts to prevent indefinite waiting
5. **Log coordination**: Add logging to understand the flow of your concurrent program

**Common Anti-Patterns to Avoid**

```go
// DON'T: Rely on sleep timing
func main() {
    go worker()
    time.Sleep(time.Second) // Unreliable and wasteful
}

// DON'T: Assume goroutines will finish quickly
func main() {
    go longRunningTask() // Might not complete before main exits
    // Main exits immediately
}

// DON'T: Forget to coordinate with all goroutines
func main() {
    for i := 0; i < 10; i++ {
        go worker(i) // Only coordinating with some workers
    }
    // Main might exit before all workers complete
}
```

**Real-World Example: File Processing Pipeline**

Here's a practical example that demonstrates proper coordination:

```go
func main() {
    // Create coordination channels
    filesProcessed := make(chan bool)
    resultsCollected := make(chan bool)
    
    // Start file processor
    go func() {
        defer func() {
            filesProcessed <- true
        }()
        
        fmt.Println("File processor: Starting to process files...")
        
        // Simulate processing multiple files
        for i := 1; i <= 5; i++ {
            fmt.Printf("File processor: Processing file %d...\n", i)
            time.Sleep(time.Millisecond * 300)
            fmt.Printf("File processor: File %d completed\n", i)
        }
        
        fmt.Println("File processor: All files processed")
    }()
    
    // Start result collector
    go func() {
        defer func() {
            resultsCollected <- true
        }()
        
        fmt.Println("Result collector: Waiting for files to be processed...")
        
        // Wait for file processing to complete
        <-filesProcessed
        
        fmt.Println("Result collector: Collecting results...")
        time.Sleep(time.Millisecond * 200)
        fmt.Println("Result collector: Results collected successfully")
    }()
    
    fmt.Println("Main: Pipeline started, waiting for completion...")
    
    // Wait for both stages to complete
    <-filesProcessed
    <-resultsCollected
    
    fmt.Println("Main: Pipeline completed successfully, exiting...")
}
```

This approach ensures that your concurrent programs are robust, predictable, and don't lose work due to premature termination. The key is to always think about the lifecycle of your goroutines and how they coordinate with the main program flow.

## Chapter 3: Goroutine Communication with Channels

### 3.1 Channels as Communication Mechanism

Channels are Go's primary mechanism for communication between goroutines. Think of them as pipes that connect different parts of your concurrent system.

#### 3.1.1 Basic Channel Communication

```go
func main() {
    // Create a channel
    ch := make(chan string)
    
    // Start goroutine that sends data
    go func() {
        fmt.Println("Goroutine: Sending message...")
        ch <- "Hello from goroutine!"
        fmt.Println("Goroutine: Message sent!")
    }()
    
    // Main function receives data
    fmt.Println("Main: Waiting for message...")
    message := <-ch
    fmt.Printf("Main: Received: %s\n", message)
}
```

**What happens:**
1. Goroutine tries to send message to channel
2. Main function receives message from channel
3. Communication is synchronized (unbuffered channel)
4. Both goroutines coordinate their execution

#### 3.1.2 Bidirectional Communication

```go
func main() {
    ch := make(chan string)
    
    // Goroutine sends and receives
    go func() {
        ch <- "Hello from goroutine!"
        response := <-ch
        fmt.Printf("Goroutine received: %s\n", response)
    }()
    
    // Main function receives and sends
    message := <-ch
    fmt.Printf("Main received: %s\n", message)
    ch <- "Hello from main!"
    
    time.Sleep(time.Millisecond * 100)
}
```

**Communication pattern:**
1. Goroutine sends "Hello from goroutine!"
2. Main receives it and sends "Hello from main!"
3. Goroutine receives the response
4. Both sides can send and receive

#### 3.1.3 Multiple Goroutines with One Channel

```go
func main() {
    ch := make(chan int)
    
    // Start multiple sender goroutines
    for i := 1; i <= 3; i++ {
        go func(id int) {
            for j := 0; j < 3; j++ {
                ch <- id*10 + j
                time.Sleep(time.Millisecond * 100)
            }
        }(i)
    }
    
    // Main function receives all messages
    for i := 0; i < 9; i++ {
        value := <-ch
        fmt.Printf("Received: %d\n", value)
    }
}
```

**Behavior:** Multiple goroutines can send to the same channel, and the receiver gets messages from any of them. The order is not guaranteed unless you implement explicit coordination.

### 3.2 Channel Types and Behavior

#### 3.2.1 Unbuffered Channels (Synchronous)

```go
func main() {
    ch := make(chan int) // Unbuffered
    
    go func() {
        fmt.Println("Goroutine: About to send...")
        ch <- 42
        fmt.Println("Goroutine: Sent successfully!")
    }()
    
    fmt.Println("Main: About to receive...")
    value := <-ch
    fmt.Printf("Main: Received %d\n", value)
}
```

**Unbuffered channel behavior:**
- Sender blocks until receiver is ready
- Receiver blocks until sender is ready
- Perfect for synchronization
- Ensures both sides are ready before communication

#### 3.2.2 Buffered Channels (Asynchronous)

```go
func main() {
    ch := make(chan int, 3) // Buffer of 3
    
    // Send multiple values without blocking
    go func() {
        for i := 1; i <= 5; i++ {
            ch <- i
            fmt.Printf("Sent: %d\n", i)
        }
        close(ch)
    }()
    
    // Receive all values
    for value := range ch {
        fmt.Printf("Received: %d\n", value)
        time.Sleep(time.Millisecond * 200) // Simulate slow processing
    }
}
```

**Buffered channel behavior:**
- Sender can send multiple values without blocking (until buffer is full)
- Receiver can read multiple values without blocking (until buffer is empty)
- Provides flow control and backpressure
- Good for handling speed differences between producer and consumer

#### 3.2.3 Channel Direction

```go
func main() {
    ch := make(chan int)
    
    // Start producer (send-only)
    go producer(ch)
    
    // Start consumer (receive-only)
    go consumer(ch)
    
    time.Sleep(time.Second)
}

// Send-only channel
func producer(out chan<- int) {
    for i := 1; i <= 5; i++ {
        out <- i
        time.Sleep(time.Millisecond * 100)
    }
    close(out)
}

// Receive-only channel
func consumer(in <-chan int) {
    for value := range in {
        fmt.Printf("Consumed: %d\n", value)
    }
}
```

**Directional channels:**
- `chan<- int`: Send-only (can only send to channel)
- `<-chan int`: Receive-only (can only receive from channel)
- `chan int`: Bidirectional (can send and receive)
- Makes code intent clear and prevents misuse

## Chapter 4: Advanced Goroutine Patterns

### 4.1 Worker Pool Pattern

The worker pool pattern is like having a team of workers who take jobs from a queue and process them.

#### 4.1.1 Basic Worker Pool

```go
func main() {
    // Create job channel
    jobs := make(chan int, 100)
    
    // Create result channel
    results := make(chan int, 100)
    
    // Start workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }
    
    // Send jobs
    go func() {
        for j := 1; j <= 9; j++ {
            jobs <- j
        }
        close(jobs)
    }()
    
    // Collect results
    for a := 1; a <= 9; a++ {
        result := <-results
        fmt.Printf("Result: %d\n", result)
    }
}

func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        
        // Simulate work
        time.Sleep(time.Millisecond * 500)
        
        // Send result
        results <- job * 2
    }
}
```

**Worker pool benefits:**
- **Controlled concurrency**: Limit number of concurrent operations
- **Load balancing**: Work is distributed among workers
- **Resource management**: Prevent system overload
- **Scalability**: Easy to adjust number of workers

#### 4.1.2 Worker Pool with Error Handling

```go
type Job struct {
    ID    int
    Data  string
    Retry int
}

type Result struct {
    JobID int
    Data  string
    Error error
}

func main() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    
    // Start workers
    for w := 1; w <= 3; w++ {
        go workerWithErrorHandling(w, jobs, results)
    }
    
    // Send jobs
    go func() {
        for j := 1; j <= 10; j++ {
            jobs <- Job{ID: j, Data: fmt.Sprintf("data_%d", j)}
        }
        close(jobs)
    }()
    
    // Collect results
    for a := 1; a <= 10; a++ {
        result := <-results
        if result.Error != nil {
            fmt.Printf("Job %d failed: %v\n", result.JobID, result.Error)
        } else {
            fmt.Printf("Job %d succeeded: %s\n", result.JobID, result.Data)
        }
    }
}

func workerWithErrorHandling(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job.ID)
        
        // Simulate work with potential errors
        result := processJob(job)
        
        // Send result
        results <- result
    }
}

func processJob(job Job) Result {
    // Simulate processing with random errors
    if rand.Float64() < 0.2 { // 20% chance of error
        return Result{
            JobID: job.ID,
            Error: fmt.Errorf("processing failed for job %d", job.ID),
        }
    }
    
    // Simulate work
    time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
    
    return Result{
        JobID: job.ID,
        Data:  fmt.Sprintf("processed_%s", job.Data),
    }
}
```

### 4.2 Pipeline Pattern with Goroutines

The pipeline pattern processes data through multiple stages, with each stage running in its own goroutine.

#### 4.2.1 Basic Pipeline

```go
func main() {
    // Create pipeline
    numbers := generate(1, 2, 3, 4, 5)
    squared := square(numbers)
    doubled := double(squared)
    
    // Consume results
    for result := range doubled {
        fmt.Printf("Final result: %d\n", result)
    }
}

func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
            time.Sleep(time.Millisecond * 100)
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
            time.Sleep(time.Millisecond * 100)
        }
    }()
    return out
}

func double(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * 2
            time.Sleep(time.Millisecond * 100)
        }
    }()
    return out
}
```

**Pipeline benefits:**
- **Modularity**: Each stage has a single responsibility
- **Concurrency**: All stages run simultaneously
- **Reusability**: Stages can be combined in different ways
- **Scalability**: Easy to add more workers to slow stages

#### 4.2.2 Pipeline with Fan-Out and Fan-In

```go
func main() {
    // Generate numbers
    numbers := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // Fan out to multiple workers
    workers := fanOut(numbers, 3)
    
    // Fan in results
    results := fanIn(workers...)
    
    // Consume results
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }
}

func fanOut(input <-chan int, workers int) []<-chan int {
    outputs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outputs[i] = worker(input, i)
    }
    return outputs
}

func worker(input <-chan int, id int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for n := range input {
            // Simulate work
            time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
            result := n * n
            fmt.Printf("Worker %d processed %d -> %d\n", id, n, result)
            output <- result
        }
    }()
    return output
}

func fanIn(channels ...<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(input <-chan int) {
            defer wg.Done()
            for n := range input {
                output <- n
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(output)
    }()
    
    return output
}
```

### 4.3 Select Statement with Goroutines

The `select` statement allows goroutines to wait on multiple channel operations simultaneously.

#### 4.3.1 Basic Select

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    // Start goroutines that send on different channels
    go func() {
        time.Sleep(time.Millisecond * 100)
        ch1 <- "Message from channel 1"
    }()
    
    go func() {
        time.Sleep(time.Millisecond * 200)
        ch2 <- "Message from channel 2"
    }()
    
    // Wait for first message from either channel
    select {
    case msg1 := <-ch1:
        fmt.Printf("Received from ch1: %s\n", msg1)
    case msg2 := <-ch2:
        fmt.Printf("Received from ch2: %s\n", msg2)
    }
}
```

#### 4.3.2 Select with Timeout

```go
func main() {
    ch := make(chan string)
    
    // Start goroutine that might be slow
    go func() {
        time.Sleep(time.Millisecond * 300)
        ch <- "Slow message"
    }()
    
    // Wait with timeout
    select {
    case msg := <-ch:
        fmt.Printf("Received: %s\n", msg)
    case <-time.After(time.Millisecond * 200):
        fmt.Println("Timeout! Message not received")
    }
}
```

#### 4.3.3 Non-blocking Select

```go
func main() {
    ch := make(chan string)
    
    // Non-blocking receive
    select {
    case msg := <-ch:
        fmt.Printf("Received: %s\n", msg)
    default:
        fmt.Println("No message available")
    }
    
    // Non-blocking send
    select {
    case ch <- "Hello":
        fmt.Println("Message sent")
    default:
        fmt.Println("Channel full, message not sent")
    }
}
```

## Chapter 5: Goroutine Synchronization

### 5.1 WaitGroup for Coordination

`sync.WaitGroup` is like a counter that tracks how many goroutines are still working.

#### 5.1.1 Basic WaitGroup Usage

```go
func main() {
    var wg sync.WaitGroup
    
    // Start multiple goroutines
    for i := 1; i <= 5; i++ {
        wg.Add(1) // Increment counter
        
        go func(id int) {
            defer wg.Done() // Decrement counter when done
            
            fmt.Printf("Worker %d starting...\n", id)
            time.Sleep(time.Millisecond * time.Duration(id*100))
            fmt.Printf("Worker %d finished!\n", id)
        }(i)
    }
    
    fmt.Println("Waiting for all workers to finish...")
    wg.Wait() // Wait until counter reaches zero
    fmt.Println("All workers finished!")
}
```

**WaitGroup operations:**
- `wg.Add(n)`: Add `n` to the counter
- `wg.Done()`: Subtract 1 from the counter
- `wg.Wait()`: Block until counter reaches zero

#### 5.1.2 WaitGroup with Error Handling

```go
func main() {
    var wg sync.WaitGroup
    errors := make(chan error, 5)
    
    // Start workers
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go workerWithError(i, &wg, errors)
    }
    
    // Wait for all workers in a separate goroutine
    go func() {
        wg.Wait()
        close(errors)
    }()
    
    // Collect errors
    for err := range errors {
        if err != nil {
            fmt.Printf("Worker error: %v\n", err)
        }
    }
    
    fmt.Println("All workers finished!")
}

func workerWithError(id int, wg *sync.WaitGroup, errors chan<- error) {
    defer wg.Done()
    
    // Simulate work with potential errors
    if rand.Float64() < 0.3 { // 30% chance of error
        errors <- fmt.Errorf("worker %d failed", id)
        return
    }
    
    fmt.Printf("Worker %d working...\n", id)
    time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
    fmt.Printf("Worker %d finished successfully\n", id)
}
```

### 5.2 Mutex for Shared Data Protection

When multiple goroutines need to access shared data, you need to protect it with a mutex.

#### 5.2.1 Basic Mutex Usage

```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) GetValue() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    counter := &Counter{}
    var wg sync.WaitGroup
    
    // Start multiple goroutines that increment the counter
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Final counter value: %d\n", counter.GetValue())
}
```

**Mutex operations:**
- `mu.Lock()`: Acquire the lock (block if already locked)
- `mu.Unlock()`: Release the lock
- `defer mu.Unlock()`: Ensure lock is released even if function panics

#### 5.2.2 RWMutex for Read/Write Operations

```go
type DataStore struct {
    mu     sync.RWMutex
    data   map[string]string
}

func (ds *DataStore) Set(key, value string) {
    ds.mu.Lock()
    defer ds.mu.Unlock()
    ds.data[key] = value
}

func (ds *DataStore) Get(key string) (string, bool) {
    ds.mu.RLock() // Read lock
    defer ds.mu.RUnlock()
    value, exists := ds.data[key]
    return value, exists
}

func main() {
    store := &DataStore{data: make(map[string]string)}
    var wg sync.WaitGroup
    
    // Start readers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                store.Get("key")
            }
        }(i)
    }
    
    // Start writers
    for i := 0; i < 2; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 50; j++ {
                store.Set(fmt.Sprintf("key_%d", j), fmt.Sprintf("value_%d", j))
            }
        }(i)
    }
    
    wg.Wait()
    fmt.Println("All operations completed!")
}
```

**RWMutex benefits:**
- Multiple readers can access data simultaneously
- Only one writer can access data at a time
- Better performance when reads are more frequent than writes

### 5.3 Context for Cancellation

Context provides a way to cancel goroutines and pass request-scoped values.

#### 5.3.1 Basic Context Usage

```go
func main() {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
    defer cancel()
    
    // Start goroutine that respects context
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Goroutine cancelled!")
                return
            default:
                fmt.Println("Working...")
                time.Sleep(time.Millisecond * 500)
            }
        }
    }()
    
    // Wait for context to be cancelled
    <-ctx.Done()
    fmt.Println("Main function: Context cancelled!")
}
```

#### 5.3.2 Context with Values

```go
func main() {
    // Create context with values
    ctx := context.WithValue(context.Background(), "user_id", "12345")
    ctx = context.WithValue(ctx, "request_id", "req_67890")
    
    // Start goroutine that uses context values
    go processRequest(ctx)
    
    time.Sleep(time.Second)
}

func processRequest(ctx context.Context) {
    userID := ctx.Value("user_id").(string)
    requestID := ctx.Value("request_id").(string)
    
    fmt.Printf("Processing request %s for user %s\n", requestID, userID)
    
    // Simulate work
    time.Sleep(time.Millisecond * 500)
    
    fmt.Printf("Request %s completed!\n", requestID)
}
```

#### 5.3.3 Context Cancellation Chain

```go
func main() {
    // Create parent context
    parentCtx, parentCancel := context.WithCancel(context.Background())
    defer parentCancel()
    
    // Create child context
    childCtx, childCancel := context.WithCancel(parentCtx)
    defer childCancel()
    
    // Start goroutines
    go worker(parentCtx, "Parent")
    go worker(childCtx, "Child")
    
    // Cancel child context after 1 second
    time.Sleep(time.Second)
    fmt.Println("Cancelling child context...")
    childCancel()
    
    // Cancel parent context after 2 seconds
    time.Sleep(time.Second)
    fmt.Println("Cancelling parent context...")
    parentCancel()
    
    time.Sleep(time.Millisecond * 500)
}

func worker(ctx context.Context, name string) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("%s worker cancelled: %v\n", name, ctx.Err())
            return
        default:
            fmt.Printf("%s worker working...\n", name)
            time.Sleep(time.Millisecond * 300)
        }
    }
}
```

## Chapter 6: Common Goroutine Patterns

### 6.1 Producer-Consumer Pattern

The producer-consumer pattern is like a factory where one group produces items and another group consumes them.

#### 6.1.1 Basic Producer-Consumer

```go
func main() {
    // Create channels
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    
    // Start producer
    go producer(jobs)
    
    // Start consumers
    for i := 1; i <= 3; i++ {
        go consumer(i, jobs, results)
    }
    
    // Collect results
    go func() {
        for i := 0; i < 10; i++ {
            result := <-results
            fmt.Printf("Result: %d\n", result)
        }
    }()
    
    time.Sleep(time.Second * 2)
}

func producer(jobs chan<- int) {
    for i := 1; i <= 10; i++ {
        jobs <- i
        fmt.Printf("Produced job %d\n", i)
        time.Sleep(time.Millisecond * 100)
    }
    close(jobs)
}

func consumer(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        fmt.Printf("Consumer %d processing job %d\n", id, job)
        
        // Simulate work
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
        
        // Send result
        results <- job * 2
        fmt.Printf("Consumer %d completed job %d\n", id, job)
    }
}
```

#### 6.1.2 Producer-Consumer with Rate Limiting

```go
func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // Start producer with rate limiting
    go rateLimitedProducer(jobs)
    
    // Start consumers
    for i := 1; i <= 5; i++ {
        go consumer(id, jobs, results)
    }
    
    // Collect results
    go resultCollector(results)
    
    time.Sleep(time.Second * 5)
}

func rateLimitedProducer(jobs chan<- int) {
    ticker := time.NewTicker(time.Millisecond * 200) // 5 jobs per second
    defer ticker.Stop()
    
    jobID := 1
    for {
        select {
        case <-ticker.C:
            if jobID <= 20 {
                jobs <- jobID
                fmt.Printf("Produced job %d\n", jobID)
                jobID++
            } else {
                close(jobs)
                return
            }
        }
    }
}

func resultCollector(results <-chan int) {
    count := 0
    for result := range results {
        count++
        fmt.Printf("Collected result %d: %d\n", count, result)
    }
    fmt.Printf("Total results collected: %d\n", count)
}
```

### 6.2 Pub-Sub Pattern

The publish-subscribe pattern is like a newspaper where publishers create content and subscribers receive it.

#### 6.2.1 Basic Pub-Sub

```go
type Message struct {
    Topic   string
    Content string
}

type PubSub struct {
    subscribers map[string][]chan Message
    mu          sync.RWMutex
}

func NewPubSub() *PubSub {
    return &PubSub{
        subscribers: make(map[string][]chan Message),
    }
}

func (ps *PubSub) Subscribe(topic string) <-chan Message {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    ch := make(chan Message, 1)
    ps.subscribers[topic] = append(ps.subscribers[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic string, content string) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()
    
    message := Message{Topic: topic, Content: content}
    
    for _, ch := range ps.subscribers[topic] {
        select {
        case ch <- message:
            // Message sent successfully
        default:
            // Channel is full, skip this subscriber
        }
    }
}

func main() {
    pubsub := NewPubSub()
    
    // Subscribe to topics
    newsCh := pubsub.Subscribe("news")
    sportsCh := pubsub.Subscribe("sports")
    weatherCh := pubsub.Subscribe("weather")
    
    // Start subscribers
    go subscriber("News Subscriber", newsCh)
    go subscriber("Sports Subscriber", sportsCh)
    go subscriber("Weather Subscriber", weatherCh)
    
    // Publish messages
    go func() {
        for i := 1; i <= 5; i++ {
            pubsub.Publish("news", fmt.Sprintf("News update %d", i))
            pubsub.Publish("sports", fmt.Sprintf("Sports update %d", i))
            pubsub.Publish("weather", fmt.Sprintf("Weather update %d", i))
            time.Sleep(time.Millisecond * 500)
        }
    }()
    
    time.Sleep(time.Second * 3)
}

func subscriber(name string, ch <-chan Message) {
    for message := range ch {
        fmt.Printf("%s received: %s - %s\n", name, message.Topic, message.Content)
    }
}
```

### 6.3 Pipeline with Error Handling

Building robust pipelines that can handle errors gracefully.

#### 6.3.1 Error-Aware Pipeline

```go
type PipelineItem struct {
    ID    int
    Data  string
    Error error
}

func main() {
    // Create pipeline with error handling
    input := generateItems(10)
    processed := processItems(input)
    validated := validateItems(processed)
    
    // Consume results
    successCount := 0
    errorCount := 0
    
    for item := range validated {
        if item.Error != nil {
            errorCount++
            fmt.Printf("Item %d failed: %v\n", item.ID, item.Error)
        } else {
            successCount++
            fmt.Printf("Item %d succeeded: %s\n", item.ID, item.Data)
        }
    }
    
    fmt.Printf("\nPipeline completed: %d successes, %d errors\n", successCount, errorCount)
}

func generateItems(count int) <-chan PipelineItem {
    out := make(chan PipelineItem)
    go func() {
        defer close(out)
        for i := 1; i <= count; i++ {
            out <- PipelineItem{
                ID:   i,
                Data: fmt.Sprintf("data_%d", i),
            }
        }
    }()
    return out
}

func processItems(input <-chan PipelineItem) <-chan PipelineItem {
    out := make(chan PipelineItem)
    go func() {
        defer close(out)
        for item := range input {
            // Simulate processing with potential errors
            if rand.Float64() < 0.2 { // 20% chance of error
                item.Error = fmt.Errorf("processing failed for item %d", item.ID)
            } else {
                item.Data = fmt.Sprintf("processed_%s", item.Data)
            }
            out <- item
        }
    }()
    return out
}

func validateItems(input <-chan PipelineItem) <-chan PipelineItem {
    out := make(chan PipelineItem)
    go func() {
        defer close(out)
        for item := range input {
            if item.Error != nil {
                // Skip validation for failed items
                out <- item
                continue
            }
            
            // Simulate validation
            if len(item.Data) < 10 {
                item.Error = fmt.Errorf("validation failed: data too short")
            }
            out <- item
        }
    }()
    return out
}
```

## Chapter 7: Goroutine Best Practices

### 7.1 Always Handle Goroutine Lifecycle

#### 7.1.1 Proper Cleanup

```go
func main() {
    // Create context for cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Start goroutines
    var wg sync.WaitGroup
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(ctx, &wg, i)
    }
    
    // Wait for user input to cancel
    fmt.Println("Press Enter to cancel all workers...")
    fmt.Scanln()
    
    // Cancel all workers
    cancel()
    
    // Wait for all workers to finish
    wg.Wait()
    fmt.Println("All workers finished!")
}

func worker(ctx context.Context, wg *sync.WaitGroup, id int) {
    defer wg.Done()
    
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d cancelled!\n", id)
            return
        default:
            fmt.Printf("Worker %d working...\n", id)
            time.Sleep(time.Millisecond * 500)
        }
    }
}
```

#### 7.1.2 Resource Management

```go
func main() {
    // Create resource pool
    pool := make(chan int, 3)
    for i := 1; i <= 3; i++ {
        pool <- i
    }
    
    var wg sync.WaitGroup
    
    // Start workers that use resources
    for i := 1; i <= 10; i++ {
        wg.Add(1)
        go resourceWorker(i, pool, &wg)
    }
    
    wg.Wait()
    fmt.Println("All workers finished!")
}

func resourceWorker(id int, pool <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    
    // Acquire resource
    resource := <-pool
    fmt.Printf("Worker %d acquired resource %d\n", id, resource)
    
    // Use resource
    time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
    
    // Release resource
    pool <- resource
    fmt.Printf("Worker %d released resource %d\n", id, resource)
}
```

### 7.2 Avoid Common Pitfalls

#### 7.2.1 Goroutine Leaks

```go
// BAD: Goroutine leak
func leakyFunction() {
    ch := make(chan int)
    go func() {
        for {
            select {
            case <-ch:
                return
            default:
                // Busy loop - never exits
            }
        }
    }()
    // Goroutine continues running forever
}

// GOOD: Proper exit condition
func properFunction() {
    ch := make(chan int)
    done := make(chan bool)
    
    go func() {
        defer close(done)
        for {
            select {
            case <-ch:
                return
            case <-done:
                return
            }
        }
    }()
    
    close(done) // Signal goroutine to exit
}
```

#### 7.2.2 Race Conditions

```go
// BAD: Race condition
func badCounter() {
    counter := 0
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // Race condition!
        }()
    }
    
    wg.Wait()
    fmt.Printf("Counter: %d\n", counter) // Result is unpredictable
}

// GOOD: Protected with mutex
func goodCounter() {
    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    fmt.Printf("Counter: %d\n", counter) // Always 1000
}
```

#### 7.2.3 Channel Deadlocks

```go
// BAD: Deadlock
func deadlock() {
    ch := make(chan int)
    ch <- 42 // Blocks forever - no receiver
}

// GOOD: Proper coordination
func noDeadlock() {
    ch := make(chan int)
    
    go func() {
        ch <- 42
    }()
    
    value := <-ch
    fmt.Printf("Received: %d\n", value)
}
```

### 7.3 Performance Considerations

#### 7.3.1 Goroutine Pooling

```go
type GoroutinePool struct {
    workers chan struct{}
}

func NewGoroutinePool(size int) *GoroutinePool {
    return &GoroutinePool{
        workers: make(chan struct{}, size),
    }
}

func (p *GoroutinePool) Submit(task func()) {
    p.workers <- struct{}{} // Acquire worker slot
    
    go func() {
        defer func() { <-p.workers }() // Release worker slot
        task()
    }()
}

func main() {
    pool := NewGoroutinePool(5)
    var wg sync.WaitGroup
    
    for i := 1; i <= 20; i++ {
        wg.Add(1)
        taskID := i
        
        pool.Submit(func() {
            defer wg.Done()
            fmt.Printf("Task %d executing...\n", taskID)
            time.Sleep(time.Millisecond * 500)
            fmt.Printf("Task %d completed!\n", taskID)
        })
    }
    
    wg.Wait()
    fmt.Println("All tasks completed!")
}
```

#### 7.3.2 Memory Management

```go
func main() {
    // Monitor memory usage
    var m runtime.MemStats
    
    // Start many goroutines
    var wg sync.WaitGroup
    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Do some work
            time.Sleep(time.Millisecond * 10)
        }(i)
    }
    
    wg.Wait()
    
    // Check memory usage
    runtime.ReadMemStats(&m)
    fmt.Printf("Memory usage: %d bytes\n", m.Alloc)
}
```

## Chapter 8: Testing Goroutines

### 8.1 Unit Testing Goroutines

#### 8.1.1 Testing with Channels

```go
func TestWorker(t *testing.T) {
    jobs := make(chan int, 5)
    results := make(chan int, 5)
    
    // Start worker
    go worker(1, jobs, results)
    
    // Send test jobs
    jobs <- 5
    jobs <- 10
    close(jobs)
    
    // Verify results
    expected := []int{10, 20}
    for i, exp := range expected {
        if got := <-results; got != exp {
            t.Errorf("Expected %d, got %d", exp, got)
        }
    }
}

func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}
```

#### 8.1.2 Testing with WaitGroup

```go
func TestMultipleWorkers(t *testing.T) {
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    
    var wg sync.WaitGroup
    
    // Start multiple workers
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- job * 2
            }
        }(i)
    }
    
    // Send jobs
    go func() {
        for i := 1; i <= 6; i++ {
            jobs <- i
        }
        close(jobs)
    }()
    
    // Wait for all workers to finish
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    count := 0
    for range results {
        count++
    }
    
    if count != 6 {
        t.Errorf("Expected 6 results, got %d", count)
    }
}
```

### 8.2 Integration Testing

#### 8.2.1 Full Pipeline Test (Continued)

```go
func TestFullPipeline(t *testing.T) {
    // Test entire pipeline
    input := generateItems(5)
    processed := processItems(input)
    validated := validateItems(processed)
    
    // Collect results
    results := []PipelineItem{}
    for item := range validated {
        results = append(results, item)
    }
    
    // Verify results
    if len(results) != 5 {
        t.Errorf("Expected 5 items, got %d", len(results))
    }
    
    // Check that some items succeeded and some failed
    successCount := 0
    for _, item := range results {
        if item.Error == nil {
            successCount++
        }
    }
    
    // Should have both successes and failures (due to random errors)
    if successCount == 0 || successCount == 5 {
        t.Logf("Success count: %d (this might be random, not necessarily an error)", successCount)
    }
}
```

#### 8.2.2 Stress Testing

```go
func TestPipelineStress(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping stress test in short mode")
    }
    
    // Test with many items
    input := generateItems(1000)
    processed := processItems(input)
    validated := validateItems(processed)
    
    // Consume all results
    count := 0
    for range validated {
        count++
    }
    
    if count != 1000 {
        t.Errorf("Expected 1000 items, got %d", count)
    }
}
```

#### 8.2.3 Race Condition Testing

```go
func TestRaceConditions(t *testing.T) {
    // Test for race conditions
    if race.Enabled {
        t.Skip("Race detector enabled, skipping race test")
    }
    
    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup
    
    // Start many goroutines that modify shared state
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++
            mu.Unlock()
        }()
    }
    
    wg.Wait()
    
    if counter != 1000 {
        t.Errorf("Expected counter to be 1000, got %d", counter)
    }
}
```

### 8.3 Benchmarking Goroutines

#### 8.3.1 Basic Benchmarking

```go
func BenchmarkWorker(b *testing.B) {
    jobs := make(chan int, b.N)
    results := make(chan int, b.N)
    
    // Fill jobs channel
    for i := 0; i < b.N; i++ {
        jobs <- i
    }
    close(jobs)
    
    b.ResetTimer()
    
    // Start worker
    go worker(1, jobs, results)
    
    // Consume results
    for i := 0; i < b.N; i++ {
        <-results
    }
}

func BenchmarkMultipleWorkers(b *testing.B) {
    jobs := make(chan int, b.N)
    results := make(chan int, b.N)
    
    // Fill jobs channel
    for i := 0; i < b.N; i++ {
        jobs <- i
    }
    close(jobs)
    
    b.ResetTimer()
    
    // Start multiple workers
    var wg sync.WaitGroup
    for w := 1; w <= 4; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- job * 2
            }
        }(w)
    }
    
    // Wait for all workers to finish
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Consume results
    for range results {
        // Consume all results
    }
}
```

#### 8.3.2 Memory Benchmarking

```go
func BenchmarkGoroutineMemory(b *testing.B) {
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        var wg sync.WaitGroup
        wg.Add(1)
        
        go func() {
            defer wg.Done()
            // Do minimal work
        }()
        
        wg.Wait()
    }
}
```

## Chapter 9: Advanced Goroutine Patterns

### 9.1 Circuit Breaker Pattern

The circuit breaker pattern prevents cascading failures by temporarily stopping operations when they're likely to fail.

#### 9.1.1 Basic Circuit Breaker

```go
type CircuitBreaker struct {
    mu          sync.RWMutex
    state       string // "closed", "open", "half-open"
    failureCount int
    lastFailure  time.Time
    threshold    int
    timeout      time.Duration
}

func NewCircuitBreaker(threshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:    "closed",
        threshold: threshold,
        timeout:   timeout,
    }
}

func (cb *CircuitBreaker) Execute(operation func() error) error {
    cb.mu.RLock()
    state := cb.state
    cb.mu.RUnlock()
    
    switch state {
    case "open":
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.mu.Lock()
            cb.state = "half-open"
            cb.mu.Unlock()
        } else {
            return fmt.Errorf("circuit breaker is open")
        }
    case "half-open":
        // Allow one attempt
    case "closed":
        // Allow all attempts
    }
    
    // Execute operation
    err := operation()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failureCount++
        cb.lastFailure = time.Now()
        
        if cb.failureCount >= cb.threshold {
            cb.state = "open"
        }
    } else {
        // Success - reset circuit breaker
        cb.state = "closed"
        cb.failureCount = 0
    }
    
    return err
}

func main() {
    cb := NewCircuitBreaker(3, time.Second*5)
    
    // Simulate operations
    for i := 0; i < 10; i++ {
        err := cb.Execute(func() error {
            // Simulate operation that sometimes fails
            if rand.Float64() < 0.7 {
                return fmt.Errorf("operation failed")
            }
            return nil
        })
        
        if err != nil {
            fmt.Printf("Operation %d failed: %v\n", i, err)
        } else {
            fmt.Printf("Operation %d succeeded\n", i)
        }
        
        time.Sleep(time.Millisecond * 500)
    }
}
```

### 9.2 Rate Limiting

Rate limiting controls how many operations can be performed in a given time period.

#### 9.2.1 Token Bucket Rate Limiter

```go
type TokenBucket struct {
    tokens    chan struct{}
    rate      time.Duration
    capacity  int
}

func NewTokenBucket(capacity int, rate time.Duration) *TokenBucket {
    tb := &TokenBucket{
        tokens:   make(chan struct{}, capacity),
        rate:     rate,
        capacity: capacity,
    }
    
    // Fill bucket initially
    for i := 0; i < capacity; i++ {
        tb.tokens <- struct{}{}
    }
    
    // Start refilling tokens
    go tb.refill()
    
    return tb
}

func (tb *TokenBucket) refill() {
    ticker := time.NewTicker(tb.rate)
    defer ticker.Stop()
    
    for range ticker.C {
        select {
        case tb.tokens <- struct{}{}:
            // Token added successfully
        default:
            // Bucket is full, skip
        }
    }
}

func (tb *TokenBucket) Take() bool {
    select {
    case <-tb.tokens:
        return true
    default:
        return false
    }
}

func (tb *TokenBucket) TakeWithTimeout(timeout time.Duration) bool {
    select {
    case <-tb.tokens:
        return true
    case <-time.After(timeout):
        return false
    }
}

func main() {
    limiter := NewTokenBucket(5, time.Millisecond*200) // 5 tokens, refill every 200ms
    
    var wg sync.WaitGroup
    
    for i := 1; i <= 20; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            if limiter.Take() {
                fmt.Printf("Worker %d got token, processing...\n", id)
                time.Sleep(time.Millisecond * 100)
                fmt.Printf("Worker %d completed\n", id)
            } else {
                fmt.Printf("Worker %d rate limited\n", id)
            }
        }(i)
    }
    
    wg.Wait()
}
```

### 9.3 Load Balancer Pattern

A load balancer distributes work across multiple workers to improve performance and reliability.

#### 9.3.1 Round-Robin Load Balancer

```go
type LoadBalancer struct {
    workers []chan int
    current int
    mu      sync.Mutex
}

func NewLoadBalancer(workerCount int) *LoadBalancer {
    lb := &LoadBalancer{
        workers: make([]chan int, workerCount),
    }
    
    // Create workers
    for i := 0; i < workerCount; i++ {
        lb.workers[i] = make(chan int, 10)
        go worker(i, lb.workers[i])
    }
    
    return lb
}

func (lb *LoadBalancer) Submit(job int) {
    lb.mu.Lock()
    worker := lb.workers[lb.current]
    lb.current = (lb.current + 1) % len(lb.workers)
    lb.mu.Unlock()
    
    worker <- job
}

func worker(id int, jobs <-chan int) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
        fmt.Printf("Worker %d completed job %d\n", id, job)
    }
}

func main() {
    lb := NewLoadBalancer(3)
    
    // Submit jobs
    for i := 1; i <= 10; i++ {
        lb.Submit(i)
    }
    
    time.Sleep(time.Second * 3)
}
```

#### 9.3.2 Weighted Load Balancer

```go
type WeightedWorker struct {
    ID     int
    Weight int
    Jobs   chan int
}

type WeightedLoadBalancer struct {
    workers []*WeightedWorker
    mu      sync.Mutex
}

func NewWeightedLoadBalancer(workers []*WeightedWorker) *WeightedLoadBalancer {
    lb := &WeightedLoadBalancer{workers: workers}
    
    // Start workers
    for _, w := range workers {
        go worker(w.ID, w.Jobs)
    }
    
    return lb
}

func (lb *WeightedLoadBalancer) Submit(job int) {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    // Find worker with highest weight and available capacity
    var selected *WeightedWorker
    maxWeight := 0
    
    for _, w := range lb.workers {
        if len(w.Jobs) < cap(w.Jobs) && w.Weight > maxWeight {
            selected = w
            maxWeight = w.Weight
        }
    }
    
    if selected != nil {
        selected.Jobs <- job
    }
}

func main() {
    workers := []*WeightedWorker{
        {ID: 1, Weight: 3, Jobs: make(chan int, 5)},
        {ID: 2, Weight: 2, Jobs: make(chan int, 5)},
        {ID: 3, Weight: 1, Jobs: make(chan int, 5)},
    }
    
    lb := NewWeightedLoadBalancer(workers)
    
    // Submit jobs
    for i := 1; i <= 15; i++ {
        lb.Submit(i)
    }
    
    time.Sleep(time.Second * 3)
}
```

## Chapter 10: Goroutines in Real-World Applications

### 10.1 Web Server with Goroutines

#### 10.1.1 Basic HTTP Server

```go
type Server struct {
    addr     string
    handlers map[string]http.HandlerFunc
}

func NewServer(addr string) *Server {
    return &Server{
        addr:     addr,
        handlers: make(map[string]http.HandlerFunc),
    }
}

func (s *Server) Handle(pattern string, handler http.HandlerFunc) {
    s.handlers[pattern] = handler
}

func (s *Server) Start() error {
    mux := http.NewServeMux()
    
    for pattern, handler := range s.handlers {
        mux.HandleFunc(pattern, handler)
    }
    
    return http.ListenAndServe(s.addr, mux)
}

func main() {
    server := NewServer(":8080")
    
    // Handle requests concurrently
    server.Handle("/", func(w http.ResponseWriter, r *http.Request) {
        // Simulate work
        time.Sleep(time.Millisecond * 100)
        fmt.Fprintf(w, "Hello, World!")
    })
    
    server.Handle("/slow", func(w http.ResponseWriter, r *http.Request) {
        // Simulate slow operation
        time.Sleep(time.Second * 2)
        fmt.Fprintf(w, "Slow operation completed!")
    })
    
    fmt.Println("Server starting on :8080...")
    if err := server.Start(); err != nil {
        log.Fatal(err)
    }
}
```

#### 10.1.2 Middleware with Goroutines

```go
type Middleware func(http.HandlerFunc) http.HandlerFunc

func LoggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Process request in goroutine for logging
        go func() {
            time.Sleep(time.Millisecond * 100) // Simulate async logging
            fmt.Printf("Request to %s completed in %v\n", r.URL.Path, time.Since(start))
        }()
        
        next(w, r)
    }
}

func RateLimitMiddleware(limiter *TokenBucket) Middleware {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Take() {
                http.Error(w, "Rate limited", http.StatusTooManyRequests)
                return
            }
            next(w, r)
        }
    }
}

func ApplyMiddleware(handler http.HandlerFunc, middlewares ...Middleware) http.HandlerFunc {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func main() {
    server := NewServer(":8080")
    limiter := NewTokenBucket(10, time.Millisecond*100)
    
    handler := func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    }
    
    // Apply middleware
    finalHandler := ApplyMiddleware(handler, 
        LoggingMiddleware,
        RateLimitMiddleware(limiter),
    )
    
    server.Handle("/", finalHandler)
    
    fmt.Println("Server starting on :8080...")
    if err := server.Start(); err != nil {
        log.Fatal(err)
    }
}
```

### 10.2 Database Connection Pool

#### 10.2.1 Connection Pool Implementation

```go
type Connection struct {
    ID        int
    CreatedAt time.Time
    LastUsed  time.Time
}

type ConnectionPool struct {
    connections chan *Connection
    maxConnections int
    mu            sync.Mutex
    closed        bool
}

func NewConnectionPool(maxConnections int) *ConnectionPool {
    pool := &ConnectionPool{
        connections:    make(chan *Connection, maxConnections),
        maxConnections: maxConnections,
    }
    
    // Initialize connections
    for i := 0; i < maxConnections; i++ {
        pool.connections <- &Connection{
            ID:        i,
            CreatedAt: time.Now(),
            LastUsed:  time.Now(),
        }
    }
    
    return pool
}

func (p *ConnectionPool) Get() (*Connection, error) {
    if p.closed {
        return nil, fmt.Errorf("connection pool is closed")
    }
    
    select {
    case conn := <-p.connections:
        conn.LastUsed = time.Now()
        return conn, nil
    case <-time.After(time.Second * 5):
        return nil, fmt.Errorf("timeout waiting for connection")
    }
}

func (p *ConnectionPool) Put(conn *Connection) error {
    if p.closed {
        return fmt.Errorf("connection pool is closed")
    }
    
    select {
    case p.connections <- conn:
        return nil
    default:
        return fmt.Errorf("connection pool is full")
    }
}

func (p *ConnectionPool) Close() {
    p.mu.Lock()
    defer p.mu.Unlock()
    
    if !p.closed {
        p.closed = true
        close(p.connections)
    }
}

func main() {
    pool := NewConnectionPool(5)
    defer pool.Close()
    
    var wg sync.WaitGroup
    
    // Simulate multiple goroutines using connections
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            conn, err := pool.Get()
            if err != nil {
                fmt.Printf("Goroutine %d failed to get connection: %v\n", id, err)
                return
            }
            
            fmt.Printf("Goroutine %d got connection %d\n", id, conn.ID)
            
            // Simulate work
            time.Sleep(time.Millisecond * time.Duration(rand.Intn(500)))
            
            // Return connection
            if err := pool.Put(conn); err != nil {
                fmt.Printf("Goroutine %d failed to return connection: %v\n", id, err)
            } else {
                fmt.Printf("Goroutine %d returned connection %d\n", id, conn.ID)
            }
        }(i)
    }
    
    wg.Wait()
}
```

### 10.3 Background Job Processor

#### 10.3.1 Job Queue Implementation

```go
type Job struct {
    ID       string
    Type     string
    Data     interface{}
    Priority int
    Created  time.Time
}

type JobProcessor struct {
    jobs     chan Job
    workers  int
    handlers map[string]func(Job) error
    mu       sync.RWMutex
}

func NewJobProcessor(workers int) *JobProcessor {
    return &JobProcessor{
        jobs:     make(chan Job, 1000),
        workers:  workers,
        handlers: make(map[string]func(Job) error),
    }
}

func (jp *JobProcessor) RegisterHandler(jobType string, handler func(Job) error) {
    jp.mu.Lock()
    defer jp.mu.Unlock()
    jp.handlers[jobType] = handler
}

func (jp *JobProcessor) Submit(job Job) error {
    select {
    case jp.jobs <- job:
        return nil
    default:
        return fmt.Errorf("job queue is full")
    }
}

func (jp *JobProcessor) Start() {
    for i := 0; i < jp.workers; i++ {
        go jp.worker(i)
    }
}

func (jp *JobProcessor) worker(id int) {
    for job := range jp.jobs {
        jp.mu.RLock()
        handler, exists := jp.handlers[job.Type]
        jp.mu.RUnlock()
        
        if !exists {
            fmt.Printf("Worker %d: No handler for job type %s\n", id, job.Type)
            continue
        }
        
        fmt.Printf("Worker %d: Processing job %s of type %s\n", id, job.ID, job.Type)
        
        if err := handler(job); err != nil {
            fmt.Printf("Worker %d: Job %s failed: %v\n", id, job.ID, err)
        } else {
            fmt.Printf("Worker %d: Job %s completed successfully\n", id, job.ID)
        }
    }
}

func main() {
    processor := NewJobProcessor(3)
    
    // Register job handlers
    processor.RegisterHandler("email", func(job Job) error {
        // Simulate email sending
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(1000)))
        return nil
    })
    
    processor.RegisterHandler("image", func(job Job) error {
        // Simulate image processing
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(2000)))
        return nil
    })
    
    processor.RegisterHandler("report", func(job Job) error {
        // Simulate report generation
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(1500)))
        return nil
    })
    
    // Start processor
    processor.Start()
    
    // Submit jobs
    jobTypes := []string{"email", "image", "report"}
    for i := 1; i <= 20; i++ {
        job := Job{
            ID:      fmt.Sprintf("job_%d", i),
            Type:    jobTypes[rand.Intn(len(jobTypes))],
            Data:    fmt.Sprintf("data_%d", i),
            Priority: rand.Intn(5),
            Created:  time.Now(),
        }
        
        if err := processor.Submit(job); err != nil {
            fmt.Printf("Failed to submit job %s: %v\n", job.ID, err)
        }
    }
    
    // Wait for jobs to complete
    time.Sleep(time.Second * 10)
}
```

## Chapter 11: Debugging and Monitoring Goroutines

### 11.1 Goroutine Profiling

#### 11.1.1 Runtime Statistics

```go
func printGoroutineStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    
    fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())
    fmt.Printf("Memory allocated: %d bytes\n", m.Alloc)
    fmt.Printf("Memory total: %d bytes\n", m.TotalAlloc)
    fmt.Printf("Memory system: %d bytes\n", m.Sys)
}

func main() {
    fmt.Println("Initial stats:")
    printGoroutineStats()
    
    // Start many goroutines
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Millisecond * 100)
        }(i)
    }
    
    fmt.Println("\nAfter starting goroutines:")
    printGoroutineStats()
    
    wg.Wait()
    
    fmt.Println("\nAfter all goroutines finished:")
    printGoroutineStats()
}
```

#### 11.1.2 Goroutine Stack Traces

```go
func printGoroutineStacks() {
    buf := make([]byte, 1<<16)
    stackLen := runtime.Stack(buf, true)
    fmt.Printf("=== Goroutine Stack Traces ===\n%s\n", buf[:stackLen])
}

func main() {
    // Start some goroutines
    for i := 0; i < 5; i++ {
        go func(id int) {
            time.Sleep(time.Second)
        }(i)
    }
    
    // Print stack traces
    printGoroutineStacks()
}
```

### 11.2 Monitoring Goroutines

#### 11.2.1 Health Check System

```go
type HealthChecker struct {
    checks map[string]func() error
    mu     sync.RWMutex
}

func NewHealthChecker() *HealthChecker {
    return &HealthChecker{
        checks: make(map[string]func() error),
    }
}

func (hc *HealthChecker) AddCheck(name string, check func() error) {
    hc.mu.Lock()
    defer hc.mu.Unlock()
    hc.checks[name] = check
}

func (hc *HealthChecker) CheckHealth() map[string]error {
    hc.mu.RLock()
    defer hc.mu.RUnlock()
    
    results := make(map[string]error)
    var wg sync.WaitGroup
    
    for name, check := range hc.checks {
        wg.Add(1)
        go func(name string, check func() error) {
            defer wg.Done()
            results[name] = check()
        }(name, check)
    }
    
    wg.Wait()
    return results
}

func main() {
    checker := NewHealthChecker()
    
    // Add health checks
    checker.AddCheck("goroutine_count", func() error {
        count := runtime.NumGoroutine()
        if count > 1000 {
            return fmt.Errorf("too many goroutines: %d", count)
        }
        return nil
    })
    
    checker.AddCheck("memory_usage", func() error {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        if m.Alloc > 100*1024*1024 { // 100MB
            return fmt.Errorf("memory usage too high: %d bytes", m.Alloc)
        }
        return nil
    })
    
    // Run health checks periodically
    ticker := time.NewTicker(time.Second * 5)
    defer ticker.Stop()
    
    for range ticker.C {
        results := checker.CheckHealth()
        
        fmt.Println("Health check results:")
        for name, err := range results {
            if err != nil {
                fmt.Printf("  %s: ERROR - %v\n", name, err)
            } else {
                fmt.Printf("  %s: OK\n", name)
            }
        }
        fmt.Println()
    }
}
```

## Chapter 12: Conclusion and Best Practices Summary

### 12.1 Key Takeaways

1. **Goroutines are lightweight**: Start with only 2KB of stack space
2. **Channels enable safe communication**: Use them for goroutine coordination
3. **Always handle lifecycle**: Ensure goroutines can be cancelled and cleaned up
4. **Use appropriate synchronization**: Mutexes for shared data, WaitGroups for coordination
5. **Test thoroughly**: Test individual goroutines and integration scenarios
6. **Monitor performance**: Watch goroutine count and memory usage
7. **Handle errors gracefully**: Implement proper error handling and recovery

### 12.2 When to Use Goroutines

**Use goroutines when:**
- You need concurrent execution
- Operations are I/O-bound
- You want to improve responsiveness
- You need to handle multiple requests simultaneously
- Operations can be performed independently

**Avoid goroutines when:**
- Operations are CPU-bound and you only have one CPU core
- The overhead of coordination exceeds the benefits
- You need strict ordering of operations
- The problem is inherently sequential

### 12.3 Performance Guidelines

1. **Start with few goroutines**: Add more only when needed
2. **Use buffered channels**: For better performance when appropriate
3. **Implement backpressure**: Prevent memory exhaustion
4. **Profile your code**: Use Go's built-in profiling tools
5. **Monitor resource usage**: Watch memory and goroutine counts
6. **Use connection pooling**: For expensive resources like database connections
7. **Implement rate limiting**: For external API calls

### 12.4 Common Patterns Summary

- **Worker Pool**: For controlled concurrency
- **Pipeline**: For data processing workflows
- **Producer-Consumer**: For work distribution
- **Pub-Sub**: For event-driven architectures
- **Circuit Breaker**: For fault tolerance
- **Rate Limiting**: For resource protection
- **Load Balancing**: For work distribution

Goroutines are one of Go's most powerful features, enabling you to write concurrent, scalable, and efficient applications. By understanding the patterns and best practices outlined in this guide, you'll be able to leverage goroutines effectively in your Go programs.
