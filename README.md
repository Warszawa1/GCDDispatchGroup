# Swift Concurrency Guide: GCD and Dispatch Groups

## Table of Contents
- [Introduction](#introduction)
- [Basic Concepts](#basic-concepts)
- [Example Code Structure](#example-code-structure)
- [Concurrency Patterns](#concurrency-patterns)
  - [Pattern 1: Fire and Forget](#pattern-1-fire-and-forget)
  - [Pattern 2: Batch Processing](#pattern-2-batch-processing)
  - [Pattern 3: Race Conditions](#pattern-3-race-conditions)
  - [Pattern 4: Protecting Shared Resources](#pattern-4-protecting-shared-resources)
- [Everyday Examples](#everyday-examples)
- [Best Practices](#best-practices)

## Introduction

Concurrency allows multiple pieces of code to run simultaneously, improving performance and responsiveness. Swift offers powerful concurrency tools through Grand Central Dispatch (GCD) and Dispatch Groups. This guide explores these concepts through practical examples.

## Basic Concepts

- **DispatchQueue**: A queue that manages the execution of tasks serially or concurrently
- **Dispatch Group**: A mechanism to track the completion of a group of tasks
- **Race Condition**: A bug that occurs when multiple threads access shared data simultaneously
- **Thread Safety**: The quality of code that functions correctly during simultaneous execution

## Example Code Structure

Our examples use this basic structure:

```swift
struct GCDDispatchGroup {
    let lettersQueue = DispatchQueue(label: "io.keepcodin.lettersQueue", attributes: .concurrent)
    let indexesQueue = DispatchQueue(label: "io.keepcodin.indexesQueue", attributes: .concurrent)
    let series = ["A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V", "W", "X", "Z"]
    let group = DispatchGroup()
    
    // Different concurrency pattern methods will be defined here
}
```

## Concurrency Patterns

### Pattern 1: Fire and Forget

This pattern dispatches all tasks at once and notifies when everything is complete, without caring about the order of execution.

```swift
func sampleDispatchGroup() {
    for letter in series {
        // Task 1: Print the letter
        group.enter()
        lettersQueue.async {
            print(letter, terminator: "")
            group.leave()
        }
        
        // Task 2: Print the index
        group.enter()
        indexesQueue.async {
            guard let index = series.firstIndex(where: {$0 == letter}) else {
                return
            }
            print(("\(index)"), terminator: "")
            group.leave()
        }
    }
    
    // This executes after all tasks complete
    group.notify(queue: .main) {
        print("\nFinished")
    }
}
```

**Everyday Example**: Like sending multiple text messages to different friends and waiting for all their replies, regardless of who responds first.

**When to Use**: When order doesn't matter and you want maximum parallelism.

### Pattern 2: Batch Processing

This pattern processes one item completely before moving to the next, creating ordered batches of work.

```swift
func sampleDispatchGroupWaitingForEveryCycle() {
    for letter in series {
        // Task 1: Print the letter
        group.enter()
        lettersQueue.async {
            print(letter, terminator: "")
            group.leave()
        }
        
        // Task 2: Print the index
        group.enter()
        indexesQueue.async {
            guard let index = series.firstIndex(where: {$0 == letter}) else {
                return
            }
            print(("\(index)"), terminator: "")
            group.leave()
        }
        
        // Wait for both tasks to complete before moving to the next letter
        group.wait()
    }
    
    group.notify(queue: .main) {
        print("\nFinished")
    }
}
```

**Everyday Example**: Like a factory assembly line that completes one product fully before moving to the next.

**When to Use**: When you need to complete all work for one unit before moving to the next.

### Pattern 3: Race Conditions

This pattern demonstrates a common concurrency bug when multiple threads modify shared state without synchronization.

```swift
func sampleDispatchGroupRaceCondition(completion: @escaping ((String) -> Void)) {
    // Shared mutable state
    var result = ""
    
    for letter in series {
        group.enter()
        lettersQueue.async {
            print(letter, terminator: "")
            
            // RACE CONDITION HERE: Unsynchronized modification
            result = result + letter
            
            group.leave()
        }
        
        group.enter()
        indexesQueue.async {
            guard let index = series.firstIndex(where: {$0 == letter}) else {
                return
            }
            print(("\(index)"), terminator: "")
            
            // RACE CONDITION HERE: Unsynchronized modification
            result = result + "\(index)"
            
            group.leave()
        }
        
        group.wait()
    }
    
    group.notify(queue: .main) {
        print("\nFinished")
        completion(result)
    }
}
```

**Everyday Example**: Two roommates both check the refrigerator, both see milk is low, and both buy milk on the way home, resulting in too much milk.

**The Problem**: This can lead to data corruption, unexpected results, or crashes because:
1. Thread 1 reads current value of result
2. Thread 2 reads current value of result (before Thread 1 finishes)
3. Thread 1 modifies and writes back
4. Thread 2 modifies and writes back, overwriting Thread 1's change

### Pattern 4: Protecting Shared Resources

This pattern fixes race conditions by synchronizing access to shared resources.

```swift
func sampleDispatchGroupRaceCondition_solved(completion: @escaping ((String) -> Void)) {
    // Create a dedicated serial queue to protect the shared resource
    let serialQueue = DispatchQueue(label: "io.keepcoding.serialQueue")
    var result = ""
    
    for letter in series {
        group.enter()
        lettersQueue.sync {
            print(letter, terminator: "")
            
            // Safe access to shared resource through serialQueue
            serialQueue.async {
                result = result + letter
            }
            
            group.leave()
        }
        
        group.enter()
        indexesQueue.async {
            guard let index = series.firstIndex(where: {$0 == letter}) else {
                return
            }
            print(("\(index)"), terminator: "")
            
            // Safe access to shared resource through serialQueue
            serialQueue.sync {
                result = result + "\(index)"
            }
            
            group.leave()
        }
        
        group.wait()
    }
    
    group.notify(queue: .main) {
        print("\nFinished")
        completion(result)
    }
}
```

**Everyday Example**: Using a shared shopping list that only one person updates at a time, preventing duplicate purchases.

**When to Use**: Whenever multiple threads need to modify shared state.

## Everyday Examples

| Concurrency Concept | Everyday Example |
|---------------------|------------------|
| Serial Queue | A single-lane road where cars must travel one after another |
| Concurrent Queue | A multi-lane highway where multiple cars travel simultaneously |
| enter()/leave() | A museum counting visitors in and out to track attendance |
| wait() | Waiting for all your friends to arrive before starting a movie |
| notify() | Setting an alarm to alert you when the laundry is finished |
| Race Condition | Two chefs adding salt to the same dish, resulting in too much salt |
| Deadlock | Two people meeting in a narrow hallway, each waiting for the other to step aside |

## Best Practices

1. **Use serial queues for shared resources**: When multiple threads need to access the same data, use a serial queue as a "lock"

2. **Balance enter() and leave()**: For every `group.enter()`, ensure there's exactly one `group.leave()` call, even in error paths

3. **Avoid nested sync calls**: Be careful with nested `sync` calls as they can lead to deadlocks

4. **Use unique queue labels**: Give your queues descriptive, unique labels for easier debugging

5. **Consider alternatives**: For newer code, consider Swift's structured concurrency with async/await which makes many of these patterns more straightforward

6. **Measure performance**: Don't assume more concurrency is always better - test and measure to find the right balance

