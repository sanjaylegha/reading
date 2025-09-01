# The Pipeline Pattern in Go: A Comprehensive Guide

## Chapter 1: Understanding the Pipeline Pattern

### 1.1 What is a Pipeline?

A pipeline is like an assembly line in a modern factory. Imagine you're building cars: the chassis goes through the welding station, then the paint booth, then the assembly line, and finally quality control. Each station transforms the car in some way before passing it to the next.

In software, a pipeline works the same way. Data flows through a series of processing stages, where each stage performs a specific transformation. The beauty of this pattern is that each stage can work independently and concurrently, making the entire system more efficient.

**Real-world analogy:** Think of a coffee shop. The barista takes orders (input), the coffee machine brews the coffee (processing), the milk steamer froths the milk (more processing), and finally everything gets assembled into a finished drink (output). Each step happens in sequence, but multiple orders can be in different stages simultaneously.

### 1.2 Why Use Pipelines?

Pipelines offer several advantages:

- **Modularity**: Each stage has a single responsibility
- **Reusability**: Stages can be combined in different ways
- **Concurrency**: Multiple stages can run simultaneously
- **Scalability**: Easy to add more workers to slow stages
- **Testability**: Each stage can be tested independently

**Example scenario:** You're building a web service that processes user uploads. Users upload images, you need to resize them, apply filters, compress them, and store them. Each of these operations can be a pipeline stage, and you can process multiple images simultaneously through different stages.

### 1.3 The Pipeline Metaphor

Let's extend the factory analogy. In a traditional factory, if one worker is slow, everyone waits. But in a pipeline, you can have multiple workers at each station, and work flows continuously.

```
Raw Materials → Station A (3 workers) → Station B (2 workers) → Station C (1 worker) → Finished Product
```

If Station B is slow, you can add more workers there. If Station C is fast, one worker might be enough. The key is that work flows continuously, and bottlenecks can be addressed independently.

## Chapter 2: Basic Pipeline Components

### 2.1 Channels - The Pipes of Data

Channels are Go's way of implementing the "pipes" in the pipeline metaphor. Think of them as actual pipes that connect different parts of your system.

#### 2.1.1 Unbuffered Channels (Synchronous)

An unbuffered channel is like a handshake between two people. When you want to give something to someone, you extend your hand. They must also extend their hand to receive it. If either person isn't ready, the handshake can't happen.

```go
ch := make(chan int) // Unbuffered channel

// This will block until someone reads from the channel
ch <- 42

// This will block until someone writes to the channel
value := <-ch
```

**Real-world example:** Imagine a relay race where the baton must be physically transferred from one runner to the next. The second runner must be ready to receive the baton before the first runner can pass it.

#### 2.1.2 Buffered Channels (Asynchronous)

A buffered channel is like a mailbox. You can put letters in the mailbox even if no one is there to collect them immediately. The mailbox can hold a certain number of letters before it's full.

```go
ch := make(chan int, 5) // Buffer of 5

// These won't block (until buffer is full)
ch <- 1
ch <- 2
ch <- 3

// Later, someone can collect all the letters
for i := 0; i < 3; i++ {
    fmt.Println(<-ch)
}
```

**Real-world example:** Think of a restaurant kitchen. The chef can prepare multiple dishes and put them on the pass (the counter where waiters pick up food). The pass acts as a buffer - the chef doesn't have to wait for a waiter to be ready before starting the next dish.

#### 2.1.3 Channel Direction

Go allows you to specify whether a channel can only send, only receive, or both. This is like having one-way streets in a city - it prevents accidents and makes the code's intent clear.

```go
// Bidirectional channel (can send and receive)
var ch chan int

// Send-only channel (can only send)
var sendOnly chan<- int

// Receive-only channel (can only receive)
var receiveOnly <-chan int
```

**Why this matters:** In a pipeline, you want to be explicit about data flow. The generator stage should only send data, and the consumer stage should only receive data. This prevents accidental misuse and makes the code self-documenting.

### 2.2 Goroutines - The Workers

Goroutines are Go's lightweight threads. Think of them as workers in your factory. Unlike traditional threads that are expensive to create and manage, goroutines are cheap - you can have thousands of them running simultaneously.

#### 2.2.1 What Makes Goroutines Special?

Traditional threads are managed by the operating system and have a significant memory overhead (typically 1-2 MB per thread). Goroutines are managed by Go's runtime and have a much smaller memory footprint (typically 2-8 KB per goroutine).

```go
// Traditional approach (expensive)
for i := 0; i < 1000; i++ {
    go func(id int) {
        // This creates 1000 OS threads - expensive!
        time.Sleep(time.Second)
    }(i)
}

// Go approach (cheap)
for i := 0; i < 1000; i++ {
    go func(id int) {
        // This creates 1000 goroutines - cheap!
        time.Sleep(time.Second)
    }(i)
}
```

**Real-world analogy:** Think of hiring workers for a factory. Traditional threads are like hiring full-time employees with benefits, office space, and equipment. Goroutines are like hiring temporary workers who can start immediately, work efficiently, and leave when the job is done.

#### 2.2.2 Goroutine Lifecycle

A goroutine starts when you call a function with the `go` keyword and ends when the function returns.

```go
func worker(id int) {
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

// Start 3 workers
go worker(1)
go worker(2)
go worker(3)

// Main function continues immediately
fmt.Println("All workers started")
```

**Important note:** The main function doesn't wait for goroutines to finish. If the main function exits, all goroutines are terminated immediately. This is why we need coordination mechanisms like channels or `sync.WaitGroup`.

## Chapter 3: Building Your First Pipeline

### 3.1 The Generator Stage - Creating the Source

Let's dive deep into the generator function from your example:

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        fmt.Printf("Gen started ...\n")
        for _, n := range nums {
            fmt.Println("gen out sending ...", n)
            out <- n
            fmt.Println("gen out sent ...", n)
        }
        close(out)
        fmt.Printf("Gen closed ...\n")
    }()
    return out
}
```

#### 3.1.1 Function Signature Analysis

```go
func gen(nums ...int) <-chan int
```

- `nums ...int`: This is Go's variadic parameter syntax. It means the function can accept any number of integers. It's like having a flexible input that can handle different amounts of data.
- `<-chan int`: This is a receive-only channel. The function returns a channel that can only be read from, not written to. This prevents the caller from accidentally sending data back to the generator.

#### 3.1.2 Channel Creation

```go
out := make(chan int)
```

This creates an unbuffered channel. Why unbuffered? Because we want the generator to work in lockstep with the consumer. Each number is sent and immediately processed, ensuring no data is lost or delayed.

#### 3.1.3 The Goroutine

```go
go func() {
    // ... work happens here
}()
```

The goroutine is essential here. Without it, the function would block at the first `out <- n` call until someone reads from the channel. By running the work in a goroutine, the function can return the channel immediately, and the work happens in the background.

#### 3.1.4 The Loop and Channel Operations

```go
for _, n := range nums {
    fmt.Println("gen out sending ...", n)
    out <- n
    fmt.Println("gen out sent ...", n)
}
```

This loop processes each input number. The `out <- n` operation sends the number through the channel. In an unbuffered channel, this operation blocks until someone reads from the channel.

**What happens step by step:**
1. Generator tries to send `2` → blocks until consumer is ready
2. Consumer reads `2` → generator unblocks and sends `3`
3. Consumer reads `3` → generator unblocks and sends `4`
4. Consumer reads `4` → generator unblocks and continues

#### 3.1.5 Channel Closing

```go
close(out)
```

Closing a channel is like shutting down a factory line. It signals that no more data will be sent. This is crucial because:
- It prevents deadlocks (receivers know when to stop waiting)
- It allows the `range` loop in the next stage to terminate properly
- It's a signal to downstream stages that processing is complete

**Important:** Only the sender should close a channel. Receivers should never close channels they receive from.

### 3.2 The Transformation Stage - Processing the Data

Now let's examine the square function:

```go
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        fmt.Printf("Sq started ...\n")
        for n := range in {
            fmt.Println("sq out sending ...", n)
            out <- n * n
            fmt.Println("sq out sent ...", n)
        }
        close(out)
        fmt.Printf("Sq closed ...\n")
    }()
    return out
}
```

#### 3.2.1 Function Signature Analysis

```go
func sq(in <-chan int) <-chan int
```

- `in <-chan int`: Input channel (receive-only)
- `<-chan int`: Output channel (receive-only from caller's perspective)

This creates a clear data flow: data comes in through `in`, gets processed, and goes out through `out`.

#### 3.2.2 The Range Loop

```go
for n := range in {
    // process n
}
```

This is one of Go's most elegant features. The `range` loop over a channel automatically reads values until the channel is closed. It's like having an automatic conveyor belt that stops when the factory shuts down.

**What happens internally:**
1. Loop reads value from `in` channel
2. If channel is closed and empty, loop exits
3. If channel has data, loop processes it and continues
4. If channel is empty but not closed, loop blocks until data arrives

#### 3.2.3 Data Transformation

```go
out <- n * n
```

This is where the actual work happens. Each input number gets squared. You could replace this with any transformation:
- `out <- n * 2` (multiply by 2)
- `out <- n + 10` (add 10)
- `out <- factorial(n)` (calculate factorial)
- `out <- processImage(n)` (process an image)

#### 3.2.4 Automatic Termination

The function automatically stops when the input channel closes. This is crucial for pipeline reliability. If the generator stage closes its channel, the square stage will automatically finish and close its output channel, propagating the "done" signal through the pipeline.

### 3.3 The Consumer Stage - Using the Results

The main function acts as the consumer:

```go
func main() {
    // Set up the pipeline
    c := gen(2, 3, 4)
    out := sq(c)
    
    // Consume the output
    fmt.Println("Output 1: ", <-out) // 4
    fmt.Println("Output 2: ", <-out) // 9
    fmt.Println("Output 3: ", <-out) // 16
}
```

#### 3.3.1 Pipeline Setup

```go
c := gen(2, 3, 4)
out := sq(c)
```

This creates the pipeline chain. Think of it like connecting pipes:
- `gen(2, 3, 4)` creates a pipe that will emit 2, 3, 4
- `sq(c)` creates a pipe that takes those numbers and squares them
- The result is a pipe that will emit 4, 9, 16

#### 3.3.2 Consuming Results

```go
fmt.Println("Output 1: ", <-out) // 4
fmt.Println("Output 2: ", <-out) // 9
fmt.Println("Output 3: ", <-out) // 16
```

Each `<-out` operation reads one value from the channel. This is like turning on the tap and collecting water in a bucket. Each read gets the next value in the sequence.

**Important observation:** The consumer controls the pace. If the consumer reads slowly, the pipeline waits. If the consumer reads quickly, the pipeline keeps up. This is called "backpressure" - the consumer's speed determines the overall pipeline speed.

## Chapter 4: Advanced Pipeline Patterns

### 4.1 Fan-Out Pattern - Distributing Work

The fan-out pattern is like having multiple workers at the same station in your factory. Instead of one worker processing all items, you distribute the work across multiple workers.

#### 4.1.1 Why Fan-Out?

Imagine you have a stage that processes images. If one worker can process 10 images per minute, but you're receiving 50 images per minute, you'll have a bottleneck. By adding more workers, you can handle the increased load.

#### 4.1.2 Implementation

```go
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
            time.Sleep(time.Millisecond * 100)
            output <- n * n
            fmt.Printf("Worker %d processed %d\n", id, n)
        }
    }()
    return output
}
```

#### 4.1.3 How It Works

1. **Input Distribution**: The same input channel is shared among all workers
2. **Concurrent Processing**: Each worker processes items independently
3. **Multiple Outputs**: Each worker has its own output channel
4. **Load Balancing**: Go's scheduler automatically distributes work among workers

**Real-world analogy:** Think of a bank with multiple tellers. All customers wait in the same line, but when a teller becomes available, they serve the next customer. Multiple tellers can serve customers simultaneously, reducing wait times.

#### 4.1.4 When to Use Fan-Out

- **CPU-intensive tasks**: Multiple workers can utilize multiple CPU cores
- **I/O-bound tasks**: Multiple workers can handle multiple concurrent I/O operations
- **Unpredictable processing times**: Some items might take longer than others
- **High throughput requirements**: Need to process more items than a single worker can handle

### 4.2 Fan-In Pattern - Combining Results

The fan-in pattern is the opposite of fan-out. It takes multiple input channels and combines them into a single output channel. This is like having multiple production lines converge into a single packaging station.

#### 4.2.1 Why Fan-In?

After fanning out work to multiple workers, you need to collect all the results. The fan-in pattern allows you to merge multiple streams of data back into a single stream.

#### 4.2.2 Implementation

```go
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

#### 4.2.3 How It Works

1. **Multiple Inputs**: Each worker's output channel becomes an input to the fan-in
2. **Concurrent Collection**: Each input channel is read in its own goroutine
3. **Single Output**: All results are sent to a single output channel
4. **Synchronization**: `sync.WaitGroup` ensures all inputs are processed before closing

**Real-world analogy:** Think of multiple conveyor belts feeding into a single sorting machine. Each belt brings items from different production lines, and the sorter combines them into a single stream.

#### 4.2.4 The Complete Fan-Out/Fan-In Pattern

```go
func main() {
    input := gen(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // Fan out to 3 workers
    workers := fanOut(input, 3)
    
    // Fan in the results
    output := fanIn(workers...)
    
    // Consume results
    for n := range output {
        fmt.Println(n)
    }
}
```

This creates a pipeline where:
1. Numbers 1-10 are generated
2. Work is distributed to 3 workers
3. Each worker squares its assigned numbers
4. All results are combined into a single output
5. The main function consumes all results

### 4.3 Pipeline with Error Handling

Real-world pipelines need to handle errors gracefully. What happens if one stage encounters an error? Should the entire pipeline fail, or should we continue processing other items?

#### 4.3.1 Error Types

```go
type Result struct {
    Value int
    Error error
}

type PipelineError struct {
    Stage string
    Value int
    Err   error
}

func (e PipelineError) Error() string {
    return fmt.Sprintf("Pipeline error in %s stage processing value %d: %v", e.Stage, e.Value, e.Err)
}
```

#### 4.3.2 Safe Processing

```go
func safeSq(in <-chan int) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for n := range in {
            if n < 0 {
                out <- Result{
                    Error: PipelineError{
                        Stage: "square",
                        Value: n,
                        Err:   fmt.Errorf("cannot square negative number"),
                    },
                }
                continue
            }
            
            result := n * n
            if result < 0 {
                out <- Result{
                    Error: PipelineError{
                        Stage: "square",
                        Value: n,
                        Err:   fmt.Errorf("overflow occurred"),
                    },
                }
                continue
            }
            
            out <- Result{Value: result}
        }
    }()
    return out
}
```

#### 4.3.3 Error Handling in Consumer

```go
func main() {
    input := gen(2, -3, 4, 1000000)
    output := safeSq(input)
    
    for result := range output {
        if result.Error != nil {
            fmt.Printf("Error: %v\n", result.Error)
            continue
        }
        fmt.Printf("Result: %d\n", result.Value)
    }
}
```

This approach allows the pipeline to continue processing even when individual items fail, providing resilience and better error reporting.

## Chapter 5: Pipeline Best Practices

### 5.1 Always Close Channels

Channel closing is like shutting down a production line. It's a signal that no more work will be sent, and it's essential for preventing deadlocks.

#### 5.1.1 Why Close Channels?

1. **Prevents Deadlocks**: Receivers know when to stop waiting
2. **Signals Completion**: Downstream stages know when processing is done
3. **Resource Cleanup**: Allows garbage collection of channel resources
4. **Range Loop Termination**: The `for range` loop automatically exits when a channel is closed

#### 5.1.2 Proper Closing

```go
// Good: Close channel in sender
func sender(out chan<- int) {
    defer close(out) // Always close, even if function panics
    for i := 0; i < 5; i++ {
        out <- i
    }
}

// Bad: Never close channel
func sender(out chan<- int) {
    for i := 0; i < 5; i++ {
        out <- i
    }
    // Channel never closed - potential deadlock
}
```

#### 5.1.3 When to Close

- **Close in the sender**: Only the sender knows when it's done sending
- **Close after all sends**: Don't close until you're completely done
- **Use defer**: Ensures channel is closed even if function panics

**Real-world analogy:** Think of a restaurant kitchen. The chef (sender) knows when all orders are prepared and can close the pass (channel). Waiters (receivers) know the kitchen is closed when they see the pass is empty and closed.

### 5.2 Use Directional Channels

Directional channels make your code's intent clear and prevent accidental misuse.

#### 5.2.1 Channel Directions

```go
// Bidirectional - can send and receive
var ch chan int

// Send-only - can only send
var sendOnly chan<- int

// Receive-only - can only receive
var receiveOnly <-chan int
```

#### 5.2.2 Function Signatures

```go
// Good: Clear intent
func processor(in <-chan int, out chan<- int) {
    // Can only read from in, only write to out
    for n := range in {
        out <- process(n)
    }
}

// Bad: Ambiguous
func processor(in chan int, out chan int) {
    // Could accidentally write to input or read from output
    // This could cause deadlocks or data corruption
}
```

#### 5.2.3 Benefits of Directional Channels

1. **Prevents Bugs**: Can't accidentally send to input channel
2. **Self-Documenting**: Code clearly shows data flow
3. **Compiler Safety**: Go compiler prevents misuse
4. **Interface Design**: Makes function contracts clear

**Real-world analogy:** Think of water pipes in a house. You have pipes that only carry water in one direction (directional channels) and pipes that can carry water both ways (bidirectional). For a pipeline, you want one-way pipes to prevent backflow.

### 5.3 Handle Channel Closing Gracefully

Receivers need to handle channel closing properly to avoid deadlocks and ensure clean shutdown.

#### 5.3.1 The Range Loop

```go
func consumer(in <-chan int) {
    for n := range in {
        process(n)
    }
    // Loop automatically exits when channel is closed
}
```

The `range` loop is the simplest and most elegant way to handle channel closing. It automatically exits when the channel is closed and empty.

#### 5.3.2 Manual Channel Checking

```go
func consumer(in <-chan int) {
    for {
        select {
        case value, ok := <-in:
            if !ok {
                // Channel closed
                return
            }
            process(value)
        }
    }
}
```

This approach gives you more control over the closing process and allows you to perform cleanup operations.

#### 5.3.3 Multiple Channel Handling

```go
func consumer(input1, input2 <-chan int) {
    for {
        select {
        case value, ok := <-input1:
            if !ok {
                input1 = nil // Mark as closed
                continue
            }
            process1(value)
        case value, ok := <-input2:
            if !ok {
                input2 = nil // Mark as closed
                continue
            }
            process2(value)
        }
        
        // Exit when both channels are closed
        if input1 == nil && input2 == nil {
            return
        }
    }
}
```

This pattern is useful when you have multiple input channels and need to handle them gracefully.

## Chapter 6: Real-World Pipeline Examples

### 6.1 Image Processing Pipeline

Let's build a realistic image processing pipeline that you might use in a web service.

#### 6.1.1 Image Types and Interfaces

```go
type Image struct {
    Data   []byte
    Width  int
    Height int
    Format string
}

type ImageProcessor interface {
    Process(img Image) (Image, error)
}

type Resizer struct {
    TargetWidth  int
    TargetHeight int
}

func (r *Resizer) Process(img Image) (Image, error) {
    // Simulate resizing
    time.Sleep(time.Millisecond * 100)
    
    // In real implementation, you'd use a library like imaging
    return Image{
        Data:   img.Data,
        Width:  r.TargetWidth,
        Height: r.TargetHeight,
        Format: img.Format,
    }, nil
}

type Filter struct {
    FilterType string
}

func (f *Filter) Process(img Image) (Image, error) {
    // Simulate filter application
    time.Sleep(time.Millisecond * 50)
    
    // In real implementation, you'd apply actual filters
    return img, nil
}

type Compressor struct {
    Quality int
    Format string
}

func (c *Compressor) Process(img Image) (Image, error) {
    // Simulate compression
    time.Sleep(time.Millisecond * 75)
    
    // In real implementation, you'd compress the image
    return img, nil
}
```

#### 6.1.2 Pipeline Builder

```go
func buildImagePipeline(processors ...ImageProcessor) func(<-chan Image) <-chan Image {
    return func(input <-chan Image) <-chan Image {
        output := input
        
        // Apply each processor in sequence
        for _, processor := range processors {
            output = applyProcessor(output, processor)
        }
        
        return output
    }
}

func applyProcessor(input <-chan Image, processor ImageProcessor) <-chan Image {
    output := make(chan Image)
    
    go func() {
        defer close(output)
        for img := range input {
            processed, err := processor.Process(img)
            if err != nil {
                fmt.Printf("Error processing image: %v\n", err)
                continue
            }
            output <- processed
        }
    }()
    
    return output
}
```

#### 6.1.3 Using the Pipeline

```go
func main() {
    // Create sample images
    images := make(chan Image)
    go func() {
        defer close(images)
        for i := 0; i < 5; i++ {
            images <- Image{
                Data:   []byte(fmt.Sprintf("image_data_%d", i)),
                Width:  1920,
                Height: 1080,
                Format: "jpeg",
            }
        }
    }()
    
    // Build pipeline
    pipeline := buildImagePipeline(
        &Resizer{TargetWidth: 800, TargetHeight: 600},
        &Filter{FilterType: "grayscale"},
        &Compressor{Quality: 80, Format: "jpeg"},
    )
    
    // Process images
    output := pipeline(images)
    
    // Consume results
    for processed := range output {
        fmt.Printf("Processed image: %dx%d %s\n", 
            processed.Width, processed.Height, processed.Format)
    }
}
```

This pipeline:
1. Takes images of any size
2. Resizes them to 800x600
3. Applies a grayscale filter
4. Compresses them to JPEG with 80% quality
5. Outputs the processed images

**Real-world application:** This could be used in a photo-sharing app where users upload high-resolution images, and the system automatically creates optimized versions for different use cases (thumbnails, previews, etc.).

### 6.2 Log Processing Pipeline

Another practical example is processing application logs to extract insights and detect issues.

#### 6.2.1 Log Data Structures

```go
type LogEntry struct {
    Timestamp time.Time
    Level     string
    Message   string
    UserID    string
    IPAddress string
    Metadata  map[string]interface{}
}

type LogProcessor interface {
    Process(entry LogEntry) (LogEntry, error)
}

type LogParser struct{}

func (p *LogParser) Process(entry LogEntry) (LogEntry, error) {
    // Parse log message for structured data
    // Extract error codes, user actions, etc.
    return entry, nil
}

type LogFilter struct {
    MinLevel string
}

func (f *LogFilter) Process(entry LogEntry) (LogEntry, error) {
    levels := map[string]int{
        "DEBUG": 0,
        "INFO":  1,
        "WARN":  2,
        "ERROR": 3,
        "FATAL": 4,
    }
    
    if levels[entry.Level] >= levels[f.MinLevel] {
        return entry, nil
    }
    
    // Return empty entry to indicate filtering
    return LogEntry{}, nil
}

type LogEnricher struct {
    UserService UserService
    GeoService  GeoService
}

func (e *LogEnricher) Process(entry LogEntry) (LogEntry, error) {
    if entry.UserID != "" {
        user, err := e.UserService.GetUser(entry.UserID)
        if err == nil {
            entry.Metadata["user_email"] = user.Email
            entry.Metadata["user_role"] = user.Role
        }
    }
    
    if entry.IPAddress != "" {
        location, err := e.GeoService.GetLocation(entry.IPAddress)
        if err == nil {
            entry.Metadata["country"] = location.Country
            entry.Metadata["city"] = location.City
        }
    }
    
    return entry, nil
}

type LogAggregator struct {
    Window time.Duration
}

func (a *LogAggregator) Process(entry LogEntry) (LogEntry, error) {
    // Aggregate logs by time window
    // Count errors, calculate response times, etc.
    return entry, nil
}
```

#### 6.2.2 Log Pipeline

```go
func buildLogPipeline(processors ...LogProcessor) func(<-chan LogEntry) <-chan LogEntry {
    return func(input <-chan LogEntry) <-chan LogEntry {
        output := input
        
        for _, processor := range processors {
            output = applyLogProcessor(output, processor)
        }
        
        return output
    }
}

func applyLogProcessor(input <-chan LogEntry, processor LogProcessor) <-chan LogEntry {
    output := make(chan LogEntry)
    
    go func() {
        defer close(output)
        for entry := range input {
            processed, err := processor.Process(entry)
            if err != nil {
                fmt.Printf("Error processing log: %v\n", err)
                continue
            }
            
            // Skip filtered entries
            if processed.Timestamp.IsZero() {
                continue
            }
            
            output <- processed
        }
    }()
    
    return output
}
```

#### 6.2.3 Using the Log Pipeline

```go
func main() {
    // Simulate log stream
    logs := make(chan LogEntry)
    go func() {
        defer close(logs)
        
        levels := []string{"DEBUG", "INFO", "WARN", "ERROR"}
        for i := 0; i < 100; i++ {
            logs <- LogEntry{
                Timestamp: time.Now().Add(time.Duration(i) * time.Second),
                Level:     levels[i%len(levels)],
                Message:   fmt.Sprintf("Log message %d", i),
                UserID:    fmt.Sprintf("user_%d", i%10),
                IPAddress: "192.168.1.1",
                Metadata:  make(map[string]interface{}),
            }
        }
    }()
    
    // Build pipeline
    pipeline := buildLogPipeline(
        &LogParser{},
        &LogFilter{MinLevel: "WARN"},
        &LogEnricher{},
        &LogAggregator{Window: time.Minute * 5},
    )
    
    // Process logs
    output := pipeline(logs)
    
    // Consume results
    for processed := range output {
        fmt.Printf("[%s] %s: %s (User: %s)\n",
            processed.Level,
            processed.Timestamp.Format("15:04:05"),
            processed.Message,
            processed.Metadata["user_email"])
    }
}
```

This pipeline:
1. Parses log entries for structured data
2. Filters out low-level logs (keeps only WARN and above)
3. Enriches logs with user and location information
4. Aggregates logs by time windows
5. Outputs processed logs for analysis

**Real-world application:** This could be used in a monitoring system to process application logs in real-time, detect anomalies, and provide insights into system performance and user behavior.

## Chapter 7: Performance Considerations

### 7.1 Buffering - Managing Flow Control

Buffering is like having a queue at a restaurant. Without a queue, customers have to wait outside until a table is available. With a queue, customers can wait inside, and the restaurant can serve them more efficiently.

#### 7.1.1 Unbuffered vs Buffered Channels

```go
// Unbuffered - synchronous
ch := make(chan int)

// Buffered - asynchronous
ch := make(chan int, 10)
```

**Unbuffered channels:**
- Sender blocks until receiver is ready
- Receiver blocks until sender is ready
- Perfect for synchronization
- Can cause deadlocks if not handled carefully

**Buffered channels:**
- Sender can send multiple values without blocking
- Receiver can read multiple values without blocking
- Provides flow control and backpressure
- Uses more memory but improves performance

#### 7.1.2 Choosing Buffer Size

```go
// Small buffer - tight coupling
ch := make(chan int, 1)

// Medium buffer - balanced
ch := make(chan int, 10)

// Large buffer - loose coupling
ch := make(chan int, 1000)
```

**Factors to consider:**
1. **Memory constraints**: Larger buffers use more memory
2. **Processing speed differences**: If stages have very different speeds, larger buffers help
3. **Latency requirements**: Smaller buffers provide lower latency
4. **Throughput requirements**: Larger buffers can improve throughput

#### 7.1.3 Buffer Size Guidelines

```go
// Rule of thumb: buffer size = processing time difference
func producer(out chan<- int) {
    for i := 0; i < 1000; i++ {
        out <- i
        time.Sleep(time.Millisecond) // 1ms per item
    }
}

func consumer(in <-chan int) {
    for n := range in {
        process(n)
        time.Sleep(time.Millisecond * 10) // 10ms per item
    }
}

// Consumer is 10x slower, so buffer size of 10-20 makes sense
ch := make(chan int, 15)
```

### 7.2 Backpressure - Managing Flow

Backpressure is like water pressure in pipes. If water flows too fast into a pipe, it can burst. If it flows too slowly, the pipe might run dry. Backpressure mechanisms ensure the flow is controlled.

#### 7.2.1 What is Backpressure?

Backpressure occurs when a downstream stage processes data slower than an upstream stage produces it. Without proper handling, this can lead to:
- Memory exhaustion (too many items in buffers)
- System crashes
- Poor performance
- Resource waste

#### 7.2.2 Implementing Backpressure

```go
// Simple backpressure with buffered channels
func producer(out chan<- int) {
    for i := 0; i < 1000; i++ {
        out <- i // Will block if buffer is full
    }
    close(out)
}

// Consumer with backpressure
func consumer(in <-chan int) {
    for n := range in {
        time.Sleep(time.Millisecond * 100) // Slow processing
        process(n)
    }
}

// Main with appropriate buffer
func main() {
    ch := make(chan int, 10) // Buffer provides backpressure
    
    go producer(ch)
    consumer(ch)
}
```

#### 7.2.3 Advanced Backpressure

```go
// Producer with backpressure awareness
func smartProducer(out chan<- int) {
    for i := 0; i < 1000; i++ {
        select {
        case out <- i:
            // Successfully sent
        default:
            // Channel full, implement backpressure strategy
            fmt.Printf("Backpressure: channel full, waiting...\n")
            time.Sleep(time.Millisecond * 10)
            i-- // Retry this item
        }
    }
    close(out)
}

// Producer with adaptive backpressure
func adaptiveProducer(out chan<- int) {
    for i := 0; i < 1000; i++ {
        // Try to send with timeout
        select {
        case out <- i:
            // Success
        case <-time.After(time.Millisecond * 100):
            // Timeout - implement backpressure
            fmt.Printf("Backpressure: timeout, skipping item %d\n", i)
            // Could implement retry logic, logging, or other strategies
        }
    }
    close(out)
}
```

#### 7.2.4 Backpressure Strategies

1. **Blocking**: Producer waits for consumer (default behavior)
2. **Dropping**: Skip items when buffer is full
3. **Retrying**: Wait and retry sending items
4. **Throttling**: Slow down production rate
5. **Load shedding**: Drop less important items first

```go
// Load shedding with priority
type Item struct {
    Data     interface{}
    Priority int
}

func priorityProducer(out chan<- Item) {
    items := []Item{
        {Data: "critical", Priority: 1},
        {Data: "important", Priority: 2},
        {Data: "normal", Priority: 3},
    }
    
    for _, item := range items {
        select {
        case out <- item:
            // Sent successfully
        default:
            // Buffer full, check if we should drop lower priority items
            if item.Priority > 2 {
                fmt.Printf("Dropping low priority item: %v\n", item.Data)
                continue
            }
            // Wait for space
            out <- item
        }
    }
    close(out)
}
```

## Chapter 8: Testing Pipelines

### 8.1 Unit Testing Pipeline Stages

Testing pipeline stages individually is crucial for ensuring reliability and maintainability.

#### 8.1.1 Testing the Generator

```go
func TestGenerator(t *testing.T) {
    // Test basic functionality
    t.Run("basic generation", func(t *testing.T) {
        input := []int{1, 2, 3}
        output := gen(input...)
        
        results := []int{}
        for n := range output {
            results = append(results, n)
        }
        
        expected := []int{1, 2, 3}
        if !reflect.DeepEqual(results, expected) {
            t.Errorf("Expected %v, got %v", expected, results)
        }
    })
    
    // Test empty input
    t.Run("empty input", func(t *testing.T) {
        output := gen()
        
        results := []int{}
        for n := range output {
            results = append(results, n)
        }
        
        if len(results) != 0 {
            t.Errorf("Expected empty output, got %v", results)
        }
    })
    
    // Test large input
    t.Run("large input", func(t *testing.T) {
        largeInput := make([]int, 1000)
        for i := range largeInput {
            largeInput[i] = i
        }
        
        output := gen(largeInput...)
        
        count := 0
        for range output {
            count++
        }
        
        if count != 1000 {
            t.Errorf("Expected 1000 items, got %d", count)
        }
    })
}
```

#### 8.2.2 Testing the Transformer

```go
func TestSquareTransformer(t *testing.T) {
    t.Run("basic transformation", func(t *testing.T) {
        input := make(chan int)
        output := sq(input)
        
        // Send test data
        go func() {
            input <- 2
            input <- 3
            input <- 4
            close(input)
        }()
        
        // Verify results
        expected := []int{4, 9, 16}
        for i, exp := range expected {
            if got := <-output; got != exp {
                t.Errorf("Expected %d, got %d", exp, got)
            }
        }
    })
    
    t.Run("empty input", func(t *testing.T) {
        input := make(chan int)
        output := sq(input)
        
        close(input)
        
        // Should not block
        select {
        case <-output:
            t.Error("Expected no output from empty input")
        default:
            // Success - no output
        }
    })
    
    t.Run("channel closing", func(t *testing.T) {
        input := make(chan int)
        output := sq(input)
        
        // Close input immediately
        close(input)
        
        // Output should also close
        count := 0
        for range output {
            count++
        }
        
        if count != 0 {
            t.Errorf("Expected 0 items, got %d", count)
        }
    })
}
```

#### 8.1.3 Testing with Mocks

```go
// Mock channel for testing
type MockChannel struct {
    values []int
    index  int
}

func NewMockChannel(values ...int) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for _, v := range values {
            ch <- v
        }
    }()
    return ch
}

func TestWithMockChannel(t *testing.T) {
    mockInput := NewMockChannel(1, 2, 3)
    output := sq(mockInput)
    
    expected := []int{1, 4, 9}
    for i, exp := range expected {
        if got := <-output; got != exp {
            t.Errorf("Expected %d, got %d", exp, got)
        }
    }
}
```

### 8.2 Integration Testing

Integration testing ensures that all pipeline stages work together correctly.

#### 8.2.1 Full Pipeline Test

```go
func TestFullPipeline(t *testing.T) {
    t.Run("complete pipeline", func(t *testing.T) {
        // Test entire pipeline
        input := gen(1, 2, 3)
        output := sq(input)
        
        results := []int{}
        for n := range output {
            results = append(results, n)
        }
        
        expected := []int{1, 4, 9}
        if !reflect.DeepEqual(results, expected) {
            t.Errorf("Expected %v, got %v", expected, results)
        }
    })
    
    t.Run("pipeline with errors", func(t *testing.T) {
        // Test pipeline with error handling
        input := gen(1, -2, 3)
        output := safeSq(input)
        
        results := []int{}
        errors := []error{}
        
        for result := range output {
            if result.Error != nil {
                errors = append(errors, result.Error)
            } else {
                results = append(results, result.Value)
            }
        }
        
        // Should have 2 successful results and 1 error
        if len(results) != 2 {
            t.Errorf("Expected 2 successful results, got %d", len(results))
        }
        
        if len(errors) != 1 {
            t.Errorf("Expected 1 error, got %d", len(errors))
        }
    })
}
```

#### 8.2.2 Performance Testing

```go
func BenchmarkPipeline(b *testing.B) {
    // Benchmark the entire pipeline
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        input := gen(1, 2, 3, 4, 5)
        output := sq(input)
        
        for range output {
            // Consume all output
        }
    }
}

func BenchmarkGenerator(b *testing.B) {
    // Benchmark just the generator
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        output := gen(1, 2, 3, 4, 5)
        for range output {
            // Consume all output
        }
    }
}

func BenchmarkTransformer(b *testing.B) {
    // Benchmark just the transformer
    input := gen(1, 2, 3, 4, 5)
    
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        output := sq(input)
        for range output {
            // Consume all output
        }
    }
}
```

#### 8.2.3 Stress Testing

```go
func TestPipelineStress(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping
